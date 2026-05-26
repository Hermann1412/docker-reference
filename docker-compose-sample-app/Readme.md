# L09-05 — Docker Compose Sample App: Lab Report

## Overview

This lab deploys a multi-service web application using Docker Compose. The stack consists of three
containers — a React **frontend**, a Node.js **backend**, and a **MariaDB** database — connected
through two isolated networks with persistent volumes and secure secret management.

---

## compose.yaml — Structure and Services

The `compose.yaml` file is the primary configuration picked up by Docker Compose (when both
`compose.yaml` and `docker-compose.yml` are present, Compose prefers `compose.yaml` and emits a
warning).

### Service: `db`

```yaml
db:
  image: mariadb:10.6.4-focal
  command: '--default-authentication-plugin=mysql_native_password'
  restart: always
  secrets:
    - db-password
  volumes:
    - db-data:/var/lib/mysql
  networks:
    - private
  environment:
    - MYSQL_DATABASE=example
    - MYSQL_ROOT_PASSWORD_FILE=/run/secrets/db-password
```

| Property | Detail |
|---|---|
| **Image** | `mariadb:10.6.4-focal` — pulled from Docker Hub at runtime |
| **Auth plugin** | Forces `mysql_native_password` for compatibility |
| **Secret** | Root password is read from `/run/secrets/db-password` (sourced from `db/password.txt`) — never stored in plain text in the YAML |
| **Volume** | `db-data` persists the database files across container restarts |
| **Network** | `private` only — the database is not reachable from the public network |
| **Restart** | `always` — restarts automatically even after a system reboot |

The `db` service starts first because `backend` declares `depends_on: db`.

---

### Service: `backend`

```yaml
backend:
  build:
    args:
      - NODE_ENV=development
    context: backend
  command: npm run start-watch
  environment:
    - DATABASE_DB=example
    - DATABASE_USER=root
    - DATABASE_PASSWORD=/run/secrets/db-password
    - DATABASE_HOST=db
    - NODE_ENV=development
  ports:
    - 3001:80
    - 9229:9229
    - 9230:9230
  secrets:
    - db-password
  volumes:
    - ./backend/src:/code/src:ro
    - ./backend/package.json:/code/package.json
    - ./backend/package-lock.json:/code/package-lock.json
    - backend-modules:/opt/app/node_modules
  networks:
    - public
    - private
  depends_on:
    - db
  restart: unless-stopped
```

| Property | Detail |
|---|---|
| **Build** | Built from `./backend/Dockerfile` with `NODE_ENV=development` as a build arg |
| **Command** | `npm run start-watch` — starts the Node.js API with file watching (hot reload) |
| **Ports** | `3001:80` exposes the API; `9229–9230` are Node.js debugger ports |
| **Secret** | Same `db-password` secret used to authenticate with the database |
| **Volumes** | Source code is bind-mounted read-only (`ro`) so changes on the host reflect immediately; `node_modules` is stored in a named volume to avoid OS conflicts |
| **Networks** | Bridges both `public` (reachable by frontend) and `private` (reaches the db) |
| **Restart** | `unless-stopped` — restarts on failure but not if manually stopped |

The backend is the only service that sits on both networks, acting as the bridge between the
frontend and the database.

---

### Service: `frontend`

```yaml
frontend:
  build:
    context: frontend
    target: development
  ports:
    - 3000:3000
  volumes:
    - ./frontend/src:/code/src
    - /code/node_modules
  networks:
    - public
  depends_on:
    - backend
  restart: unless-stopped
```

| Property | Detail |
|---|---|
| **Build** | Built from `./frontend/Dockerfile` using the `development` multi-stage target |
| **Port** | `3000:3000` — the React dev server, accessible at http://localhost:3000 |
| **Volumes** | `./frontend/src` is bind-mounted for live reloading; `node_modules` is an anonymous volume to isolate container packages from the host |
| **Network** | `public` only — talks to the backend but cannot reach the database directly |
| **Restart** | `unless-stopped` |

---

### Networks

```yaml
networks:
  public:
  private:
```

