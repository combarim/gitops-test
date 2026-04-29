# Observability Stack

Stack complète de monitoring : collecte de métriques, agrégation de logs et visualisation.

## Composants

| App | Rôle | Sync Wave | Replicas | Storage |
|-----|------|-----------|----------|---------|
| observability-base | Namespace + Prometheus CRDs | 100 | - | - |
| minio-operator | Opérateur pour stockage S3 | 101 | 1 | - |
| minio-tenant | Instance MinIO (buckets S3) | 102 | 1 | 6Gi |
| victoria-metrics | TSDB - Stockage métriques | 103 | 1 | 8Gi |
| loki | Agrégation et stockage logs | 103 | 3 | 8Gi |
| grafana | Visualisation métriques + logs | 104 | 1 | 2Gi |
| grafana-alloy | Collecteur métriques (DaemonSet) | 105 | DS | - |

**Total Storage** : ~24Gi (StorageClass: `longhorn-observability`)
**Total CPU** : ~750m requests
**Total RAM** : ~2Gi requests / ~5.5Gi limits

## Architecture

### Flux Métriques
```
Applications
    ↓ (annotations prometheus.io/scrape: "true")
Grafana Alloy (DaemonSet)
    ↓ (Kubernetes Service Discovery)
    ↓ (prometheus.scrape → 30s interval)
VictoriaMetrics
    ↓ (Prometheus API compatible)
Grafana
```

### Flux Logs
```
Applications (stdout/stderr)
    ↓
Loki Write (ingestion)
    ↓
MinIO S3 (stockage chunks + index)
    ↓
Loki Read (query)
    ↓
Grafana
```

### Intégrations
```
┌─────────────┐
│   Grafana   │
│ (port 80)   │
└──────┬──────┘
       │
       ├──→ VictoriaMetrics (:8428) ─→ Métriques
       └──→ Loki Gateway (:80) ─────→ Logs
```

## Dépendances

- **Longhorn** : StorageClass `longhorn-observability` pour tous les PVCs
- **Network** : Aucune (accès interne ClusterIP uniquement)

## Configuration Critique

### VictoriaMetrics (`victoria-metrics/base/values.yaml`)

**Rétention et stockage** :
- `retentionPeriod: "7"` - Durée conservation des métriques (jours)
- `persistentVolume.size: 8Gi` - Taille du TSDB
- Impact : Augmenter la rétention augmente l'usage disque proportionnellement

**Resources** :
- `resources.requests.memory: 512Mi` - RAM minimum
- `resources.limits.memory: 1Gi` - RAM maximum
- `extraArgs.memory.allowedPercent: "80"` - Limite interne 80% de la RAM allouée
- Impact : Métriques perdues si RAM insuffisante

**Limites de requêtes** :
- `extraArgs.search.maxQueryDuration: 60s` - Timeout requêtes
- `extraArgs.search.maxConcurrentRequests: "8"` - Max requêtes parallèles
- Impact : Protège contre les requêtes abusives

### Loki (`loki/base/values.yaml`)

**Stockage S3** :
- `storage_config.aws.endpoint: minio.observability.svc:80` - Backend MinIO
- `storage_config.aws.s3: s3://minio.observability.svc:80/loki-chunks` - Bucket chunks
- Variables env : `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY` (via secret `minio-root-creds`)
- Impact : Loki ne démarre pas sans accès S3

**Rétention** :
- `limits_config.retention_period: 168h` - 7 jours de logs
- `compactor.retention_enabled: true` - Nettoyage automatique
- Impact : Logs > 7j sont supprimés définitivement

**Limites ingestion** :
- `limits_config.ingestion_rate_mb: 10` - Max 10MB/s par tenant
- `limits_config.max_streams_per_user: 10000` - Max streams simultanés
- `limits_config.max_line_size: 256KB` - Taille max d'une ligne
- Impact : Logs dépassant les limites sont rejetés

**Composants** (SimpleScalable mode) :
- Gateway : nginx reverse proxy
- Read : traitement des requêtes
- Write : ingestion des logs
- Backend : compaction, ruler, index

### Grafana Alloy (`grafana-alloy/base/values.yaml`)

**Déploiement** :
- `controller.type: daemonset` - 1 pod par nœud
- Impact : Collecte locale, réduit la latence réseau

**Auto-discovery** :
- `discovery.kubernetes` pour pods, services, nodes, endpoints
- `discovery.relabel` pour filtrage via annotations
- Scrape uniquement si `prometheus.io/scrape: "true"`
- Impact : Aucune config manuelle requise

**Remote Write** :
- `prometheus.remote_write.endpoint.url` - VictoriaMetrics
- `queue_config.max_shards: 10` - Max queues parallèles
- `queue_config.capacity: 10000` - Buffer des samples
- Impact : Ajuster si métriques perdues sous forte charge

### Grafana (`grafana/base/values.yaml`)

**Datasources** :
- VictoriaMetrics (défaut) : `http://victoria-metrics-victoria-metrics-single-server.observability.svc:8428`
- Loki : `http://loki-gateway.observability.svc:80`
- Impact : Modifier les endpoints si renommage des services

