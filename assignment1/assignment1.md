# CS-548: Cloud-native Software Architectures — Assignment 1: Docker

**Student:** Nikos Kanakousakis (csd5133@csd.uoc.gr)

---

## Exercise 1 — Nginx

### 1.1 — Pull the images

```bash
docker pull nginx:1.29.5
docker pull nginx:1.29.5-alpine
```

### 1.2 — Compare sizes

```bash
docker images nginx
```

```
IMAGE                 ID             DISK USAGE   CONTENT SIZE
nginx:1.29.5          0236ee02dcbc        240MB         65.8MB
nginx:1.29.5-alpine   1d13701a5f9f       93.4MB         26.9MB
```

The `alpine` image is ~2.5× smaller because Alpine Linux is a minimal distribution, whereas the standard image is Debian-based and includes many more system libraries.

### 1.3 — Start nginx in the background with port forwarding

```bash
docker run -d -p 8000:80 --name mynginx nginx:1.29.5
curl http://localhost:8000
```

`-d` runs the container in the background, `-p 8000:80` forwards port 8000 on the host to port 80 inside the container. The response is the default nginx welcome page:

```html
<h1>Welcome to nginx!</h1>
```

### 1.4 — Confirm the container is running

```bash
docker ps
```

```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                                     NAMES
fa780e48c034   nginx:1.29.5   "/docker-entrypoint.…"   41 seconds ago   Up 40 seconds   0.0.0.0:8000->80/tcp, [::]:8000->80/tcp   mynginx
```

### 1.5 — Get the logs

```bash
docker logs mynginx
```

The logs show nginx startup messages and one GET request entry from the curl command:

```
172.17.0.1 - - [28/Feb/2026:15:57:21 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0" "-"
```

### 1.6 — Stop the container

```bash
docker stop mynginx
```

### 1.7 — Start the stopped container

```bash
docker start mynginx
```

No flags needed — the port forwarding and other settings are remembered from when the container was created with `docker run`.

### 1.8 — Stop and remove the container

```bash
docker stop mynginx
docker rm mynginx
```

`stop` halts the container gracefully, `rm` deletes it. The image (`nginx:1.29.5`) remains on disk.

---

## Exercise 2 — Modifying the Nginx Container

Started the container again with:

```bash
docker run -d -p 8000:80 --name mynginx nginx:1.29.5
```

### 2.1 — Open a shell inside the container and change the default page

Opened an interactive shell with `docker exec -it`, modified the page with `sed`, then exited:

```bash
docker exec -it mynginx bash
sed -i 's/Welcome to nginx!/Welcome to MY nginx!/' /usr/share/nginx/html/index.html
exit
```

Validated from the host:

```bash
curl http://localhost:8000 | grep h1
# <h1>Welcome to MY nginx!</h1>
```

### 2.2 — Download the default page and upload a new one from the host

Downloaded the current page from the container:

```bash
docker cp mynginx:/usr/share/nginx/html/index.html ./index.html
```

Created a new page and uploaded it into the container:

```bash
cat > ./mypage.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>My Custom Page</title></head>
<body><h1>Hello from the host machine!</h1></body>
</html>
EOF

docker cp ./mypage.html mynginx:/usr/share/nginx/html/index.html
```

Validated:

```bash
curl http://localhost:8000 | grep h1
# <h1>Hello from the host machine!</h1>
```

### 2.3 — Remove the container and start a new one

```bash
docker rm -f mynginx
docker run -d -p 8000:80 --name mynginx nginx:1.29.5
curl http://localhost:8000 | grep h1
# <h1>Welcome to nginx!</h1>
```

**The changes do not persist.** All modifications made via `docker exec` or `docker cp` are written only to the container's writable layer, which is deleted with `docker rm`. The image itself is read-only and never modified — a new container always starts from a clean copy of the image.

---

## Exercise 3 — Serve a Custom Page via Bind Mount

Created a local folder with a custom HTML file:

