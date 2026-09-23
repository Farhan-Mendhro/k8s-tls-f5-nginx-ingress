# Securing Ingress with F5 NGINX Ingress Controller, cert-manager TLS, and Basic Authentication

A hands-on Kubernetes lab demonstrating TLS termination, automated certificate management, and access control on a local Minikube cluster using the **official F5 NGINX Ingress Controller** (`nginx.org`) — not the community `ingress-nginx` controller.

## Architecture

```
<img width="1411" height="736" alt="tls-ingress" src="https://github.com/user-attachments/assets/3a0ee31d-ed56-4c91-9035-10f6d7faa1fd" />

```

TLS is automated end-to-end via **cert-manager**: a `ClusterIssuer` issues a self-signed certificate, stored as a Kubernetes `Secret`, which the Ingress references directly — no manual `openssl` cert generation or renewal.

## Stack

- Minikube (local Kubernetes cluster)
- F5 NGINX Ingress Controller (`nginx.org` annotations — distinct from community `ingress-nginx`)
- cert-manager (automated TLS certificate lifecycle)
- Two backend apps (`nginx:alpine`, `httpd:alpine`) behind path-based routing

## Repository Structure

| File | Purpose |
|---|---|
| `issuer.yml` | `ClusterIssuer` — self-signed certificate authority for cert-manager |
| `app1-deploy.yml` / `app1-service.yml` | Deployment + Service for `/app1` backend |
| `app2-deploy.yml` / `app2-service.yml` | Deployment + Service for `/app2` backend |
| `secure-ingress.yml` | Ingress resource — TLS termination, Basic Auth, path routing |

## Setup

### 1. Configure local DNS resolution

```bash
minikube ip
# retrieves the Minikube cluster node's IP address
```

Add the result to `/etc/hosts`:

```
<MINIKUBE_IP> lab.devops.local
```

### 2. Create the ClusterIssuer

```bash
kubectl apply -f issuer.yml
# registers a self-signed ClusterIssuer with cert-manager
```

### 3. Create the Basic Auth secret

```bash
htpasswd -c auth admin
# generates an htpasswd credentials file for user 'admin'

kubectl create secret generic basic-auth --from-file=auth=auth --type=nginx.org/basic-auth
# creates the secret with the type required specifically by F5 NGINX Ingress Controller
```

> **Note:** F5 NGINX Ingress Controller requires `--type=nginx.org/basic-auth` on this secret. The default `Opaque` type (from a plain `kubectl create secret generic` with no `--type`) is silently ignored by the F5 controller — this is a common point of confusion since it differs from the community `ingress-nginx` controller's requirements.

### 4. Deploy backend applications

```bash
kubectl apply -f app1-deploy.yml -f app1-service.yml -f app2-deploy.yml -f app2-service.yml
```

### 5. Deploy the secure Ingress

```bash
kubectl apply -f secure-ingress.yml
# deploys F5 NGINX Ingress with TLS termination, Basic Auth, and path-based rewriting
```

The Ingress annotations tie everything together:
- `cert-manager.io/cluster-issuer` — triggers automatic certificate issuance via the ClusterIssuer
- `nginx.org/basic-auth-secret` / `nginx.org/basic-auth-realm` — enforces Basic Auth using the secret above
- `nginx.org/rewrites` — rewrites `/app1` and `/app2` to `/` for each backend

## Verification

```bash
kubectl get certificate secure-ingress-tls
# confirms cert-manager issued the certificate (READY=True)

kubectl get svc -l app.kubernetes.io/name=nginx-ingress
# identifies the HTTPS NodePort mapped to container port 443
```

**Unauthenticated requests are rejected (401):**

```bash
curl -k -v https://lab.devops.local:<HTTPS_NODEPORT>/app1
curl -k -v https://lab.devops.local:<HTTPS_NODEPORT>/app2
```

**Authenticated requests succeed (200):**

```bash
curl -k -v -u admin https://lab.devops.local:<HTTPS_NODEPORT>/app1
curl -k -v -u admin https://lab.devops.local:<HTTPS_NODEPORT>/app2
```

## What This Demonstrates

- Automated TLS certificate issuance and renewal via cert-manager, replacing manual `openssl` certificate generation
- TLS termination at the Ingress layer using a Kubernetes `Secret` of type `kubernetes.io/tls`
- Layer 7 path-based routing to multiple backend services under one host
- HTTP Basic Authentication enforced at the Ingress layer via a controller-specific secret type
- Correct annotation and secret-type usage for the **F5 NGINX Ingress Controller**, as distinct from the community `ingress-nginx` controller
