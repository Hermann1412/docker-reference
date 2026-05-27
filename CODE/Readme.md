# L06-02 — Containerizing an Express App & Publishing to Docker Hub

This lab walks through building a Docker image from a Node.js/Express application, troubleshooting common errors, and publishing versioned images to Docker Hub.

---

## Project Structure

```
CODE/
├── Dockerfile          # Container build instructions
├── app.js              # Express application entry point
├── package.json        # Node.js dependencies
├── bin/www             # HTTP server bootstrap
└── node_modules/       # Installed dependencies
```

---

## The Dockerfile

```dockerfile
FROM node:lts-alpine
ENV NODE_ENV=production
WORKDIR /usr/src/app
COPY ["package.json", "package-lock.json*", "npm-shrinkwrap.json*", "./"]
RUN npm install --production --silent && mv node_modules ../
COPY . .
EXPOSE 3000
RUN chown -R node /usr/src/app
USER node
CMD ["npm", "start"]
```

**What each line does:**

| Instruction | Purpose |
|---|---|
| `FROM node:lts-alpine` | Uses the lightweight Alpine-based Node.js LTS image as the base |
| `ENV NODE_ENV=production` | Sets the environment to production (disables dev tools, enables optimizations) |
| `WORKDIR /usr/src/app` | Sets the working directory inside the container |
| `COPY [package*.json, ...]` | Copies only dependency files first (enables Docker layer caching) |
| `RUN npm install --production --silent` | Installs only production dependencies; moves `node_modules` one level up to keep it outside the app directory |
| `COPY . .` | Copies the rest of the application source code |
| `EXPOSE 3000` | Documents that the app listens on port 3000 |
| `RUN chown -R node /usr/src/app` | Gives ownership of the app directory to the non-root `node` user |
| `USER node` | Switches to the non-root `node` user (security best practice) |
| `CMD ["npm", "start"]` | Default command to run the app when a container starts |

---

## Step-by-Step: What Was Done

### Step 1 — Navigate to the Project Directory

```powershell
cd ..
cd code
```

Changed directory from the parent folder into the `CODE` project folder containing the Express app and Dockerfile.

---

### Step 2 — First Build Attempt (Failed — Uppercase in Image Name)

```powershell
docker build -t YourUsername/express:v1 .
```

**Error:**
```
ERROR: failed to solve: invalid reference format: repository name must be lowercase
```

**Why it failed:** Docker image names (repository names) must be entirely lowercase. The capital letter in `YourUsername` violated Docker's naming rules.

---

### Step 3 — Second Build Attempt (Typo in Command)

```powershell
docker built -t yourusername/express:v1 .
```

**Error:**
```
unknown shorthand flag: 't' in -t
```

**Why it failed:** The command was `docker built` instead of `docker build`. Docker interpreted `built` as an unknown sub-command and rejected the `-t` flag.

---

### Step 4 — Successful Build (Correct Lowercase Name)

```powershell
docker build -t yourusername/express:v1 .
```

**Result:** Build succeeded in 5.6s (12/12 steps FINISHED). Most layers were served from cache since the base image had been pulled before. The final image was tagged as `docker.io/yourusername/express:v1`.

---

### Step 5 — First Push Attempt (Failed — Wrong Credentials)

```powershell
docker push yourusername/express:v1
```

**Error:**
```
denied: requested access to the resource is denied
```

**Why it failed:** The Docker Hub account used did not match the authenticated session. Docker Hub rejected the push because the credentials on file did not match the target namespace.

---

### Step 6 — Login with Cached Credentials (Still Wrong Account)

```powershell
docker login
```

**Result:** `Authenticating with existing credentials... Login Succeeded` — but this logged in with the previously saved credentials, which were not for the intended account. The push therefore still failed.

---

### Step 7 — System Inspection

```powershell
docker info
docker info | findstr Username
```

Ran `docker info` to inspect the Docker daemon state (34 containers, 19 images, overlay2 storage driver, WSL2 kernel). The `findstr Username` attempt was to check which user was logged in, but Docker Desktop on Windows does not include a `Username` field in `docker info` output — that field only appears in older Docker CLI versions or Linux environments.

