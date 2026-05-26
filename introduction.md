# Docker CLI Introduction

A walkthrough of common Docker CLI commands demonstrated in this session.

---

## 1. Logging In

```powershell
docker login
```

Authenticates with Docker Hub using existing credentials. Required before pulling or pushing images.

---

## 2. Running a Container

```powershell
docker run -d -p 8080:80 --name webserver nginx
```

| Flag | Meaning |
|------|---------|
| `-d` | Run the container in detached (background) mode |
| `-p 8080:80` | Map port 8080 on the host to port 80 inside the container |
| `--name webserver` | Assign a custom name to the container |
| `nginx` | The image to use (pulled from Docker Hub if not found locally) |

> **Note:** If a container with the same name already exists, Docker will throw a conflict error. You must remove or rename the old container first.

---

## 3. Listing Containers

```powershell
docker ps        # Show only running containers
docker ps -a     # Show all containers (including stopped ones)
```

`docker ps -a` is useful for finding containers that have exited or crashed.

---

## 4. Removing a Container

```powershell
docker rm webserver
```

Deletes a stopped container by name or container ID. The container must be stopped before it can be removed.

---

## 5. Executing Commands Inside a Running Container

```powershell
docker container exec -it webserver bash
```

| Flag | Meaning |
|------|---------|
| `-i` | Keep STDIN open (interactive mode) |
| `-t` | Allocate a pseudo-TTY (terminal) |
| `bash` | The command to run inside the container |

This opens an interactive shell session inside the running `webserver` container. From there, standard Linux commands like `ls` can be used to explore the container filesystem.

```bash
# Inside the container
ls        # List root directory
ls bin    # List all binaries available in /bin
exit      # Exit back to the host shell
```

---

## 6. Stopping a Container

```powershell
docker stop webserver
```

Gracefully stops a running container by sending a SIGTERM signal. The container still exists in a stopped state after this command.

---

## 7. Listing Images

```powershell
docker images
```

Displays all Docker images stored locally, along with their repository, tag, image ID, creation date, and size.

---

## 8. Removing an Image

```powershell
docker rmi nginx
```

Deletes a local Docker image. All associated layers are removed. The image must not be in use by any container (running or stopped) before it can be deleted.

---

## Full Workflow Summary

```
docker login
  → docker run -d -p 8080:80 --name webserver nginx
  → docker ps / docker ps -a
  → docker container exec -it webserver bash
  → docker stop webserver
  → docker rm webserver
  → docker rmi nginx
```

This sequence covers the full lifecycle of a Docker container: **pull → run → inspect → exec → stop → remove**, and finally removing the image itself.
