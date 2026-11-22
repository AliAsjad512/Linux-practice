# Docker Day-2: Multi-Container Apps, Networking, Volumes & Docker Compose

> 📝 The original images weren't included in the text, so the diagrams here are plain-text versions redrawn from the explanation around them.

---

## Part 2 — Why Multiple Containers?

Take **Project 1**:

- React
- Spring Boot
- MySQL

Technically, someone could build **one huge image** containing:

- Node.js
- Nginx
- Java
- MySQL
- Application code

and run everything inside one container.

**But that is a bad container architecture.** Instead:

| Container | Responsibility |
|-----------|----------------|
| Container 1 | Frontend |
| Container 2 | Backend |
| Container 3 | Database |

This immediately gives us some advantages. The frontend can be:

- built independently
- restarted independently
- updated independently
- scaled independently
- secured independently

The same goes for the backend and the database.

---

## Part 3 — One Main Responsibility Per Container

It is commonly summarized as:

> **One main responsibility per container.**

It does **not** literally mean *"only one Linux process is ever allowed inside a container."* For example, Nginx itself creates worker processes internally. That's fine.

The point is:

```
Frontend responsibility  → Frontend container
Backend responsibility   → Backend container
Database responsibility  → Database container
```

We do **not** want:

```
One container
├── nginx
├── java
├── mysql
├── redis
├── python
└── cron
```

because then lifecycle management becomes complicated.

---

## Part 4 — The First Big Problem: How Do Containers Communicate?

Suppose:

- **Container A** = Backend
- **Container B** = MySQL

The backend has to connect to MySQL. A beginner often writes this inside the backend container:

```
jdbc:mysql://localhost:3306/projectops
```

**Question:** What does `localhost` mean here?
**Answer:** `localhost` = **the current container.**

```
Inside backend container:

localhost:3306
     ↓
backend container itself
```

It does **not** mean the MySQL container. This is one of the most common Docker networking mistakes.

---

## Part 5 — Understanding localhost Properly

Each container has its **own network namespace**. Therefore:

```
Backend localhost  !=  MySQL localhost
```

```
┌──── Backend container ────┐      ┌───── MySQL container ─────┐
│  own network namespace    │      │  own network namespace    │
│  localhost → backend      │      │  localhost → mysql        │
└───────────────────────────┘      └───────────────────────────┘
```

The backend needs some other way to reach MySQL. That's where **Docker networking** comes in.

---

## Part 6 — Docker Network

```bash
docker network ls
```

Typical output:

```
NETWORK ID     NAME      DRIVER
xxxxx          bridge    bridge
xxxxx          host      host
xxxxx          none      null
```

| Driver | Meaning | Notes |
|--------|---------|-------|
| **bridge** | Normal Docker networking. Containers get isolated IP addresses and communicate over a software bridge. | What our applications use |
| **host** | Container uses the host network directly, e.g. `docker run --network host ...` | Less isolation; not our normal application architecture |
| **none** | No network connectivity (`Container ✗ Network`) | Useful for highly isolated workloads |

For our applications, we primarily care about **bridge** networking.

---

## Part 7 — Default Bridge vs User-Defined Bridge

**This is very important.** Instead of simply running:

```bash
docker run backend
docker run mysql
```

create a network:

```bash
docker network create projectops-network
docker network ls
docker network inspect projectops-network
```

Then run the containers on it:

```bash
docker run \
  --name mysql \
  --network projectops-network \
  ...
```

```bash
docker run \
  --name backend \
  --network projectops-network \
  ...
```

Now Docker provides **internal DNS / service-name resolution**. The backend can use `mysql` instead of trying to know MySQL's IP:

```
Backend
   │
   │  jdbc:mysql://mysql:3306/projectops
   ▼
Docker DNS
   │
   ▼
MySQL Container
```

> 🔑 **Key point:** on the **default** `bridge` network, containers can't find each other by name. On a **user-defined** network, Docker's built-in DNS resolves container names.

---

## Part 8 — Never Depend on Container IP Addresses

Someone may ask: *why don't we inspect the MySQL container and use its IP?*

```bash
docker inspect mysql
```

Maybe Docker gives `172.18.0.2`. Then the backend could technically use:

```
jdbc:mysql://172.18.0.2:3306/projectops
```

**But this is bad**, because container IPs are **dynamic**:

```
Today:              mysql → 172.18.0.2
After recreating:   mysql → 172.18.0.5
```

Meanwhile, `mysql` stays the same **logical service name**.

| ❌ Bad | ✅ Good |
|--------|---------|
| `172.18.0.2` | `mysql` |

The IP a Docker container gets is normally an **internal/private virtual network IP**. It is **not** the EC2/public IP, and it is not usually the host's private IP.

Think of it like this:

```
Host / EC2 instance
├── Public IP:   e.g. 54.x.x.x      ← reachable from the internet
├── Private IP:  e.g. 10.0.1.25     ← reachable inside the VPC
└── Docker bridge network: 172.17.0.0/16   ← exists only inside this host
    ├── Gateway:     172.17.0.1
    ├── Container 1: 172.17.0.2
    ├── Container 2: 172.17.0.3
    └── Container 3: 172.17.0.4
```

### Where does `172.17.0.2` come from?

When Docker is installed, it normally creates a virtual bridge network:

```bash
docker network ls
```

```
NETWORK ID     NAME      DRIVER
xxxx           bridge    bridge
```

Inspect it:

```bash
docker network inspect bridge
```

You may see something like:

```
Subnet:  172.17.0.0/16
Gateway: 172.17.0.1
```

So Docker has effectively created its own little private network, `172.17.0.0/16`. When containers start:

```
Container 1 → 172.17.0.2
Container 2 → 172.17.0.3
Container 3 → 172.17.0.4
```

Docker's **IPAM (IP Address Management)** picks an available address from that network.

