# Clusters

Configuration spécifique par cluster Kubernetes.

## Structure

```
clusters/
├── management/     # Cluster de management (prod-like)
├── preprod/        # Cluster de préproduction
└── prod/           # Cluster de production (futur)
```

Chaque cluster contient :

```
{cluster-name}/
├── root/                   # Applications racine ArgoCD
│   ├── global-root.yaml    # Root app (wave 0)
│   ├── management-root.yaml # Management app (wave 2)
│   └── projects.yaml       # Projects ArgoCD
│
└── apps/                   # ApplicationSets
    └── management-appset.yaml
```

## Pattern de Déploiement

### 1. Bootstrap Initial (Ansible)

Le rôle Ansible `argocd` déploie `global-root.yaml` :

```bash
ansible-playbook -i inventories/management_cluster.yaml \
  playbooks/bootstrap_argocd.yml
```

### 2. Root App (Wave 0)

`global-root.yaml` pointe vers `clusters/management/root/` et déploie :
- Projects ArgoCD (`projects.yaml`)
- Management Root App (`management-root.yaml`)

### 3. Management Root App (Wave 2)

`management-root.yaml` pointe vers `clusters/management/apps/` et déploie les ApplicationSets.

### 4. ApplicationSets

`management-appset.yaml` découvre automatiquement les apps via `config.json` :

**Pattern de découverte** : `apps/core/*/*/config.json`
**Path de déploiement** : `apps/core/{category}/{app}/overlays/mgmt`

## Projects ArgoCD

Définis dans `clusters/{cluster}/root/projects.yaml` :

### core-infrastructure
- **Scope** : Apps d'infrastructure (network, storage, observability)
- **Permissions** : Ressources cluster-wide autorisées
- **Namespaces** : Tous

### workloads
- **Scope** : Applications métier
- **Permissions** : Ressources namespace uniquement
- **Namespaces** : Tous

## ApplicationSets

Configuration automatique des applications :

```yaml
spec:
  goTemplate: true
  generators:
    - git:
        repoURL: https://github.com/combarim/gitops-test.git
        revision: main
        files:
          - path: "apps/core/*/*/config.json"
  template:
    metadata:
      name: '{{.path.basenameNormalized}}-mgmt'
      annotations:
        argocd.argoproj.io/sync-wave: '{{default "0" .syncWave}}'
    spec:
      project: core-infrastructure
      source:
        path: 'apps/core/{{index .path.segments 2}}/{{index .path.segments 3}}/overlays/mgmt'
```

**Résultat** :
- Toute app avec `config.json` est déployée automatiquement
- Nom : `{app-name}-mgmt`
- Sync wave depuis `config.json`
- Overlay : `overlays/mgmt/`

## Configuration par Cluster

### Management

**Réseau** :
- Nodes : `10.0.0.20-22`
- MetalLB pool : `10.0.0.0-10.0.0.250`
- Gateway API : Activée

**Apps déployées** :
- Toutes les apps core
- Pas de workloads (cluster infrastructure uniquement)

**Spécificités** :
- ArgoCD auto-géré
- Observability complète
- Tous les CRDs et operators

### Preprod

**Réseau** : À configurer selon infrastructure
**Apps déployées** : Apps core + workloads preprod
**Spécificités** : Environnement de test avant production

### Prod

**Statut** : Non déployé
**Prévu** : Apps core + workloads production
**Haute disponibilité** : Replicas augmentés, multi-zones

## Ajouter un Nouveau Cluster

1. **Créer la structure** :
```bash
mkdir -p clusters/new-cluster/{root,apps}
```

2. **Copier les manifestes root** :
```bash
cp clusters/management/root/* clusters/new-cluster/root/
```

3. **Adapter `global-root.yaml`** :
```yaml
spec:
  source:
    path: clusters/new-cluster/root
```

4. **Adapter `management-root.yaml`** :
```yaml
spec:
  source:
    path: clusters/new-cluster/apps
```

5. **Créer ApplicationSet** dans `apps/` :
```yaml
# Adapter pattern et overlay
path: 'apps/core/{{...}}/overlays/new-cluster'
```

6. **Créer overlays** pour chaque app :
```bash
mkdir -p apps/core/{category}/{app}/overlays/new-cluster
```

7. **Bootstrap ArgoCD** via Ansible avec variables adaptées

## Synchronisation Multi-Cluster

Actuellement chaque cluster est indépendant :
- Management : overlay `mgmt`
- Preprod : overlay `preprod`
- Prod : overlay `prod`

Pour apps multi-cluster, partager la base et personnaliser l'overlay.

## Sécurité

- Root apps protégées contre prune (`Prune=false`)
- Projects isolent core vs workloads
- RBAC ArgoCD par project
- Credentials stockés dans secrets K8s

## Opérations

### Vérifier les Root Apps

```bash
kubectl get app -n argocd | grep root
```

### Vérifier ApplicationSets

```bash
kubectl get applicationset -n argocd
```

### Lister Apps d'un Cluster

```bash
kubectl get app -n argocd -l cluster=management
```

### Troubleshooting Bootstrap

Si root app ne démarre pas :
1. Vérifier déploiement ArgoCD : `kubectl get pods -n argocd`
2. Vérifier Application créée : `kubectl get app -n argocd root-app`
3. Vérifier logs : `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller`
4. Vérifier accès Git : tester clone manuel depuis un pod
