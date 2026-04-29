# Core Infrastructure

Applications d'infrastructure fondamentale déployées sur tous les clusters.

## Stacks

| Stack | Rôle | Sync Waves | Doc |
|-------|------|------------|-----|
| [GitOps](gitops/) | ArgoCD - Auto-gestion du cluster | 0 | [README](gitops/README.md) |
| [Storage](storage/) | Longhorn - Stockage persistant distribué | 100-110 | [README](storage/README.md) |
| [Certificates](certificates/) | PKI - Gestion des certificats TLS | 110-120 | [README](certificates/README.md) |
| [Network](network/) | Réseau - LoadBalancer, Ingress, Gateway API | 100-130 | [README](network/README.md) |
| [Observability](observability/) | Monitoring - Métriques, logs, visualisation | 100-105 | [README](observability/README.md) |

## Ordre de Déploiement

Les sync waves garantissent l'ordre correct :

```
Wave 0     → ArgoCD (auto-gestion)
Wave 100   → Longhorn (stockage) + Network base (MetalLB)
Wave 101   → MinIO operator (observability)
Wave 102   → MinIO tenant (observability)
Wave 103   → VictoriaMetrics + Loki (observability)
Wave 104   → Grafana (observability)
Wave 105   → Grafana Alloy (observability)
Wave 110   → cert-manager (certificats)
Wave 120   → Envoy Gateway (network)
Wave 130   → Ingress Gateway (network)
```

## Dépendances entre Stacks

```
Longhorn
    ↓
    ├─→ Observability (PVCs pour Grafana, Loki, VictoriaMetrics, MinIO)
    └─→ Network (PVCs potentiels)

Network (MetalLB)
    ↓
    ├─→ Ingress Gateway (LoadBalancer IPs)
    └─→ Services externes

Certificates (cert-manager)
    ↓
    └─→ Ingress Gateway (TLS certificates)

Observability
    └─→ (Indépendant - monitoring de toutes les autres stacks)
```

## Project ArgoCD

Toutes les apps core utilisent le project `core-infrastructure` :

**Permissions** :
- Création ressources cluster-wide (CRDs, ClusterRoles, etc.)
- Déploiement dans tous les namespaces
- Gestion des StorageClasses, IngressClasses, etc.

## StorageClasses

Les stacks utilisent des StorageClasses dédiées :

| StorageClass | Usage | Provider |
|--------------|-------|----------|
| `longhorn-observability` | Observability stack | Longhorn |
| `longhorn` | Usage général | Longhorn |

## Configuration Globale

### Namespaces

| Namespace | Stack | Annotations |
|-----------|-------|-------------|
| `argocd` | ArgoCD | `Prune=false` |
| `observability` | Observability | `Prune=false` |
| `cert-manager` | Certificates | - |
| `longhorn-system` | Longhorn | - |
| `metallb-system` | Network (MetalLB) | - |
| `envoy-gateway-system` | Network (Envoy) | - |
| `gateway-system` | Network (Ingress) | - |

### Ressources Totales (Cluster Management)

| Stack | CPU Requests | RAM Requests | Storage |
|-------|--------------|--------------|---------|
| ArgoCD | ~300m | ~1Gi | 2Gi |
| Observability | ~750m | ~2Gi | ~24Gi |
| Longhorn | Variable | Variable | - |
| Network | ~200m | ~500Mi | - |
| Certificates | ~100m | ~200Mi | - |
| **Total** | **~1.35 CPU** | **~3.7Gi RAM** | **~26Gi** |

## Catégories

### gitops/
Applications GitOps et déploiement continu :
- **argocd** : Contrôleur GitOps principal

Future : Argo Workflows, Argo Rollouts, Flux, etc.

### storage/
Solutions de stockage persistant :
- **longhorn** : Stockage distribué cloud-native

Future : Rook-Ceph, OpenEBS, etc.

### observability/
Stack complète de monitoring :
- **observability-base** : Namespace + CRDs
- **minio-operator** / **minio-tenant** : Backend S3
- **victoria-metrics** : TSDB métriques
- **loki** : Agrégation logs
- **grafana** : Visualisation
- **grafana-alloy** : Collecteur

### network/
Infrastructure réseau :
- **metallb** : LoadBalancer L2
- **envoy-gateway** : Implémentation Gateway API
- **ingress-gateway** : Instances Gateway

### certificates/
PKI et gestion TLS :
- **cert-manager** : Contrôleur de certificats

## Documentation Détaillée

Voir les README de chaque stack pour :
- Architecture et flux de données
- Configuration critique
- Endpoints et services
- Intégrations avec autres stacks
- Opérations courantes
