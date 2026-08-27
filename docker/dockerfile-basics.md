# 2026-08-27 — Docker: Image, Container and Dockerfile Basics

**Topic:** Writing a Dockerfile and containerizing a FastAPI app
**Why:** Started the Docker step for the resume-chatbot project

---

## 1. Three different things

| Thing | What it is |
|---|---|
| **Dockerfile** | A text file — a list of instructions describing how to build an image |
| **Image** | The result of the build — a read-only template (like an app installer) |
| **Container** | A running instance of an image (like the installed app actually running) |

One image can produce as many containers as you want.

---

## 2. Layer caching — the most important concept

Every instruction in a Dockerfile creates a **layer**, and Docker caches those layers.
If one layer changes, every layer **after** it is rebuilt.

That is why `requirements.txt` is copied first and the code afterwards:

- Code changes every day
- Dependencies change only occasionally
- With this order `pip install` does not re-run on every build → faster builds

---

## 3. Working Dockerfile for FastAPI

```dockerfile
# Base image - slim variant is smaller than the full python image
FROM python:3.11-slim

# Set working directory inside the container
WORKDIR /app

# Copy dependency file FIRST so pip install layer stays cached
COPY requirements.txt .

# --no-cache-dir keeps the image size down
RUN pip install --no-cache-dir -r requirements.txt

# Now copy the actual application code
COPY . .

# Documents which port the app listens on (does not publish it)
EXPOSE 8000

# Bind to 0.0.0.0 so the app is reachable from outside the container
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Things I noted:
- `--host 0.0.0.0` is required. `127.0.0.1` only listens inside the container, so nothing from outside can reach it.
- `EXPOSE` is documentation only — the port is actually published with the `-p` flag.

---

## 4. `.dockerignore`

This file works like `.gitignore` — it keeps files out of the build context, so the image is smaller and the build is faster:

```
__pycache__/
*.pyc
.git/
.env
venv/
.venv/
*.md
```

Ignoring `.env` matters for security — secrets should never end up baked into the image.

---

## 5. Commands used today

```bash
# Build an image and tag it
docker build -t resume-chatbot:v1 .

# Run a container, map host port 8000 to container port 8000
docker run -d -p 8000:8000 --name chatbot resume-chatbot:v1

# List running containers
docker ps

# List all containers, including stopped ones
docker ps -a

# View logs of a container
docker logs chatbot

# Follow logs live
docker logs -f chatbot

# Open a shell inside a running container
docker exec -it chatbot /bin/bash

# Stop and remove a container
docker stop chatbot
docker rm chatbot

# List images
docker images

# Remove an image
docker rmi resume-chatbot:v1

# Clean up unused containers, networks, images
docker system prune
```

---

## 6. Errors hit and how they were fixed

**Error:** `docker: Error response from daemon: port is already allocated`
**Fix:** An older container was still holding that port. Checked with `docker ps`, then ran `docker stop <name>`. Alternatively, change the host port: `-p 8001:8000`.

**Error:** The build succeeded but `localhost:8000` showed nothing in the browser
**Fix:** The `CMD` had host `127.0.0.1`. Changed it to `0.0.0.0` and it worked.

**Error:** `COPY failed: file not found in build context`
**Fix:** The last argument of `docker build` (`.`) is the build context. Any file being copied must live inside that folder — you cannot copy something from outside it with `../file`.

---

## 7. For tomorrow

- [ ] Try a multi-stage build to reduce image size
- [ ] Write a `docker-compose.yml` (app + one database service)
- [ ] Add a non-root user to the Dockerfile (security best practice)
