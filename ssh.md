# Ubuntu Server SSH Hardening: Passwordless Key Authentication

This document summarizes the steps taken to configure an Ubuntu Server system to **disable password-based SSH authentication** and **enable public key authentication**, including troubleshooting and final hardening.

---

## 1. Initial Problem

Update sshd_config
```bash
sudo nano /etc/ssh/sshd_config
```
Change the following lines
```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

---

## 2. Verifying Client-Side Keys (Windows 11)

On the Windows client:

```powershell
ssh-keygen -lf C:\Users\brand\.ssh\id_ed25519.pub
ssh-keygen -lf C:\Users\brand\.ssh\id_rsa.pub
```

Confirmed:
- Keys existed and were valid
- Client attempted multiple key types (RSA, ED25519)

---

## 3. Fixing Server-Side Ownership and Permissions

On the Ubuntu server:

```bash
chown -R brandon:brandon /home/brandon/.ssh
chmod 700 /home/brandon
chmod 700 /home/brandon/.ssh
chmod 600 /home/brandon/.ssh/authorized_keys
```

Verification:

```bash
ls -ld ~ ~/.ssh ~/.ssh/authorized_keys
```



---

## 5. Optional Key Cleanup (Recommended)

To standardize on ED25519 (preferred):

1. Add ED25519 public key to the server:
   ```bash
   nano ~/.ssh/authorized_keys
   ```

## 6. Disabling Password Authentication (LAN-Scoped)

Created a dedicated SSH configuration override:

```bash
sudo nano /etc/ssh/sshd_config.d/10-local-network.conf
```
Disable existing cloud configuration:

```bash
sudo rm /etc/ssh/sshd_config.d/50-cloud-init.conf
```

Contents:

```conf
Match Address 10.10.0.0/16
    PasswordAuthentication no
    PubkeyAuthentication yes
Match Address 10.50.0.0/16
    PasswordAuthentication no
    PubkeyAuthentication yes
```

Reload SSH:

```bash
sudo systemctl reload ssh
```

**Effect:**
- Password authentication disabled on internal networks
- Public key authentication enforced
- External/default behavior unaffected unless explicitly configured

---

## 7. Final State

- Password authentication: **Disabled (LAN)**
- Public key authentication: **Enabled**
- File permissions: **Secure and compliant**
- SSH behavior: **Deterministic and hardened**

---

