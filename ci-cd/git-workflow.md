# Git Workflow, Secrets and Environment Config

Notes from building a FastAPI + React chatbot with a proper branch and PR workflow.

---

## 1. Keeping secrets out of Git

**The rule:** two files, one committed, one never.

| File | Contains | Committed? |
| ---- | -------- | ---------- |
| `.env` | The real API key | Never |
| `.env.example` | Placeholder values only | Yes |

`.env.example` documents which variables the project needs, without leaking their values.
Anyone cloning the repo copies it to `.env` and fills in their own key.

### GitHub push protection

GitHub scans pushes for known secret patterns and rejects the push if it finds one:

```
remote: - GITHUB PUSH PROTECTION
remote:   Push cannot contain secrets
remote:   —— Groq API Key ——
remote:    path: .env:1
```

The error message includes a link to "allow the secret" — **never click it**. That
whitelists the key and lets it become public. Fix the commit instead.

### The mistake I made

I created `.gitignore` *after* the first commit, so `.env` was already tracked.

**Important:** adding a file to `.gitignore` does not untrack it. Git ignores untracked
files only. Once a file is tracked, it stays tracked until you explicitly remove it.

**Lesson: create `.gitignore` before the first commit.**

### Cleaning history (only safe when nothing is pushed yet)

```powershell
Remove-Item -Recurse -Force .git   # PowerShell
# rm -rf .git                      # bash

git init
git branch -M main
git add .
git status                         # verify .env and venv are NOT listed
git commit -m "chore: initialize project structure"
```

If the commit is already on a remote, this is not enough — the key must be rotated and
history rewritten with a tool like `git filter-repo` or BFG.

**Either way: rotate the key.** Once a secret is written to disk in a commit, treat it
as compromised.

### A working `.gitignore`

```
.env
venv/
.venv/
__pycache__/
node_modules/
dist/
```

`venv/` and `.venv/` are different strings — list both.

---

## 2. Branch and PR workflow

Working directly on `main` is fine for a solo script, but every real team gates `main`
behind pull requests.

```bash
git checkout -b feat/react-frontend   # create branch and switch to it
# ... make changes ...
git add .
git commit -m "feat: replace static UI with React frontend"
git push -u origin feat/react-frontend
# open a Pull Request on GitHub, then merge
git checkout main
git pull
```

### Branch naming

| Prefix | Purpose | Example |
| ------ | ------- | ------- |
| `feat/` | New feature | `feat/react-frontend` |
| `fix/` | Bug fix | `fix/empty-message-crash` |
| `chore/` | Setup, config, deps | `chore/add-dockerfile` |
| `docs/` | Documentation | `docs/add-readme` |
| `ci/` | Pipeline changes | `ci/add-github-actions` |

### The mistake I made

I committed to `main` first and created the branch afterwards. Both branches then pointed
at the same commit, so there was nothing to open a PR against.

Fix — move `main` back to where the remote is, without touching the working files:

```bash
git branch -f main origin/main
```

**Lesson: branch first, then work.**

### Commit messages

Conventional Commits format — `type: what changed`, imperative mood:

```
feat: add chat endpoint with config, schemas and Groq client
chore: scaffold React frontend with Vite and Tailwind
fix: handle empty messages array with a 400
docs: add architecture section to README
```

Bad: `update`, `fix`, `changes`, `final`, `final2`.

### Commits not showing on the profile graph

GitHub attributes a commit by its author email. If the local email is not registered on
the account, the commit shows up in the repo but not on the contribution graph.

```bash
git log -1 --format="%an <%ae>"          # check what the last commit used
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

Only affects future commits.

---

## 3. Configuration belongs in the environment

Never hardcode a model name, URL, or key in application code.

```python
import os

API_KEY = os.getenv("GROQ_API_KEY")
MODEL = os.getenv("LLM_MODEL", "openai/gpt-oss-20b")
```

**Why this matters — a real example:** the model I coded against was retired by the
provider and started returning:

```json
{"error": {"message": "The model does not exist or you do not have access to it.",
           "code": "model_not_found"}}
```

Because the name came from an environment variable, the fix was one variable change and a
restart. No code edit, no rebuild, no redeploy.

Providers expose a `/models` endpoint that always reflects reality, even when the docs are
stale:

```bash
curl https://api.groq.com/openai/v1/models -H "Authorization: Bearer $GROQ_API_KEY"
```

Same principle in a container: config is injected at runtime via `env_file` or
`environment`, never baked into the image. One image, many environments.

---

## 4. Dev environment vs production

**Development** — two servers running at once:

| Process | Port | Role |
| ------- | ---- | ---- |
| `uvicorn app.main:app --reload` | 8000 | FastAPI backend |
| `npm run dev` | 5173 | Vite dev server, React |

The browser only ever opens 5173. Vite forwards API calls to the backend:

```javascript
server: {
  proxy: {
    "/api": "http://localhost:8000",
    "/health": "http://localhost:8000",
  },
}
```

This means the frontend code calls `/api/chat` — a relative path — and never hardcodes a
host. The same code works in dev and in production without changes. It also avoids CORS,
since to the browser everything comes from one origin.

**Production** — one process. `npm run build` compiles React into static files, and the
backend serves them. One port, one container.

### Why the API key stays server-side

The browser never talks to the LLM provider directly. Anything shipped to the browser is
readable by the user, so a key used in frontend code is a public key. The backend acts as
a proxy and holds the secret.

---

## Commands worth remembering

```bash
git checkout -b feat/thing        # create branch and switch
git branch                        # which branch am I on
git log --oneline                 # compact history
git log -1 --format="%ae"         # author email of last commit
git branch -f main origin/main    # reset main to the remote (no file changes)
git status                        # ALWAYS run before committing
```

---

## Takeaways

- `.gitignore` before the first commit, always.
- A leaked key is a rotated key. Removing the file is not enough.
- Branch first, then work.
- Config comes from the environment so it can change without a rebuild.
- Relative API paths in the frontend; the proxy handles the rest.