> ⚠️ The subnet isn't always `172.17.x.x`. Docker chooses a private subnet that doesn't conflict with existing networks.

---

## Part 9 — Internal Port vs Host Port

People often confuse these two.

Suppose the MySQL container listens on port `3306` and the backend is on the same Docker network. The backend can use `mysql:3306` **even if you do not publish** `-p 3306:3306`.

**Why?** Because `-p 3306:3306` is for **Host → Container** traffic. It is **not** required for **Container → Container** traffic.

```
HOST → CONTAINER  (needs -p)
Browser / host ──▶ host:3306 ──(-p 3306:3306)──▶ mysql container :3306

CONTAINER → CONTAINER  (no -p needed)
backend container ──(mysql:3306 over the Docker network)──▶ mysql container :3306
```

These are different.

---

## Part 11 — Three-Tier Networking Using the Java Project

```
Browser
   │  http://localhost:3000
   ▼
frontend   React build served by Nginx (:8080 inside the container)
   │  /api/ → http://backend:8080
   ▼
backend    Spring Boot (:8080)
   │  jdbc:mysql://mysql:3306/projectops
   ▼
mysql      MySQL (:3306)
```

Two extremely important names:

- `backend`
- `mysql`

These aren't hard-coded IP addresses. They become **Docker DNS names**.

---

## Part 12 — First Demo Without Docker Compose

Do this **before** Compose. It shows what Compose actually solves.

Go to the project:

```bash
cd 3-Tier-Java-NodeJS-MySQL-Project-NEW
```

---

## Part 13 — Improve the Java Backend Dockerfile

The existing Dockerfile is already multi-stage, but this improved version replaces it.

**`backend/Dockerfile`**

```dockerfile
# syntax=docker/dockerfile:1

# ============================
# Stage 1 - Build
# ============================

FROM maven:3.9.11-eclipse-temurin-21 AS build

WORKDIR /workspace

# Copy dependency definition first
COPY pom.xml .

# Cache Maven dependencies
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B dependency:go-offline

# Now copy application source
COPY src ./src

# Build application
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B clean package


# ============================
# Stage 2 - Runtime
# ============================

FROM eclipse-temurin:21-jre-jammy AS runtime

# Create dedicated non-root user
RUN groupadd --gid 10001 appgroup \
    && useradd \
       --uid 10001 \
       --gid appgroup \
       --no-create-home \
       --shell /usr/sbin/nologin \
       appuser

WORKDIR /app

COPY --from=build \
     --chown=appuser:appgroup \
     /workspace/target/projectops-api.jar \
     /app/app.jar

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

---

## Part 14 — Backend `.dockerignore`

Create **`backend/.dockerignore`**:

```gitignore
target/
.git/
.gitignore
README.md
*.log
```

**Reason:** without `.dockerignore`:

```
docker build
     ↓
entire directory
     ↓
sent as build context
```

We don't need things like `target/`, `.git/` or log files in the build context.

---

## Part 15 — Better React Frontend Image

The frontend is interesting: React/Vite is needed to **build** the UI, but Node.js is **not** needed to **serve** the compiled files.

| Command | Best use | Behavior |
|---------|----------|----------|
| `npm install` | Local development | Can update `package-lock.json` |
| `npm ci` | CI/CD, Docker builds | Installs exactly from `package-lock.json` |

Therefore:

```
Build stage:   Node.js → npm run build → dist/
Runtime:       Nginx (serves dist/)
```

### Revised frontend Dockerfile

**`frontend/Dockerfile`**

```dockerfile
# syntax=docker/dockerfile:1

# ============================
# Stage 1 - Build React App
# ============================

FROM node:22-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./

RUN --mount=type=cache,target=/root/.npm \
    npm ci \
    --no-audit \
    --no-fund

COPY . .

RUN npm run build


# ============================
# Stage 2 - Serve with Nginx
# ============================

FROM nginxinc/nginx-unprivileged:stable-alpine

COPY nginx.conf /etc/nginx/conf.d/default.conf

COPY --from=build \
     /app/dist \
     /usr/share/nginx/html

EXPOSE 8080
```

> 💡 The NGINX-maintained **unprivileged** image is used here because it's designed to run Nginx as a non-root user. Its default unprivileged listen port is **8080**.

### Revised `nginx.conf`

Because Nginx now runs unprivileged, use port **8080**:

```nginx
server {

    listen 8080;

    server_name _;

    root /usr/share/nginx/html;

    index index.html;


    location /api/ {

        proxy_pass http://backend:8080;

        proxy_http_version 1.1;

        proxy_set_header Host $host;

        proxy_set_header X-Real-IP $remote_addr;

        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_set_header X-Forwarded-Proto $scheme;
    }


    location / {

        try_files $uri $uri/ /index.html;
    }
}
```

### Architecture

```
Browser
   │  http://localhost:3000
   ▼
frontend container (nginx-unprivileged, :8080)
   ├── /       → static React files from /usr/share/nginx/html
   │             (unknown paths fall back to index.html for client-side routing)
   └── /api/   → proxy_pass http://backend:8080
                    │
                    ▼
              backend container (:8080)
```

### Frontend `.dockerignore`

```gitignore
node_modules/
dist/
.git/
.gitignore
npm-debug.log*
*.log
```

---

## Part 16 — Build Both Images

From the project root, build the backend:

```bash
docker build \
  -t devopsshack/projectops-backend:v2 \
  ./backend
```

Frontend:

```bash
docker build \
  -t devopsshack/projectops-frontend:v2 \
  ./frontend
```

Check:

```bash
docker images
```

---

## Part 17 — Create the Docker Network Manually

```bash
docker network create projectops-network
```

Check:

```bash
docker network ls
```

Inspect:

```bash
docker network inspect projectops-network
```

At this point, no containers are attached yet.

---

## Part 18 — Docker Volumes Before Starting MySQL

**Where does MySQL actually store database files?** Inside the container filesystem, at `/var/lib/mysql`.

**What happens when the container is deleted?** If the data exists only in the writable container layer:

```
docker rm mysql
        ↓
