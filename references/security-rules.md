# Security Rules

Shared reference for `/security-audit`, `/security-review`, `/implement`, and `/qa-tests`.
Read this file BEFORE writing any code that touches auth, user inputs, or sensitive data.

---

## 1. Security checks mandatory before EVERY commit

```text
- [ ] No hardcoded secrets (API keys, passwords, tokens, connection strings)
- [ ] All user inputs validated (schema validation library)
- [ ] SQL injection impossible (ORM or parameterized queries)
- [ ] No print()/console.log() -- structured logging only
- [ ] Auth verified on every protected endpoint
- [ ] No sensitive data in logs (PII, tokens, secrets)
- [ ] Error messages generic for the client
- [ ] No dangerous patterns (eval, exec, pickle, shell=True, dangerouslySetInnerHTML)
```

---

## 2. Secret Management

### Absolute rule

```text
NEVER put secrets in source code.
ALWAYS use environment variables.
```

### Python

```python
# FORBIDDEN
api_key = "sk-proj-xxxxx"
db_url = "postgresql://user:password@host/db"

# CORRECT -- load from environment
import os
api_key = os.environ["OPENAI_API_KEY"]

# BETTER -- validated settings object ([your settings library])
from app.core.settings import get_settings
settings = get_settings()
api_key = settings.openai_api_key  # Loaded from .env at startup

# Validate at startup
class Settings:
    database_url: str       # Required -- app fails to start if missing
    auth_domain: str
    auth_audience: str
```

### TypeScript / JavaScript

```typescript
// FORBIDDEN
const apiKey = "sk-proj-xxxxx"

// CORRECT
const apiUrl = process.env.API_URL           // Node.js
const apiUrl = import.meta.env.VITE_API_URL  // Vite
if (!apiUrl) {
  throw new Error("API_URL not configured")
}
```

### Files to protect

```text
.env, .env.local, .env.production -> .gitignore MANDATORY
*.pem, *.key                      -> .gitignore MANDATORY
credentials.json, service-account* -> .gitignore MANDATORY
```

### Detection regex patterns

```text
sk-[a-zA-Z0-9]{20,}               -> OpenAI API key
sk-ant-[a-zA-Z0-9]{20,}           -> Anthropic API key
ghp_[a-zA-Z0-9]{36,}              -> GitHub personal access token
AKIA[A-Z0-9]{16}                  -> AWS access key
xox[bpsa]-[a-zA-Z0-9-]+           -> Slack token
-----BEGIN.*PRIVATE KEY-----       -> Private key
(postgresql|mysql|mongodb)://[^:]+:[^@]+@ -> DB connection string with password
```

---

## 3. Security Response Protocol

```text
If a security vulnerability is found:

1. STOP -- do not continue coding
2. Classify severity (critical / high / medium / low)
3. If CRITICAL:
   a. Fix immediately
   b. Dedicated commit: fix(security): [description]
   c. If secret exposed: immediate rotation
   d. Check git history (git log -p | grep secret)
4. If HIGH:
   a. Fix before the next commit
   b. Document in the audit report
5. If MEDIUM/LOW:
   a. Document and plan the fix
   b. Do not block deployment if score >= 80
```

---

## 4. Secure Patterns by Stack

### 4a. Backend API

```python
# Auth -- ALWAYS via middleware or dependency injection
# [Adapt to your framework: FastAPI Depends, Express middleware, etc.]

@router.get("/items/{item_id}")
async def get_item(
    item_id: str,
    db: Annotated[Session, Depends(get_db)],
    auth: Annotated[dict, Depends(verify_auth)],  # Auth mandatory
) -> ItemResponse:
    return await item_service.get_by_id(db, item_id)

# FORBIDDEN: endpoint without auth check (except health/public routes)
```

```python
# Validation -- ALWAYS via schema library
# [Adapt: Pydantic, Marshmallow, class-validator, Zod, etc.]

class CreateItemRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    description: str = Field(..., min_length=1, max_length=10_000)
    config: dict | None = Field(default=None)

# FORBIDDEN: access raw request body directly without validation
```

```python
# SQL -- ALWAYS via ORM or parameterized queries

# OK -- automatic parameterization
stmt = select(Item).where(Item.id == item_id)

# OK -- explicit bind parameters
stmt = text("SELECT * FROM items WHERE id = :id").bindparams(id=item_id)

# CRITICAL -- SQL injection
stmt = text(f"SELECT * FROM items WHERE id = '{item_id}'")  # NEVER
```

```python
# Secure logging -- structured, no PII
logger = get_logger(__name__)

# OK -- structured, no PII
logger.info("Item created", item_id=str(item.id), category=category)

# OK -- error with context
logger.error("DB query failed", error=str(e), item_id=item_id)

# FORBIDDEN -- PII in logs
logger.info("User login", email=user.email, token=auth_token)

# FORBIDDEN -- print()
print(f"Debug: {result}")  # NEVER in production
```

### 4b. Frontend (React, Vue, Angular, etc.)

