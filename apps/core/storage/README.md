# Longhorn

Stockage persistant distribué cloud-native pour Kubernetes.

## Rôle

Fournit des volumes persistants (PVC) répliqués sur plusieurs nœuds avec snapshots, backups et disaster recovery.

## Composants

- **Longhorn Manager** : Contrôleur principal
- **Longhorn Engine** : Moteur de stockage par volume
- **Longhorn UI** : Interface web de gestion
- **CSI Driver** : Intégration Kubernetes (StorageClass, PVC)

**Sync Wave** : Varie selon config (généralement 100-110)
**Namespace** : `longhorn-system`

## Architecture

```
PVC (PersistentVolumeClaim)
    ↓
StorageClass (longhorn / longhorn-*)
    ↓
Longhorn CSI Driver
    ↓
Longhorn Manager (contrôleur)
    ↓
Longhorn Engine (1 par volume)
    ↓
Replicas (distribués sur nœuds)
    ↓
Disks locaux (/var/lib/longhorn)
```

## Configuration Critique

### StorageClasses

**StorageClass par défaut** : `longhorn`
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
provisioner: driver.longhorn.io
parameters:
  numberOfReplicas: "3"
  staleReplicaTimeout: "30"
  fsType: "ext4"
```

**StorageClass observability** : `longhorn-observability`
- Dédiée à la stack observability
- Configuration identique mais séparée

### Paramètres Importants

**numberOfReplicas** :
- Nombre de copies du volume
- Défaut : 3 (haute disponibilité)
- PoC : peut être réduit à 1 ou 2
- Impact : Moins de replicas = moins de résilience mais économie d'espace

**staleReplicaTimeout** :
- Délai avant considérer un replica comme obsolète (minutes)
- Impact : Trop court = reconstruction fréquente, trop long = données potentiellement périmées

**dataLocality** :
- `disabled` : replicas distribués librement
- `best-effort` : préfère placer un replica sur le nœud du pod
- Impact : `best-effort` améliore perf mais réduit la distribution

## Dépendances

- **iscsi-tools** : Requis sur les nœuds worker
- **Aucune app** : Longhorn est fondation pour toutes les autres stacks

## Utilisé par

- Observability (PVCs pour Grafana, Loki, VictoriaMetrics, MinIO)
- Toute app nécessitant stockage persistant

## Intégrations

### avec PVCs

Toute app peut créer un PVC utilisant Longhorn :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 10Gi
```

### avec StatefulSets

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: longhorn
        resources:
          requests:
            storage: 10Gi
```

## Endpoints

- **Longhorn UI** : `http://longhorn-frontend.longhorn-system.svc:80`
- Accès via port-forward : `kubectl port-forward -n longhorn-system svc/longhorn-frontend 8000:80`

## Usage

### Vérifier l'Installation

```bash
# Pods Longhorn
kubectl get pods -n longhorn-system

# StorageClasses
kubectl get storageclass

# Volumes provisionnés
kubectl get pv
```

### Créer un PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

### Créer un Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: longhorn-snapshot-vsc
  source:
    persistentVolumeClaimName: my-pvc
```

### Accéder à l'UI

```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80
```

Ouvrir `http://localhost:8080`

## Operations

### Changer le Nombre de Replicas par Défaut

Via l'UI Longhorn : **Settings → General → Default Replica Count**

Ou via ConfigMap/Settings CRD si déployé via Helm.

### Expansion de Volume

Longhorn supporte l'expansion de PVCs à chaud :

```bash
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'
```

### Backup vers S3

Configurer backup target dans Settings :
```
s3://bucket-name@region/path?endpoint=https://s3.example.com
```

Créer backup :
```bash
# Via UI ou kubectl
kubectl create -f backup.yaml
```

## Troubleshooting

**PVC en Pending** :
1. Vérifier Longhorn Manager running : `kubectl get pods -n longhorn-system`
2. Vérifier nœuds avec disks : Longhorn UI → Node
3. Vérifier events : `kubectl describe pvc <name>`

**Volume dégradé** :
1. Vérifier replicas : Longhorn UI → Volume
2. Vérifier santé nœuds : `kubectl get nodes`
3. Vérifier espace disque nœuds : `df -h /var/lib/longhorn`

**Performance lente** :
1. Vérifier `dataLocality: best-effort` activé
2. Vérifier nombre de replicas (réduire si acceptable)
3. Vérifier I/O des disques sous-jacents

## Sécurité

- Volumes chiffrés au niveau filesystem (ext4/xfs)
- Pas de chiffrement at-rest natif (utiliser LUKS sur nœuds si requis)
- RBAC pour accès UI et ressources
- Network policies recommandées pour isoler trafic iSCSI

## Ressources

- CPU/RAM : Variable selon nombre de volumes
- Espace disque : Par défaut `/var/lib/longhorn` sur chaque nœud
- Recommandation : Dédier un disque séparé pour Longhorn en production
