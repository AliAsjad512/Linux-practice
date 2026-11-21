# Lab Notes: Harden a Container Step by Step

> **Goal:** Start from an insecure container, find the risks, then add one security control at a time. Check each control with `docker inspect` and test that the restriction actually works.
> **Time:** ~35 min | **Images used:** `nginx:latest`, `nginx:alpine`

> 💻 **Where to run these commands:** in a **bash** shell, such as the XLabs lab terminal or a Linux/WSL distro shell. They use bash syntax (`\` line breaks, `cat << 'EOF'`, `awk`, `/dev/null`) that **won't work in Windows PowerShell**. Make sure Docker Desktop shows *Engine running* first.

---

## 1. Big Picture: The 5 Hardening Controls

| # | Control | Flag / Setting | What it protects against |
|---|---------|----------------|--------------------------|
| 1 | Don't run privileged | *(don't use)* `--privileged` | Container getting near-full control of the host |
| 2 | Run as non-root | `USER` in Dockerfile or `--user UID:GID` | A break-in giving the attacker root inside the container |
| 3 | Drop capabilities | `--cap-drop ALL` (+ `--cap-add` only what's needed) | Abuse of the kernel privileges root normally gets |
| 4 | Block privilege escalation | `--security-opt no-new-privileges:true` | setuid binaries (like `su`/`sudo`) gaining privileges |
| 5 | Read-only filesystem | `--read-only` + `--tmpfs <dir>` | Attackers modifying binaries/config or dropping malware |
| + | Resource limits | `--memory`, `--memory-swap`, `--cpus`, `--pids-limit` | DoS: memory exhaustion, CPU hogging, fork bombs |

**Defense in depth:** each layer covers what the others miss. Use them together.

### Key terms
- **Capabilities:** Linux splits "root power" into ~40 separate privileges (e.g. `CHOWN`, `NET_RAW`). A process can hold some without holding all.
- **Docker's default capability set** (what a normal container gets):
  `CHOWN, DAC_OVERRIDE, FOWNER, FSETID, KILL, SETGID, SETUID, SETPCAP, SETFCAP, NET_BIND_SERVICE, NET_RAW, SYS_CHROOT, MKNOD, AUDIT_WRITE`
- **`--privileged`:** gives **all** capabilities, access to **all host devices** (`/dev/*`), and turns off seccomp/AppArmor confinement. This is close to root on the host.
- **tmpfs:** an in-memory filesystem. It's writable, gets wiped when the container stops, and is size-limited with `size=`.

---

## 2. Step 1: Audit an Insecure Container

```bash
docker run -d --name insecure-app --privileged nginx:latest
```
- `-d` runs it in the background (detached). `--privileged` is the bad practice being demonstrated.

```bash
docker inspect insecure-app --format '{{.HostConfig.Privileged}}'   # → true
docker exec insecure-app whoami                                       # → root
docker exec insecure-app touch /etc/bad-file && echo "INSECURE: wrote to /etc"
docker rm -f insecure-app                                             # -f = stop + remove
```

| Finding | Why it's bad |
|---------|--------------|
| `Privileged: true` | All capabilities and host devices; container isolation is mostly gone |
| User = `root` | Any exploit in the app runs as root |
| Can write `/etc` | Attacker can change config or plant files |

> 📝 **Note:** writing to `/etc` would work even **without** `--privileged`, because root plus a writable filesystem is enough. `--privileged` is a separate, bigger problem. To see the difference, try:
> ```bash
> docker exec insecure-app grep CapEff /proc/1/status   # many f's = all capabilities
> docker exec insecure-app ls /dev                      # lots of host devices visible
> ```

---

## 3. Step 2: Run as a Non-Root User

### The lab's Dockerfile, annotated
```dockerfile
FROM nginx:alpine
RUN addgroup -S webgroup && adduser -S webuser -G webgroup   # -S = system user/group (Alpine syntax)
RUN chown -R webuser:webgroup /var/cache/nginx /var/run \     # give the user dirs nginx writes to
    && chmod -R 755 /var/cache/nginx
USER webuser                                                 # everything after this runs as webuser
EXPOSE 8080                                                  # documentation only; doesn't open a port
```

### ⚠️ Problems with the lab version
1. **`nginx.conf` is created but never copied into the image.** There's no `COPY` line, so nginx still uses its default config on port 80.
2. **On Alpine, `/var/run` is a symlink to `/run`.** `chown -R` on a symlink may only change the link itself, not the real directory. nginx then can't write its PID file and fails at startup.
3. **The `whoami` / `id` checks don't prove nginx works.** `docker run secure-nginx whoami` *replaces* the nginx command, so the image passes even if nginx itself would crash.

### ✅ Corrected Dockerfile
Create `nginx.conf` first (the lab's version listening on 8080 is fine), then:
```dockerfile
FROM nginx:alpine
RUN addgroup -S webgroup && adduser -S webuser -G webgroup
COPY nginx.conf /etc/nginx/conf.d/default.conf
RUN chown -R webuser:webgroup /var/cache/nginx /run \
    && chmod -R 755 /var/cache/nginx
USER webuser
EXPOSE 8080
```
```bash
docker build -t secure-nginx .
docker run --rm secure-nginx whoami     # → webuser
docker run --rm secure-nginx id         # → uid=100(webuser) gid=... (exact numbers may differ)

# Prove nginx itself actually runs as non-root:
docker run -d --name web-test -p 8080:8080 secure-nginx
docker ps --filter name=web-test        # STATUS should say "Up"
# open http://localhost:8080 in your browser → nginx welcome page
docker rm -f web-test
```
> 📝 **Why port 8080?** Traditionally only root can bind ports below 1024. Modern Docker relaxes this inside containers (it sets `net.ipv4.ip_unprivileged_port_start=0`), but 8080 is still the portable habit, and it's needed on Podman/Kubernetes setups that don't do this.

---

## 4. Step 3: Drop Capabilities

### Flags
- `--cap-drop ALL`: remove every capability.
- `--cap-add NET_BIND_SERVICE`: add one back (binding ports below 1024).
- `--security-opt no-new-privileges:true`: processes can never gain more privileges than they started with (blocks setuid tricks).
- `cmd1 2>/dev/null || cmd2`: hide cmd1's errors, and run cmd2 if cmd1 fails (a fallback pattern).

### ⚠️ Problems with the lab version
1. **Wrong image name.** The image was built as `secure-nginx`, but the command uses `security-lab/secure-nginx`, which doesn't exist. The fallback **always** runs.
2. **The fallback (`nginx:alpine` + `--cap-drop ALL`) will probably crash.** Stock nginx starts as root and needs `CHOWN`, `SETUID`, and `SETGID` to set up and switch to its worker user. With none of them, it exits.
3. **The ping test can give a false "BLOCKED".** If the container has exited, `docker exec` fails and the `|| echo "BLOCKED..."` still prints. Also, modern Docker allows unprivileged "ping sockets" by default, so ping may **work** even without `NET_RAW`.

### ✅ Better version: non-root image + zero capabilities
```bash
docker run -d --name secure-app \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  secure-nginx                           # non-root on 8080 → needs no capabilities at all

docker ps --filter name=secure-app      # 1. confirm it's actually running ("Up")
docker inspect secure-app --format 'CapDrop: {{.HostConfig.CapDrop}}'   # → CapDrop: [ALL]

# 2. Definitive check: effective capabilities of the main process
docker exec secure-app grep CapEff /proc/1/status    # → CapEff: 0000000000000000

# 3. Reliable test: even ROOT can't chown without CAP_CHOWN
docker exec -u 0 secure-app chown nobody /tmp        # → Operation not permitted ✅

docker rm -f secure-app
```
> If you must use stock `nginx:alpine` (which starts as root), the usual minimum is:
> `--cap-drop ALL --cap-add CHOWN --cap-add SETUID --cap-add SETGID --cap-add NET_BIND_SERVICE`

---

## 5. Step 4: Read-Only Root Filesystem

```bash
docker run -d --name readonly-app \
  --read-only \
  --tmpfs /tmp:size=50m \
  --tmpfs /var/cache/nginx:size=50m \
  --tmpfs /var/run:size=10m \
  nginx:alpine
```
| Flag | Purpose |
|------|---------|
| `--read-only` | The whole root filesystem is read-only |
| `--tmpfs /tmp:size=50m` | Writable in-memory scratch space, capped at 50 MB |
| `--tmpfs /var/cache/nginx` | nginx temp/cache files |
| `--tmpfs /var/run` | nginx PID file |

```bash
docker inspect readonly-app --format '{{.HostConfig.ReadonlyRootfs}}'   # → true
docker exec readonly-app touch /etc/hacked     # → Read-only file system ✅ blocked
docker exec readonly-app touch /tmp/allowed && echo "OK: tmpfs is writable"   # ✅ works
docker rm -f readonly-app
```
- **Pattern:** read-only everywhere, with tmpfs only where the app must write.
- You may see `can not modify /etc/nginx/conf.d/default.conf (read-only file system?)` in `docker logs`. It's harmless: the nginx startup script skips an optional IPv6 tweak.

---

## 6. Step 5: Full Hardened Run (Everything Together)

| Flag | Meaning |
|------|---------|
| `--user 101:101` | Run as UID/GID 101 (the `nginx` user in `nginx:alpine`) |
| `--cap-drop ALL` | No capabilities |
| `--read-only` + `--tmpfs` | Immutable filesystem with small writable areas |
| `--security-opt no-new-privileges:true` | No privilege escalation |
| `--memory 128m` | RAM cap; the container is OOM-killed above this |
| `--memory-swap 128m` | Memory + swap total; equal to `--memory` means **no swap** |
| `--cpus 0.5` | At most half of one CPU core |
| `--pids-limit 50` | Max 50 processes, which stops fork bombs |
| `--network bridge` | Default isolated network (stated explicitly) |

### ⚠️ Problem with the lab version
Only `/tmp` is writable. nginx (as UID 101) also needs to write **`/var/cache/nginx`** and its **PID file in `/run`**, so the container will probably exit right away. The `id` line in the audit then prints **blank**, because its error is hidden by `2>/dev/null`. `docker inspect` still shows the settings, because it works on stopped containers, so the audit can *look* fine while nothing is running.

### ✅ Corrected command
```bash
docker run -d \
  --name hardened-app \
  --user 101:101 \
  --cap-drop ALL \
  --read-only \
  --tmpfs /tmp:size=50m \
  --tmpfs /var/cache/nginx:size=50m \
  --tmpfs /run:size=10m \
  --security-opt no-new-privileges:true \
  --memory 128m \
  --memory-swap 128m \
  --cpus 0.5 \
  --pids-limit 50 \
  --network bridge \
  nginx:alpine

docker ps --filter name=hardened-app    # must say "Up" before auditing
docker logs hardened-app                # if it exited, the reason is here
```

### Security audit (from the lab)
```bash
echo "=== Security Audit ==="
echo -n "User: ";         docker exec hardened-app id 2>/dev/null | head -1
echo -n "Privileged: ";   docker inspect hardened-app --format '{{.HostConfig.Privileged}}'
echo -n "ReadOnly FS: ";  docker inspect hardened-app --format '{{.HostConfig.ReadonlyRootfs}}'
echo -n "Cap Drop: ";     docker inspect hardened-app --format '{{.HostConfig.CapDrop}}'
echo -n "Memory limit: "; docker inspect hardened-app --format '{{.HostConfig.Memory}}' | awk '{printf "%d MB\n", $1/1024/1024}'
echo -n "CPU limit: ";    docker inspect hardened-app --format '{{.HostConfig.NanoCpus}}' | awk '{printf "%.1f cores\n", $1/1e9}'
echo -n "PID limit: ";    docker inspect hardened-app --format '{{.HostConfig.PidsLimit}}'
```
**Expected:**
```
User: uid=101(nginx) gid=101(nginx)
Privileged: false
ReadOnly FS: true
Cap Drop: [ALL]
Memory limit: 128 MB
CPU limit: 0.5 cores      ← NanoCpus is stored in billionths of a CPU (0.5 = 500000000)
PID limit: 50
```
```bash
docker rm -f hardened-app && rm -rf security-lab    # clean up
```
> 💡 **Shortcut:** the image `nginxinc/nginx-unprivileged:alpine` already runs as UID 101 on port 8080 with writable dirs set up. It's a good base for hardened nginx.

---

## 7. Completion Checklist

- [ ] Identified insecure config: root user, privileged mode
- [ ] Built a Dockerfile with a non-root user *(with `COPY nginx.conf`)*
- [ ] Dropped all capabilities and **verified with `CapEff: 0000000000000000`**
- [ ] Applied read-only root filesystem with tmpfs for writable dirs
- [ ] Confirmed writes to the root filesystem are blocked
- [ ] Confirmed tmpfs directories are writable
- [ ] Applied full hardening (user + cap-drop + read-only + limits + no-new-privileges)
- [ ] **Confirmed the container is still `Up`** after hardening
- [ ] Audited the hardened container with `docker inspect`

> 🧠 **Golden rule:** after every hardening step, run `docker ps` and `docker logs`. A security test that "passes" on a crashed container proves nothing.

---

## 8. Troubleshooting Notes

### Problem 1: App crashes after `--cap-drop ALL` ("operation not permitted")
**Read the logs first:** `docker logs <name>`, then match the error to a capability:

| Error in logs | Capability to add |
|---------------|-------------------|
| `bind: permission denied` (port < 1024) | `NET_BIND_SERVICE` |
| `chown ... operation not permitted` | `CHOWN` |
| `setuid` / `setgid ... not permitted` | `SETUID` / `SETGID` |
| Can't read root-owned protected files | `DAC_OVERRIDE` |
| Can't signal processes | `KILL` |

- **Best approach (allowlist):** start from `--cap-drop ALL` and add back **one capability at a time** until the app works.
- The lab's other method (start with all caps and drop one at a time until it breaks) also works, but it's slower and easier to get wrong.
- `strace` (from the lab) shows the exact failing system call, but it must be installed in the image and needs `--cap-add SYS_PTRACE` *while debugging only*.

### Problem 2: App fails after `--read-only` ("read-only file system")
Find what the app writes using **`docker diff`** on a normal (writable) run:
```bash
docker run -d --name test-app my-app && sleep 5 && docker diff test-app
```
| Letter | Meaning |
|--------|---------|
| `A` | Added file/dir |
| `C` | Changed |
| `D` | Deleted |

**Fix:** add a `--tmpfs` for each path it writes to. **Better long-term fix:** have the app log to **stdout/stderr** instead of files (Docker collects these with `docker logs`).

### Problem 3: CI/CD container needs to build images
| Option | Verdict |
|--------|---------|
| Mount `/var/run/docker.sock` | ❌ **Equivalent to root on the host.** Anyone with the socket can start a privileged container that mounts the host's disk. |
| Docker-in-Docker (`docker:dind`) | ⚠️ Still needs `--privileged`. Only run it on separate, non-production build hosts. |
| Kaniko | ⚠️ Daemonless builds, but the **original Google project was archived in June 2025**, so `gcr.io/kaniko-project/executor` gets no security fixes. Use a maintained fork (e.g. Chainguard's) if you choose Kaniko. |
| Buildah / rootless BuildKit | ✅ Builds images without root and without a Docker daemon |

---

## 9. Cheat Sheet: Inspect Formats

```bash
docker inspect <c> --format '{{.HostConfig.Privileged}}'      # privileged?
docker inspect <c> --format '{{.HostConfig.CapDrop}}'         # dropped caps
docker inspect <c> --format '{{.HostConfig.CapAdd}}'          # added caps
docker inspect <c> --format '{{.HostConfig.ReadonlyRootfs}}'  # read-only FS?
docker inspect <c> --format '{{.HostConfig.SecurityOpt}}'     # no-new-privileges, etc.
docker inspect <c> --format '{{.Config.User}}'                # configured user
docker inspect <c> --format '{{.HostConfig.Memory}}'          # bytes
docker inspect <c> --format '{{.HostConfig.NanoCpus}}'        # 1e9 = 1 CPU
docker inspect <c> --format '{{.HostConfig.PidsLimit}}'       # process limit
docker exec <c> grep Cap /proc/1/status                       # live capabilities (CapEff)
```

## 10. Gotchas Summary
1. `docker inspect` works on **stopped** containers, so always check `docker ps` too.
2. `2>/dev/null` and `|| echo "BLOCKED"` can **hide failures** and print false success messages.
3. `docker run image whoami` overrides the app's command, so it proves the user but **not** that the app runs.
4. Files created in a lab step must be `COPY`'d in the Dockerfile to end up in the image.
5. On Alpine, `/var/run` → `/run` (symlink).
6. Modern Docker lets non-root bind ports < 1024 and use `ping` inside containers. Don't rely on these as capability tests; use `CapEff` and a `chown` test instead.