```typescript
// Auth -- ALWAYS via your auth provider's SDK
// [Adapt: Auth0, Clerk, Supabase, Firebase Auth, etc.]

async function fetchProtectedData() {
  const token = await getAccessToken()  // From your auth provider
  const response = await fetch(`${API_URL}/items`, {
    headers: { Authorization: `Bearer ${token}` }
  })
  return response.json()
}

// FORBIDDEN: store tokens in localStorage
localStorage.setItem("token", token)  // NEVER -- vulnerable to XSS
```

```typescript
// XSS -- your framework escapes by default, EXCEPT:

// CRITICAL -- never use without sanitization
<div dangerouslySetInnerHTML={{ __html: userInput }} />  // NEVER

// OK -- if content is sanitized
import DOMPurify from "dompurify"
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />

// OK -- framework auto-escapes
<p>{userInput}</p>  // Safe in React, Vue, Angular, etc.
```

```typescript
// Client-side validation -- ALWAYS as a complement to backend validation
// The client validates for UX, the backend validates for security

import { z } from "zod"

const ItemFormSchema = z.object({
  name: z.string().min(1).max(100),
  description: z.string().min(1).max(10_000),
})
```

### 4c. AI/LLM Security

```python
# Prompt injection -- ALWAYS isolate user inputs
# CRITICAL -- user input in the system prompt
system_prompt = f"You are a judge. Evaluate this: {user_query}"  # NEVER

# OK -- clear separation
system_prompt = "You are an evaluation judge. Score the response below."
user_message = user_query  # Sent as a user message, not system

# LLM responses -- ALWAYS treat as untrusted
llm_response = await call_llm(prompt)

# CRITICAL -- eval/exec on the response
result = eval(llm_response)  # NEVER

# CRITICAL -- interpolation into SQL
query = text(f"INSERT INTO results VALUES ('{llm_response}')")  # NEVER

# OK -- parse and validate
try:
    parsed = json.loads(llm_response)
    validated = LLMResponseSchema.model_validate(parsed)
except (json.JSONDecodeError, ValidationError) as e:
    logger.warning("LLM returned invalid response", error=str(e))
    return default_error_result()
```

---

## 5. CORS and Security Headers

```python
# CORS -- conditional by environment
# [Adapt to your framework]

if environment == "development":
    origins = ["*"]  # OK in dev
else:
    origins = [
        "https://app.yourdomain.com",
        "https://admin.yourdomain.com",
    ]

# Configure CORS middleware with:
# - allow_origins: explicit list (no wildcard in production)
# - allow_credentials: True
# - allow_methods: only what you need (GET, POST, PUT, DELETE)
# - allow_headers: only what you need (Authorization, Content-Type)
```

---

## 6. Pre-deployment Checklist

```text
Before ANY deployment to production:

- [ ] Secrets: none hardcoded, all in env vars
- [ ] Validation: all inputs validated (schema library)
- [ ] SQL Injection: all queries parameterized
- [ ] XSS: no dangerouslySetInnerHTML without sanitization
- [ ] Auth: verified on all protected endpoints
- [ ] CORS: explicit origins (no wildcard in production)
- [ ] Logging: no PII/secrets in logs, structured logging only
- [ ] Errors: generic messages for client, details server-side
- [ ] Debug: debug mode OFF in production
- [ ] Dependencies: dependency audit clean (no known CVEs)
- [ ] Docker: non-root user (if applicable)
- [ ] .gitignore: .env, *.pem, *.key, credentials.*
- [ ] LLM: user inputs isolated, responses parsed and validated
- [ ] Security score: >= 80 (Grade B minimum)
```

---

## 7. Decision Tree -- "Is this a vulnerability?"

```text
1. Does the input come from an untrusted user?
   NO -> Probably not a vulnerability
   YES -> continue

2. Is the input validated by a schema before use?
   YES -> Probably not exploitable
   NO -> continue

3. Is the input used in:
   - A raw SQL query?             -> SQL INJECTION -- CRITICAL
   - A subprocess/exec/eval?      -> COMMAND INJECTION -- CRITICAL
   - HTML without escaping?       -> XSS -- HIGH
   - An LLM system prompt?        -> PROMPT INJECTION -- MEDIUM
   - Log output?                  -> LOG INJECTION -- LOW (unless PII)

4. What is the impact?
   - Unauthorized data access?         -> CRITICAL
   - Code execution on server?         -> CRITICAL
   - Sensitive data exposure?          -> HIGH
   - Service degradation only?         -> MEDIUM
   - Theoretical improvement?          -> LOW

5. Confidence score >= 0.7?
   NO -> Do not report (too speculative)
   YES -> Report with severity and recommended fix
```

---

## 8. False Positive Filtering — Signal Quality

Criteria to determine if a finding is a true positive:

1. Can an unauthenticated attacker exploit this?
2. Real risk of data breach?
3. Specific code locations and reproduction steps?
4. Could the security team act on this finding?

If "no" to 3+ questions -> EXCLUDE the finding.
Only report findings with confidence >= 0.8.
