# Certificates Stack

Gestion automatique des certificats TLS via cert-manager.

## Composants

| App | Rôle | Sync Wave |
|-----|------|-----------|
| cert-manager | Contrôleur de certificats TLS | 110 |

## Architecture

```
Certificate (CRD)
    ↓
cert-manager (contrôleur)
    ↓
Issuer / ClusterIssuer
    ↓
ACME / CA / Vault
    ↓
Secret (certificat TLS)
    ↓
Ingress / Gateway (référence secret)
```

## Dépendances

- **Aucune** : cert-manager fonctionne de manière autonome
- **Utilisé par** : Network (Gateways), applications exposées en HTTPS

## Configuration Critique

### cert-manager

**Namespace** : `cert-manager`

**CRDs installées** :
- `Certificate` - Demande de certificat
- `Issuer` - Émetteur de certificats (namespace-scoped)
- `ClusterIssuer` - Émetteur cluster-wide
- `CertificateRequest` - Requête interne
- `Order`, `Challenge` - Pour ACME (Let's Encrypt)

**Ressources** :
- Installation via Helm chart
- CRDs incluses
- Webhooks pour validation

## Intégrations

### avec Gateways (Gateway API)

Les Gateways référencent directement les Secrets générés :

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
spec:
  listeners:
    - name: https
      protocol: HTTPS
      tls:
        certificateRefs:
          - name: my-cert-tls  # Secret créé par Certificate
```

### avec Certificate CRD

Création automatique de certificats :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cert
  namespace: my-namespace
spec:
  secretName: my-cert-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - www.example.com
```

## Usage

### Créer un ClusterIssuer (Let's Encrypt)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: envoy-gateway
```

### Créer un Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls
  namespace: my-app
spec:
  secretName: app-tls-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - app.example.com
  duration: 2160h  # 90 days
  renewBefore: 720h  # 30 days
```

### Vérifier les Certificats

```bash
# Lister tous les certificats
kubectl get certificate -A

# Détails d'un certificat
kubectl describe certificate my-cert -n my-namespace

# Vérifier le secret généré
kubectl get secret my-cert-tls -n my-namespace
```

### Vérifier cert-manager

```bash
# Pods cert-manager
kubectl get pods -n cert-manager

# Logs
kubectl logs -n cert-manager -l app=cert-manager

# CRDs installées
kubectl get crd | grep cert-manager
```

## Troubleshooting

**Certificate en NotReady** :
1. Vérifier CertificateRequest : `kubectl get certificaterequest -n <namespace>`
2. Vérifier Order/Challenge (ACME) : `kubectl get order,challenge -n <namespace>`
3. Vérifier logs : `kubectl logs -n cert-manager -l app=cert-manager`
4. Vérifier Issuer existe : `kubectl get issuer,clusterissuer`

**ACME Challenge échoue** :
1. Vérifier DNS pointe vers LoadBalancer
2. Vérifier HTTPRoute/Ingress pour `/.well-known/acme-challenge/`
3. Tester accessibilité externe : `curl http://domain/.well-known/acme-challenge/test`

**Renouvellement bloqué** :
1. Vérifier date d'expiration : `kubectl describe certificate`
2. Forcer renouvellement : `kubectl delete certificaterequest -n <namespace> <name>`
3. Vérifier quota Let's Encrypt (5 certs/domaine/semaine)

## Sécurité

- Secrets TLS générés automatiquement
- Private keys stockées dans Secrets K8s
- Rotation automatique avant expiration
- Webhook validation pour CRDs

## Opérations

### Forcer Renouvellement

```bash
kubectl delete secret my-cert-tls -n my-namespace
# cert-manager va recréer automatiquement
```

### Backup des Issuers

Les ClusterIssuers contiennent les private keys ACME :

```bash
kubectl get secret -n cert-manager letsencrypt-prod-key -o yaml > acme-key-backup.yaml
```

### Migration vers nouvel Issuer

1. Créer nouveau ClusterIssuer
2. Mettre à jour les Certificates pour pointer vers nouveau Issuer
3. Supprimer ancien Issuer après validation
