# Guide de Test — GitOps Corp Platform (Todo API)

Tests complets couvrant les 5 étapes du projet : validation Kustomize, déploiement ArgoCD, déploiements progressifs (canary + blue/green), pipeline CI, Terraform IaC et sécurité.

---

## Prérequis

| Outil | Version | Vérification |
|---|---|---|
| `kubectl` | 1.25+ | `kubectl version --client` |
| `kustomize` | 5.0+ | `kustomize version` |
| `argocd` | 2.8+ | `argocd version --client` |
| `kubeseal` | any | `kubeseal --version` |
| `curl` | any | `curl --version` |
| `k3d` | 5.0+ | `k3d version` |

```bash
# Vérifier l'accès au cluster
kubectl cluster-info
kubectl get nodes
```

---

## Étape 1 — Validation locale des manifests (sans cluster)

### 1.1 Build des overlays

```bash
# Overlay dev
kubectl kustomize apps/todo-api/overlays/dev

# Overlay prod
kubectl kustomize apps/todo-api/overlays/prod
```

**Résultat attendu dev :**
- `kind: Rollout` (pas `Deployment`)
- `namespace: todo-api-dev`
- `replicas: 1`
- `image: docker.io/shatri/todo-api-node:v1`
- `DB_PASSWORD` référencé via `secretKeyRef` (jamais en clair)
- `kind: SealedSecret` présent

**Résultat attendu prod :**
- `kind: Rollout` avec stratégie `blueGreen`
- `namespace: todo-api-prod`
- `replicas: 3`
- `activeService: todo-api-stable`, `previewService: todo-api-canary`
- `kind: SealedSecret` présent

### 1.2 Diff entre les deux overlays

```bash
diff \
  <(kubectl kustomize apps/todo-api/overlays/dev) \
  <(kubectl kustomize apps/todo-api/overlays/prod)
```

Différences attendues : namespace, replicas, stratégie Rollout (canary vs blueGreen), `NODE_ENV=production` en prod.

### 1.3 Vérifier l'absence de secrets en clair

```bash
grep -r "password:\|token:\|secret:" apps/ --include="*.yaml" \
  | grep -v "secretKeyRef\|sealed\|SealedSecret\|encryptedData"
```

**Attendu : aucune ligne.** Toute sortie indique un secret en clair à corriger immédiatement.

### 1.4 Valider la structure ArgoCD

```bash
ls argocd/applications/
# Attendu : todo-api-dev.yaml  todo-api-prod.yaml

cat argocd/applications/todo-api-dev.yaml | grep -E "name:|path:|namespace:"
cat argocd/applications/todo-api-prod.yaml | grep -E "name:|path:|namespace:"
```

---

## Étape 2 — Bootstrap et déploiement ArgoCD

### 2.1 Vérifier qu'ArgoCD est opérationnel

```bash
kubectl get pods -n argocd
```

Tous les pods doivent être `Running` / `1/1 Ready` :
- `argocd-server`
- `argocd-application-controller`
- `argocd-repo-server`
- `argocd-dex-server`
- `argocd-redis`

### 2.2 Se connecter à l'interface

```bash
# Port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
# → https://localhost:8080

# Récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Login CLI
argocd login localhost:8080 --insecure --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d)
```

### 2.3 Appliquer les Applications ArgoCD

```bash
kubectl apply -f argocd/applications/ -n argocd
```

Résultat attendu :
```
application.argoproj.io/todo-api-dev created
application.argoproj.io/todo-api-prod created
```

### 2.4 Vérifier les applications

```bash
argocd app list
```

Attendu : les deux apps `todo-api-dev` et `todo-api-prod` en `Synced` / `Healthy`.

```bash
argocd app get todo-api-dev
argocd app get todo-api-prod
```

Champs à vérifier :
- `Repo` : `https://github.com/abdallahsidibe/gitops-corp-platform`
- `Sync Policy` : `Automated (Prune, Self Heal)`
- `Health Status` : `Healthy`

---

## Étape 3 — Vérification du déploiement sur le cluster

### 3.1 Attendre la synchronisation

```bash
argocd app wait todo-api-dev --sync --health --timeout 180
argocd app wait todo-api-prod --sync --health --timeout 180
```

### 3.2 Vérifier les ressources déployées

```bash
# Dev (Rollout + pods + services)
kubectl -n todo-api-dev get rollout,po,svc -l app=todo-api

# Prod
kubectl -n todo-api-prod get rollout,po,svc -l app=todo-api
```