container filesystem removed
        ↓
database data lost
```

This is why we use a **volume**.

---

## Part 19 — Types of Docker Storage

| # | Type | Managed by | Lifetime | Typical use |
|---|------|------------|----------|-------------|
| 1 | Container writable layer | Docker, inside the container | Deleted with the container | Temporary only, **not** for database data |
| 2 | Named volume | Docker | Survives container removal | MySQL, PostgreSQL, MongoDB data |
| 3 | Bind mount | You (a host path) | Lives on the host | Config files, source code during development, init scripts |
| 4 | tmpfs | Memory | Gone when the container stops | Temporary writable paths in read-only containers |

### 1. Writable layer

Temporary. Lives with the container. Not suitable for persistent database data.

### 2. Named volume

Docker manages the storage:

```bash
docker volume create projectops-mysql-data
```

Perfect for MySQL, PostgreSQL and MongoDB.

### 3. Bind mount

A host path mounted directly into the container:

```bash
-v ./database/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
```

| Part | Meaning |
|------|---------|
| `-v` | Attach a volume / bind mount |
| `./database/init.sql` | File on your **host** machine |
| `/docker-entrypoint-initdb.d/init.sql` | Location of that file **inside** the container |
| `:ro` | **Read-only:** the container can read it but cannot modify it |

Useful for configuration files, source code during development, and initialization scripts.

### 4. tmpfs

Data is stored in memory:

```bash
--tmpfs /tmp
```

Useful for temporary writable paths in otherwise read-only containers.

---

## Part 20 — Create the MySQL Volume

```bash
docker volume create projectops-mysql-data
```

Check:

```bash
docker volume ls
```

Inspect:

```bash
docker volume inspect projectops-mysql-data
```

---

## Part 21 — Start MySQL Manually

From the project root:

```bash
docker run -d \
  --name mysql \
  --network projectops-network \
  -e MYSQL_DATABASE=projectops \
  -e MYSQL_USER=projectops \
  -e MYSQL_PASSWORD=projectops123 \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -v projectops-mysql-data:/var/lib/mysql \
  -v "$PWD/database/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro" \
  mysql:8.4
```

- **`/docker-entrypoint-initdb.d/`** is a special initialization directory in many official database images, especially MySQL and PostgreSQL.
- The **`01-`** prefix is useful because initialization files are usually processed in **alphabetical order**:

```
01-init.sql
02-schema.sql
03-seed.sql
```

> 🔒 **Important:** we did **not** need `-p 3306:3306`, because only the backend needs MySQL. That immediately reduces unnecessary exposure.

---

## Part 22 — Check MySQL

```bash
docker ps
```

Logs:

```bash
docker logs -f mysql
```

Then:

```bash
docker exec mysql \
  mysqladmin ping \
  -uroot \
  -proot123
```

Expected:

```
mysqld is alive
```

---

## Part 23 — Start the Backend

```bash
docker run -d \
  --name backend \
  --network projectops-network \
  -e 'DB_URL=jdbc:mysql://mysql:3306/projectops?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC' \
  -e DB_USERNAME=projectops \
  -e DB_PASSWORD=projectops123 \
  -p 8080:8080 \
  devopsshack/projectops-backend:v2
```

Focus on:

```
jdbc:mysql://mysql:3306
              ↑
        not localhost
```

---

## Part 26 — Start the Frontend

```bash
docker run -d \
  --name frontend \
  --network projectops-network \
  -p 3000:8080 \
  devopsshack/projectops-frontend:v2
```

Open:

- `http://<SERVER-IP>:3000`
- or locally: `http://localhost:3000`

The architecture is now:

```
Browser
   │  localhost:3000   (published: -p 3000:8080)
   ▼
[frontend :8080] ─┐
   │              │
   ▼              │  all attached to: projectops-network
[backend  :8080]  │  (Docker DNS names: frontend, backend, mysql)
   │              │
   ▼              │
[mysql    :3306] ─┘
   │
   ▼
volume: projectops-mysql-data → /var/lib/mysql
```

---

## Part 27 — Notice the Problem

We needed all of this:

```bash
docker network create ...
docker volume create ...
docker run mysql ...
docker run backend ...
docker run frontend ...
```

And each command contains:

- environment variables
- networks
- volumes
- ports
- image names
- restart configuration
- security settings
- dependencies

Now imagine 3 containers, then 7, 20 or 50. Managing that with individual `docker run` commands becomes painful.

**This is where Docker Compose comes in.**

---

## Part 28 — What Is Docker Compose?

- A **Dockerfile** describes how to **build an image**.
- **Docker Compose** describes how **multiple containers should run together**.

| | Dockerfile | Docker Compose |
|---|------------|----------------|
| Describes | How to build **one image** | How **multiple containers** run together |
| File | `Dockerfile` | `compose.yaml` / `docker-compose.yml` |
| Main command | `docker build` | `docker compose up` |
| Result | An image | A running multi-container application |

---

## Part 29 — Docker Compose File Structure

Complete conceptual structure:

```yaml
services:

  frontend:
    image: frontend

  backend:
    image: backend

  database:
    image: mysql

volumes:
  database-data:

networks:
  frontend-network:
  backend-network:
```

---

## Part 30 — Service vs Container

Compose introduces a very important concept:

```yaml
services:
  backend:
```

`backend` is the **service name**, and Docker networking makes that name resolvable. Therefore:

```
services:
  backend:                          ← service name (in the Compose file)
        ↓
DNS name "backend"                  ← what other containers use, e.g. http://backend:8080
        ↓
container(s), e.g. projectops-java-backend-1   ← the actual runtime instance(s)
```

---

## Part 31 — Don't Use `container_name` Unless You Actually Need It

