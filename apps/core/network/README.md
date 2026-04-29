# Network Stack

Infrastructure réseau : LoadBalancer, Gateway API et Ingress.

## Composants

| App | Rôle | Sync Wave |
|-----|------|-----------|
| [MetalLB](metallb/) | LoadBalancer L2 - Attribution IPs externes | 1 |
| [Envoy Gateway](envoy-gateway/) | Implémentation Gateway API (contrôleur) | 3 |
| [Ingress Gateway](ingress-gateway/) | Instances Gateway pour routing HTTP/TLS | 4 |

## Architecture

```
Services (type: LoadBalancer)
    ↓
MetalLB (allocation IP pool)
    ↓
IP externe annoncée (L2 ARP)


HTTPRoutes / TLSRoutes
    ↓
Gateway (via GatewayClass)
    ↓
Envoy Gateway (contrôleur)
    ↓
Envoy Proxy (data plane)
    ↓
Services backend
```

## Dépendances

- **Aucune** : Stack réseau est autonome
- **Utilisé par** : Toutes les apps exposant des services externes

## Configuration Critique

### MetalLB (`metallb/overlays/mgmt/patches.yaml`)

**IP Address Pool** :
- `addresses: 10.0.0.0-10.0.0.250` - Plage d'IPs disponibles pour LoadBalancer
- Impact : Services LoadBalancer reçoivent IPs de cette plage

**L2 Advertisement** :
- Mode Layer 2 (ARP)
- Impact : IPs annoncées sur le réseau local uniquement

**Configuration par cluster** :
- Management : `10.0.0.0-10.0.0.250`
- À adapter selon réseau de chaque cluster

### Envoy Gateway (`envoy-gateway/base/`)

**GatewayClass** :
- `controllerName: gateway.envoyproxy.io/gatewayclass-controller`
- Implémentation officielle Envoy
- Impact : Tous les Gateways utilisent Envoy comme data plane

**Namespace** :
- Déployé dans `envoy-gateway-system`
- Impact : Les Gateways créent automatiquement des Deployments Envoy dans leur namespace

### Ingress Gateway (`ingress-gateway/`)

**Structure Components** :
- Base : namespace uniquement
- Component `internal-gateway` : Gateway + Certificate

**Gateway interne** (`components/internal-gateway/gateway.yaml`) :
- `gatewayClassName: envoy-gateway` - Utilise Envoy Gateway
- Listeners : HTTP (80) + HTTPS (443)
- TLS mode : Terminate (déchiffrement au Gateway)
- Impact : Point d'entrée unique pour tout le trafic HTTP(S) interne

**Certificate** (`components/internal-gateway/certificate.yaml`) :
- Intégration cert-manager
- Génère certificat TLS pour le Gateway
- Impact : HTTPS fonctionnel uniquement si cert-manager déployé

## Endpoints

### MetalLB
Pas de service direct - alloue IPs aux services `type: LoadBalancer`

### Envoy Gateway
- Contrôleur : `envoy-gateway.envoy-gateway-system.svc`
- Pas d'exposition externe

### Ingress Gateway
- Gateway interne : IP allouée par MetalLB
- Accessible via LoadBalancer IP
- Ports : 80 (HTTP), 443 (HTTPS)

## Intégrations

### MetalLB ← Services
Tout service avec `type: LoadBalancer` reçoit une IP du pool MetalLB.

Exemple :
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  ports:
    - port: 80
```

### Envoy Gateway ← Gateways
Gateways utilisant `gatewayClassName: envoy-gateway` sont gérés par Envoy.

Exemple :
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      port: 80
      protocol: HTTP
```

### HTTPRoute → Gateway → Service
Routes HTTP attachées aux Gateways pour router le trafic.

Exemple :
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
spec:
  parentRefs:
    - name: internal-gateway
      namespace: gateway-system
  rules:
    - matches:
        - path:
            value: /app
      backendRefs:
        - name: my-service
          port: 80
```

### Certificate ← Gateway
Certificates cert-manager référencés dans les Gateways pour TLS.

Le Gateway référence le certificat :
```yaml
tls:
  mode: Terminate
  certificateRefs:
    - name: internal-gateway-cert
```

## Usage

### Exposer une Application via Gateway API

1. **Créer un HTTPRoute** :
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: observability
spec:
  parentRefs:
    - name: internal-gateway
      namespace: gateway-system
  hostnames:
    - grafana.example.com
  rules:
    - matches:
        - path:
            value: /
      backendRefs:
        - name: grafana
          port: 80
```

2. **Accès** : `https://grafana.example.com` (via IP LoadBalancer du Gateway)

### Créer un Nouveau Gateway

1. Ajouter component dans `ingress-gateway/components/my-gateway/`
2. Créer Gateway + Certificate
3. Référencer le component dans overlay

### Vérifier MetalLB

```bash
# Vérifier IP pool
kubectl get ipaddresspool -n metallb-system

# Vérifier IPs allouées
kubectl get svc --all-namespaces -o wide | grep LoadBalancer
```

### Vérifier Envoy Gateway

```bash
# Vérifier GatewayClass
kubectl get gatewayclass

# Vérifier Gateways
kubectl get gateway -A

# Logs contrôleur
kubectl logs -n envoy-gateway-system -l control-plane=envoy-gateway
```

### Vérifier Routes

```bash
# Lister HTTPRoutes
kubectl get httproute -A

# Détails d'une route
kubectl describe httproute grafana -n observability
```

## Sécurité

### TLS
- Certificats gérés par cert-manager
- Mode Terminate au Gateway (déchiffrement)
- Backend en HTTP (pas de re-encryption interne)

### Network Policies
- À définir selon besoins
- Par défaut : tout trafic autorisé

## Opérations

### Changer le Pool d'IPs MetalLB

Modifier `metallb/overlays/{cluster}/patches.yaml` :
```yaml
spec:
  addresses:
    - 10.0.1.0-10.0.1.100
```

### Troubleshooting

**Service LoadBalancer en Pending** :
1. Vérifier MetalLB running : `kubectl get pods -n metallb-system`
2. Vérifier IP pool disponible : `kubectl get ipaddresspool -n metallb-system`
3. Vérifier logs : `kubectl logs -n metallb-system -l component=controller`

**Gateway non Ready** :
1. Vérifier GatewayClass existe : `kubectl get gatewayclass`
2. Vérifier Envoy Gateway running : `kubectl get pods -n envoy-gateway-system`
3. Vérifier listeners : `kubectl describe gateway -n gateway-system internal-gateway`

**HTTPRoute non fonctionnel** :
1. Vérifier parentRef valide : `kubectl describe httproute`
2. Vérifier service backend existe : `kubectl get svc`
3. Tester depuis pod Envoy : `kubectl exec -n gateway-system <envoy-pod> -- curl http://backend-svc`
