# ArgoCD

Plateforme GitOps pour déploiement continu Kubernetes.

## Rôle

ArgoCD synchronise automatiquement l'état du cluster avec les manifestes Git, implémentant le pattern GitOps.

**Sync Wave** : 0 (déployé en premier)
**Namespace** : `argocd`

## Architecture

```
Git Repository (source of truth)
    ↓
ArgoCD Application Controller (polling/webhook)
    ↓
Kubernetes API (apply manifests)
    ↓
Cluster (état désiré = état actuel)
```

## Pattern App-of-Apps

Ce dépôt utilise le pattern App-of-Apps à 3 niveaux :

```
global-root (Root App)
    ↓
management-root (Management App)
    ↓
ApplicationSet (Auto-discovery)
    ↓
Applications individuelles
```

### global-root
- Déploie Projects ArgoCD
- Déploie management-root App
- Sync wave : 0-2

### management-root
- Pointe vers `clusters/management/apps/`
- Déclenche ApplicationSets
- Sync wave : 2

### ApplicationSets
- Découvre apps via `config.json`
- Pattern : `apps/core/*/*/config.json`
- Crée automatiquement Applications ArgoCD

## Projects ArgoCD

### core-infrastructure
```yaml
spec:
  destinations:
    - namespace: "*"
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: "*"
      kind: "*"
```

Peut créer ressources cluster-wide (CRDs, ClusterRoles, StorageClasses, etc.)

### workloads
```yaml
spec:
  destinations:
    - namespace: "*"
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
    - group: "*"
      kind: "*"
```

Limité aux ressources namespace uniquement.

## Configuration Critique

### Sync Policies

**Automated** :
```yaml
syncPolicy:
  automated:
    prune: true        # Supprime ressources non présentes dans Git
    selfHeal: true     # Corrige drift automatiquement
```

**ServerSideApply** :
```yaml
syncOptions:
  - ServerSideApply=true
```

Recommandé pour ressources avec multi-ownership (CRDs, etc.)

### Retry Backoff

```yaml
syncPolicy:
  retry:
    limit: 5
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

Gère les échecs temporaires (ressources non disponibles, etc.)

## Endpoints

- **UI** : `http://argocd-server.argocd.svc:80`
- **API** : `http://argocd-server.argocd.svc:80`
- **Metrics** : `http://argocd-metrics.argocd.svc:8082/metrics`

## Intégrations

### avec Git Repository
- Repo : `https://github.com/combarim/gitops-test.git`
- Branch : `main`
- Polling interval ou webhooks

### avec Kustomize
Applications utilisent Kustomize pour templating :
```yaml
source:
  repoURL: https://github.com/combarim/gitops-test.git
  targetRevision: main
  path: apps/core/observability/grafana/overlays/mgmt
```

### avec Helm
Via Kustomize `helmCharts` pour intégration native.

## Usage

### Accéder à l'UI

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:80
```

Login :
```bash
# Récupérer password initial
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# User : admin
```

### Lister les Applications

```bash
kubectl get applications -n argocd
```

### Synchroniser Manuellement

```bash
kubectl patch app <app-name> -n argocd -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}' --type merge
```

Ou via UI : **App → SYNC**

### Ajouter une Nouvelle App

1. Créer structure `apps/core/{category}/{app}/`
2. Ajouter `config.json`
3. Commit + push
4. ApplicationSet découvre et déploie automatiquement

### Voir les Sync Waves

```bash
kubectl get app -n argocd -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.argocd\.argoproj\.io/sync-wave}{"\n"}{end}' | sort -k2 -n
```

## Opérations

### Refresh App (Re-scan Git)

```bash
kubectl patch app <app-name> -n argocd -p '{"operation":{"initiatedBy":{"username":"admin"},"info":[{"name":"Reason","value":"manual refresh"}]}}' --type merge
```

### Hard Refresh (Ignore Cache)

Via UI : **App → REFRESH HARD**

### Sync avec Prune

Force suppression des ressources non présentes dans Git :
```bash
kubectl patch app <app-name> -n argocd -p '{"operation":{"sync":{"prune":true}}}' --type merge
```

### Rollback

ArgoCD garde historique des syncs. Via UI :
**App → HISTORY → SELECT REVISION → SYNC**

## Troubleshooting

**App en OutOfSync** :
1. Comparer Git vs Cluster : UI → App → DIFF
2. Vérifier dernière sync : UI → App → LAST SYNC
3. Sync manuellement si selfHeal désactivé

**App en Progressing** (bloquée) :
1. Vérifier hooks : `kubectl get app <name> -o yaml | grep -A 10 operation`
2. Vérifier sync waves : Apps avec wave plus élevée attendent wave précédente
3. Vérifier logs : `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller`

**Resource Health Unknown** :
1. ArgoCD ne connaît pas le type de ressource
2. Ajouter health check custom si nécessaire
3. Ou ignorer : `argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true`

**CRD Install Failed** :
1. Utiliser `Replace=true` : `argocd.argoproj.io/sync-options: Replace=true`
2. Ou ServerSideApply
3. Ou séparer CRDs dans app dédiée (wave 0)

## Sécurité

- RBAC intégré
- Projects pour isolation
- SSO possible (OIDC, SAML, LDAP)
- Audit logs complets
- Secrets chiffrés dans etcd (natif K8s)

## Auto-Gestion

ArgoCD se gère lui-même via GitOps :
- App ArgoCD présente dans `apps/core/argocd/`
- Modifications ArgoCD dans Git sont auto-appliquées
- Attention aux erreurs de config (risque de casser ArgoCD)

## Ressources

- **UI + API Server** : ~100m CPU, ~256Mi RAM
- **Application Controller** : ~100m CPU, ~512Mi RAM
- **Repo Server** : ~50m CPU, ~256Mi RAM
- **Redis** : ~50m CPU, ~256Mi RAM
