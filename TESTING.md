# Guide de Test — GitOps ArgoCD Platform

Tests de bout en bout : validation Kustomize → déploiement ArgoCD → vérification applicative → self-heal → sécurité.

**Cluster :** minikube
**Image déployée :** `kennethreitz/httpbin`
**Environnements :** `dev` (1 replica) · `prod` (3 replicas)
**Endpoint de santé :** `GET /get`
**Applications ArgoCD :** `todo-api-dev` · `todo-api-prod` · `infrastructure`

---

## Prérequis

```bash
kubectl version --client
kubectl kustomize --help | head -3
argocd version --client
kubeseal --version
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

Chaque commande doit se terminer sans erreur et sans warning.

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
- Probes sur `/get` (pas `/health`)

**Infrastructure :**
- Namespaces `todo-api-prod` et `monitoring` présents
- RBAC : `ServiceAccount`, `Role`, `RoleBinding` dans `todo-api-prod`
- `SealedSecret` présent avec `encryptedData`
- `NetworkPolicy` ingress sur port `80`

### 1.3 Audit secrets en clair

```bash
grep -r "password:\|token:\|secret:\|apiKey:" apps/ argocd/ infrastructure/ .github/ --include="*.yaml" | grep -v "secretKeyRef\|sealed\|encryptedData"
```

**Attendu : aucune ligne.**

### 1.4 Vérifier les Applications ArgoCD

```bash
grep -E "repoURL:|targetRevision:|path:" argocd/applications/*.yaml
```

**Attendu :**
- `todo-api-dev` → `apps/todo-api/overlays/dev` · `targetRevision: main`
- `todo-api-prod` → `apps/todo-api/overlays/prod` · `targetRevision: main`
- `infrastructure` → `infrastructure/kubernetes` · `targetRevision: main`

---

## 2. Bootstrap ArgoCD

### 2.1 Installation ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side
kubectl wait --for=condition=available --timeout=120s deployment/argocd-server -n argocd
kubectl get pods -n argocd
```

> `--server-side` évite l'erreur `metadata.annotations: Too long` sur les CRDs volumineux.

### 2.2 Installation Sealed Secrets

```bash
curl -sL https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml | kubectl apply -f -
kubectl wait --for=condition=available --timeout=60s deployment/sealed-secrets-controller -n kube-system
kubectl get pods -n kube-system | grep sealed
```

### 2.3 Login ArgoCD

```bash
# Vérifier si le port 8080 est déjà utilisé
lsof -i :8080 -sTCP:LISTEN
# Si occupé par un kubectl précédent : kill <PID>

kubectl port-forward svc/argocd-server -n argocd 8080:443 &
ARGOCD_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
argocd login localhost:8080 --insecure --username admin --password $ARGOCD_PWD
echo "UI → https://localhost:8080  (admin / $ARGOCD_PWD)"
```

### 2.4 Déploiement des Applications ArgoCD

```bash
kubectl apply -f argocd/applications/ -n argocd
argocd app list
# Attendu : 3 apps — todo-api-dev, todo-api-prod, infrastructure
```

---

## 3. Vérification du déploiement

### 3.1 Attendre la synchronisation

> Les apps synchonisent depuis `main`. Si les changements ne sont pas encore sur `main`, le statut sera `OutOfSync` — c'est normal.

```bash
argocd app wait todo-api-dev --sync --health --timeout 180
argocd app wait todo-api-prod --sync --health --timeout 180
argocd app wait infrastructure --sync --health --timeout 180
```

### 3.2 Ressources déployées — dev

```bash
kubectl -n todo-api-dev get deploy,po,svc -l app=todo-api
```

**Attendu :**
```
NAME                      READY   STATUS    RESTARTS
pod/todo-api-<hash>       1/1     Running   0

NAME               TYPE        PORT(S)
service/todo-api   ClusterIP   80/TCP
```

### 3.3 Ressources déployées — prod

```bash
kubectl -n todo-api-prod get deploy,po,svc -l app=todo-api
```

**Attendu :**
```
NAME                    READY   STATUS    RESTARTS
pod/todo-api-<hash>-0   1/1     Running   0
pod/todo-api-<hash>-1   1/1     Running   0
pod/todo-api-<hash>-2   1/1     Running   0

NAME               TYPE        PORT(S)
service/todo-api   ClusterIP   80/TCP
```

### 3.4 Probes et resource limits

```bash
kubectl -n todo-api-dev describe pod -l app=todo-api | grep -A5 "Liveness\|Readiness\|Limits\|Requests"
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
kubectl get ns todo-api-dev todo-api-prod monitoring
kubectl get networkpolicies -A
kubectl get roles,rolebindings -n todo-api-prod
kubectl get sealedsecrets -A
```

---

## 4. Tests fonctionnels de l'application (httpbin)

`kennethreitz/httpbin` est un service HTTP de test. Chaque requête retourne les informations de la requête elle-même — pas d'état, pas de base de données.

### 4.1 Environnement dev

```bash
kubectl port-forward svc/todo-api -n todo-api-dev 3000:80 &

curl -s http://localhost:3000/get
# Attendu : JSON avec origin, headers, url

curl -s http://localhost:3000/headers
# Attendu : {"headers": {"Host": "localhost:3000", ...}}

curl -s http://localhost:3000/ip
# Attendu : {"origin": "..."}

curl -s -X POST http://localhost:3000/post -H "Content-Type: application/json" -d '{"env": "dev", "test": true}'
# Attendu : JSON avec json.env = "dev", json.test = true

curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/status/200
# Attendu : 200

curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/status/404
# Attendu : 404

curl -s http://localhost:3000/delay/2
# Attendu : réponse après ~2 secondes

kill %1
```

### 4.2 Environnement prod

```bash
kubectl port-forward svc/todo-api -n todo-api-prod 3001:80 &

curl -s http://localhost:3001/get
# Attendu : JSON valide, même comportement que dev

curl -s -X POST http://localhost:3001/post -H "Content-Type: application/json" -d '{"env": "prod", "test": true}'
# Attendu : json.env = "prod"

kill %1
```

### 4.3 Endpoint `/anything`

```bash
kubectl port-forward svc/todo-api -n todo-api-dev 3000:80 &

curl -s -X PUT http://localhost:3000/anything/custom-path -H "X-Custom-Header: gitops-test" -d "payload=test"
# Attendu : json avec method=PUT, headers.X-Custom-Header=gitops-test

kill %1
```

---

## 5. Test du self-heal et du prune ArgoCD

### 5.1 Self-heal (correction de drift)

```bash
kubectl -n todo-api-dev scale deployment todo-api --replicas=5
kubectl -n todo-api-dev get deploy todo-api
# Attendu temporaire : replicas = 5

watch kubectl -n todo-api-dev get deploy todo-api
# Attendu après ~1 minute : revient automatiquement à 1 replica (selfHeal ArgoCD)
```

### 5.2 Auto-prune (suppression des ressources orphelines)

```bash
kubectl -n todo-api-dev create configmap orphan-test --from-literal=key=value
kubectl -n todo-api-dev get configmap orphan-test
# Attendu : présente

sleep 180
kubectl -n todo-api-dev get configmap orphan-test
# Attendu : Error from server (NotFound)
```

---

## 6. Tests sécurité

### 6.1 Absence de secrets en clair

```bash
grep -r "password:\|token:\|apiKey:\|secret:" apps/ argocd/ infrastructure/ .github/ --include="*.yaml" | grep -v "secretKeyRef\|sealed\|encryptedData"
# Attendu : aucune ligne
```

### 6.2 Sealed Secrets

```bash
kubectl get pods -n kube-system | grep sealed-secrets
# Attendu : sealed-secrets-controller Running

kubectl get sealedsecrets -A
# Attendu : todo-api-secret dans todo-api-prod
```

### 6.3 Network Policies

```bash
kubectl get networkpolicies -n todo-api-prod
kubectl describe networkpolicy todo-api-netpol -n todo-api-prod
# Attendu : ingress sur port 80 depuis namespace environment=prod
```

### 6.4 RBAC

```bash
kubectl get roles,rolebindings -n todo-api-prod
# Attendu : todo-api-role, todo-api-rolebinding, todo-api-sa
```

### 6.5 Historique Git propre

```bash
git log --oneline -15
# Vérifier : aucun "wip", "fix", "test" sans contexte
```

---

## 7. Logs et diagnostic

```bash
kubectl -n todo-api-dev logs -l app=todo-api --tail=50
kubectl -n todo-api-prod logs -l app=todo-api --tail=50

kubectl -n todo-api-dev get events --sort-by='.lastTimestamp' | tail -10
kubectl -n todo-api-prod get events --sort-by='.lastTimestamp' | tail -10

kubectl -n argocd logs deploy/argocd-application-controller --tail=30

argocd app get todo-api-dev
argocd app get todo-api-prod
argocd app get infrastructure
```

---

## 8. Nettoyage

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
| 6 | ArgoCD opérationnel | `argocd app list` | 3 apps Synced + Healthy |
| 7 | Sealed Secrets running | `kubectl get pods -n kube-system \| grep sealed` | 1/1 Running |
| 8 | Pod dev running | `kubectl -n todo-api-dev get po` | 1/1 Running |
| 9 | Pod prod running | `kubectl -n todo-api-prod get po` | 3/3 Running |
| 10 | httpbin répond (dev) | `curl localhost:3000/get` | JSON valide |
| 11 | httpbin répond (prod) | `curl localhost:3001/get` | JSON valide |
| 12 | POST fonctionne | `curl -X POST .../post` | JSON avec body |
| 13 | Codes HTTP | `curl .../status/200` et `.../status/404` | 200 et 404 |
| 14 | Self-heal | Scale manuel → watch | Revient à 1 replica |
| 15 | Auto-prune | ConfigMap orphelin | Supprimé par ArgoCD |
| 16 | Sealed Secrets déployé | `kubectl get sealedsecrets -A` | todo-api-secret présent |
| 17 | Network Policies | `kubectl get netpol -A` | Présente dans todo-api-prod |
| 18 | Historique Git | `git log --oneline` | Commits propres |
