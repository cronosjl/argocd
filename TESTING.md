# Guide de Test — GitOps ArgoCD Platform

Tests de bout en bout : validation Kustomize → déploiement ArgoCD → vérification applicative → self-heal → sécurité.

**Image déployée :** `kennethreitz/httpbin`
**Environnements :** `dev` (1 replica) · `prod` (3 replicas)
**Endpoint de santé :** `GET /get`

---

## Prérequis

```bash
# Vérifier les outils installés
kubectl version --client
# Kustomize est intégré à kubectl — pas besoin d'installation séparée
kubectl kustomize --help | head -3
argocd version --client
kubeseal --version

# Vérifier l'accès au cluster (minikube)
kubectl cluster-info
kubectl get nodes
```

---

## 1. Validation locale des manifests (sans cluster)

### 1.1 Build de tous les overlays

```bash
kubectl kustomize apps/todo-api/base
kubectl kustomize apps/todo-api/overlays/dev
kubectl kustomize apps/todo-api/overlays/prod
kubectl kustomize infrastructure/kubernetes
```

Chaque commande doit se terminer sans erreur.

### 1.2 Vérifications attendues

**Base :**
- `image: kennethreitz/httpbin`
- `containerPort: 80`
- `livenessProbe.httpGet.path: /get`
- `readinessProbe.httpGet.path: /get`
- `resources.requests` et `resources.limits` présents

**Overlay dev :**
- `namespace: todo-api-dev`
- `replicas: 1`
- `environment: dev`
- CPU request : `50m`, limit : `200m`
- Memory request : `64Mi`, limit : `128Mi`

**Overlay prod :**
- `namespace: todo-api-prod`
- `replicas: 3`
- `environment: prod`, `tier: production`
- CPU request : `200m`, limit : `1000m`
- Memory request : `256Mi`, limit : `512Mi`
- `podAntiAffinity` présent (pods répartis sur les nœuds)

### 1.3 Audit secrets en clair

```bash
grep -r "password:\|token:\|secret:\|apiKey:" \
  apps/ argocd/ infrastructure/ .github/ \
  --include="*.yaml" \
  | grep -v "secretKeyRef\|sealed\|encryptedData"
```

**Attendu : aucune ligne.**

### 1.4 Vérifier les Applications ArgoCD

```bash
grep -E "repoURL:|targetRevision:|path:" argocd/applications/*.yaml
```

**Attendu :**
- `todo-api-dev` → `apps/todo-api/overlays/dev` · `targetRevision: main`
- `todo-api-prod` → `apps/todo-api/overlays/prod`
- `infrastructure` → `infrastructure/kubernetes` · `targetRevision: main`

---

## 2. Bootstrap ArgoCD

### 2.1 Installation ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=available --timeout=120s deployment/argocd-server -n argocd
kubectl get pods -n argocd
# Attendu : tous les pods Running
```

### 2.2 Login ArgoCD

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

argocd login localhost:8080 --insecure --username admin --password "$ARGOCD_PWD"
echo "UI → https://localhost:8080  (admin / $ARGOCD_PWD)"
```

### 2.3 Déploiement des Applications ArgoCD

```bash
kubectl apply -f argocd/applications/ -n argocd

argocd app list
# Attendu : todo-api-dev, todo-api-prod, infrastructure (+ todo-api-rollout si présent)
```

---

## 3. Vérification du déploiement

### 3.1 Attendre la synchronisation

```bash
argocd app wait todo-api-dev   --sync --health --timeout 180
argocd app wait todo-api-prod  --sync --health --timeout 180
argocd app wait infrastructure --sync --health --timeout 180
```

### 3.2 Ressources déployées — dev

```bash
kubectl -n todo-api-dev get deploy,po,svc -l app=todo-api
```

**Attendu :**
```
NAME                        READY   STATUS    RESTARTS
pod/todo-api-<hash>         1/1     Running   0

NAME               TYPE        PORT(S)
service/todo-api   ClusterIP   80/TCP
```

### 3.3 Ressources déployées — prod

```bash
kubectl -n todo-api-prod get deploy,po,svc -l app=todo-api
```

**Attendu :**
```
NAME                        READY   STATUS    RESTARTS
pod/todo-api-<hash>-0       1/1     Running   0
pod/todo-api-<hash>-1       1/1     Running   0
pod/todo-api-<hash>-2       1/1     Running   0

NAME               TYPE        PORT(S)
service/todo-api   ClusterIP   80/TCP
```

### 3.4 Probes et resource limits

