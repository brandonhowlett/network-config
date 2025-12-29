# SOPS + Age + Kubernetes

This document describes installing and configuring **SOPS with Age
encryption** and the **isindir SOPS Secrets Operator** for Kubernetes
(k3s).

## 1. Install Age

``` bash
sudo apt update
sudo apt install -y age
age --version
```

## 2. Generate an Age key for SOPS

``` bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

Strip comments and store only private key:

``` bash
sed '/^#/d' ~/.config/sops/age/keys.txt > ~/.config/sops/age/key.txt
chmod 600 ~/.config/sops/age/key.txt
```

## 3. Install SOPS

``` bash
SOPS_LATEST=$(curl -s https://api.github.com/repos/getsops/sops/releases/latest | jq -r '.assets[] | select(.name | test("linux.amd64$")) | .browser_download_url') && \
sudo curl -L "$SOPS_LATEST" -o /usr/local/bin/sops && \
sudo chmod +x /usr/local/bin/sops
sops --version
```

## 4. Install SOPS Secrets Operator

``` bash
helm repo add sops-operator https://isindir.github.io/sops-secrets-operator/
helm repo update
```

## 5. Create Age Key Secret

``` bash
kubectl create namespace sops-operator
kubectl create secret generic age-key \
  -n sops-operator \
  --from-file=key.txt="$HOME/.config/sops/age/key.txt"
```

## 6. Operator values.yaml

``` bash
nano ~/k3s-cluster/infrastructure/sops-operator/values.yaml
```
``` yaml
secretsAsFiles:
  - name: sops-age-key
    secretName: age-key
    mountPath: /etc/sops-age-key

extraEnv:
  - name: SOPS_AGE_KEY_FILE
    value: /etc/sops-age-key/key.txt
```

## 7. Install Operator

``` bash
helm install sops-operator sops-operator/sops-secrets-operator \
  --namespace sops-operator \
  --values ~/k3s-cluster/infrastructure/sops-operator/values.yaml
```

## 8. .sops.yaml

``` bash
nano ~/k3s-cluster/.sops.yaml
```
``` yaml
creation_rules:
  - path_regex: infrastructure/.*/secrets\.enc\.yaml$
    encrypted_regex: '^(data|stringData)$'
    age:
      - age1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## 9. Example SopsSecret

``` yaml
apiVersion: isindir.github.com/v1alpha3
kind: SopsSecret
metadata:
  name: test-sopssecret
  namespace: default
spec:
  suspend: false
  secretTemplates:
    - name: some-token
      stringData:
        password: test
```

Encrypt:

``` bash
sops -e test-secret.yaml > test-secret.enc.yaml
```

Apply:

``` bash
kubectl apply -f test-secret.enc.yaml
```

## 10. Verify

``` bash
kubectl get secret some-token -n default -o jsonpath='{.data.password}' | base64 -d
```
