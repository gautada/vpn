# vpn

A containerised L2TP/IPsec VPN server that runs on the `gautada/debian` base image inside a Kubernetes cluster. Exposes a native system-VPN endpoint over the internet — no third-party app required. Connects macOS and iOS devices to your home network using the built-in VPN client on each platform.

[![Build](https://github.com/gautada/vpn/actions/workflows/build.yml/badge.svg)](https://github.com/gautada/vpn/actions/workflows/build.yml)
[![Image Version](https://ghcr.io/gautada/vpn)](https://github.com/gautada/vpn/pkgs/container/vpn)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## About

`gautada/vpn` runs an L2TP/IPsec server using [Libreswan](https://libreswan.org/) (IKE daemon) and [xl2tpd](https://github.com/xelerance/xl2tpd) (L2TP daemon), protected by [Fail2Ban](https://www.fail2ban.org/).

Key properties:

- **Protocol:** L2TP over IPsec (PSK) — natively supported by macOS and iOS with no extra software
- **Base image:** [`gautada/debian`](https://github.com/gautada/debian) — the shared Debian 13 foundation for the entire container fleet
- **Runtime:** Kubernetes — designed as a `Deployment` with a `LoadBalancer` service for internet exposure
- **Role:** Home network gateway — k8s pods can already reach the home network; this container extends that access to external macOS and iOS clients

---

## Prerequisites

- Kubernetes cluster (1.27+) with a `LoadBalancer` provider (or a `NodePort` on a publicly routable node)
- A public IP address assigned to the LoadBalancer service
- Required environment variables set as a Kubernetes `Secret` (see [Configuration](#configuration))
- Container image from `ghcr.io/gautada/vpn`

**`gautada/*` dependencies:**

| Image | Purpose |
|---|---|
| [`gautada/debian`](https://github.com/gautada/debian) | Base OS image |

---

## Deployment

### 1. Create the credentials Secret

```bash
kubectl create secret generic vpn-credentials \
  --from-literal=VPN_IPSEC_PSK='<your-ipsec-psk>' \
  --from-literal=VPN_USER='<your-vpn-username>' \
  --from-literal=VPN_PASSWORD='<your-vpn-password>'
```

### 2. Apply the Deployment and Service

```yaml
# vpn.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpn
spec:
  replicas: 1
  selector:
    matchLabels:
      app: vpn
  template:
    metadata:
      labels:
        app: vpn
    spec:
      containers:
        - name: vpn
          image: ghcr.io/gautada/vpn:latest
          securityContext:
            privileged: true        # required for iptables and kernel modules
          envFrom:
            - secretRef:
                name: vpn-credentials
          env:
            - name: VPN_PUBLIC_IP
              value: "<your-public-ip>"  # set explicitly; auto-detect is unreliable in k8s
          ports:
            - containerPort: 500
              protocol: UDP
              name: ike
            - containerPort: 4500
              protocol: UDP
              name: ike-nat
            - containerPort: 1701
              protocol: UDP
              name: l2tp
---
apiVersion: v1
kind: Service
metadata:
  name: vpn
spec:
  type: LoadBalancer
  selector:
    app: vpn
  ports:
    - name: ike
      port: 500
      targetPort: 500
      protocol: UDP
    - name: ike-nat
      port: 4500
      targetPort: 4500
      protocol: UDP
    - name: l2tp
      port: 1701
      targetPort: 1701
      protocol: UDP
```

```bash
kubectl apply -f vpn.yaml
```

### 3. Confirm the service is running

```bash
kubectl get pods -l app=vpn
kubectl get svc vpn
```

The `EXTERNAL-IP` column of the service is your VPN server address.

---

## Configuration

| Variable | Required | Default | Description |
|---|---|---|---|
| `VPN_IPSEC_PSK` | ✅ | — | IPsec pre-shared key |
| `VPN_USER` | ✅ | — | VPN username |
| `VPN_PASSWORD` | ✅ | — | VPN password |
| `VPN_PUBLIC_IP` | Recommended | auto-detect | Server's public IP address. Set explicitly in k8s — auto-detect is unreliable. |
| `VPN_DNS_SRV1` | ❌ | `8.8.8.8` | Primary DNS server pushed to VPN clients |
| `VPN_DNS_SRV2` | ❌ | `8.8.4.4` | Secondary DNS server pushed to VPN clients |
| `VPN_L2TP_NET` | ❌ | `192.168.42.0/24` | L2TP internal network |
| `VPN_L2TP_LOCAL` | ❌ | `192.168.42.1` | L2TP local (server-side) IP |
| `VPN_L2TP_POOL` | ❌ | `192.168.42.10-192.168.42.250` | IP range assigned to L2TP clients |
| `VPN_XAUTH_NET` | ❌ | `192.168.43.0/24` | IKEv1 XAuth internal network |
| `VPN_XAUTH_POOL` | ❌ | `192.168.43.10-192.168.43.250` | IP range assigned to XAuth clients |

---

## Usage

### macOS

1. Open **System Settings → VPN → Add VPN Configuration → L2TP over IPSec**
2. Fill in the fields:
   - **Server Address:** Your `EXTERNAL-IP` from `kubectl get svc vpn`
   - **Account Name:** Value of `VPN_USER`
3. Click **Authentication Settings…**
   - **Password:** Value of `VPN_PASSWORD`
   - **Shared Secret:** Value of `VPN_IPSEC_PSK`
4. Tick **Send all traffic over VPN connection** if you want full tunnel
5. Click **Connect**

### iOS

1. Open **Settings → General → VPN & Device Management → VPN → Add VPN Configuration**
2. Set **Type** to **L2TP**
3. Fill in the fields:
   - **Description:** Home VPN (or any label)
   - **Server:** Your `EXTERNAL-IP` from `kubectl get svc vpn`
   - **Account:** Value of `VPN_USER`
   - **Password:** Value of `VPN_PASSWORD`
   - **Secret:** Value of `VPN_IPSEC_PSK`
4. Tap **Done**, then toggle the VPN on

---

## Architecture

```
Internet
   │
   ▼
LoadBalancer Service (UDP 500, 4500, 1701)
   │
   ▼
vpn Pod  (gautada/vpn)
   ├── Libreswan (ipsec) — IKE / IPsec keying
   ├── xl2tpd — L2TP tunnel daemon
   └── Fail2Ban — brute-force protection
   │
   ▼
k8s cluster network → home network
```

**Container fleet position:**

| Layer | Image |
|---|---|
| OS base | [`gautada/debian`](https://github.com/gautada/debian) |
| VPN (this repo) | [`gautada/vpn`](https://github.com/gautada/vpn) |
| Mesh overlay (separate) | [`gautada/tailscale`](https://github.com/gautada/tailscale) |

`gautada/tailscale` handles Tailscale-based mesh access (a separate concern). This container handles the internet-facing native-VPN endpoint.

---

## License

MIT