```bash
# Vérifier en dev
kubectl -n todo-api-dev describe pod -l app=todo-api \
  | grep -A5 "Liveness\|Readiness\|Limits\|Requests"
```

**Attendu :**
```
Limits:    cpu: 200m   memory: 128Mi
Requests:  cpu: 50m    memory: 64Mi
Liveness:  http-get http://:http/get
Readiness: http-get http://:http/get
```

### 3.5 Infrastructure

```bash
kubectl get ns todo-api-dev todo-api-prod
kubectl get networkpolicies -A
kubectl get roles,rolebindings -n todo-api-dev
kubectl get roles,rolebindings -n todo-api-prod
```

---

## 4. Tests fonctionnels de l'application (httpbin)

`kennethreitz/httpbin` est un service HTTP de test. Il n'a pas d'état — chaque requête retourne les informations de la requête elle-même.

### 4.1 Environnement dev

```bash
kubectl port-forward svc/todo-api -n todo-api-dev 3000:80 &

# Santé de l'application
curl -s http://localhost:3000/get
# Attendu : JSON avec origin, headers, url

# Vérifier les headers
curl -s http://localhost:3000/headers
# Attendu : {"headers": {"Host": "localhost:3000", ...}}

# Vérifier l'IP source
curl -s http://localhost:3000/ip
# Attendu : {"origin": "..."}

# Tester une requête POST
curl -s -X POST http://localhost:3000/post \
  -H "Content-Type: application/json" \
  -d '{"env": "dev", "test": true}'
# Attendu : JSON avec json.env = "dev", json.test = true

# Tester les codes HTTP
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/status/200
# Attendu : 200

curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/status/404
# Attendu : 404

# Tester la réponse avec délai
curl -s http://localhost:3000/delay/2
# Attendu : réponse après ~2 secondes

kill %1
```

### 4.2 Environnement prod

```bash
kubectl port-forward svc/todo-api -n todo-api-prod 3001:80 &

curl -s http://localhost:3001/get
# Attendu : JSON valide, même comportement que dev

curl -s -X POST http://localhost:3001/post \
  -H "Content-Type: application/json" \
  -d '{"env": "prod", "test": true}'
# Attendu : json.env = "prod"

kill %1
```

### 4.3 Endpoint `/anything`

```bash
kubectl port-forward svc/todo-api -n todo-api-dev 3000:80 &

# Renvoie tout : méthode, headers, body, args
curl -s -X PUT http://localhost:3000/anything/custom-path \
  -H "X-Custom-Header: gitops-test" \
  -d "payload=test"
# Attendu : json avec method=PUT, headers.X-Custom-Header=gitops-test

kill %1
```

---

## 5. Test du self-heal et du prune ArgoCD

### 5.1 Self-heal (correction de drift)

```bash
# Modifier manuellement le nombre de replicas en dev
kubectl -n todo-api-dev scale deployment todo-api --replicas=5
kubectl -n todo-api-dev get deploy todo-api
# Attendu temporaire : replicas = 5

# Attendre ~1 minute (selfHeal ArgoCD)
watch kubectl -n todo-api-dev get deploy todo-api
# Attendu : revient automatiquement à 1 replica
```

### 5.2 Auto-prune (suppression des ressources orphelines)

```bash
# Créer une ressource hors GitOps
kubectl -n todo-api-dev create configmap orphan-test --from-literal=key=value
kubectl -n todo-api-dev get configmap orphan-test
# Attendu : présente

# Attendre ~3 minutes (ArgoCD détecte le drift et prune)
sleep 180
kubectl -n todo-api-dev get configmap orphan-test
# Attendu : Error from server (NotFound)
```

---

## 6. Déploiement Blue/Green (Argo Rollouts)

> Applicable si Argo Rollouts est installé et `todo-api-rollout` déployé.

### 6.1 Installation Argo Rollouts

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

kubectl wait --for=condition=available --timeout=120s \
  deployment/argo-rollouts -n argo-rollouts
kubectl argo rollouts version
```

### 6.2 État initial du Rollout

```bash
kubectl argo rollouts list rollouts -A
kubectl argo rollouts get rollout todo-api -n todo-api-staging
# Attendu : status = Healthy, strategy = BlueGreen
```

### 6.3 Déclencher un blue/green

```bash
kubectl argo rollouts set image todo-api \
  todo-api=kennethreitz/httpbin:latest \
  -n todo-api-staging