**Credentials** :
- `admin.existingSecret: grafana-admin-secret` - À créer manuellement
- Clés : `admin-user`, `admin-password`
- Impact : Grafana ne démarre pas sans ce secret

**Persistence** :
- `persistence.size: 2Gi` - Stockage dashboards et configuration
- Impact : Dashboards perdus si PVC supprimé

### MinIO Tenant (`minio-tenant/base/`)

**Buckets** :
- `loki-chunks` - Stockage des logs compressés
- `loki-ruler` - Règles Loki (alertes logs)
- Impact : Loki ne fonctionne pas sans ces buckets

**Credentials** :
- Secret `minio-root-creds` (minio-creds.yaml)
- Clés : `username`, `password`, `config.env`
- Impact : À modifier en production (actuellement credentials par défaut)

**Storage** :
- `pools.size: 3Gi` × 2 volumes = 6Gi total
- Impact : Augmenter si logs volumineux

## Endpoints

| Service | Port | URL interne | Usage |
|---------|------|-------------|-------|
| VictoriaMetrics | 8428 | `victoria-metrics-victoria-metrics-single-server.observability.svc:8428` | Prometheus API (write/query) |
| Loki Gateway | 80 | `loki-gateway.observability.svc:80` | Loki API (push/query) |
| Grafana | 80 | `grafana.observability.svc:80` | Interface web |
| MinIO | 80 | `minio.observability.svc:80` | S3 API |

## Intégrations

### Grafana ← VictoriaMetrics
- Datasource Prometheus-compatible
- Requêtes PromQL
- URL : `/api/v1/query` et `/api/v1/query_range`

### Grafana ← Loki
- Datasource Loki
- Requêtes LogQL
- URL : `/loki/api/v1/query` et `/loki/api/v1/query_range`

### Loki → MinIO
- Stockage S3 des chunks (logs compressés)
- Stockage index (TSDB)
- Bucket ruler pour règles d'alerte

### Alloy → VictoriaMetrics
- Remote write Prometheus
- Endpoint : `/api/v1/write`
- Format : Prometheus Remote Write Protocol

## Usage

### Activer le Scraping sur une Application

Ajouter annotations sur le **Pod** ou **Service** :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"        # optionnel (défaut: port du service)
    prometheus.io/path: "/metrics"    # optionnel (défaut: /metrics)
spec:
  ports:
    - port: 8080
      name: metrics
```

Grafana Alloy découvrira et scrapera automatiquement l'endpoint.

### Créer le Secret Grafana

```bash
kubectl create secret generic grafana-admin-secret \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=CHANGE_ME \
  -n observability
```

### Accéder à Grafana

```bash
kubectl port-forward -n observability svc/grafana 3000:80
```

Ouvrir `http://localhost:3000`

### Requêtes PromQL Exemple

```promql
# CPU usage par pod
rate(container_cpu_usage_seconds_total[5m])

# Mémoire usage
container_memory_working_set_bytes

# Requêtes HTTP par code
rate(http_requests_total[5m])
```

### Requêtes LogQL Exemple

```logql
# Tous les logs d'un namespace
{namespace="default"}

# Logs avec erreur
{namespace="default"} |~ "error|ERROR"

# Rate de logs
rate({namespace="default"}[5m])
```

## Sécurité

- Tous les pods en non-root
- SecurityContext : `runAsNonRoot: true`
- Capabilities : `drop: [ALL]`
- Seccomp : `RuntimeDefault`
- Read-only filesystem (Alloy)

## Opérations

### Vérifier la Collecte de Métriques

```bash
# Vérifier les targets Alloy
kubectl logs -n observability -l app.kubernetes.io/name=alloy

# Vérifier VictoriaMetrics ingestion
kubectl port-forward -n observability svc/victoria-metrics-victoria-metrics-single-server 8428:8428
curl http://localhost:8428/api/v1/status/tsdb
```

### Vérifier Loki

```bash
# Status Loki
kubectl logs -n observability -l app.kubernetes.io/component=gateway

# Tester ingestion
kubectl port-forward -n observability svc/loki-gateway 3100:80
curl http://localhost:3100/ready
```

### Troubleshooting

**Métriques non collectées** :
1. Vérifier annotations `prometheus.io/scrape: "true"`
2. Vérifier logs Alloy : `kubectl logs -n observability -l app.kubernetes.io/name=alloy`
3. Vérifier endpoint accessible : `curl http://POD_IP:PORT/metrics`

**Logs non visibles** :
1. Vérifier Loki Write : `kubectl logs -n observability -l app.kubernetes.io/component=write`
2. Vérifier MinIO accessible : `kubectl get pods -n observability -l app.kubernetes.io/name=minio`
3. Vérifier credentials S3 dans secret `minio-root-creds`

**Grafana inaccessible** :
1. Vérifier secret `grafana-admin-secret` existe
2. Vérifier PVC bound : `kubectl get pvc -n observability`
3. Vérifier logs : `kubectl logs -n observability -l app.kubernetes.io/name=grafana`
