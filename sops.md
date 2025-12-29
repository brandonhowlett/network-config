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
  - path_regex: .*/.*secrets?\.yaml$
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
    - name: test-secret
      stringData:
        password: test
```

Encrypt:

``` bash
sops -e secret.yaml > secret.enc.yaml
```

Apply:

``` bash
kubectl apply -f secret.enc.yaml
```

## 10. Verify

``` bash
kubectl get secret test-secret -n default -o jsonpath='{.data.password}' | base64 -d
```

# Executable install script
``` bash
#!/bin/bash
set -e

echo "=== SOPS + Age + Kubernetes Installer ==="

# --- Prompt for cluster directories ---
read -rp "Enter the cluster root directory [~/k3s-cluster]: " CLUSTER_ROOT
CLUSTER_ROOT=${CLUSTER_ROOT:-~/k3s-cluster}
CLUSTER_ROOT=$(realpath "$CLUSTER_ROOT")

read -rp "Enter the sops-operator Helm chart directory relative to cluster root [./infrastructure/sops-operator]: " HELM_CHART_DIR
HELM_CHART_DIR=${HELM_CHART_DIR:-infrastructure/sops-operator}
HELM_CHART_PATH=$(realpath "$CLUSTER_ROOT/$HELM_CHART_DIR")


read -rp "Enter the path to .sops.yaml relative to cluster root [./.sops.yaml]: " SOPS_YAML_REL
SOPS_YAML_REL=${SOPS_YAML_REL:-.sops.yaml}
SOPS_YAML=$(realpath "$CLUSTER_ROOT/$SOPS_YAML_REL")

# --- 1. Install Age ---
echo "==> Installing Age"
sudo apt update
sudo apt install -y age
age --version

# --- 2. Generate Age key ---
AGE_DIR="$HOME/.config/sops/age"
mkdir -p "$AGE_DIR"
echo "==> Generating Age key"
age-keygen -o "$AGE_DIR/keys.txt"
chmod 600 "$AGE_DIR/keys.txt"

# Extract private key
sed '/^#/d' "$AGE_DIR/keys.txt" > "$AGE_DIR/key.txt"
chmod 600 "$AGE_DIR/key.txt"

# Extract public key
AGE_PUB="$AGE_DIR/pub.txt"
grep '^# public key:' "$AGE_DIR/keys.txt" | awk '{print $4}' > "$AGE_PUB"
PUBLIC_KEY=$(cat "$AGE_PUB")

# --- 3. Install SOPS ---
echo "==> Installing SOPS"
SOPS_LATEST=$(curl -s https://api.github.com/repos/getsops/sops/releases/latest | \
              jq -r '.assets[] | select(.name | test("linux.amd64$")) | .browser_download_url')
sudo curl -L "$SOPS_LATEST" -o /usr/local/bin/sops
sudo chmod +x /usr/local/bin/sops
sops --version

# --- 4. Add Helm repo ---
echo "==> Adding sops-operator Helm repo"
helm repo add sops-operator https://isindir.github.io/sops-secrets-operator/
helm repo update

# --- 5. Create Age key secret ---
echo "==> Creating Kubernetes secret for Age key"
kubectl create namespace sops-operator --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic age-key \
    -n sops-operator \
    --from-file=key.txt="$AGE_DIR/key.txt" \
    --dry-run=client -o yaml | kubectl apply -f -

# --- 6. Write operator values.yaml ---
VALUES_FILE="$HELM_CHART_PATH/values.yaml"
mkdir -p "$(dirname "$VALUES_FILE")"
cat > "$VALUES_FILE" <<EOF
secretsAsFiles:
  - name: sops-age-key
    secretName: age-key
    mountPath: /etc/sops-age-key

extraEnv:
  - name: SOPS_AGE_KEY_FILE
    value: /etc/sops-age-key/key.txt
EOF

# --- 7. Install SOPS Secrets Operator ---
echo "==> Installing SOPS Secrets Operator"
helm upgrade --install sops-operator sops-operator/sops-secrets-operator \
    --namespace sops-operator \
    --values "$VALUES_FILE"

# --- 8. Create .sops.yaml with the public key ---
mkdir -p "$(dirname "$SOPS_YAML")"
cat > "$SOPS_YAML" <<EOF
creation_rules:
  - path_regex: .*/.*secrets?\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: '$PUBLIC_KEY'
