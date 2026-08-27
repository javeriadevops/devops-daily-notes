# 2026-08-27 — GitHub Actions: First CI Workflow

**Topic:** How a GitHub Actions workflow is structured, and writing a basic CI pipeline
**Why:** Next step for the resume-chatbot project after containerizing it with Docker

---

## 1. Where the file goes

GitHub only looks in one place:

```
.github/workflows/ci.yml
```

The folder name and path are fixed. The filename can be anything ending in `.yml` or `.yaml`. You can have multiple workflow files in that folder and each one runs independently.

---

## 2. The hierarchy

Four levels, and they always nest in this order:

```
Workflow          the whole file
└── Job           runs on its own fresh virtual machine
    └── Step      one task inside a job
        └── Action or run command
```

Key point: **each job gets a completely fresh VM.** Two jobs do not share a filesystem. If job A installs something, job B does not have it. To pass data between jobs you need artifacts or outputs.

---

## 3. A basic workflow

```yaml
name: CI

# Events that trigger this workflow
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    # The VM image this job runs on
    runs-on: ubuntu-latest

    steps:
      # Pre-built action that clones the repo into the runner
      - name: Checkout code
        uses: actions/checkout@v4

      # Pre-built action that installs a specific Python version
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      # Shell commands run directly on the runner
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests
        run: pytest -v
```

---

## 4. `uses` vs `run` — the distinction I kept mixing up

- **`uses`** — pulls in a pre-built action that someone else wrote and published. `actions/checkout@v4` means: the `checkout` action from the `actions` org, version 4.
- **`run`** — executes a shell command directly on the runner, exactly as if you typed it in a terminal.

A step uses one or the other, never both.

---

## 5. The checkout step is not optional

By default the runner starts with an **empty** filesystem. The repository is not there. Without `actions/checkout`, any step referencing your files fails with "file not found."

This was not obvious to me — I assumed the code was already present because the workflow lives inside the repo.

---

## 6. `runs-on` options

| Value | What it is |
|---|---|
| `ubuntu-latest` | Most common, fastest, free tier friendly |
| `windows-latest` | Windows runner |
| `macos-latest` | macOS runner, consumes free minutes much faster |

For a Python or Docker project, `ubuntu-latest` is the default choice.

---

## 7. Secrets

Never put API keys in the YAML file — the file is committed to the repo and is public.

Add them in the repo: **Settings → Secrets and variables → Actions → New repository secret**

Then reference them:

```yaml
      - name: Run with API key
        env:
          GROQ_API_KEY: ${{ secrets.GROQ_API_KEY }}
        run: python main.py
```

GitHub masks secret values in the logs, so they show as `***` in the output.

---

## 8. Things to watch out for

**YAML is whitespace-sensitive.** Indentation must use spaces, never tabs. A single wrong indent breaks the whole file, and the error message is usually unhelpful.

**Workflows only run after the file is on the default branch.** Pushing a workflow file to a side branch does not trigger it for `push: branches: [main]`.

**Check the Actions tab after pushing.** A red X gives you the full log of every step, which is where the actual error is.

---

## 9. For tomorrow

- [ ] Add a step that builds the Docker image inside the pipeline
- [ ] Try a `matrix` strategy to test against multiple Python versions
- [ ] Look into caching pip dependencies to speed up the run