Two custom bridge networks provide **network segmentation**:

- **`public`** — shared by `frontend` and `backend`. The frontend reaches the API through this network.
- **`private`** — shared by `backend` and `db`. The database is completely isolated from the frontend.

This means the database is never directly accessible from the frontend container, which is a
security best practice.

---

### Volumes

```yaml
volumes:
  backend-modules:
  db-data:
```

- **`backend-modules`** — stores the backend's `node_modules` inside Docker. This prevents the host OS's file system (Windows) from interfering with native Node.js binaries compiled for Linux.
- **`db-data`** — persists all MariaDB data files. Containers can be stopped and restarted without losing the database contents.

---

### Secrets

```yaml
secrets:
  db-password:
    file: db/password.txt
```

The database password is loaded from `db/password.txt` at runtime and injected into containers
as a file at `/run/secrets/db-password`. This avoids hardcoding credentials in the YAML or
environment variables.

---

## Lab Steps

### Step 1 — Build the images

```
docker compose build
```

Docker Compose built two custom images:

- `docker-compose-sample-app-backend` — from `./backend/Dockerfile`
- `docker-compose-sample-app-frontend` — from `./frontend/Dockerfile` (target: `development`)

Both images are based on `node:lts`. The initial pull of the base image took ~956s (~16 min) due
to its size (~360 MB compressed). Total build time: **1231 seconds**.

> **Note:** A warning was shown on every command because both `compose.yaml` and `docker-compose.yml`
> exist in the directory. Compose automatically used `compose.yaml` as the preferred file.

---

### Step 2 — Start the app

```
docker compose up -d
```

Docker pulled the `mariadb:10.6.4-focal` image (10 layers, ~198s), then created:

| Resource | Name |
|---|---|
| Network | `docker-compose-sample-app_public` |
| Network | `docker-compose-sample-app_private` |
| Volume | `docker-compose-sample-app_backend-modules` |
| Volume | `docker-compose-sample-app_db-data` |
| Container | `docker-compose-sample-app-db-1` |
| Container | `docker-compose-sample-app-backend-1` |
| Container | `docker-compose-sample-app-frontend-1` |

All three containers started successfully in detached mode.

---

### Step 3 — List running containers

```
docker compose ps
```

Output:

| Container | Image | Ports | Status |
|---|---|---|---|
| `docker-compose-sample-app-backend-1` | `docker-compose-sample-app-backend` | `0.0.0.0:3001->80/tcp`, `9229-9230->9229-9230/tcp` | Up (health: starting) |
| `docker-compose-sample-app-db-1` | `mariadb:10.6.4-focal` | `3306/tcp` (internal only) | Up |
| `docker-compose-sample-app-frontend-1` | `docker-compose-sample-app-frontend` | `0.0.0.0:3000->3000/tcp` | Up |

The app was accessible at **http://localhost:3000**.

The backend health check was still initializing (`health: starting`) at the time of the snapshot —
this is normal for a Node.js app that needs a few seconds to connect to the database.

---

### Step 4 — Stop and remove the app

```
docker compose down
```

Compose stopped and removed all three containers and both networks in ~4 seconds:

```
✔ Container docker-compose-sample-app-frontend-1  Removed
✔ Container docker-compose-sample-app-backend-1   Removed
✔ Container docker-compose-sample-app-db-1        Removed
✔ Network docker-compose-sample-app_private       Removed
✔ Network docker-compose-sample-app_public        Removed
```

> The **volumes** (`db-data` and `backend-modules`) were **not** removed by `docker compose down`.
> To also delete volumes, use `docker compose down -v`.

### Step 5 — Verify cleanup

```
docker ps
```

Returned an empty list — all containers were successfully removed.

---

## Architecture Summary

```
Browser
  │
  └─► frontend (port 3000) ──[public network]──► backend (port 80/3001)
                                                       │
                                              [private network]
                                                       │
                                                    db (port 3306)
```

The frontend never communicates with the database directly. All data access goes through the
backend API, which is the only service with access to both networks.
