# RemnaNode Helm Chart

A highly configurable, production-ready Helm chart for deploying a Remnawave Xray node on Kubernetes.

## Prerequisites

* Kubernetes cluster (K3s, EKS, GKE, etc.)
* Helm 3.0+
* **Traefik Ingress Controller** installed (this chart uses Traefik's `IngressRoute` and `IngressRouteTCP` CRDs)

## Features

* **Multi-Protocol Support:**
  * **Reality/Trojan:** TCP Passthrough via Traefik (SNI routing)
  * **WebSocket:** Secure Path routing with automatic WS headers
  * **XHTTP:** Native HTTP routing
  * **gRPC:** Full HTTP/2 (h2c) support
  * **Shadowsocks:** Direct HostPort access (TCP + UDP)
* **Website Camouflage:** Optional lightweight web server for probe resistance.
* **Auto-Updates:** Configurable CronJob to securely update `geoip.dat`, `geosite.dat`, and `zapret.dat`.
* **Security First:** Principle of least privilege applied. The main pods do not have Kubernetes API access.

## Installation

1. Add the repository (if hosted) or clone this repo:

   ```bash
   git clone https://github.com/TheroDev-Corp/remnanode-helm.git
   cd remnanode-helm
   ```

2. Create a custom values file (e.g., `my-values.yaml`). See the configuration example below.

3. Install the chart:

   ```bash
   helm upgrade --install remnanode . \
     --namespace remnanode \
     --create-namespace \
     -f my-values.yaml
   ```

## Configuration Example (`my-values.yaml`)

You only need to define what you want to override from the default `values.yaml`.

```yaml
global:
  domain: "node1.your-domain.com"

node:
  secretKey: "YOUR_SUPER_SECRET_KEY_FROM_PANEL"
  # existingSecret: "my-vault-secret" # Use this if you manage secrets via external tools
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1024Mi"

website:
  enabled: true
  image:
    repository: th3ro/remnanode_website
    tag: latest

autoUpdate:
  enabled: true
  schedule: "30 3 * * *" # Every day at 03:30 AM

ingress:
  enabled: true
  tls:
    enabled: true
    redirect: true # Automatically redirect HTTP to HTTPS
    certResolver: "le" # Name of your Traefik Let's Encrypt resolver
    # secretName: "my-custom-tls-secret" # Use this instead if using cert-manager

inbounds:
  # VLESS TCP REALITY (Layer 4 Passthrough)
  - name: vless-vk
    port: 10001
    mode: ingress-sni
    sni:
      - "www.vk.com"

  # VLESS WebSocket (Layer 7)
  - name: vless-ws
    port: 10003
    mode: ingress-path
    path: "/api/media/" 

  # TROJAN gRPC (Layer 7 HTTP/2)
  - name: trojan-grpc
    port: 10005
    mode: ingress-grpc
    serviceName: "k8sSecurityTraffic"

  # SHADOWSOCKS (Direct Host Port)
  - name: ss2022-std
    port: 993
    mode: direct
```

## Detailed Configuration Reference

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `global.domain` | Primary domain name for the node. | `example.com` |
| **node** | **Core Node Settings** | |
| `node.image.repository` | Xray-core node image repository. | `remnawave/node` |
| `node.image.tag` | Xray-core node image tag. | `2.6.1` |
| `node.apiPort` | Port for the Xray API (internal communication). | `2222` |
| `node.secretKey` | Secret key from the Remnawave Panel. | `""` |
| `node.existingSecret` | Name of an existing secret for the secret key. | `""` |
| `node.resources` | CPU/Memory requests and limits for the node pod. | See `values.yaml` |
| **website** | **Camouflage Website (Probe Resistance)** | |
| `website.enabled` | Enable the fallback/camouflage website. | `true` |
| `website.image.repository` | Image for the camouflage website. | `th3ro/remnanode_website` |
| `website.image.tag` | Tag for the camouflage website image. | `latest` |
| `website.resources` | CPU/Memory requests and limits for the website pod. | See `values.yaml` |
| **autoUpdate** | **Update Automation (geoip/geosite/zapret)** | |
| `autoUpdate.enabled` | Enable the update CronJob. | `true` |
| `autoUpdate.schedule` | Cron schedule for the updates. | `0 4 * * *` |
| `autoUpdate.image.repository` | Kubectl image used to trigger rollouts. | `bitnami/kubectl` |
| `autoUpdate.image.tag` | Tag for the kubectl image. | `latest` |
| **ingress** | **Routing & TLS (Traefik Specific)** | |
| `ingress.enabled` | Enable Ingress/IngressRoute creation. | `true` |
| `ingress.tls.enabled` | Enable TLS for all ingresses. | `true` |
| `ingress.tls.redirect` | Enable HTTP to HTTPS redirection. | `true` |
| `ingress.tls.certResolver` | Traefik cert resolver (e.g., `le`). | `le` |
| `ingress.tls.secretName` | Existing Kubernetes TLS secret (ignores certResolver). | `""` |
| **inbounds** | **Xray Inbounds Configuration** | |
| `inbounds` | List of protocol-specific inbound configurations. | `[]` |

## Supported Inbound Modes

| Mode | Description | Use Case |
| :--- | :--- | :--- |
| `ingress-sni` | TCP Passthrough (Layer 4). Routing based on SNI. | VLESS Reality, Trojan Reality |
| `ingress-path` | HTTP/WS Routing (Layer 7). Adds WS headers. | VLESS WebSocket, VMess WS |
| `ingress-xhttp` | HTTP Routing (Layer 7). No extra headers. | Trojan XHTTP, VLESS XHTTP |
| `ingress-grpc` | HTTP/2 h2c Routing. Matches ServiceName. | VLESS gRPC, Trojan gRPC |
| `direct` | Direct HostPort mapping. Opens TCP & UDP on the node. | Shadowsocks, Fallbacks |

## Panel Configuration (Remnawave)

When configuring Inbounds in your Remnawave Panel:

1. **Listen IP:** `0.0.0.0`
2. **Port:** Must match the `port` defined in your `values.yaml` (e.g., `10001`).
3. **Security:** `none` (TLS termination is handled by Traefik). Configure TLS inside the panel's Advanced Host Settings instead.
4. **Sniffing:**
   * **Enable** for Reality/Shadowsocks.
   * **DISABLE** for gRPC/XHTTP/WS.

## Troubleshooting

Restart the node manually:

```bash
kubectl rollout restart deployment remnanode -n remnanode
```

Check node logs:

```bash
kubectl logs -f -l app.kubernetes.io/name=remnanode -c remnanode -n remnanode
```