Résultat attendu dev :
```
NAME                          DESIRED   CURRENT   UP-TO-DATE   READY
rollout.argoproj.io/todo-api  1         1         1            1

NAME                    READY   STATUS    RESTARTS
pod/todo-api-<hash>     1/1     Running   0

NAME                     TYPE        PORT(S)
service/todo-api         ClusterIP   80/TCP
service/todo-api-stable  ClusterIP   80/TCP
service/todo-api-canary  ClusterIP   80/TCP
```

### 3.3 Vérifier la configuration sécurité du pod

```bash
# Variables d'env (DB_PASSWORD ne doit pas apparaître en clair)
kubectl -n todo-api-dev exec deploy/todo-api -- env | grep DB_

# User d'exécution (attendu: uid=1000)
kubectl -n todo-api-dev exec deploy/todo-api -- id

# Filesystem root en lecture seule
kubectl -n todo-api-dev exec deploy/todo-api -- touch /test-file 2>&1
# Attendu : "Read-only file system"

# /tmp accessible en écriture
kubectl -n todo-api-dev exec deploy/todo-api -- touch /tmp/test-file && echo "OK"
```

### 3.4 Vérifier les probes et les resource limits

```bash
kubectl -n todo-api-dev describe pod -l app=todo-api \
  | grep -A5 "Liveness\|Readiness\|Limits\|Requests"
```

Attendu :
```
Liveness:  http-get http://:http/ delay=10s period=10s
Readiness: http-get http://:http/ delay=5s  period=5s
Limits:    cpu: 500m  memory: 256Mi
Requests:  cpu: 100m  memory: 128Mi
```

### 3.5 Vérifier le Sealed Secret

```bash
# Le SealedSecret doit avoir été déchiffré en Secret par le contrôleur
kubectl -n todo-api-dev get secret todo-api-db-secret
# Attendu : secret de type Opaque

# Vérifier que la valeur n'est PAS lisible en clair
kubectl -n todo-api-dev get secret todo-api-db-secret \
  -o jsonpath='{.data.db-password}' | base64 -d
# → doit afficher votre mot de passe (c'est normal, il est dans etcd chiffré)
```

---

## Étape 4 — Tests fonctionnels de l'API

### 4.1 Port-forward et tests de base

```bash
kubectl port-forward svc/todo-api -n todo-api-dev 3000:80 &

# Test root
curl -i http://localhost:3000/
# Attendu : 200 OK

# Lister les todos (liste vide au départ)
curl http://localhost:3000/todos
# Attendu : []

# Créer un todo
curl -X POST http://localhost:3000/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Test GitOps Étape 2", "completed": false}'
# Attendu : {"id": 1, "title": "...", "completed": false}

# Lister après création
curl http://localhost:3000/todos

# Supprimer
curl -X DELETE http://localhost:3000/todos/1

kill %1
```

### 4.2 Test depuis l'intérieur du cluster

```bash
kubectl -n todo-api-dev run tmp --rm -it \
  --image=curlimages/curl --restart=Never -- \
  curl -sS http://todo-api.todo-api-dev.svc.cluster.local/todos
```

---

## Étape 5 — Test du workflow GitOps (bout en bout CI → cluster)

### 5.1 Vérifier que le pipeline CI est configuré

```bash
cat .github/workflows/ci.yaml | grep -E "name:|on:|jobs:"
```

Vérifier que les 3 jobs sont présents : `build-and-push`, `update-gitops`, `validate-manifests`.

### 5.2 Simuler une mise à jour de tag (sans pipeline CI)

```bash
# Mettre à jour le tag image manuellement dans l'overlay prod
cd apps/todo-api/overlays/prod
kustomize edit set image docker.io/shatri/todo-api-node:test-sha123
cd -

# Committer
git add apps/todo-api/overlays/prod/kustomization.yaml
git commit -m "chore: update todo-api image to test-sha123"
git push origin main
```

### 5.3 Observer la synchronisation ArgoCD

```bash
# ArgoCD poll toutes les 3 minutes — forcer immédiatement :
argocd app sync todo-api-prod

# Surveiller la convergence
watch kubectl -n todo-api-prod get rollout,po -l app=todo-api
```

### 5.4 Test du self-heal (ArgoCD corrige les drifts)

```bash
# Modifier manuellement le nombre de replicas (hors GitOps)
kubectl -n todo-api-dev patch rollout todo-api \
  --type=merge -p '{"spec":{"replicas":5}}'

# Attendre ~1 minute (selfHeal ArgoCD)
watch kubectl -n todo-api-dev get rollout todo-api
# Attendu : revient à 1 replica (valeur dans Git)
```

