# Whoami Test Deployment Guide

This guide will walk you through deploying the `whoami` application to test the functionality of Traefik, Cert-Manager, and Cloudflare DNS in your Kubernetes cluster.

---

## 1. Namespace and Deployment (`whoami.yaml`)

Create a namespace and deploy the `whoami` application.

```bash
kubectl apply -f whoami.yaml
```

`whoami.yaml` contents:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: whoami
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: whoami
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
          image: containous/whoami:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami
spec:
  selector:
    app: whoami
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Verify the deployment and service:

```bash
kubectl get pods -n whoami
kubectl get svc -n whoami
```

---

## 2. IngressRoute Configuration (`ingress.yaml`)

Configure Traefik to route traffic to `whoami` for internal and external access.

```bash
kubectl apply -f ingress.yaml
```

`ingress.yaml` contents:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami-internal
  namespace: whoami
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`whoami.smart.home`)
      kind: Rule
      services:
        - name: whoami
          port: 80
  tls:
    secretName: smarthome-wildcard-tls

---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: whoami-external
  namespace: whoami
spec:
  entryPoints:
    - web
  routes:
    - match: Host(`whoami.brandonhowlett.com`)
      kind: Rule
      services:
        - name: whoami
          port: 80
```

Verify IngressRoutes:

```bash
kubectl get ingressroutes.traefik.containo.us -n whoami
kubectl describe ingressroute whoami-internal -n whoami
```

---

## 3. Certificates (`certificates.yaml`)

Use Cert-Manager to issue TLS certificates for internal and external access.

```bash
kubectl apply -f certificates.yaml
```

`certificates.yaml` contents:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: whoami-internal
  namespace: whoami
spec:
  secretName: whoami-internal-tls
  duration: 2160h
  renewBefore: 360h
  issuerRef:
    name: clusterissuer-lan-smarthome
    kind: ClusterIssuer
  commonName: whoami.smart.home
  dnsNames:
    - whoami.smart.home
    - whoami.whoami.svc.cluster.local

---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: whoami-external
  namespace: whoami
spec:
  secretName: whoami-external-tls
  duration: 2160h
  renewBefore: 360h
  issuerRef:
    name: clusterissuer-cloudflare
    kind: ClusterIssuer
  commonName: whoami.brandonhowlett.com
  dnsNames:
    - whoami.brandonhowlett.com
```

Verify certificates:

```bash
kubectl get certificates -n whoami
kubectl describe certificate whoami-internal -n whoami
kubectl describe certificate whoami-external -n whoami
```

---

## 4. Testing Access

1. **Internal Access:**  
   ```bash
   curl -k https://whoami.smart.home
   ```
2. **External Access:**  
   ```bash
   curl http://whoami.brandonhowlett.com
   ```

3. **TLS Verification:**  
   ```bash
   openssl s_client -connect whoami.smart.home:443 -showcerts
   ```

---

## Notes

- Ensure your DNS records (`*.smart.home` and `whoami.brandonhowlett.com`) point to your cluster or Cloudflare tunnel.
- The internal certificate uses your local root CA, while the external certificate is issued by
