Install age
```bash
sudo apt install age
age --version
```
Generate an age key for SOPS
```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
sed '/^#/d' ~/.config/sops/age/keys.txt > ~/.config/sops/age/key.txt
```
Install SOPS
https://github.com/getsops/sops/releases
```bash
curl -LO https://github.com/getsops/sops/releases/download/v3.11.0/sops-v3.11.0.linux.amd64
mv sops-v3.11.0.linux.amd64 /usr/local/bin/sops
chmod +x /usr/local/bin/sops
```

Install SOPS Operator
[https://github.com/craftypath/kubectl-sops/releases/](https://github.com/isindir/sops-secrets-operator)
```bash
helm repo add sops-operator https://isindir.github.io/sops-secrets-operator/
helm repo update
```
```bash
cat << EOF | helm upgrade --install sops-operator sops-operator/sops-secrets-operator \
  --namespace sops-operator --create-namespace --values -
secretsAsFiles:
- mountPath: /etc/sops-age-key
  name: sops-age-key
  secretName: age-key
extraEnv:
- name: SOPS_AGE_KEY_FILE
  value: /etc/sops-age-key/key.txt
EOF
```

or create values.yaml and install
```bash
nano values.yaml
```
```bash
secretsAsFiles:
- mountPath: /etc/sops-age-key
  name: sops-age-key
  secretName: age-key

extraEnv:
- name: SOPS_AGE_KEY_FILE
  value: /etc/sops-age-key/key.txt
```
```bash
helm install sops-operator sops-operator/sops-secrets-operator \
  --namespace sops-operator --create-namespace --values=infrastructure/sops-operator/values.yaml
```
Create age-key secret
```bash
 kubectl create secret generic -n sops-operator age-key --from-file=~/.config/sops/age/key.txt
```

In k3s-cluster root
```bash
nano .sops.yaml
```
```bash
creation_rules:
  - path_regex: infrastructure/.*/secrets\.enc\.yaml$
    encrypted_regex: '^(data|stringData)$'
    age: age1ns7vp2xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
Encrypt with:
```bash
sops -e secret.yaml > secret.enc.yaml
```
or
```bash
sops -e ~/k3s-cluster/infrastructure/<dir>/secrets.yaml > ~/k3s-cluster/infrastructure/<dir>/secrets.enc.yaml
```
Verify with:
```bash
grep ENC secret.enc.yaml
kubectl apply -f secret.enc.yaml
kubectl get secret some-token -n default -o jsonpath='{.data.password}' | base64 -d
```
