# Security Reference

Security rules for Python code. Most of these are caught by `ruff S` (bandit)
rules: enable them. Apply defence-in-depth: validate at every boundary.

---

## Input Validation

### SQL injection: parameterised queries always

```python
# NEVER: f-string or format into SQL
cursor.execute(f"SELECT * FROM users WHERE name = '{user_input}'")      # CRITICAL BUG
cursor.execute("SELECT * FROM users WHERE name = '%s'" % user_input)    # CRITICAL BUG

# ALWAYS: parameterised
cursor.execute("SELECT * FROM users WHERE name = ?", (user_input,))     # SQLite
cursor.execute("SELECT * FROM users WHERE name = %s", (user_input,))    # psycopg2
db.execute(select(User).where(User.name == user_input))                  # SQLAlchemy ORM

# LiteralString (3.11+): type-level enforcement
from typing import LiteralString

def query(sql: LiteralString, params: tuple) -> list:
    """Type checker prevents f-string/user-string from being passed as sql."""
    return cursor.execute(sql, params).fetchall()
```

### t-strings: injection-safe interpolation (3.14+)

```python
# For SQL, HTML, shell — when a driver/renderer supports Template
from string.templatelib import Template

query = t"SELECT * FROM {table} WHERE id = {user_id}"
result = db.execute(query)  # driver extracts interpolations as params

html = t"<div>{user_content}</div>"
output = html_render(html)  # renderer escapes user_content
```

### Path traversal

```python
from pathlib import Path

BASE_DIR = Path("/var/data/uploads").resolve()

def get_file(filename: str) -> Path:
    # Resolve to absolute, then verify it's under BASE_DIR
    requested = (BASE_DIR / filename).resolve()
    if not requested.is_relative_to(BASE_DIR):
        raise PermissionError(f"Path {filename!r} is outside allowed directory")
    return requested
```

### HTML/XSS: always escape user content

```python
import html

# Escaping for HTML content
safe = html.escape(user_content)

# In Jinja2 — autoescaping must be ON (default for HTML templates)
from jinja2 import Environment
env = Environment(autoescape=True)   # safe default for HTML
```

---

## Secrets and Credentials

```python
import os
import secrets

# Reading secrets — fail fast on startup if missing
DATABASE_URL = os.environ["DATABASE_URL"]   # KeyError if not set — intentional
SECRET_KEY = os.environ["SECRET_KEY"]

# Generating cryptographically secure tokens
token = secrets.token_hex(32)            # 64-char hex string
csrf = secrets.token_urlsafe(16)         # 22-char URL-safe base64
api_key = secrets.token_bytes(32)        # raw bytes

# Timing-safe comparison (prevents timing attacks)
if not secrets.compare_digest(provided_token.encode(), expected_token.encode()):
    raise Unauthorized()

# NEVER use random for security purposes
import random
bad_token = random.randbytes(16)   # NOT cryptographically secure
```

### Never log secrets

```python
import logging
logger = logging.getLogger(__name__)

# GOOD: log what happened, not the secret
logger.info("User %s authenticated successfully", username)
logger.warning("Failed login for user %s from %s", username, ip_address)

# BAD
logger.debug("Auth with password=%s", password)    # logs credential
logger.info("Token: %s", token)                    # logs secret
```

### Environment variable loading

```python
# Use python-dotenv for local dev (NEVER in production)
# .env is gitignored; production uses real env vars / secrets manager
from dotenv import load_dotenv
load_dotenv()   # only call in __main__, not in library code

# For production secrets: AWS Secrets Manager, GCP Secret Manager, Vault
import boto3

def get_secret(name: str) -> str:
    client = boto3.client("secretsmanager")
    return client.get_secret_value(SecretId=name)["SecretString"]
```

---

## Subprocess Safety