```bash
mkdir -p ~/mysite
cat > ~/mysite/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>My Nginx Site</title></head>
<body>
  <h1>Hello from my local folder!</h1>
  <p>Served via a bind mount.</p>
</body>
</html>
EOF
```

Started nginx with the bind mount using `-v`:

```bash
docker run -d -p 8000:80 --name mynginx \
  -v ~/mysite:/usr/share/nginx/html:ro \
  nginx:1.29.5
```

`-v ~/mysite:/usr/share/nginx/html:ro` mounts the local folder into the container at the path nginx reads from. `:ro` means read-only.

Validated:

```bash
curl http://localhost:8000 | grep h1
# <h1>Hello from my local folder!</h1>
```

---

## Exercise 4 — Extended Django Application

### Dockerfile

The original Dockerfile from the course repo was extended to add `vim-tiny`, a persistent volume for the database, and cache-friendly layer ordering (dependencies copied and installed before application code):

```dockerfile
FROM python:3.13.2-slim-bookworm

ENV PYTHONUNBUFFERED=1

# Install vim-tiny — rarely changes, placed early for caching
RUN apt-get update && \
    apt-get install -y --no-install-recommends vim-tiny && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy only requirements first so pip layer is cached independently of app code
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code last — changes most often
COPY . .

# Create DB directory and declare as volume for persistent storage
RUN mkdir -p /app/db
VOLUME /app/db

CMD ["./start.sh"]
```

### 4.1 — Build the image

```bash
docker build -t nikoskan/hy548-django:latest .
```

Size comparison:

```
nikoskan/hy548-django:latest   259MB
```

The base image `python:3.13.2-slim-bookworm` is ~45MB. Our image is 259MB — the extra ~214MB comes from `vim-tiny` (~5MB), Django and its Python dependencies (~170MB), and the application source code. We used `slim-bookworm` to keep the size down; the full `bookworm` image would have been ~1.55GB.

### 4.2 — Start with debug on and debug off

```bash
docker run -d -p 8001:8000 \
  --name django_debug \
  -v django_db_debug:/app/db \
  nikoskan/hy548-django:latest

docker run -d -p 8002:8000 \
  --name django_nodebug \
  -e DJANGO_DEBUG=0 \
  -v django_db_prod:/app/db \
  nikoskan/hy548-django:latest
```

Confirmed both running:

```
CONTAINER ID   IMAGE                          PORTS                     NAMES
6a1b9c691c62   nikoskan/hy548-django:latest   0.0.0.0:8002->8000/tcp    django_nodebug
92ed3727a2f7   nikoskan/hy548-django:latest   0.0.0.0:8001->8000/tcp    django_debug
```

Validated responses:

```bash
curl http://localhost:8001
# Hello world! (DEBUG is on)

curl http://localhost:8002
# Hello world!
```

With `DJANGO_DEBUG=1` (default) the app appends "(DEBUG is on)" to the response. With `DJANGO_DEBUG=0` it serves a plain response. In a real application, debug mode also exposes detailed error pages with full stack traces, while production mode shows only a generic error page — important for security.

### 4.3 — Push to Docker Hub

```bash
docker login
docker push nikoskan/hy548-django:latest
```

```
latest: digest: sha256:90136747ae78d004975880de3229b24953bf28913a8d7f716c4bffacbedeca2a size: 856
```

Image available at: https://hub.docker.com/r/nikoskan/hy548-django

---

## Exercise 5 — GitHub Actions Workflow

Created `.github/workflows/docker-build-push.yml`. The workflow is triggered manually via `workflow_dispatch` and builds and pushes the Django image to Docker Hub.

```yaml
name: Build and Push Docker Image

on:
  workflow_dispatch:

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./django
          file: ./django/Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/hy548-django:latest
```

Two repository secrets are configured in GitHub repo settings: `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`.

The workflow was triggered manually and completed successfully in 26 seconds.