The existing projects contain:

```yaml
container_name: projectops-backend
```

**Let Compose name the containers.** Compose already provides service DNS names:

- `backend`
- `mysql`
- `frontend`

A fixed `container_name` also prevents scaling that service to more than one container, because container names must be unique.

---

## Part 32 — Compose Automatically Creates a Network

With:

```bash
docker compose up
```

Compose automatically creates a project network, for example `projectops_default`. Then `frontend`, `backend` and `mysql` all join that network.

That's why `backend` and `mysql` become **resolvable by name**.

---

## Part 33 — Environment Variables in Compose

Example:

```yaml
environment:
  DB_USERNAME: projectops
  DB_PASSWORD: projectops123
```

These become environment variables inside the container. Check:

```bash
docker compose exec backend env
```

Spring Boot then reads them:

```properties
spring.datasource.username=${DB_USERNAME:projectops}
spring.datasource.password=${DB_PASSWORD:projectops123}
```

Flow:

```
docker-compose.yml  (DB_PASSWORD)
        ↓
Container environment
        ↓
Spring Boot
        ↓
application.properties
```

---

## Part 34 — `.env` and Compose

Instead of hardcoding:

```yaml
MYSQL_PASSWORD: projectops123
```

create a **`.env`** file:

```env
MYSQL_DATABASE=projectops
MYSQL_USER=projectops
MYSQL_PASSWORD=projectops123
MYSQL_ROOT_PASSWORD=root123
```

Then reference the variables:

```yaml
MYSQL_DATABASE: ${MYSQL_DATABASE}
MYSQL_USER: ${MYSQL_USER}
MYSQL_PASSWORD: ${MYSQL_PASSWORD}
```

> ⚠️ **Important:** the Compose `.env` file is mainly for **Compose variable substitution**. It is **not** a secure secret-management system.
>
> - Don't commit real production passwords.
> - Add `.env` to `.gitignore`.

Production secrets belong in systems like these, depending on the architecture:

- HashiCorp Vault
- AWS Secrets Manager
- Azure Key Vault
- Kubernetes Secrets + an external secret system

---

## Part 35 — `depends_on`

Example:

```yaml
backend:

  depends_on:
    mysql:
      condition: service_healthy
```

Meaning:

```
Start MySQL
    ↓
wait until the MySQL health check succeeds
    ↓
start backend
```

> ⚠️ **Without** a health condition, `depends_on` only controls **startup order**. It doesn't mean the dependency is actually ready to accept connections.

---

## Part 36 — Docker Health Checks

For MySQL:

```yaml
healthcheck:

  test:
    [
      "CMD",
      "mysqladmin",
      "ping",
      "-h",
      "localhost",
      "-uroot",
      "-proot123"
    ]

  interval: 5s

  timeout: 5s

  retries: 20
```

The important part is:

```bash
mysqladmin ping -h localhost -uroot -proot123
```

It basically asks MySQL: *"Are you alive and accepting connections?"*

- If MySQL responds successfully, Docker marks the container **`healthy`**.
- If it keeps failing, Docker eventually marks it **`unhealthy`**.

| Setting | Meaning |
|---------|---------|
| `interval: 5s` | Docker runs the health check every 5 seconds |
| `timeout: 5s` | Each health check gets at most 5 seconds to respond |
| `retries: 20` | After 20 consecutive failures, the container is marked `unhealthy` |

---

## Part 37 — Complete Improved Java Compose

```yaml
name: projectops-java

services:

  mysql:
    image: mysql:8.4

    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE:-projectops}
      MYSQL_USER: ${MYSQL_USER:-projectops}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-projectops123}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-root123}

    volumes:
      - mysql-data:/var/lib/mysql
      - ./database/init.sql:/docker-entrypoint-initdb.d/01-init.sql:ro

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "mysqladmin ping -h localhost -uroot -p\"$${MYSQL_ROOT_PASSWORD}\""
        ]
      interval: 5s
      timeout: 5s
      retries: 20
      start_period: 20s

    networks:
      - database-network

    restart: unless-stopped


  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile

    environment:
      DB_URL: >-
        jdbc:mysql://mysql:3306/${MYSQL_DATABASE:-projectops}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC
      DB_USERNAME: ${MYSQL_USER:-projectops}
      DB_PASSWORD: ${MYSQL_PASSWORD:-projectops123}

    depends_on:
      mysql:
        condition: service_healthy

    networks:
      - application-network
      - database-network

    read_only: true

    tmpfs:
      - /tmp

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

    restart: unless-stopped


  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile

    ports:
      - "3000:8080"

    depends_on:
      - backend

    networks:
      - application-network

    read_only: true

    tmpfs:
      - /tmp

    security_opt:
      - no-new-privileges:true

    cap_drop:
      - ALL

    restart: unless-stopped


volumes:
  mysql-data:


networks:
  application-network:

  database-network:
    internal: true
```

> 📝 **Syntax notes:**
> - `${VAR:-default}` uses `VAR` from `.env` or the shell, or `default` if it isn't set.
> - `$$` escapes the `$`, so **Compose doesn't substitute it**. `$${MYSQL_ROOT_PASSWORD}` is expanded by the shell **inside the container** instead.
> - `CMD-SHELL` runs the test through a shell, which is what makes that `$` expansion work. Plain `CMD` runs the command directly.
> - `>-` is a YAML folded string that joins the lines into one value with no trailing newline.

---

## Part 38 — Why Two Networks?

Instead of putting the frontend, backend and MySQL **all in the same network**, we create:

- `application-network`
- `database-network`

| Service | Joins |
|---------|-------|
| Frontend | `application-network` |
| Backend | `application-network` + `database-network` |
| MySQL | `database-network` only |

```
application-network:   frontend ◀──▶ backend
database-network:                    backend ◀──▶ mysql     (internal: true)

frontend ──✗──▶ mysql    (no shared network → no direct path)
```