### 5.5 Test du pruning automatique

```bash
# Créer une ressource orpheline directement sur le cluster
kubectl -n todo-api-dev create configmap orphan-test \
  --from-literal=key=value

# Attendre ~3 minutes (ArgoCD prune)
sleep 180
kubectl -n todo-api-dev get configmap orphan-test
# Attendu : "Error from server (NotFound)"
```

---

## Étape 6 — Tests déploiements progressifs (Argo Rollouts)

### 6.1 Vérifier qu'Argo Rollouts est installé

```bash
kubectl get pods -n argo-rollouts
kubectl argo rollouts version
```

### 6.2 Canary deployment (dev)

```bash
# État initial du rollout
kubectl argo rollouts get rollout todo-api -n todo-api-dev

# Déclencher un canary en changeant le tag image
kubectl argo rollouts set image todo-api \
  todo-api=docker.io/shatri/todo-api-node:v2 \
  -n todo-api-dev

# Observer : 10% du trafic sur v2
kubectl argo rollouts get rollout todo-api -n todo-api-dev --watch

# Promouvoir vers 50%
kubectl argo rollouts promote todo-api -n todo-api-dev

# Promouvoir vers 100%
kubectl argo rollouts promote todo-api -n todo-api-dev
```

### 6.3 Test du rollback canary

```bash
# Déclencher un canary
kubectl argo rollouts set image todo-api \
  todo-api=docker.io/shatri/todo-api-node:v-bad \
  -n todo-api-dev

# Observer 10% sur la nouvelle version
kubectl argo rollouts get rollout todo-api -n todo-api-dev

# Simuler une anomalie → rollback
kubectl argo rollouts abort todo-api -n todo-api-dev

# Vérifier le retour à la version stable
kubectl argo rollouts get rollout todo-api -n todo-api-dev
# Attendu : status = Degraded → Healthy (après undo)

kubectl argo rollouts undo todo-api -n todo-api-dev
```

### 6.4 Blue/Green deployment (prod)

```bash
# État initial
kubectl argo rollouts get rollout todo-api -n todo-api-prod

# Déclencher un blue/green
kubectl argo rollouts set image todo-api \
  todo-api=docker.io/shatri/todo-api-node:v2 \
  -n todo-api-prod

# Observer : version preview active sur todo-api-canary
kubectl argo rollouts get rollout todo-api -n todo-api-prod --watch

# Vérifier les services : stable pointe sur v1, preview sur v2
kubectl -n todo-api-prod get svc todo-api-stable todo-api-canary \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.selector}{"\n"}{end}'

# Valider la version preview
kubectl port-forward svc/todo-api-canary -n todo-api-prod 3002:80 &
curl http://localhost:3002/todos
kill %1

# Bascule (blue → green) : promote
kubectl argo rollouts promote todo-api -n todo-api-prod
```

### 6.5 Dashboard Argo Rollouts

```bash
kubectl argo rollouts dashboard -n todo-api-prod
# → http://localhost:3100
```

---

## Étape 7 — Tests Terraform IaC

### 7.1 Vérifier la syntaxe Terraform

```bash
cd infrastructure/terraform
terraform init -backend=false
terraform validate
cd -
```

Attendu : `Success! The configuration is valid.`

### 7.2 Vérifier le Terraform Workspace CRD

```bash
# Vérifier que le Terraform Controller est installé
kubectl get ns flux-system 2>/dev/null || echo "Flux TF Controller non installé"

# Appliquer le Workspace
kubectl apply -f infrastructure/terraform-workspace.yaml

# Suivre l'état
kubectl get terraform -n flux-system
kubectl describe terraform corp-platform-infra -n flux-system
```

### 7.3 Vérifier les ressources créées par Terraform

```bash
# Namespaces créés par Terraform
kubectl get ns todo-api-dev todo-api-prod -o yaml \
  | grep -A3 "labels:"

# ServiceAccounts
kubectl get serviceaccount todo-api -n todo-api-dev
kubectl get serviceaccount todo-api -n todo-api-prod

# NetworkPolicies
kubectl get networkpolicies -n todo-api-dev
kubectl get networkpolicies -n todo-api-prod

# RBAC
kubectl get role,rolebinding -n todo-api-dev
kubectl get role,rolebinding -n todo-api-prod
```

---

## Étape 8 — Tests sécurité

### 8.1 Audit secrets en clair

```bash
# Aucun credential ne doit être en clair dans le repo
grep -r "password:\|token:\|apiKey:\|secret:" \
  apps/ argocd/ infrastructure/ .github/ \
  --include="*.yaml" --include="*.env" \
  | grep -v "secretKeyRef\|sealed\|SealedSecret\|encryptedData\|#"
# Attendu : aucune ligne
```

