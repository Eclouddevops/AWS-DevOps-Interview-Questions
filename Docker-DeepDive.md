# Docker — Deep-Dive Interview Q&A

## Table of Contents
1. [Docker Architecture & Internals](#docker-architecture--internals)
2. [Dockerfile Best Practices](#dockerfile-best-practices)
3. [Networking](#networking)
4. [Storage & Volumes](#storage--volumes)
5. [Security](#security)
6. [Docker Compose & Orchestration](#docker-compose--orchestration)
7. [Troubleshooting & Tricky Scenarios](#troubleshooting--tricky-scenarios)

---

## Docker Architecture & Internals


**Q1: Explain Docker architecture. What are the key components and how do they interact?**

**A:**

```
Docker Client (CLI/API)
    ↓ REST API
Docker Daemon (dockerd)
    ↓ gRPC
containerd (container runtime)
    ↓
runc (OCI runtime - creates containers)
    ↓
Linux Kernel (namespaces + cgroups + UnionFS)
```

**Components:**
| Component | Role |
|-----------|------|
| Docker CLI | User interface, sends commands to daemon |
| dockerd | API server, image management, networking, volumes |
| containerd | Container lifecycle management (start/stop/delete) |
| runc | Low-level OCI runtime, creates namespaces/cgroups |
| containerd-shim | Keeps container running after runc exits (daemonless) |

**Tricky**: When you run `docker run`, runc creates the container and EXITS. The containerd-shim takes over as parent process. This means you can restart dockerd without killing running containers!

---

**Q2: What are Linux namespaces and cgroups? How does Docker use them?**

**A:**

**Namespaces** (isolation — what a container CAN SEE):
| Namespace | Isolates |
|-----------|----------|
| PID | Process IDs (container sees PID 1 as its init) |
| NET | Network stack (interfaces, routing, ports) |
| MNT | Filesystem mounts |
| UTS | Hostname and domain name |
| IPC | Inter-process communication (shared memory, semaphores) |
| USER | User/Group IDs (UID mapping) |
| CGROUP | Cgroup root directory visibility |

**Cgroups** (limitation — what a container CAN USE):
| Resource | Control |
|----------|---------|
| CPU | cpu.shares, cpu.cfs_quota_us, cpuset |
| Memory | memory.limit_in_bytes, memory.oom_control |
| I/O | blkio.throttle.read_bps_device |
| PIDs | pids.max (prevent fork bombs) |
| Network | net_cls (traffic classification) |

**Tricky**: A container is NOT a VM. It's just a process with isolated namespaces and resource limits. `docker exec` creates a new process in the SAME namespaces. If you kill PID 1 inside a container, the container dies (just like killing init on a machine).

---


**Q3: Explain Docker image layers. How does the Union Filesystem work?**

**A:**

```
Image Layers (Read-Only):
┌─────────────────────────┐
│ Layer 4: COPY app.js    │  (only the file)
├─────────────────────────┤
│ Layer 3: RUN npm install│  (node_modules)
├─────────────────────────┤
│ Layer 2: WORKDIR /app   │  (directory creation)
├─────────────────────────┤
│ Layer 1: FROM node:18   │  (base OS + node)
└─────────────────────────┘

Container Layer (Read-Write):
┌─────────────────────────┐
│ Thin writable layer     │  (runtime changes)
└─────────────────────────┘
```

**How UnionFS (overlay2) works:**
- Each Dockerfile instruction creates a new layer
- Layers are stacked using overlay2 (or aufs, devicemapper)
- Reading: Check top layer first, then go down (copy-on-write for modifications)
- Writing: Copy file to writable layer, modify there (original layer unchanged)
- Deleting: "Whiteout" file marks deletion in upper layer

**Why layers matter:**
- Shared between images (node:18 layer reused by all node apps)
- Cached during builds (unchanged layers = instant rebuild)
- Smaller push/pull (only transfer changed layers)

**Tricky**: If you modify a 100MB file in a new layer, the entire 100MB is duplicated. Even deleting a file in a later layer doesn't reduce image size — the original layer still contains it! Use multi-stage builds to solve this.

---

**Q4: What is the difference between `CMD` and `ENTRYPOINT`? How do they interact?**

**A:**

| Feature | ENTRYPOINT | CMD |
|---------|-----------|-----|
| Purpose | Defines the executable | Provides default arguments |
| Override at runtime | `--entrypoint` flag | Append args to `docker run` |
| Forms | exec `["cmd"]` or shell `cmd` | exec `["cmd"]` or shell `cmd` |

**Interaction matrix:**

| ENTRYPOINT | CMD | docker run | Actual Command |
|-----------|-----|------------|----------------|
| `["/app"]` | `["--help"]` | (nothing) | `/app --help` |
| `["/app"]` | `["--help"]` | `--debug` | `/app --debug` |
| (none) | `["python","app.py"]` | (nothing) | `python app.py` |
| (none) | `["python","app.py"]` | `bash` | `bash` |
| `["/app"]` | (none) | (nothing) | `/app` |

**Best practices:**
```dockerfile
# Prefer exec form (no shell wrapping, signals pass correctly)
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]

# Users can override: docker run myimage --port 9090
# Result: python app.py --port 9090
```

**Tricky**: Shell form `ENTRYPOINT python app.py` wraps in `/bin/sh -c "python app.py"`. PID 1 is `sh`, not `python`. SIGTERM goes to `sh` (which doesn't forward it) → app never gracefully shuts down → gets SIGKILL after timeout!

---


**Q5: Explain multi-stage builds. Why are they important?**

**A:**

```dockerfile
# Stage 1: Build (large image with all build tools)
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server

# Stage 2: Runtime (minimal image)
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
COPY --from=builder /app/server /usr/local/bin/server
EXPOSE 8080
USER 1000:1000
ENTRYPOINT ["server"]
```

**Result:**
- Build stage: ~1.2GB (Go toolchain, source code, dependencies)
- Final image: ~15MB (only binary + ca-certificates)

**Benefits:**
1. Smaller images = faster deployments, less attack surface
2. No build tools in production image (compiler, package managers)
3. Secrets used during build don't leak to final image
4. Parallel builds: `docker build --target=test` runs only test stage

**Advanced patterns:**
```dockerfile
# Shared base for testing and production
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dependencies
RUN npm ci --production

FROM base AS test
RUN npm ci
COPY . .
RUN npm test

FROM base AS production
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .
USER node
CMD ["node", "server.js"]
```

---

## Dockerfile Best Practices

**Q6: Optimize this Dockerfile. What's wrong with it?**

```dockerfile
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y python3 python3-pip
COPY . /app
RUN pip3 install -r /app/requirements.txt
EXPOSE 5000
CMD python3 /app/app.py
```

**A:** Problems and fixes:

```dockerfile
# PROBLEM 1: "latest" tag is non-deterministic
# PROBLEM 2: apt-get update in separate layer (can be cached stale)
# PROBLEM 3: Installing unnecessary packages
# PROBLEM 4: COPY . too early (busts cache on any file change)
# PROBLEM 5: Running as root
# PROBLEM 6: Shell form CMD (PID 1 signal issue)
# PROBLEM 7: No .dockerignore (copies node_modules, .git, etc.)
# PROBLEM 8: No WORKDIR set

# FIXED:
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .
RUN useradd -r -u 1000 appuser
USER appuser
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:5000/health || exit 1
ENTRYPOINT ["python3", "app.py"]
```

**Key optimizations:**
1. Pinned version, slim base image
2. Multi-stage build
3. `requirements.txt` copied BEFORE source (layer caching)
4. Non-root user
5. Exec form for proper signal handling
6. HEALTHCHECK for orchestrator integration
7. `--no-cache-dir` reduces image size

---

**Q7: What is the Docker build cache? How does cache invalidation work?**

**A:**

**Cache rules:**
1. Each instruction checks if a cached layer exists
2. Starting from the FIRST changed layer, ALL subsequent layers are rebuilt
3. For `COPY`/`ADD`: Cache checks file content checksums (not timestamps)
4. For `RUN`: Cache checks the command string only (not external state!)

**Dangerous cache scenarios:**
```dockerfile
# BAD: apt-get update cached from weeks ago, install gets old packages
RUN apt-get update        # Layer cached from 2 weeks ago
RUN apt-get install nginx # Uses stale package list!

# GOOD: Combined = both re-run together
RUN apt-get update && apt-get install -y nginx && rm -rf /var/lib/apt/lists/*
```

**Cache-busting techniques:**
```bash
# Force no cache
docker build --no-cache .

# Bust specific layer with ARG
ARG CACHEBUST=1
RUN git clone https://github.com/repo.git  # Rebuilds when CACHEBUST changes

# BuildKit cache mounts (keep cache across builds)
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
```

**Tricky**: `RUN curl https://example.com/file` will use cache even if the remote file changed! Docker only checks the command STRING, not the result. Use `ADD` with URL (checks Content-Length/Last-Modified) or `--no-cache`.

---


## Networking

**Q8: Explain Docker networking modes. What are the differences?**

**A:**

| Mode | Description | Use Case |
|------|-------------|----------|
| bridge (default) | Private network, NAT to host | Standard containers |
| host | Shares host network stack | Performance-critical (no NAT overhead) |
| none | No networking | Security-isolated batch jobs |
| overlay | Multi-host networking (Swarm) | Swarm/multi-node clusters |
| macvlan | Container gets MAC on physical network | Legacy apps needing L2 access |

**Bridge networking packet flow:**
```
Container (172.17.0.2) → veth pair → docker0 bridge → iptables NAT → eth0 (host) → Internet
```

**Tricky**: With `--network=host`, container uses host's port space directly. Port conflicts are possible, no isolation, but zero networking overhead. Container can see ALL host network traffic.

---

**Q9: How does Docker DNS resolution work between containers?**

**A:**

- **Default bridge**: No automatic DNS. Must use `--link` (deprecated) or container IP
- **User-defined bridge**: Built-in DNS server (127.0.0.11). Containers resolved by name
- **Docker Compose**: Automatic DNS using service names

```bash
# Default bridge - no DNS
docker run --name app1 nginx
docker run --name app2 busybox ping app1  # FAILS

# User-defined bridge - DNS works
docker network create mynet
docker run --name app1 --network mynet nginx
docker run --name app2 --network mynet busybox ping app1  # WORKS
```

**Tricky**: Containers on the default bridge CAN communicate by IP but NOT by name. Always use user-defined networks in production.

---

## Storage & Volumes

**Q10: Explain the difference between volumes, bind mounts, and tmpfs.**

**A:**

| Type | Managed by | Location | Persistence | Performance |
|------|-----------|----------|-------------|-------------|
| Volume | Docker | /var/lib/docker/volumes/ | Survives container removal | Best |
| Bind mount | User | Anywhere on host | Host filesystem | Good |
| tmpfs | Kernel | RAM only | Lost on container stop | Fastest |

```bash
# Volume (recommended for data)
docker run -v mydata:/app/data nginx

# Bind mount (for development - live code reload)
docker run -v $(pwd)/src:/app/src nginx

# tmpfs (for secrets/temp data)
docker run --tmpfs /tmp:rw,noexec,nosuid,size=100m nginx
```

**Tricky**: Bind mounts with SELinux need `:z` (shared) or `:Z` (private) suffix. Without it, container gets "permission denied" even if file permissions look correct.

---

## Security

**Q11: How do you secure Docker containers in production?**

**A:**

1. **Run as non-root**: `USER 1000` in Dockerfile
2. **Read-only filesystem**: `--read-only` flag
3. **Drop all capabilities**: `--cap-drop=ALL --cap-add=NET_BIND_SERVICE`
4. **No new privileges**: `--security-opt=no-new-privileges`
5. **Resource limits**: `--memory=512m --cpus=1`
6. **Image scanning**: Trivy, Snyk, Docker Scout
7. **Content trust**: `DOCKER_CONTENT_TRUST=1`
8. **Minimal base images**: distroless, alpine, scratch
9. **No secrets in images**: Use build secrets or runtime injection
10. **PID limits**: `--pids-limit=100` (prevent fork bombs)

```bash
# Hardened container run
docker run \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges \
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  --user=1000:1000 \
  myapp:latest
```

---

**Q12: What is container escape? How do you prevent it?**

**A:**

**Container escape** = Breaking out of container isolation to access the host.

**Common attack vectors:**
1. **Privileged mode** (`--privileged`): Full host access, mount host filesystem
2. **Docker socket mount** (`-v /var/run/docker.sock:/var/run/docker.sock`): Can create new privileged containers
3. **Writable hostPath**: Write to host's /etc/cron.d for code execution
4. **Kernel exploits**: Container shares kernel with host
5. **CAP_SYS_ADMIN**: Allows mount namespace escape

**Prevention:**
- Never use `--privileged` in production
- Never mount Docker socket (use Docker-in-Docker with rootless if needed)
- Use seccomp profiles to restrict syscalls
- Use AppArmor/SELinux profiles
- Keep kernel updated (patches escape CVEs)
- Use gVisor/Kata containers for untrusted workloads
- Enable user namespaces (UID remapping)

---

## Docker Compose & Orchestration

**Q13: Explain Docker Compose v2 features. How do you manage multi-environment deployments?**

**A:**

```yaml
# docker-compose.yml (base)
services:
  web:
    build: .
    ports:
      - "8080:80"
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 10s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  db:
    image: postgres:15
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s

volumes:
  pgdata:
```

**Multi-environment with override files:**
```bash
# docker-compose.override.yml (auto-loaded for dev)
# docker-compose.prod.yml (explicit for production)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

**Tricky**: `depends_on` with `condition: service_healthy` waits for healthcheck to pass. Without condition, it only waits for container START (not readiness). This was the #1 source of startup race conditions.

---

## Troubleshooting & Tricky Scenarios

**Q14: Docker build is extremely slow. How do you diagnose and fix?**

**A:**

**Diagnosis:**
```bash
# Check which layers are slow
DOCKER_BUILDKIT=1 docker build --progress=plain .

# Check build context size
docker build . 2>&1 | head -1
# "Sending build context to Docker daemon 500MB" ← TOO BIG
```

**Fixes:**
1. **Add .dockerignore**: Exclude `.git`, `node_modules`, `dist`, `*.log`
2. **Order layers by change frequency**: Static deps first, source code last
3. **Use BuildKit cache mounts**:
```dockerfile
RUN --mount=type=cache,target=/root/.npm npm install
```
4. **Parallel builds**: BuildKit builds independent stages in parallel
5. **Use `COPY --link`**: Doesn't invalidate cache when previous layers change (BuildKit)
6. **Remote cache**: `docker buildx build --cache-from=type=registry,ref=myrepo/cache`

---

**Q15: Container runs fine locally but fails in production. OOMKilled with 512MB limit but app only uses 200MB. Why?**

**A:**

Common causes of unexpected OOM:
1. **Memory accounting includes**: RSS + cache + swap + kernel memory
2. **Child processes**: Forked workers not visible in main process memory
3. **Memory fragmentation**: Allocator overhead
4. **Shared libraries loaded**: glibc, openssl loaded but not in app metrics
5. **JVM/Go runtime**: Garbage collector needs headroom (2x working set typical)

**Debugging:**
```bash
# Check actual memory usage from cgroup
docker stats my-container
# Shows MEM USAGE / LIMIT

# Detailed breakdown
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.stat

# Check if OOM killed
docker inspect my-container | jq '.[0].State.OOMKilled'
# dmesg on host shows which process was killed
```

**Fix for JVM:**
```dockerfile
# JVM respects container memory limits since Java 10+
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0"
```

---

**Q16: Explain `docker system prune` and disk space management.**

**A:**

```bash
# Remove stopped containers, unused networks, dangling images, build cache
docker system prune

# Remove ALL unused images (not just dangling)
docker system prune -a

# Remove volumes too (DANGEROUS - data loss!)
docker system prune -a --volumes

# Check disk usage breakdown
docker system df
# TYPE          TOTAL   ACTIVE  SIZE    RECLAIMABLE
# Images        50      5       15GB    12GB (80%)
# Containers    10      3       500MB   200MB
# Volumes       20      5       8GB     6GB
# Build Cache   -       -       5GB     5GB

# Selective cleanup
docker image prune -a --filter "until=168h"  # Images older than 7 days
docker container prune --filter "until=24h"   # Stopped containers >24h old
docker volume prune --filter "label!=keep"    # Volumes without "keep" label
```

**Tricky**: `docker system prune -a` removes ALL images not used by a running container. If you stop containers for maintenance and then prune, you'll delete images you need! Always verify with `docker system df` first.

---

**Q17: How do Docker health checks work? What happens when a container is unhealthy?**

**A:**

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

**States:** `starting` → `healthy` → `unhealthy`

**What happens when unhealthy:**
- **Standalone Docker**: NOTHING! Docker doesn't restart unhealthy containers by default
- **Docker Swarm**: Reschedules/restarts the task
- **Docker Compose**: `depends_on: condition: service_healthy` prevents dependent startup
- **Kubernetes**: Ignores Docker HEALTHCHECK (uses its own probes)

**Tricky**: You MUST add `--restart=always` or `--restart=on-failure` for Docker to act on unhealthy status. Or use `autoheal` container that monitors and restarts unhealthy containers.

---

**Q18: Explain the difference between `COPY` and `ADD` in Dockerfile.**

**A:**

| Feature | COPY | ADD |
|---------|------|-----|
| Local files | Yes | Yes |
| URL download | No | Yes (but discouraged) |
| Auto-extract tar | No | Yes (.tar, .tar.gz, .tar.bz2) |
| Predictability | High | Lower (magic behavior) |
| Best practice | Preferred | Only for tar extraction |

```dockerfile
# COPY - explicit, predictable
COPY app.tar.gz /app/  # Copies the archive AS-IS

# ADD - auto-extracts
ADD app.tar.gz /app/   # Extracts contents into /app/

# ADD with URL (discouraged - no cache control, adds layer)
ADD https://example.com/file /app/file

# Better alternative for URLs:
RUN curl -o /app/file https://example.com/file
```

**Tricky**: `ADD` with a URL doesn't extract archives! Auto-extraction only works for LOCAL tar files. This inconsistency is why `COPY` is preferred for everything except local tar extraction.

---

**Q19: How do you pass secrets during Docker build without leaking them in the image?**

**A:**

```dockerfile
# BAD - secret baked into layer (visible with docker history)
COPY .npmrc /root/.npmrc
RUN npm install
RUN rm /root/.npmrc  # Still in previous layer!

# GOOD - BuildKit secret mounts (never stored in layer)
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm install

# Build command:
docker build --secret id=npmrc,src=.npmrc .
```

**Other secure patterns:**
```dockerfile
# SSH agent forwarding for private git repos
RUN --mount=type=ssh git clone git@github.com:private/repo.git

# Build with: docker build --ssh default .

# Multi-stage (secret only in build stage, not final image)
FROM node AS build
ARG NPM_TOKEN
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc
RUN npm install
RUN rm .npmrc

FROM node:slim
COPY --from=build /app/node_modules ./node_modules
# NPM_TOKEN not present in final image
```

**Tricky**: `ARG` values are visible in `docker history`! Even in multi-stage builds, the build stage layers are cached locally. Use `--mount=type=secret` for true secret isolation.

---

**Q20: What is the PID 1 zombie process problem in Docker? How do you solve it?**

**A:**

**Problem**: In Linux, PID 1 (init) has special responsibilities:
- Reap zombie child processes (wait for orphaned children)
- Forward signals to child processes

If your app runs as PID 1 and spawns children that die, zombies accumulate (can't be killed, consume PID table entries).

**Solutions:**

1. **Use `--init` flag** (Docker's built-in tini):
```bash
docker run --init myapp
```

2. **Use tini in Dockerfile:**
```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["python", "app.py"]
```

3. **Use dumb-init:**
```dockerfile
RUN apt-get install -y dumb-init
ENTRYPOINT ["dumb-init", "--"]
CMD ["python", "app.py"]
```

4. **Handle signals in your app** (if no child processes):
```python
import signal
signal.signal(signal.SIGTERM, lambda *args: sys.exit(0))
```

**Tricky**: Shell form `CMD python app.py` → PID 1 is `/bin/sh`, python is PID 2. SIGTERM goes to `sh` which doesn't forward it. Python never gets shutdown signal → hard killed after timeout. Always use exec form OR tini.