So the frontend has no direct database connectivity. That's **network segmentation**.

---

## Part 39 — Why `database-network` Is Internal

```yaml
database-network:
  internal: true
```

This isolates the database-side network from ordinary external connectivity:

| Traffic | Allowed? |
|---------|----------|
| Backend → MySQL | ✅ |
| MySQL → Backend | ✅ |
| MySQL → Internet | ❌ |
| Internet → MySQL | ❌ |

---

## Part 40 — Why We Don't Publish MySQL

Notice there's **no**:

```yaml
ports:
  - "3306:3306"
```

- Our browser doesn't need database access.
- `ports:` is only needed when something **outside Docker** has to reach the container.
- Publishing database ports unnecessarily expands exposure.

---

## Part 41 — Why We Don't Publish the Backend Either

There's no `8080:8080` on the backend. The browser communicates like this:

```
Browser
   ↓
Frontend
   ↓
Backend
```

**Frontend/Nginx is the entry point**, so only port **3000** needs to be published.

---

## Part 42 — Read-Only Containers

Backend:

```yaml
read_only: true
```

This means the container's root filesystem **isn't writable** during normal execution. But Java may need temporary storage, so:

```yaml
tmpfs:
  - /tmp
```

```
Root filesystem  →  read-only
/tmp             →  temporary writable memory filesystem
```

By default, `/tmp` would also be read-only, since it's part of the container's root filesystem. But Docker mounts a **separate tmpfs filesystem on top of `/tmp`**:

```
Container root filesystem
/
├── app      read-only
├── etc      read-only
├── usr      read-only
└── tmp
     ↓
  separate tmpfs mount
     ↓
  writable ✅
```

---

## Part 43 — `cap_drop: ALL`

```yaml
cap_drop:
  - ALL
```

Linux breaks root privileges into smaller units called **capabilities**. Rather than giving applications capabilities they don't need:

1. **Drop everything.**
2. Add a capability back **only** if the application genuinely needs it.

These ordinary HTTP applications don't need any powerful Linux capabilities.

---

## Part 44 — `no-new-privileges`

```yaml
security_opt:
  - no-new-privileges:true
```

Processes inside the container **cannot gain additional privileges**, for example through setuid executables.

Again: **least privilege.**

---

## Part 45 — `restart: unless-stopped`

```yaml
restart: unless-stopped
```

If the container crashes, Docker restarts it automatically. It stays stopped only if you stopped it yourself.

---

## Part 46 — Run Compose

Before running anything:

```bash
docker compose config
```

**This is extremely useful.** It validates the Compose file and prints the final rendered configuration.

Then:

```bash
docker compose build
```

or:

```bash
docker compose up -d --build
```

Using a specific Compose file:

```bash
docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml up -d --build
docker compose -f docker-compose.dev.yml down

# General form:
docker compose -f <filename> <command>
```

| Part | Meaning |
|------|---------|
| `up` | Create and start the application |
| `-d` | Detached (runs in the background) |
| `--build` | Build local images first |

---

## Part 47 — Check Services

```bash
docker compose ps
```

All logs:

```bash
docker compose logs
```

A specific service:

```bash
docker compose logs backend
```

Follow live:

```bash
docker compose logs -f backend
```

Last 100 lines:

```bash
docker compose logs \
  --tail=100 \
  backend
```

---

## Part 48 — Verify the Application

Open: `http://localhost:3000`

---

## Part 49 — Inspect the Networks

```bash
docker network ls
```

Compose creates networks like:

```
projectops-java_application-network
projectops-java_database-network
```

Inspect the application network:

```bash
docker network inspect \
  projectops-java_application-network
```

→ You should see **`frontend`** and **`backend`**.

Inspect the database network:

```bash
docker network inspect \
  projectops-java_database-network
```

→ You should see **`backend`** and **`mysql`**.

---

## Part 50 — Inspect the Volume

```bash
docker volume ls
```

You'll see something like:

```
projectops-java_mysql-data
```

Inspect it:

```bash
docker volume inspect \
  projectops-java_mysql-data
```

---

## Part 52 — Now Destroy the Volume

```bash
docker compose down -v
```

| Command | Removes | Named volumes |
|---------|---------|---------------|
| `docker compose down` | Containers + networks | ✅ Normally kept |
| `docker compose down -v` | Containers + networks + Compose volumes | ❌ Deleted |

Run again:

```bash
docker compose up -d
```

The database is fresh, and the seed data is recreated. **This is a perfect demonstration of persistence.**

---

## Part 53 — `docker compose exec`

Enter MySQL:

```bash
docker compose exec mysql \
  mysql \
  -uprojectops \
  -pprojectops123 \
  projectops
```

Then:

```sql
SHOW TABLES;
SELECT * FROM project;   -- table name depends on Hibernate's naming
```

For a shell (if one is available in the image):

```bash
docker compose exec backend sh
```

---

## Part 54 — `docker compose run` vs `exec`

| Command | What it does |
|---------|--------------|
| `exec` | Runs a command inside an **existing**, running container, e.g. `docker compose exec backend sh` |
| `run` | Creates a **new one-off container** for a service |

---

## Part 55 — Project 2

Once Project 1 is fully understood, Project 2 becomes much easier.

Architecture:

```
Browser
    │
    │ :5000
    ▼
Python / Flask
    │
    │ HTTP
    ▼
ASP.NET Core / .NET 8
    │
    │ Npgsql
    ▼
PostgreSQL 16
```

The application already uses the **`BACKEND_URL`** variable to tell Python where .NET is located. Inside Docker:

```
BACKEND_URL=http://backend:8080
```

Again, `backend` is **Docker service discovery**.

---

## Part 56 — Hardened .NET Dockerfile

Replace the existing backend Dockerfile with:

```dockerfile
# syntax=docker/dockerfile:1

# ============================
# Stage 1 - Restore
# ============================

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /src

COPY ProjectOps.Api.csproj .

RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore

COPY . .

RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet publish \
      -c Release \
      -o /app/publish \
      --no-restore \
      /p:UseAppHost=false


# ============================
# Stage 2 - Runtime
# ============================

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime

WORKDIR /app

ENV ASPNETCORE_URLS=http://+:8080

COPY --from=build \
     --chown=app:app \
     /app/publish .

USER app

EXPOSE 8080

ENTRYPOINT ["dotnet", "ProjectOps.Api.dll"]
```

> 💡 Starting with **.NET 8**, Microsoft's Linux container images include a non-root **`app`** user, so we can switch to it with `USER app` instead of creating another runtime user.

### .NET `.dockerignore`

```gitignore
bin/
obj/
.git/
.gitignore
*.user
*.log
```

---

## Part 57 — Hardened Python Frontend Dockerfile

The current Python frontend is straightforward, but it can be improved with a **builder stage** and an **isolated virtual environment**.

```dockerfile
# syntax=docker/dockerfile:1

# ============================
# Stage 1 - Dependencies
# ============================

FROM python:3.13-slim AS builder

ENV VIRTUAL_ENV=/opt/venv

RUN python -m venv ${VIRTUAL_ENV}

ENV PATH="${VIRTUAL_ENV}/bin:${PATH}"

WORKDIR /build

COPY requirements.txt .

RUN pip install \
      --no-cache-dir \
      --upgrade pip \
    && pip install \
      --no-cache-dir \
      -r requirements.txt


# ============================
# Stage 2 - Runtime
# ============================

FROM python:3.13-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1

ENV PYTHONUNBUFFERED=1

ENV VIRTUAL_ENV=/opt/venv

ENV PATH="${VIRTUAL_ENV}/bin:${PATH}"


RUN groupadd \
      --gid 10001 \
      appgroup \
    && useradd \
      --uid 10001 \
      --gid appgroup \
      --no-create-home \
      --shell /usr/sbin/nologin \
      appuser


WORKDIR /app


COPY --from=builder \
     --chown=appuser:appgroup \
     /opt/venv \
     /opt/venv


COPY --chown=appuser:appgroup \
     . \
     .


USER 10001:10001


EXPOSE 5000


CMD ["gunicorn", \
     "--bind", "0.0.0.0:5000", \
     "--workers", "2", \
     "--threads", "4", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "app:app"]
```

> ⚠️ **Fixed from the original:** in a Dockerfile, an instruction only continues onto the next line if the line ends with `\`. The original `CMD [` written across lines **without** `\` would fail to build, so each line here ends with `\`. You could also write the whole `CMD [...]` on a single line.

### Why `PYTHONDONTWRITEBYTECODE`?

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
```

Python normally may generate `__pycache__/` and `*.pyc` files. We don't need those written into the application filesystem, and this also works nicely with `read_only: true`.

### Why `PYTHONUNBUFFERED`?

```dockerfile
ENV PYTHONUNBUFFERED=1
```

Without it, stdout may be **buffered**: Python may hold `print()` output in memory for a while instead of showing it immediately.

`PYTHONUNBUFFERED=1` makes Python send console output and logs **immediately**. Inside containers, we want logs available right away through `docker logs`:

```
Application
   ↓
stdout / stderr
   ↓
Docker logging
```

### Python `.dockerignore`

```gitignore
.venv/
__pycache__/
*.pyc
*.pyo
*.log
.git/
```

---

## Part 58 — .NET Configuration Naming

This application gives us a great example. In **`appsettings.json`**:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "..."
  }
}
```

Docker environment variable:

```yaml
ConnectionStrings__DefaultConnection: ...
```

Notice the **`__` (double underscore)**. .NET maps:

```
ConnectionStrings__DefaultConnection  →  ConnectionStrings:DefaultConnection
```

So Compose can **override application configuration without changing the source code**.

---

## Part 59 — Improved Python / .NET / Postgres Compose

```yaml
name: projectops-dotnet

services:

  postgres:
    image: postgres:16-alpine

    environment:
      POSTGRES_DB: ${POSTGRES_DB:-projectops}
      POSTGRES_USER: ${POSTGRES_USER:-projectops}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-projectops123}

    volumes:
      - postgres-data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U ${POSTGRES_USER:-projectops} -d ${POSTGRES_DB:-projectops}"
        ]
      interval: 5s
      timeout: 5s
      retries: 20
      start_period: 10s

    networks:
      - database-network

    restart: unless-stopped


  backend:
    build:
      context: ./backend-dotnet
      dockerfile: Dockerfile

    environment:
      ASPNETCORE_URLS: http://+:8080
      ASPNETCORE_ENVIRONMENT: Production
      ConnectionStrings__DefaultConnection: >-
        Host=postgres;Port=5432;Database=${POSTGRES_DB:-projectops};Username=${POSTGRES_USER:-projectops};Password=${POSTGRES_PASSWORD:-projectops123}

    depends_on:
      postgres:
        condition: service_healthy

    networks:
      - application-network
      - database-network

    read_only: true

    tmpfs:
      - /tmp

    cap_drop:
      - ALL

    security_opt:
      - no-new-privileges:true

    restart: unless-stopped


  frontend:
    build:
      context: ./frontend-python
      dockerfile: Dockerfile

    environment:
      BACKEND_URL: http://backend:8080
      PORT: 5000
      FLASK_ENV: production

    ports:
      - "5000:5000"

    depends_on:
      - backend

    networks:
      - application-network

    read_only: true

    tmpfs:
      - /tmp

    cap_drop:
      - ALL

    security_opt:
      - no-new-privileges:true

    restart: unless-stopped


volumes:
  postgres-data:


networks:
  application-network:

  database-network:
    internal: true
