# Building an LLM App: FastAPI Backend + React Frontend

Notes from building a chatbot end to end — API integration, service structure, and the
frontend toolchain.

---

## 1. How an LLM API actually works

Most providers (Groq, OpenRouter, Together, OpenAI) expose the same endpoint shape, so the
same client code works across all of them:

```
POST {base_url}/chat/completions
Authorization: Bearer {api_key}
```

Request body:

```json
{
  "model": "openai/gpt-oss-20b",
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "What is Docker?" }
  ]
}
```

The response is a large object; the reply is buried inside it:

```python
data["choices"][0]["message"]["content"]
```

The rest is metadata — `usage`, `finish_reason`, `total_tokens`. `total_tokens` is what
providers bill on, so it is worth logging in a real service.

### The model has no memory

This is the single most important thing to understand.

Each request is completely independent. The model does not remember the previous turn.
"The chatbot remembers our conversation" only means **the client resends the entire
conversation history every time**.

```json
"messages": [
  { "role": "user", "content": "My name is Javeria." },
  { "role": "assistant", "content": "Nice to meet you!" },
  { "role": "user", "content": "What is my name?" }
]
```

Drop the first two entries and the model cannot answer.

Consequence: history grows with every turn, and so does cost and prompt size. Production
apps cap it — keep the last N turns, or summarize older ones.

### Roles

| Role | Who writes it | Purpose |
| ---- | ------------- | ------- |
| `system` | The backend | Sets behaviour; the user never controls it |
| `user` | The person | Their message |
| `assistant` | The model | Its previous replies |

Keeping the `system` prompt server-side means a user cannot override the bot's persona by
sending their own system message.

---

## 2. Backend structure

One file per responsibility, instead of everything in `main.py`:

```
app/
├── config.py     # reads environment variables, nothing else
├── schemas.py    # request/response shapes (Pydantic)
├── llm.py        # talks to the provider, raises LLMError on failure
└── main.py       # routes only
```

Why it matters: swapping providers touches only `llm.py`. Changing the API contract
touches only `schemas.py`. Nothing else needs to be read or retested.

### Validation is free with Pydantic

```python
class Message(BaseModel):
    role: Literal["user", "assistant"]
    content: str


class ChatRequest(BaseModel):
    messages: list[Message]
```

FastAPI validates the body against this before the route function runs. A bad `role` or a
missing field returns `422` automatically — no manual `if` checks.

### async and await

```python
async with httpx.AsyncClient(timeout=30) as client:
    response = await client.post(url, headers=headers, json=payload)
```

The provider takes 1–3 seconds to answer. With synchronous code, the server blocks for
that whole time and other users queue behind it. `await` releases the worker to handle
other requests while waiting.

Rule of thumb: `await` goes wherever the code waits on the network or disk.

### Mapping failures to status codes

Never let an upstream failure surface as a stack trace.

```python
try:
    reply = await generate_reply(request.messages)
except LLMError as error:
    raise HTTPException(status_code=502, detail=str(error))
```

| Code | Meaning | When |
| ---- | ------- | ---- |
| `400` | Bad request | Empty messages array |
| `422` | Validation failed | Pydantic rejected the body |
| `502` | Bad gateway | The provider failed, not us |
| `500` | Server error | A real bug in our code |

`502` vs `500` matters operationally: `502` says "we are fine, the dependency is not."
That points on-call at the right system.

### The health endpoint

```python
@app.get("/health")
async def health():
    return {"status": "ok", "model": config.MODEL}
```

Small, but it is what Docker's `HEALTHCHECK`, Kubernetes liveness probes, load balancers,
and uptime monitors all call. Build it before you need it.

Keep it cheap — no database calls, no provider calls. A probe that hits an external API
turns a slow dependency into a restart loop.

---

## 3. Frontend toolchain

```powershell
npm create vite@latest frontend -- --template react
cd frontend
npm install
npm install tailwindcss @tailwindcss/vite
```

`vite.config.js`:

```javascript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    proxy: {
      "/api": "http://localhost:8000",
      "/health": "http://localhost:8000",
    },
  },
});
```

`src/index.css` — one line, nothing else:

```css
@import "tailwindcss";
```

### Problems hit, and what caused them

**`Cannot find package '@tailwindcss/vite'`**
The package was never installed, or `npm install` ran in the wrong directory. `npm`
installs into the folder you are standing in — always check with `pwd` first.

**Tailwind classes did nothing; everything was centred on a white background**
Vite's starter `index.css` was still there with `text-align: center` and a fixed-width
`#root`. It has to be deleted entirely, not appended to. When CSS "doesn't apply", it is
almost always something else overriding it — `F12` → Elements shows which rules won and
which were struck through.

**Config changes seemed to have no effect**
`vite.config.js` is read at startup. Hot reload does not cover it. Restart the dev server.

---

## 4. Two dev servers, one production process

**Development:**

| Process | Port |
| ------- | ---- |
| `uvicorn app.main:app --reload` | 8000 |
| `npm run dev` | 5173 |

The browser only opens 5173. The proxy forwards `/api` and `/health` to 8000. Because the
frontend uses relative paths, the same code works in production without an edit — and
there is no CORS setup, since the browser sees a single origin.

**Production:** `npm run build` produces static files in `dist/`, the backend serves them,
one process on one port.

---

## 5. Debugging approach that saved time

**Test one layer at a time.** Before wiring the provider into FastAPI, I called it from a
throwaway `test_groq.py`. When it failed, the cause was unambiguous — there was nothing
else in the path. Only once it returned `200` did the logic move into `llm.py`.

**Read the status code first.** It localizes the fault before you read a single line of
the message:

| Code | Where the problem is |
| ---- | -------------------- |
| `401` | Authentication — key wrong or not loaded |
| `404` | The resource does not exist — wrong path or a retired model |
| `422` | Our request body is malformed |
| `429` | Rate limited — back off |
| `502` | The dependency failed |

`404` on a model name that used to work meant it had been retired — confirmed by listing
`/v1/models`, which reflects reality even when the docs are stale.

**A `404` is not always a bug.** Opening `localhost:8000/` returned `404` because no route
was registered at `/`. The server was working exactly as written.

---

## Takeaways

- LLM APIs are stateless; conversation memory is the client resending history.
- Keep the system prompt and the API key server-side.
- One file per responsibility makes the provider swappable.
- Map upstream failures to `502` so alerts point at the right system.
- Ship a cheap `/health` endpoint from day one.
- Isolate a layer before integrating it — failures become unambiguous.
