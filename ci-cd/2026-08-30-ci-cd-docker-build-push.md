# 2026-08-30 — CI/CD: Building and Pushing Docker Images in GitHub Actions

**Topic:** Adding a Docker build and registry push to the pipeline
**Why:** Follow-up from the first workflow notes — tests passing is only half a pipeline

---

## 1. What this adds

The earlier workflow ran tests. This one:

1. Runs tests
2. Builds a Docker image
3. Logs into Docker Hub
4. Pushes the image with a tag

That turns CI into something that actually produces a deployable artifact.

---

## 2. The workflow

```yaml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # Sets up Buildx, the extended docker build engine
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Authenticate against Docker Hub using repository secrets
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          # Build context - the folder containing the Dockerfile
          context: .
          push: true
          # Two tags: one moving, one pinned to this exact commit
          tags: |
            javeriadevops/resume-chatbot:latest
            javeriadevops/resume-chatbot:${{ github.sha }}
```

---

## 3. Never use a password for the registry

`DOCKERHUB_TOKEN` should be an **access token**, not the account password.

Create it at: Docker Hub → Account Settings → Personal access tokens → Generate.

Reasons: it can be scoped to read/write only, revoked on its own without changing your password, and it works when 2FA is enabled.

Both values go in the repo under Settings → Secrets and variables → Actions.

---

## 4. Why two tags

```
:latest          moves to whatever was built most recently
:<commit-sha>    permanently points at one specific build
```

`:latest` is convenient but ambiguous — it tells you nothing about what is actually inside. The SHA tag means you can identify exactly which commit produced a running image, and roll back to a known one.

Deploying `:latest` in production is a common way to lose track of what is actually running.

---

## 5. Useful context variables

GitHub injects a `github` context into every workflow:

| Variable | Contains |
|---|---|
| `${{ github.sha }}` | Full commit SHA |
| `${{ github.ref_name }}` | Branch or tag name |
| `${{ github.actor }}` | Who triggered the run |
| `${{ github.repository }}` | `owner/repo` |
| `${{ github.run_number }}` | Incrementing build number |

---

## 6. Only push on main

Building on every pull request is useful. Pushing an image on every pull request is not — the registry fills with junk.

A conditional handles it:

```yaml
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
```

On a PR the image builds (so a broken Dockerfile still fails the check) but nothing is pushed.

---

## 7. Caching

Docker builds in CI start with an empty cache every time, so every layer rebuilds. Buildx can cache into GitHub's cache store:

```yaml
        with:
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

This makes a real difference on repeated builds, especially when dependency installation is slow.

---

## 8. Things that went wrong / to watch for

**`denied: requested access to the resource is denied`** — the image name must start with your actual Docker Hub username. `resume-chatbot:latest` alone will not push; it needs `username/resume-chatbot:latest`.

**Secret shows as empty** — secret names are case sensitive and must match exactly. Also, secrets are not available to workflows triggered by pull requests from forks.

**Workflow did not trigger** — the file has to be on `main` for a `push: branches: [main]` trigger to apply.

---

## 9. For tomorrow

- [ ] Add a step that scans the image for vulnerabilities (Trivy)
- [ ] Look into GitHub Container Registry as an alternative to Docker Hub
- [ ] Understand how deployment would follow this — pull the image on a server