kubectl argo rollouts get rollout todo-api -n todo-api-staging --watch
```

**Attendu :**
- Pods preview créés avec la nouvelle image
- Service `todo-api-active` : ancienne version (stable)
- Service `todo-api-preview` : nouvelle version

### 6.4 Valider la version preview

```bash
kubectl port-forward svc/todo-api-preview -n todo-api-staging 3002:80 &
curl -s http://localhost:3002/get
kill %1
```

### 6.5 Promouvoir (bascule active → preview)

```bash
kubectl argo rollouts promote todo-api -n todo-api-staging
kubectl argo rollouts get rollout todo-api -n todo-api-staging --watch
# Attendu : status = Healthy, pods active = nouvelle version
```

### 6.6 Rollback

```bash
kubectl argo rollouts abort todo-api -n todo-api-staging
kubectl argo rollouts undo todo-api -n todo-api-staging

kubectl argo rollouts get rollout todo-api -n todo-api-staging
# Attendu : Healthy sur la version précédente
```

### 6.7 Dashboard

```bash
kubectl argo rollouts dashboard -n todo-api-staging
# → http://localhost:3100
```

---

## 7. Tests sécurité

### 7.1 Absence de secrets en clair

```bash
grep -r "password:\|token:\|apiKey:\|secret:" \
  apps/ argocd/ infrastructure/ .github/ \
  --include="*.yaml" \
  | grep -v "secretKeyRef\|sealed\|encryptedData"
# Attendu : aucune ligne
```

### 7.2 Sealed Secrets

```bash
kubectl get pods -n kube-system | grep sealed-secrets
kubectl get sealedsecrets -A
# Attendu : sealed-secrets-controller Running, SealedSecret présent dans infrastructure
```

### 7.3 Network Policies

```bash
kubectl get networkpolicies -n todo-api-dev
kubectl get networkpolicies -n todo-api-prod
kubectl describe networkpolicy -n todo-api-prod
```

### 7.4 RBAC

```bash
kubectl get roles,rolebindings -n todo-api-dev
kubectl get roles,rolebindings -n todo-api-prod
```

### 7.5 Historique Git propre

```bash
git log --oneline -15
# Vérifier : aucun "wip", "fix", "test" sans contexte
```

---

## 8. Logs et diagnostic

```bash
# Logs de l'application
kubectl -n todo-api-dev  logs -l app=todo-api --tail=50
kubectl -n todo-api-prod logs -l app=todo-api --tail=50

# Events récents
kubectl -n todo-api-dev  get events --sort-by='.lastTimestamp' | tail -10
kubectl -n todo-api-prod get events --sort-by='.lastTimestamp' | tail -10

# Logs ArgoCD
kubectl -n argocd logs deploy/argocd-application-controller --tail=30

# Statut ArgoCD détaillé
argocd app get todo-api-dev
argocd app get todo-api-prod
argocd app get infrastructure
```

---

## 9. Nettoyage

```bash
argocd app delete todo-api-dev todo-api-prod infrastructure --yes
kubectl delete ns todo-api-dev todo-api-prod
```

---

## Checklist de validation

| # | Test | Commande | Attendu |
|---|------|----------|---------|
| 1 | Kustomize build base | `kubectl kustomize apps/todo-api/base` | Pas d'erreur, image httpbin |
| 2 | Kustomize build dev | `kubectl kustomize apps/todo-api/overlays/dev` | namespace: todo-api-dev, replicas: 1 |
| 3 | Kustomize build prod | `kubectl kustomize apps/todo-api/overlays/prod` | namespace: todo-api-prod, replicas: 3 |
| 4 | Kustomize build infra | `kubectl kustomize infrastructure/kubernetes` | Pas d'erreur |
| 5 | Pas de secrets en clair | `grep password: ...` | 0 ligne |
| 6 | ArgoCD opérationnel | `argocd app list` | Apps Synced + Healthy |
| 7 | Pod dev running | `kubectl -n todo-api-dev get po` | 1/1 Running |
| 8 | Pod prod running | `kubectl -n todo-api-prod get po` | 3/3 Running |
| 9 | httpbin répond (dev) | `curl localhost:3000/get` | JSON valide |
| 10 | httpbin répond (prod) | `curl localhost:3001/get` | JSON valide |
| 11 | POST fonctionne | `curl -X POST .../post` | JSON avec body |
| 12 | Codes HTTP | `curl .../status/200` et `.../status/404` | 200 et 404 |
| 13 | Self-heal | Scale manuel → watch | Revient à la valeur Git |
| 14 | Auto-prune | ConfigMap orphelin | Supprimé par ArgoCD |
| 15 | Sealed Secrets | `kubectl get sealedsecrets` | Présents |
| 16 | Network Policies | `kubectl get netpol -A` | Présentes dans chaque ns |
| 17 | Historique Git | `git log --oneline` | Commits propres |