```

### Clean rebuild

```bash
docker compose down -v
docker compose build --no-cache
docker compose up
```

| Command | Meaning |
|---------|---------|
| `down -v` | Remove old containers + networks + volumes |
| `build --no-cache` | Rebuild the frontend/backend images from scratch |
| `up` | Create and start the complete application stack |

### Exploring PostgreSQL

First enter the PostgreSQL container:

```bash
docker compose exec postgres sh
```

Then connect to PostgreSQL:

```bash
psql -U projectops -d projectops
```

Useful `psql` commands:

| Command | What it does |
|---------|--------------|
| `\l` | List all databases |
| `\dt` | List tables in the current database (you should see `projects`) |
| `\d projects` | Show the structure of the `projects` table |
| `SELECT * FROM projects;` | Show all rows |
| `\du` | List users/roles |
| `\conninfo` | Show which database/user you're connected as |
| `\q` | Quit `psql` |

Then leave the container with `exit`.

**Shortcut:** skip the shell and open PostgreSQL directly:

```bash
docker compose exec postgres \
  psql -U projectops -d projectops
```

---

## Part 60 — The Complete Second Flow

1. The browser sends `GET http://localhost:5000/api/projects`.
2. Python receives `/api/projects`.
3. Python has `BACKEND_URL=http://backend:8080`, so it calls the .NET backend.

```
Browser
   │  GET http://localhost:5000/api/projects
   ▼
frontend   Python/Flask (:5000)
   │  BACKEND_URL → http://backend:8080
   ▼
backend    .NET 8 (:8080)
   │  Host=postgres;Port=5432   (Npgsql)
   ▼
postgres   PostgreSQL 16 (:5432)
```

Again:

- **No container IP addresses.**
- **No `localhost` between containers.**

---

## Part 61 — Start Project 2

```bash
docker compose config
```

Then:

```bash
docker compose up -d --build
```

Check:

```bash
docker compose ps
```

Open `http://localhost:5000`.

Health check:

```bash
curl http://localhost:5000/health
```

The Python `/health` endpoint also checks the backend's status.

---

## Part 62 — Verify PostgreSQL

```bash
docker compose exec postgres \
  psql \
  -U projectops \
  -d projectops
```

Then:

```sql
\dt

SELECT
    id,
    title,
    category,
    status,
    progress
FROM projects;
```

Exit:

```sql
\q
```

---

## Part 63 — Important Docker Compose Commands

*(Practice these by actually running them, not just reading them.)*

| Task | Command | Notes |
|------|---------|-------|
| Start | `docker compose up` | Foreground |
| Start detached | `docker compose up -d` | Background |
| Build and start | `docker compose up -d --build` | |
| Only build | `docker compose build` | |
| Build one service | `docker compose build backend` | |
| Force clean build | `docker compose build --no-cache backend` | Ignores the cache |
| Status | `docker compose ps` | |
| Logs | `docker compose logs` | |
| Logs for one service | `docker compose logs backend` | |
| Follow logs | `docker compose logs -f backend` | |
| Execute a command | `docker compose exec backend sh` | |
| Restart one service | `docker compose restart backend` | You don't have to restart the whole stack |
| Stop | `docker compose stop` | Containers remain |
| Start again | `docker compose start` | |
| Remove app containers | `docker compose down` | |
| Remove containers + volumes | `docker compose down -v` | ⚠️ Use carefully |
| See final configuration | `docker compose config` | |

---

## Part 64 — Compose Rebuild Behaviour

Suppose the Java code changes. You can run:

```bash
docker compose up -d --build backend
```

Then:

```
Maven dependencies unchanged  →  cached
Source changed                →  compile/package again
Runtime image                 →  recreated
```

---

## Part 65 — Service Lifecycle

| Concept | What it is |
|---------|------------|
| Compose configuration | The **desired** application definition |
| Container | A **runtime instance** |

If the backend image changes:

```bash
docker compose up -d --build backend
```

Compose may recreate the **backend container**, but it doesn't need to recreate the **database volume**. This separation is fundamental.

---

## Part 66 — The Container Is Disposable, the Data Is Not

- Containers should be **disposable**.
- Persistent data should live **outside** the container's writable layer.

```
Container  →  ✗ disposable
Volume     →  ✓ persistent
```

For MySQL:

```
container
    │
    ▼
/var/lib/mysql
    │
    ▼
named volume
```

---

## Part 77 — Secrets Should Not Be Baked Into Images

**Never do this:**

```dockerfile
ENV DB_PASSWORD=myproductionpassword
```

**Why?** An image can be:

- shared
- pushed
- inspected
- cached
- reused

**Configuration belongs at runtime.**

```
✅ Better:  Image  +  Runtime configuration
❌ Not:     Image containing production secrets
```

---

## Part 78 — Build-Time `ARG` vs Runtime `ENV`