### 8.2 Vérifier Sealed Secrets

```bash
# Controller opérationnel
kubectl get deployment sealed-secrets-controller -n kube-system

# SealedSecrets déchiffrés en Secrets
kubectl get sealedsecret -n todo-api-dev
kubectl get sealedsecret -n todo-api-prod
kubectl get secret todo-api-db-secret -n todo-api-dev
kubectl get secret todo-api-db-secret -n todo-api-prod
```

### 8.3 Vérifier le RBAC ArgoCD

```bash
# Vérifier la configuration des rôles
kubectl get configmap argocd-rbac-cm -n argocd \
  -o jsonpath='{.data.policy\.csv}'
```

Doit contenir les rôles `view`, `sync`, `override` et les bindings de groupe.

### 8.4 Vérifier les signed commits

```bash
# Vérifier la signature des derniers commits
git log --show-signature -5 | grep -E "Good signature|BAD signature|commit "

# Vérifier la config locale
git config commit.gpgsign
# Attendu : true
```

### 8.5 Vérifier les NetworkPolicies

```bash
kubectl describe networkpolicy -n todo-api-prod
```

Vérifier que le trafic entrant est limité au port 3000 et que le cross-namespace est bloqué.

---

## Étape 9 — Vérification des logs et événements

```bash
# Logs de l'application dev
kubectl -n todo-api-dev logs -l app=todo-api --tail=50

# Logs si crash
kubectl -n todo-api-dev logs -l app=todo-api --previous

# Événements du namespace
kubectl -n todo-api-prod get events --sort-by='.lastTimestamp' | head -20

# Logs ArgoCD (controller)
kubectl -n argocd logs deploy/argocd-application-controller --tail=30

# Logs Argo Rollouts
kubectl -n argo-rollouts logs deploy/argo-rollouts --tail=30
```

---

## Étape 10 — Nettoyage complet

```bash
# Supprimer les applications ArgoCD (prune automatique des ressources)
argocd app delete todo-api-dev --yes
argocd app delete todo-api-prod --yes

# Supprimer les namespaces
kubectl delete ns todo-api-dev todo-api-prod

# Supprimer le cluster k3d
k3d cluster delete corp-platform

# Vérifier que tout est propre
kubectl get ns | grep todo-api
# Attendu : aucune ligne
```

---

## Checklist de validation complète

| # | Étape | Test | Résultat attendu |
|---|---|---|---|
| 1 | Étape 1 | `kubectl kustomize overlays/dev` | Rollout, tag v1, secretKeyRef |
| 2 | Étape 1 | `kubectl kustomize overlays/prod` | Rollout blueGreen, 3 replicas |
| 3 | Étape 1 | `grep password: apps/**/*.yaml` | Aucune ligne |
| 4 | Étape 2 | `kubectl apply -f argocd/applications/` | 2 apps créées |
| 5 | Étape 2 | `argocd app list` | Synced + Healthy |
| 6 | Étape 2 | Pod running | `1/1 Running` |
| 7 | Étape 2 | Filesystem root read-only | `touch /test-file` → erreur |
| 8 | Étape 2 | `/tmp` accessible | `touch /tmp/file` → OK |
| 9 | Étape 2 | Self-heal | Replicas manuels → revient à Git |
| 10 | Étape 2 | Auto-prune | Objet orphelin supprimé |
| 11 | Étape 2 | API répond | `curl localhost:3000/` → 200 |
| 12 | Étape 2 | CRUD Todo | POST/GET/DELETE → OK |
| 13 | Étape 3 | Canary 10% → 50% → 100% | Promote sans erreur |
| 14 | Étape 3 | Rollback canary | `abort` → retour version stable |
| 15 | Étape 3 | Blue/Green promote | Bascule instantanée |
| 16 | Étape 3 | Blue/Green rollback | `abort` → retour version active |
| 17 | Étape 4 | `terraform validate` | Configuration is valid |
| 18 | Étape 4 | NetworkPolicies en place | `kubectl get netpol -n todo-api-prod` |
| 19 | Étape 5 | Aucun secret en clair | `grep password:` → 0 résultats |
| 20 | Étape 5 | SealedSecrets déchiffrés | `kubectl get secret todo-api-db-secret` |
| 21 | Étape 5 | RBAC ArgoCD configuré | 3 rôles distincts dans argocd-rbac-cm |
| 22 | Étape 5 | Signed commits | `git log --show-signature` → Good signature |
