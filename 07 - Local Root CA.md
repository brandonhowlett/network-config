``` bash
openssl genrsa -out smarthome-root-ca.key 4096
```
``` bash
openssl req -x509 -new -nodes \
  -key smarthome-root-ca.key \
  -sha256 \
  -days 3650 \
  -out smarthome-root-ca.crt \
  -subj "/C=US/O=Smarthome/CN=Smarthome Root CA"
```
``` bash
cp smarthome-root-ca.crt local-ca.crt
openssl genrsa -out local-key.pem 2048
```
``` bash
nano wildcard.cnf
```
``` ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
CN = *.smart.home

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.smart.home
DNS.2 = smart.home
```
``` bash
openssl req -new \
  -key local-key.pem \
  -out wildcard.csr \
  -config wildcard.cnf
```
``` bash
openssl x509 -req \
  -in wildcard.csr \
  -CA smarthome-root-ca.crt \
  -CAkey smarthome-root-ca.key \
  -CAcreateserial \
  -out local-cert.pem \
  -days 825 \
  -sha256 \
  -extensions req_ext \
  -extfile wildcard.cnf
```
Verify
``` bash
openssl x509 -in local-cert.pem -text -noout
```
``` bash
nano secret.yaml
```
``` yaml
apiVersion: isindir.github.com/v1alpha3
kind: SopsSecret
metadata:
  name: local-certs
  namespace: traefik
spec:
  suspend: false
  secretTemplates:
    - name: local-certs
      stringData:
        local-ca.crt: |
          -----BEGIN CERTIFICATE-----
          xxxxxxxxxxxxxxxxxxx
          -----END CERTIFICATE-----
        local-cert.pem: |
          -----BEGIN CERTIFICATE-----
          xxxxxxxxxxxxxxxxxxx
          -----END CERTIFICATE-----
        local-key.pem: |
          -----BEGIN PRIVATE KEY-----
          xxxxxxxxxxxxxxxxxx
          -----END PRIVATE KEY-----
```
``` bash
sops --encrypt secret.yaml > secret.enc.yaml
kubectl apply -f secret.enc.yaml
kubectl get secret local-certs -n traefik
kubectl describe secret local-certs -n traefik
kubectl get secret local-certs -n traefik -o jsonpath='{.data}' | jq keys
```
