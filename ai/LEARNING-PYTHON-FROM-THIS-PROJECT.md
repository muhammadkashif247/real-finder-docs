# Learning Python from realfinder-ai (for JS Developers)

A walkthrough of the **realfinder-ai** codebase that maps Python/FastAPI concepts to JavaScript equivalents. Assumes ~4 years of JS experience.

---

## 1. Project Structure: Think of it like a Node/Express app

```
realfinder-ai/                   # Like your Node.js project root
├── app/                         # Like "src/"
│   ├── main.py                  # Like "app.ts" (Express app factory)
│   ├── config.py                # Like "config.ts" (env vars)
│   ├── dependencies.py          # Like Express middleware functions
│   ├── api/v1/                  # Like "routes/"
│   ├── core/                    # Like "utils/" or "lib/"
│   ├── schemas/                 # Like Zod schemas or DTO classes
│   └── services/                # Like "services/"
├── tests/                       # Like "__tests__/"
├── requirements.txt             # Like "package.json" dependencies
└── .env                         # Same as JS
```

---

## 2. Variables, Types & Syntax — The Quick Wins

| JS | Python (from this project) | File |
|---|---|---|
| `const x = 5` | `x = 5` (no keyword needed) | everywhere |
| `let score = 1.0` | `score = RULES_BASE_SCORE` | `text_analyzer.py` |
| `x: number` (TS) | `x: int` or `x: float` | `config.py` |
| `x: string \| null` | `x: str \| None` | `verification.py` |
| `// comment` | `# comment` | everywhere |
| `"""docstring"""` | no JS equivalent (JSDoc is closest) | every file |
| `true / false / null` | `True / False / None` | throughout |
| `` template `literal` `` | `f"string {var}"` | `text_analyzer.py:98` |
| `===` | `==` (Python only has `==`, and it's strict for types) | `dependencies.py:29` |
| `!==` | `!=` | — |
| `&&` / `\|\|` | `and` / `or` | `text_analyzer.py:90` |
| `!value` | `not value` | `gemini_client.py:100` |

---

## 3. `config.py` — Environment Variables

In JS you'd do something like:

```javascript
// JS equivalent
const config = {
  appName: process.env.APP_NAME || 'realfinder-ai',
  port: parseInt(process.env.PORT) || 8000,
  geminiApiKey: process.env.GEMINI_API_KEY || '',
};
```

Python uses **Pydantic BaseSettings**: a class that auto-reads from `.env` and environment variables. Think of it like a Zod schema that auto-parses `process.env`.

Key concepts:

- **`class Settings(BaseSettings)`** — Python uses classes for structured data.
- **Type hints** — `port: int = 8000` means "this is an integer, default 8000". Like TypeScript but built into the language.
- **`@lru_cache`** — A **decorator** (like JS decorators in NestJS). It means "call this function once, cache the result forever." Like wrapping a function in a memoize.
- **`def` vs `function`** — Python uses `def` instead of `function`. No curly braces — indentation IS the block.

---

## 4. `main.py` — App Factory (Like creating an Express app)

```javascript
// JS equivalent of create_app():
// const app = express();
// app.use(cors());
// app.use('/api/v1', v1Router);
// return app;
```

Key differences from Express:

- **`-> FastAPI`** after the function means "this function returns a FastAPI instance" (like TypeScript return types).
- **No semicolons** — Python doesn't use them.
- **`lifespan`** — Like `app.on('listening', ...)` in Express. Runs code on startup/shutdown.

### The `lifespan` — Startup/Shutdown hooks

The `yield` keyword splits the function: everything **before** it runs on startup, everything **after** runs on shutdown. Think of it like a generator function.

```javascript
// JS equivalent
app.on('listening', async () => { /* startup */ });
process.on('SIGTERM', async () => { /* shutdown */ });
```

---

## 5. `dependencies.py` — Like Express Middleware

FastAPI uses **dependency injection** instead of global middleware. You inject dependencies into route handlers.

**JS equivalent:**

```javascript
// Express middleware
function verifyServiceKey(req, res, next) {
  const key = req.headers['x-service-key'];
  if (key !== process.env.SERVICE_API_KEY) {
    return res.status(401).json({ detail: 'Invalid service key' });
  }
  next();
}
```

Key Python concepts:

- **`Header(...)`** — FastAPI automatically extracts the `X-Service-Key` header. The `...` (Ellipsis) means "required".
- **`Depends(get_settings)`** — Dependency injection. FastAPI calls `get_settings()` for you and passes the result. Think NestJS `@Inject()`.
- **`raise`** — Like `throw` in JS. `raise HTTPException(...)` = `throw new HttpException(...)`.
- **`-> None`** — This function returns nothing (like `: void` in TypeScript).

---

## 6. `schemas/verification.py` — Like Zod Schemas or DTOs

In JS you might write:

```javascript
// Zod
const TextAnalysisRequest = z.object({
  listing_id: z.string(),
  title: z.string().min(1),
  price: z.number().gte(0),
  currency: z.string().default('AED'),
});
```

Python uses **Pydantic BaseModel** with `Field()` for validation and defaults.

**Type mapping cheat sheet:**

| TypeScript | Python |
|---|---|
| `string` | `str` |
| `number` | `int` or `float` |
| `boolean` | `bool` |
| `string[]` | `list[str]` |
| `Record<string, string>` | `dict[str, str]` |
| `string \| null` | `str \| None` |
| `Partial<T>` | fields with `= None` |

---

## 7. `api/v1/analyze.py` — Route Handlers (Like Express Routes)

What FastAPI does automatically that you'd do manually in Express:

- **Request validation** — `body: TextAnalysisRequest` auto-validates the JSON body.
- **Dependency injection** — `client: GeminiClient = Depends(get_gemini_client)` auto-provides the client.
- **Response serialization** — Just `return` the object; FastAPI converts to JSON.
- **API docs** — Auto-generates Swagger at `/docs`.

---

## 8. `core/exceptions.py` — Error Handling (Like custom Error classes)

Almost identical to JS! Key differences:

- **`def __init__(self, ...)`** — Like JS `constructor(...)`.
- **`self`** — Like JS `this`, but you must pass it explicitly as the first parameter.
- **`super().__init__(...)`** — Like JS `super(...)`.

---

## 9. `services/text_analyzer.py` — Core Python Patterns

### Lists, Comprehensions & Counters

```python
# JS: const letters = [...text].filter(c => /[a-zA-Z]/.test(c));
letters = [c for c in text if c.isalpha()]   # List comprehension

# JS: const upperRatio = letters.filter(c => c === c.toUpperCase()).length / letters.length;
upper_ratio = sum(1 for c in letters if c.isupper()) / len(letters)

# JS: const words = text.toLowerCase().match(/\b[a-zA-Z]{2,}\b/g);
words = re.findall(r"\b[a-zA-Z]{2,}\b", text.lower())

# JS: lodash.countBy(words) — Python has this built in!
counts = Counter(words)   # Counter({'dubai': 5, 'bedroom': 3, ...})
```

**List comprehensions** replace `.filter().map()` in one line: `[x for x in items if condition]`.

### Tuple Returns (No JS equivalent)

Python can return multiple values as a **tuple** and destructure them:

```python
def _run_rules(req) -> tuple[float, list[RuleResult]]:
    # ...
    return rules_score, triggered

# Destructuring (like JS: const [score, rules] = runRules(req))
rules_score, rules_triggered = _run_rules(req)
```

In JS you'd return `{ score, triggered }` as an object.

### Private Functions (Convention)

- Leading `_` = "private" (convention only, not enforced).
- Like naming something `_internal` in JS.

---

## 10. `services/gemini_client.py` — Async/Await (Almost identical to JS)

- **`async def`** = `async function`
- **`await`** = `await`
- Python uses `asyncio` instead of Node's built-in event loop.

### Try/Except (= Try/Catch)

```python
try:
    response = await self._http.get(url)
    response.raise_for_status()
except httpx.HTTPError as exc:            # "as exc" = JS "catch (exc)"
    raise GeminiError(f"Failed: {exc}") from exc   # "from exc" chains errors
```

Python's `except` can match specific exception types. JS has one generic `catch`.

---

## 11. Tests — `pytest` (Like Jest)

- **`@pytest.fixture`** — Like `beforeEach` that returns a value.
- Test functions start with `test_`.
- **`assert x == y`** — Like `expect(x).toBe(y)`.
- Fixtures are injected by parameter name (pytest magic).

---

## 12. Imports (The Biggest Difference)

```python
# Python                                    # JS equivalent
from fastapi import APIRouter, Depends      # import { Router, Depends } from 'fastapi'
from app.config import get_settings        # import { getSettings } from './config'
import logging                              # import * as logging from 'logging'
from collections import Counter             # stdlib — no npm equivalent
```

Python uses **dots** for package paths (`app.schemas.verification`), not slashes. The `__init__.py` file makes a directory a "package" (like `index.js` in a folder).

---

## Quick Reference Card

| Concept | JavaScript | Python (this project) |
|---|---|---|
| Package manager | `npm` / `yarn` | `pip` |
| Deps file | `package.json` | `requirements.txt` |
| Run dev server | `npm run dev` | `uvicorn app.main:app --reload` |
| Run tests | `npm test` / `jest` | `pytest` |
| Linter | `eslint` | `ruff` |
| Web framework | Express / NestJS | FastAPI |
| Validation | Zod / class-validator | Pydantic |
| HTTP client | axios / fetch | `httpx` |
| Env vars | `dotenv` + `process.env` | `pydantic-settings` + `.env` |
| `console.log()` | — | `logger.info()` via `logging` |

---

## Where to Start Exploring

Suggested order:

1. **`app/config.py`** — Simplest file; learn Pydantic BaseSettings.
2. **`app/api/v1/health.py`** — Simplest route; see how endpoints work.
3. **`app/dependencies.py`** — Learn the `Depends()` pattern (FastAPI's killer feature).
4. **`app/schemas/verification.py`** — Learn Pydantic models (like Zod).
5. **`app/api/v1/analyze.py`** — See how a protected route + DI + validation come together.
6. **`app/services/text_analyzer.py`** — Core Python: regex, list comprehensions, tuples, `Counter`.
7. **`app/services/gemini_client.py`** — Async patterns, retry logic, error handling.
8. **`tests/test_health.py`** — Simplest test; learn pytest basics.
9. **`app/main.py`** — App factory, lifespan, putting it all together.