| Instruction | Example | Available |
|-------------|---------|-----------|
| `ARG` | `ARG APP_VERSION` | Mainly **during the image build** |
| `ENV` | `ENV APP_ENV=production` | During the build (after it's declared) **and** at container runtime |

### Demo Dockerfile

```dockerfile
FROM alpine:3.20

ARG APP_VERSION=1.0
ENV APP_ENV=production

RUN echo "During BUILD:" && \
    echo "APP_VERSION = $APP_VERSION" && \
    echo "APP_ENV = $APP_ENV"

CMD ["sleep", "3600"]
```

### 1. Build the image

```bash
docker build \
  --build-arg APP_VERSION=2.5 \
  -t arg-env-demo .
```

During the build you'll see:

```
During BUILD:
APP_VERSION = 2.5
APP_ENV = production
```

So at **build time**, `ARG` ✅ and `ENV` ✅ are both available.

### 2. Create the container

```bash
docker run -d \
  --name arg-env-container \
  arg-env-demo
```

Go inside:

```bash
docker exec -it arg-env-container sh
```

Check both:

```sh
echo $APP_VERSION    # → (nothing — ARG is gone)
echo $APP_ENV        # → production
```

### Summary

| | Build time | Runtime |
|---|------------|---------|
| `ARG APP_VERSION` | ✅ `2.5` | ❌ Not available |
| `ENV APP_ENV` | ✅ `production` | ✅ `production` |

```
ARG  →  build-time configuration
ENV  →  runtime environment
```

`ARG` is **build-time only** by default. `ENV` is available during the build after it's declared, **and** it persists into the container at runtime.

> ⚠️ Don't use either one casually for secrets.

---

## Part 79 — Image Configuration vs Runtime Configuration

| Dockerfile (intrinsic to the image) | Compose (environment-specific runtime config) |
|-------------------------------------|-----------------------------------------------|
| Runtime | Database URL |
| Application binary | Ports |
| Dependencies | Volumes |
| Default command | Network relationships |
| Application user | Runtime environment |
| | Restart policy |

This gives you **the same image** for Development, QA and Production, each with **different runtime configuration**.

---

## Part 82 — Very Important Interview Questions From Day 2

*Be able to answer these from understanding, not memorization.*

**Q: Why doesn't `localhost` work between containers?**
Each container has its own network namespace. Inside the backend, `localhost` = the backend itself.

**Q: How do containers find each other?**
Through Docker networking and DNS/service names, e.g. `backend`, `mysql`, `postgres`.

**Q: Do we need `-p` for container-to-container traffic?**
No. `-p` publishes a container port to the **host**. Containers on the same network communicate directly.

**Q: Why use volumes?**
To separate persistent data from disposable containers.

**Q: `docker compose down` vs `docker compose down -v`?**
- `down` → removes containers and networks; named volumes are normally kept.
- `down -v` → removes volumes too.

**Q: Dockerfile vs Compose?**
- Dockerfile → builds **one image**.
- Compose → runs and configures a **multi-container application**.

**Q: Why use health checks?**
A container that's **running** isn't necessarily **ready**.

**Q: Why use multi-stage builds?**
To keep build dependencies out of the runtime image.

**Q: Why run as non-root?**
To reduce the privileges available to compromised or malfunctioning application code.

**Q: Why shouldn't database ports always be published?**
Only services that need external access should be externally reachable.

**Q: Why use service names instead of IP addresses?**
Container IPs are dynamic. Service names give stable, logical discovery.

---

## Part 84 — `docker stats`

Once everything is running:

```bash
docker stats
```

This shows live CPU, memory, network and disk I/O usage per container.

---

## Part 85 — Container Logs

Applications should preferably log to **stdout** and **stderr**, so Docker can collect the logs with `docker logs`, instead of the application only writing to a file like `/app/logs/application.log`.

Examples:

```bash
docker compose logs frontend
docker compose logs backend
docker compose logs mysql
```

---

## Part 86 — Final Architecture: Java Demo

```
Browser
   │  http://localhost:3000        (only published port: 3000 → 8080)
   ▼
frontend   nginx-unprivileged :8080          [application-network]
   │  /api/ → http://backend:8080
   ▼
backend    Spring Boot :8080                 [application-network + database-network]
   │  jdbc:mysql://mysql:3306/projectops
   ▼
mysql      MySQL 8.4 :3306                   [database-network — internal]
   │
   ▼
volume     mysql-data → /var/lib/mysql

frontend & backend: read_only + tmpfs /tmp, cap_drop ALL, no-new-privileges
all services:       restart: unless-stopped
```

---

## Part 87 — Final Architecture: Python / .NET Demo

```
Browser
   │  http://localhost:5000        (only published port: 5000 → 5000)
   ▼
frontend   Python/Flask + gunicorn :5000     [application-network]
   │  BACKEND_URL = http://backend:8080
   ▼
backend    ASP.NET Core .NET 8 :8080         [application-network + database-network]
   │  ConnectionStrings__DefaultConnection → Host=postgres;Port=5432
   ▼
postgres   PostgreSQL 16 :5432               [database-network — internal]
   │
   ▼
volume     postgres-data → /var/lib/postgresql/data

frontend & backend: read_only + tmpfs /tmp, cap_drop ALL, no-new-privileges
all services:       restart: unless-stopped
```

---

## Debugging Scenarios

### Part 71 — Scenario 2: Backend Cannot Connect to the Database

Typical log:

```
Connection refused
```

Check the services:

```bash
docker compose ps
```

Check the database logs:

```bash
docker compose logs mysql
# or
docker compose logs postgres
```

Then check the configuration:

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| `localhost:3306` | `mysql:3306` |
| `localhost:5432` | `postgres:5432` |

### Part 72 — Scenario 3: Port Already Allocated

Error:

```
port is already allocated
0.0.0.0:3000 failed
```

Find what's using it:

```bash
docker ps

# Linux:
sudo ss -lntp | grep 3000
```

Then either stop the existing process or change the host port:

```yaml
ports:
  - "3001:8080"
```

Remember: **`3001` = host**, **`8080` = container**.

### Part 73 — Scenario 4: Container Exits Immediately

```bash
docker compose ps -a
docker compose logs backend
```

Check for:

- the application crashed
- a missing environment variable
- the database being unavailable
- an incorrect command
- file permissions
- the wrong CPU architecture
- a missing dependency

> 🧠 **Don't blindly restart. Look at the logs first.**

### Part 74 — Scenario 5: Data Disappeared

Ask: did you run `docker compose down -v` or `docker volume rm ...`?

Check:

```bash
docker volume ls
docker volume inspect ...
```

### Part 75 — Scenario 6: Environment Variable Not Working

Check the final Compose configuration:

```bash
docker compose config
```

Then check inside the container:

```bash
docker compose exec backend env
```

These answer two separate questions:

1. **Did Compose resolve the variable correctly?** → `docker compose config`
2. **Did the container receive it correctly?** → `docker compose exec backend env`