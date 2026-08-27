# 2026-08-28 — Docker Compose: Multi-Container Setup

**Topic:** Running an app and its database together with one command
**Why:** Follow-up from the Dockerfile notes — a single container is rarely enough

---

## 1. The problem it solves

Running an app plus a database with plain `docker run` means:

- Two long commands with flags you have to remember
- Manually creating a network so they can talk
- Starting them in the right order
- Repeating all of it on every machine

Compose puts all of that in one file, and `docker compose up` runs it.

---

## 2. A working `docker-compose.yml`

```yaml
services:
  # The application service
  app:
    # Build from the Dockerfile in the current directory
    build: .
    ports:
      - "8000:8000"
    environment:
      # Note the hostname is "db" - the service name, not localhost
      DATABASE_URL: postgresql://postgres:secret@db:5432/chatbot
    depends_on:
      - db
    # Mount local code into the container for live reload during development
    volumes:
      - .:/app

  # The database service
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: chatbot
    # Named volume so data survives container removal
    volumes:
      - pgdata:/var/lib/postgresql/data

# Declares the named volume used above
volumes:
  pgdata:
```

---

## 3. Services talk by service name, not localhost

This is the part that confused me most.

Inside the `app` container, `localhost` means **the app container itself**, not the host machine and not the database. To reach Postgres, the hostname is `db` — the service name from the compose file.

Compose creates a network automatically and registers each service name as a DNS entry on it. So `db:5432` resolves correctly.

---

## 4. `build` vs `image`

- **`build: .`** — build from a local Dockerfile. Used for code you wrote.
- **`image: postgres:16`** — pull a ready-made image from Docker Hub. Used for databases, caches, and other off-the-shelf software.

You never write a Dockerfile for Postgres. The official image already exists.

---

## 5. Volumes — two different kinds

```yaml
# Bind mount: maps a host folder into the container
volumes:
  - .:/app

# Named volume: Docker manages the storage location
volumes:
  - pgdata:/var/lib/postgresql/data
```

**Bind mount** is for development — edit code on your machine and the container sees it immediately, no rebuild.

**Named volume** is for data that must survive. Without it, removing the database container deletes the entire database.

---

## 6. Commands

```bash
# Build if needed and start everything
docker compose up

# Start in the background
docker compose up -d

# Force a rebuild of the images
docker compose up --build

# Stop and remove containers and networks
docker compose down

# Also delete the named volumes - this deletes your data
docker compose down -v

# Logs from all services
docker compose logs -f

# Logs from just one service
docker compose logs -f app

# Run a command inside a running service
docker compose exec app /bin/bash

# List the services and their state
docker compose ps
```

Note: it is `docker compose` (with a space) in current versions. The older `docker-compose` with a hyphen is the deprecated standalone tool.

---

## 7. `depends_on` does less than it sounds like

`depends_on: [db]` controls **start order only**. It waits for the db container to start, not for Postgres inside it to actually be ready to accept connections.

So the app can still crash on startup with a connection refused error, because the database process is still initialising.

Real fixes are a healthcheck or retry logic in the app. Something to look at properly later.

---

## 8. Secrets do not belong here

The passwords above are hardcoded, which is fine for a local scratch file but not for anything committed. Compose reads a `.env` file in the same folder automatically:

```yaml
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
```

And `.env` goes in `.gitignore`.

---

## 9. For tomorrow

- [ ] Add a healthcheck to the db service and see if it fixes the startup race
- [ ] Try `docker compose up --scale app=3`
- [ ] Read about override files for separate dev and prod configs
