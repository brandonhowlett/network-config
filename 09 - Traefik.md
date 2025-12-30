# Traefik Helm Installation Guide

This guide outlines installing Traefik in a Kubernetes cluster using Helm with a separate installation of CRDs. It also covers supplemental configuration files, secrets, and verification.

---

## Prerequisites

- Kubernetes cluster ready
- Helm installed (`v3+`)
- `kubectl` configured
- Optional: sops for encrypted secrets

---

## 1. Directory Structure

Ensure your Traefik Helm chart directory has the following structure:

```
➜  traefik git:(main) ✗ tree ./
./
├── certificates.yaml       # Optional TLS certificates
├── Chart.yaml
├── secret.enc.yaml         # Optional encrypted secret
├── secret.yaml             # Optional unencrypted secret
├── templates
│   ├── dynamic-configmap.yaml
│   └── plugins-pvc.yaml
└── values.yaml             # Helm values configuration
```

- `values.yaml` contains your main configuration, including volumes, dynamic config, ports, TLS options, and plugin configuration.
- `templates/dynamic-configmap.yaml` is for Traefik dynamic configuration.
- `templates/plugins-pvc.yaml` creates a PVC for Traefik plugins.
- `certificates.yaml` can be used to create cluster or namespace certificates.
- `secret.yaml` or `secret.enc.yaml` can store sensitive information.

---

## 2. Install Traefik CRDs


Install the traefik repo:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

Traefik’s CRDs must be installed first:

```bash
helm upgrade --install traefik-crds traefik/traefik-crds -n traefik --create-namespace
```

- This will install all required CRDs such as:
  - `IngressRoute`
  - `Middleware`
  - `TraefikService`
  - `TLSOption`
  - `ServerTransport`
- Verify CRDs are present:

```bash
kubectl get crds | grep traefik.io
```

---

## 3. Install Traefik Helm Chart

Install Traefik using Helm, skipping CRDs (already installed):

```bash
helm upgrade --install traefik traefik/traefik \
  -n traefik \
  --skip-crds \
  --values values.yaml
```

- Verify Helm releases:

```bash
helm list -n traefik
```

You should see both `traefik-crds` and `traefik` listed.

---

## 4. Apply Supplemental Files

Apply dynamic configuration, secrets, and certificates:

```bash
kubectl apply -f templates
kubectl apply -f certificates.yaml -n traefik
kubectl apply -f secret.enc.yaml -n traefik
```
```bash
# kubectl apply -f templates/dynamic-configmap.yaml -n traefik
# kubectl apply -f templates/plugins-pvc.yaml -n traefik
```
---

## 5. Verify Traefik Deployment

Check pods and service status:

```bash
kubectl get pods -n traefik -o wide
kubectl get svc -n traefik
```

Expected output:

- Traefik pod(s) in `Running` state
- `LoadBalancer` or `ClusterIP` service with ports `80`, `443`, etc.

---

## 6. Verify Ingress Function

1. Create a test `IngressRoute`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami
  namespace: traefik
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`whoami.smart.home`)
      kind: Rule
      services:
        - name: whoami
          port: 80
```

2. Deploy a simple `whoami` service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: traefik
spec:
  selector:
    app: whoami
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: traefik
spec:
  replicas: 1
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: containous/whoami
          ports:
            - containerPort: 80
```

3. Verify IngressRoute resolution:

```bash
kubectl get ingressroutes.traefik.io -n traefik
kubectl get traefikservices.traefik.io -n traefik
curl -k https://whoami.smart.home
```

- You should get a response from the `whoami` container.
- TLS certificate should be applied if configured in `certificates.yaml` or via `values.yaml`.

---

## 7. Verify Certificates

Check that the secrets for TLS exist:

```bash
kubectl get secret -n traefik
kubectl describe secret <secret-name> -n traefik
```

- Certificates should match what is mounted in `values.yaml` under `tls-certs` volume.

---

## Notes

- Keep Traefik CRDs installed before upgrading Traefik itself.
- Always apply dynamic configuration and plugin PVCs after Helm installation.
- Use sops for encrypted secrets, particularly API tokens or TLS keys.
- Validate IngressRoutes and TraefikServices to ensure proper routing.

---

This completes the Traefik Helm installation with CRDs, supplemental files, and verification.

