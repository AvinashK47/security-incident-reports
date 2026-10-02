# Remediation, Hardening & Prevention Guide: Attack-1

This guide documents the exact configuration steps, script templates, and architecture rules to eliminate the `r2sh` Redis vulnerability and protect public Linux VMs from automated worm exploitation.

---

## 1. Redis Security Hardening

Never install Redis and leave it on default configuration. Always configure `/etc/redis/redis.conf` before starting the service:

```conf
# /etc/redis/redis.conf

# 1. Bind exclusively to local loopback addresses
bind 127.0.0.1 -::1

# 2. Enable protected mode
protected-mode yes

# 3. Disable or rename dangerous administrative commands
rename-command CONFIG ""
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command DEBUG ""
rename-command SHUTDOWN ""
rename-command SLAVEOF ""
rename-command REPLICAOF ""

# 4. Require strong authentication (even on loopback)
requirepass YourUltraSecureRandomSecretHere!
```

---

## 2. Linux Filesystem Immutability (`chattr +i`)

The most resilient protection against unauthorized SSH key injection:

```bash
# Ensure correct ownership and permissions
sudo chown -R ubuntu:ubuntu /home/ubuntu/.ssh
sudo chmod 700 /home/ubuntu/.ssh
sudo chmod 600 /home/ubuntu/.ssh/authorized_keys

# Lock the authorized_keys file with the immutable attribute
sudo chattr +i /home/ubuntu/.ssh/authorized_keys

# Verification:
lsattr /home/ubuntu/.ssh/authorized_keys
# Output: ----i---------e-- /home/ubuntu/.ssh/authorized_keys
```

> **Note**: To legitimately add a new key in the future:
> ```bash
> sudo chattr -i /home/ubuntu/.ssh/authorized_keys
> echo "ssh-ed25519 ..." >> /home/ubuntu/.ssh/authorized_keys
> sudo chattr +i /home/ubuntu/.ssh/authorized_keys
> ```

---

## 3. Firewall Rules (UFW)

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed

# Allow only standard ports
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Drop known malicious subnets
sudo ufw insert 1 deny from 103.230.144.0/24 to any comment 'Block botnet attacker'
sudo ufw insert 1 deny from 195.178.110.0/24 to any comment 'Block malware server'
sudo ufw insert 1 deny from 85.215.219.0/24 to any comment 'Block mining pool'

# Enable firewall
sudo ufw --force enable
sudo ufw status verbose
```

---

## 4. Virtual Memory & Swap Configuration

```bash
# Allocate 4GB swap space
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

# Enable unconditional virtual memory overcommit
sudo sysctl -w vm.overcommit_memory=1
echo "vm.overcommit_memory=1" | sudo tee -a /etc/sysctl.d/99-overcommit.conf
```

---

## 5. Docker Sandbox Hardening for Online Judges

In `apps/worker/src/docker.ts`, untrusted code MUST be restricted:

```typescript
const container = await docker.createContainer({
  Image: image,
  Cmd: cmd,
  WorkingDir: "/app",
  HostConfig: {
    Memory: 256 * 1024 * 1024,      // 256MB RAM cap
    MemorySwap: 256 * 1024 * 1024,  // Disable swap access
    CpuQuota: 100000,               // 1.0 CPU core cap
    NetworkMode: "none",            // ZERO network access
    AutoRemove: false,
    PidsLimit: 100,                 // Prevent fork bombs
    SecurityOpt: ["no-new-privileges"],
  },
  StopTimeout: 3,
});
```