---

### Step 8 — Failed Login (Wrong Username Attempted)

```powershell
docker login --username <wrong-username>
```

**Error (twice):**
```
Error response from daemon: Get "https://registry-1.docker.io/v2/": unauthorized: incorrect username or password
```

**Why it failed:** The Docker Hub account attempted either does not exist or the password entered was incorrect.

---

### Step 9 — Successful Login with Correct Account

```powershell
docker login --username <your-dockerhub-username>
```

**Result:** `Login Succeeded` — authenticated with the correct Docker Hub account.

---

### Step 10 — Rebuild Image with Correct Username

```powershell
docker build -t your-dockerhub-username/express:v1 .
```

**Result:** Build succeeded (12/12 FINISHED, all layers from cache). The image was tagged as `docker.io/your-dockerhub-username/express:v1`. Since the code had not changed, Docker reused all cached layers — making the build nearly instant.

---

### Step 11 — Push v1 to Docker Hub

```powershell
docker push your-dockerhub-username/express:v1
```

**Result:** All 9 layers pushed successfully.

```
v1: digest: sha256:426fe082db42c04b51523b6b7b87a4d4cd244ee83e079513abb1b8c1eb63514d size: 2202
```

The image is now publicly available on Docker Hub at `docker.io/your-dockerhub-username/express:v1`.

---

### Step 12 — Remove Local v1 Image

```powershell
docker rmi your-dockerhub-username/express:v1
```

Removed the local tag `your-dockerhub-username/express:v1`. The image data was deleted from the local machine (since no other tag pointed to the same digest at that moment). The remote copy on Docker Hub was unaffected.

---

### Step 13 — Pull v1 Back from Docker Hub

```powershell
docker pull your-dockerhub-username/express:v1
```

**Result:** Successfully pulled from Docker Hub, confirming the push in Step 11 worked correctly.

```
Status: Downloaded newer image for your-dockerhub-username/express:v1
```

---

### Step 14 — Build v2

```powershell
docker build -t your-dockerhub-username/express:v2 .
```

**Note:** A first attempt was made without the `.` context argument and failed:
```
ERROR: "docker buildx build" requires exactly 1 argument.
```
After adding `.`, the build succeeded. Because no source files were changed between v1 and v2, Docker reused every cached layer and produced an image with the **identical digest** as v1 (`sha256:426fe082...`). The only difference is the tag label.

---

### Step 15 — Push v2 to Docker Hub

```powershell
docker push your-dockerhub-username/express:v2
```

**Result:** All 9 layers already existed on Docker Hub (from the v1 push). Docker skipped re-uploading them and simply created a new tag pointing to the same manifest.

```
v2: digest: sha256:426fe082db42c04b51523b6b7b87a4d4cd244ee83e079513abb1b8c1eb63514d size: 2202
```

---

### Step 16 — Remove Both Local Images

```powershell
docker rmi your-dockerhub-username/express:v1
docker rmi your-dockerhub-username/express:v2
```

Both local tags were removed. Since both tags pointed to the same underlying image digest, removing v1 only removed its tag (the image data was kept because v2 still referenced it). Removing v2 then freed the actual image data from local storage.

---

## Key Concepts Demonstrated

| Concept | Observation |
|---|---|
| **Image naming rules** | Repository names must be fully lowercase |
| **Docker layer caching** | Unchanged layers are reused across builds, making rebuilds fast |
| **Image tags** | Multiple tags (v1, v2) can point to the same underlying image digest |
| **Docker Hub authentication** | You can only push to namespaces you are authenticated for |
| **`docker rmi` vs remote images** | Removing a local image does not delete it from Docker Hub |
| **Non-root container user** | The Dockerfile uses `USER node` — a security best practice |
| **Build context (`.`)** | The `.` at the end of `docker build` specifies the build context directory |

---

## Final State on Docker Hub

Both tags are available at:
- `docker.io/your-dockerhub-username/express:v1`
- `docker.io/your-dockerhub-username/express:v2`

Both resolve to the same image digest:
```
sha256:426fe082db42c04b51523b6b7b87a4d4cd244ee83e079513abb1b8c1eb63514d
```