EOF

# --- 9. Generate a test secret ---
TEST_SECRET="$HELM_CHART_PATH/secret.yaml"
cat > "$TEST_SECRET" <<EOF
apiVersion: isindir.github.com/v1alpha3
kind: SopsSecret
metadata:
  name: test-sopssecret
  namespace: default
spec:
  suspend: false
  secretTemplates:
    - name: test-secret
      stringData:
        password: test
EOF

# Encrypt the test secret
ENC_SECRET="${TEST_SECRET%.yaml}.enc.yaml"
sops -e "$TEST_SECRET" > "$ENC_SECRET"

echo "==> Test secret generated at $ENC_SECRET"

# --- Apply the encrypted secret ---
kubectl apply -f "$ENC_SECRET"

# --- Wait for Secret to be created ---
echo "==> Waiting for secret 'test-secret' to be created..."
TIMEOUT=30
for i in $(seq 1 $TIMEOUT); do
    if kubectl get secret test-secret -n default >/dev/null 2>&1; then
        break
    fi
    sleep 1
done
if ! kubectl get secret test-secret -n default >/dev/null 2>&1; then
    echo "Error: Secret test-secret not created after $TIMEOUT seconds"
    exit 1
fi

# --- Verify and decode the secret ---
echo "==> Verifying secret content:"
kubectl get secret test-secret -n default -o jsonpath='{.data.password}' | base64 -d
echo

# --- Cleanup ---
echo "==> Cleaning up test resources..."
kubectl delete secret test-secret -n default
kubectl delete sopssecret test-sopssecret -n default
rm -f "$ENC_SECRET"

# Rename original secret.yaml to example-secret.yaml
mv "$TEST_SECRET" "${HELM_CHART_PATH}/example-secret.yaml"

echo "==> Test completed and cleaned up. Original secret.yaml renamed to example-secret.yaml."
echo ""
cat <<EOF

Usage Example:
1. Create secret.yaml

apiVersion: isindir.github.com/v1alpha3
kind: SopsSecret
metadata:
  name: <sopssecret-name>
  namespace: <namespace>
spec:
  suspend: false
  secretTemplates:
    - name: <secret-name>
      stringData:
        <key>: test
        
2. Encrypt:      sops -e secret.yaml > secret.enc.yaml
3. Apply:        kubectl apply -f secret.enc.yaml
4. Verify:       kubectl get secret <secret-name> -n <namespace>
5. Decode field: kubectl get secret <secret-name> -n <namespace> -o jsonpath='{.data.<key>}' | base64 -d
6. Cleanup:      kubectl delete secret <secret-name> -n <namespace>
                 kubectl delete sopssecret <sopssecret-name> -n <namespace>

EOF
```

# Uninstall script
``` bash
#!/bin/bash
set -e

echo "==> Uninstalling SOPS Secrets Operator Helm chart"
helm uninstall sops-operator -n sops-operator || true

echo "==> Removing SOPS Operator Repo"
helm repo remove sops-operator || true

echo "==> Deleting sops-operator namespace"
kubectl delete namespace sops-operator --ignore-not-found

echo "==> Deleting SopsSecret custom resources"
kubectl get sopssecrets --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name | tail -n +2 | while read ns name; do
    echo "Deleting SopsSecret $name in namespace $ns"
    kubectl delete sopssecret "$name" -n "$ns" --ignore-not-found
done

echo "==> Deleting any SOPS-related secrets"
kubectl get secrets --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name | grep -i 'age\|sops' | while read ns name; do
    echo "Deleting secret $name in namespace $ns"
    kubectl delete secret "$name" -n "$ns" --ignore-not-found
done

echo "==> Removing SOPS binary"
sudo rm -f /usr/local/bin/sops

echo "==> Removing Age binary"
sudo apt remove -y age || true
sudo rm -f /usr/local/bin/age

echo "==> Cleaning up local configuration"
rm -rf ~/.config/sops
rm -rf ~/.config/age
rm -f ~/k3s-cluster/.sops.yaml
rm -rf ~/k3s-cluster/infrastructure/sops-operator

echo "==> Removing encrypted SOPS files"
find ~/k3s-cluster/infrastructure -type f -name '*.enc.yaml' -exec rm -f {} \;

echo "==> Cleanup complete"
```
