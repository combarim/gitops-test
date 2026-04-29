# GitOps Repository

Dépôt GitOps pour la gestion déclarative de l'infrastructure Kubernetes via ArgoCD.

## Architecture

Pattern **App-of-Apps** à trois niveaux :

```
Root App (global-root)
    ↓
Management Root App
    ↓
ApplicationSets (auto-discovery via config.json)
    ↓
Applications individuelles
```

## Structure

```
gitops/
├── apps/               # Définitions des applications
│   ├── core/           # Infrastructure (network, storage, observability)
│   └── workloads/      # Applications métier
│
└── clusters/           # Configuration par cluster
    ├── management/     # Cluster de management
    ├── preprod/        # Cluster de préproduction
    └── prod/           # Cluster de production
```

## Fonctionnement

### ApplicationSets

Les applications sont découvertes automatiquement via leurs fichiers `config.json` :

```json
{
  "syncWave": "100",
  "namespace": "observability"
}
```

**Pattern de découverte** : `apps/core/*/*/config.json`

Chaque app trouvée est déployée depuis `apps/core/{category}/{app}/overlays/{cluster}/`

### Structure d'une Application

```
apps/core/{category}/{app}/
├── base/
│   ├── kustomization.yaml      # Helm chart ou ressources K8s
│   └── values.yaml             # Configuration Helm
├── overlays/
│   └── mgmt/                   # Overlay spécifique cluster management
│       └── kustomization.yaml
└── config.json                 # Metadata ArgoCD
```

### Sync Waves

Contrôle l'ordre de déploiement des applications :

- **0-99** : Fondations (namespace, CRDs)
- **100-199** : Infrastructure core (storage, network, PKI)
- **200-299** : Applications métier
- **300+** : Post-installation

## Projects ArgoCD

- **core-infrastructure** : Apps d'infrastructure (peut créer ressources cluster-wide)
- **workloads** : Applications métier (limité aux ressources namespace)

## Ajouter une Nouvelle Application

1. Créer la structure dans `apps/core/{category}/{app}/`
2. Ajouter `config.json` avec syncWave et namespace
3. Créer `base/kustomization.yaml` avec Helm chart ou ressources
4. Créer overlay dans `overlays/{cluster}/`
5. Commit et push → ArgoCD découvre et déploie automatiquement

## Documentation

- [Applications](apps/README.md) - Organisation et structure des apps
- [Clusters](clusters/README.md) - Configuration par cluster
- [Apps Core](apps/core/README.md) - Infrastructure de base

## Liens Utiles

- **Repo Git** : `https://github.com/combarim/gitops-test.git`
- **Branch** : `main`
- **ArgoCD** : Déployé dans le namespace `argocd`