```python
import subprocess
from pathlib import Path

# NEVER: shell=True with user-controlled input
subprocess.run(f"ls {user_path}", shell=True)              # command injection!
os.system(f"convert {filename} output.png")                # command injection!

# ALWAYS: pass as list, no shell interpolation
subprocess.run(["ls", user_path], check=True, capture_output=True, timeout=30)

# Validate paths before use
def convert_image(input_path: str, output_dir: Path) -> Path:
    safe_input = Path(input_path).resolve()
    if not safe_input.exists():
        raise FileNotFoundError(input_path)
    if safe_input.suffix not in {".jpg", ".png", ".webp"}:
        raise ValueError(f"Unsupported format: {safe_input.suffix}")
    output = (output_dir / safe_input.stem).with_suffix(".png")
    subprocess.run(
        ["convert", str(safe_input), str(output)],
        check=True,
        capture_output=True,
        timeout=60,
    )
    return output
```

---

## Avoiding `eval` and `exec`

```python
# NEVER: executes arbitrary user code
eval(user_expression)
exec(user_code)
exec(compile(user_source, "<string>", "exec"))

# Safe alternatives:
import ast
safe_value = ast.literal_eval(user_string)   # only parses literals (str, int, list, dict, etc.)

# For expressions — use a real parser (e.g., pyparsing, lark)
# For config loading — use tomllib, json.loads, yaml.safe_load (NOT yaml.load)
import yaml
config = yaml.safe_load(config_string)   # safe; yaml.load is NOT
import json
data = json.loads(json_string)           # already safe
```

---

## Cryptography

```python
# Use the `cryptography` package — never implement your own
from cryptography.fernet import Fernet
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC

# NEVER use MD5 or SHA1 for security (only for checksums where collision resistance is not needed)
import hashlib
insecure = hashlib.md5(data).hexdigest()       # NEVER for passwords, signatures

# Password hashing — use bcrypt or argon2
from argon2 import PasswordHasher
ph = PasswordHasher()
hash = ph.hash(password)
ph.verify(hash, password)   # raises VerifyMismatchError on failure
```

---

## SSL/TLS

```python
import ssl

# ALWAYS: use create_default_context() — validates certs, modern settings
ctx = ssl.create_default_context()

# 3.13+: stricter by default (VERIFY_X509_PARTIAL_CHAIN + VERIFY_X509_STRICT)
# Don't override unless you have a very specific reason

# NEVER: disable cert verification
ctx.check_hostname = False       # DANGEROUS
ctx.verify_mode = ssl.CERT_NONE  # DANGEROUS

# For httpx / aiohttp / requests:
import httpx
client = httpx.Client()                      # cert verification on by default
client = httpx.Client(verify=False)          # NEVER in production
```

---

## Dependency Security

```bash
# Audit for known CVEs
uv pip audit
pip-audit                     # installs with: pip install pip-audit

# In CI — fail the build on known vulnerabilities
uv pip audit --strict
```

```yaml
# .github/workflows/security.yml
- name: Audit dependencies
  run: uv pip audit --strict
```

---

## OWASP Top 10 Python Checklist

| Vulnerability | Mitigation in Python |
|---|---|
| Injection (SQL, XSS, shell) | Parameterised queries, `html.escape()`, `subprocess` list form |
| Broken auth | `secrets` module, `bcrypt`/`argon2`, constant-time comparison |
| Sensitive data exposure | Env vars for secrets, never log credentials, TLS everywhere |
| Security misconfiguration | `ssl.create_default_context()`, `yaml.safe_load()` |
| Vulnerable components | `uv pip audit` in CI |
| Broken access control | Validate ownership at every boundary, not just the entry point |
| Crypto failures | `cryptography` package; never homebrew crypto |
| SSRF | Allowlist outbound URLs; validate before `requests.get(user_url)` |
| Insecure deserialisation | Never `pickle.loads(untrusted)`, `yaml.load(untrusted)` |
| Insufficient logging | Log auth events, errors, anomalies: never the secrets themselves |
