# Applications

Organisation des applications Kubernetes déployées via ArgoCD.

## Structure

```
apps/
├── core/           # Infrastructure de base (obligatoire)
└── workloads/      # Applications métier
```

## Apps Core vs Workloads

### Core
Infrastructure fondamentale du cluster :
- Réseau (LoadBalancer, Ingress, Gateway API)
- Stockage persistant (Longhorn)
- Certificats (cert-manager)
- Observabilité (métriques, logs, monitoring)
- ArgoCD (auto-gestion)

**Project ArgoCD** : `core-infrastructure` (peut créer ressources cluster-wide)

### Workloads
Applications métier et services :
- APIs backend
- Frontends web
- Services de données
- Jobs et CronJobs

**Project ArgoCD** : `workloads` (limité aux ressources namespace uniquement)

## Structure d'une Application

Chaque application suit ce pattern :

```
{category}/{app-name}/
├── base/
│   ├── kustomization.yaml      # Définition Helm ou ressources K8s
│   └── values.yaml             # Configuration (si Helm)
│
├── overlays/
│   ├── mgmt/                   # Overlay cluster management
│   ├── preprod/                # Overlay cluster preprod
│   └── prod/                   # Overlay cluster prod
│
├── components/                 # (optionnel) Composants réutilisables
│   └── feature-x/
│
└── config.json                 # Metadata ArgoCD (syncWave, namespace)
```

## config.json

Fichier de métadonnées utilisé par ApplicationSets pour découvrir et déployer l'app :

```json
{
  "syncWave": "100",
  "namespace": "my-namespace"
}
```

**Champs** :
- `syncWave` : Ordre de déploiement (0-999)
- `namespace` : Namespace cible Kubernetes

## Kustomize Base

### Avec Helm Chart

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

helmCharts:
  - name: chart-name
    repo: https://charts.example.com
    version: 1.2.3
    releaseName: my-app
    namespace: my-namespace
    valuesFile: values.yaml
```

### Avec Ressources K8s

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
```

## Overlays

Les overlays héritent de la base et ajoutent des personnalisations par environnement :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

# Optionnel : patches, replicas, images, etc.
```

## Sync Waves Recommandées

| Range | Usage | Exemples |
|-------|-------|----------|
| 0-99 | Fondations | Namespaces, CRDs, Operators |
| 100-149 | Infrastructure base | Storage, Network |
| 150-199 | Services partagés | Monitoring, Logging |
| 200-299 | Applications métier | APIs, Frontends |
| 300+ | Post-installation | Jobs, Migrations |

## Découverte Automatique

ApplicationSets découvrent automatiquement les apps via :

**Pattern** : `apps/core/*/*/config.json`

Toute app avec un `config.json` valide est déployée automatiquement sur le cluster correspondant.

## Documentation

- [Core Infrastructure](core/README.md) - Apps d'infrastructure
- [Workloads](workloads/README.md) - Applications métier
