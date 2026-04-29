# Workloads

Applications métier déployées sur les clusters.

## Rôle

Applications de production et services métier, séparées de l'infrastructure core.

**Project ArgoCD** : `workloads`
**Permissions** : Ressources namespace uniquement (pas de cluster-wide)

## Structure

```
workloads/
├── {app-name}/
│   ├── base/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ...
│   ├── overlays/
│   │   ├── preprod/
│   │   └── prod/
│   └── config.json
```

## Différences avec Core

| Aspect | Core | Workloads |
|--------|------|-----------|
| **Scope** | Infrastructure | Applications métier |
| **Permissions** | Cluster-wide | Namespace uniquement |
| **Exemples** | Storage, Network, Monitoring | APIs, Frontends, Services |
| **Project** | core-infrastructure | workloads |
| **Sync Waves** | 0-199 | 200+ |

## Restrictions

Les workloads **NE PEUVENT PAS** créer :
- CRDs
- ClusterRoles / ClusterRoleBindings
- StorageClasses
- IngressClasses
- PriorityClasses (cluster-wide)
- Namespaces (doivent être pré-créés)

## Sync Waves Recommandées

| Wave | Usage |
|------|-------|
| 200 | Namespaces et resources de base |
| 210 | ConfigMaps, Secrets |
| 220 | Deployments, StatefulSets |
| 230 | Services |
| 240 | Ingress / HTTPRoutes |
| 250+ | Jobs, CronJobs |

## Exemple d'Application

### Structure Complète

```
workloads/my-api/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── httproute.yaml
├── overlays/
│   ├── preprod/
│   │   ├── kustomization.yaml
│   │   └── patches.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patches.yaml
└── config.json
```

### config.json

```json
{
  "syncWave": "220",
  "namespace": "my-api"
}
```

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: my-api

resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - httproute.yaml
```

### overlays/prod/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: my-api
    count: 3

images:
  - name: my-api
    newTag: v1.2.3
```

## Bonnes Pratiques

### Gestion des Secrets

Ne **JAMAIS** commiter secrets en clair. Options :

1. **Sealed Secrets** (recommandé)
2. **External Secrets Operator**
3. **SOPS + Age/GPG**
4. **Vault**

### Limites de Ressources

Toujours définir `requests` et `limits` :

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Health Checks

Définir `livenessProbe` et `readinessProbe` :

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

### Labels Standards

```yaml
metadata:
  labels:
    app.kubernetes.io/name: my-api
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: my-platform
```

## Intégrations avec Core

### Stockage (Longhorn)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

### Réseau (Gateway API)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
spec:
  parentRefs:
    - name: internal-gateway
      namespace: gateway-system
  rules:
    - backendRefs:
        - name: my-api
          port: 80
```

### Observability (Metrics)

Ajouter annotations pour scraping Alloy :

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

### Certificats (cert-manager)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-api-tls
spec:
  secretName: my-api-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.example.com
```

## Ajouter une Nouvelle Workload

1. Créer structure dans `workloads/{app-name}/`
2. Ajouter `config.json` avec syncWave >= 200
3. Créer manifestes dans `base/`
4. Créer overlays par environnement
5. Commit + push → ArgoCD découvre et déploie

## Troubleshooting

**App non découverte par ApplicationSet** :
- Vérifier `config.json` présent et valide
- Vérifier pattern ApplicationSet match le path
- Vérifier project `workloads` existe

**Permission denied cluster-wide resource** :
- Workloads ne peuvent pas créer ressources cluster-wide
- Déplacer ressource dans core ou demander création manuelle

**PVC non provisionné** :
- Vérifier StorageClass existe (core/longhorn)
- Vérifier quotas namespace
