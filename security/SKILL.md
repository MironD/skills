---
name: security
description: Use when reviewing code for security vulnerabilities — authentication, authorization, injection, secrets, data protection, infrastructure and dependency risks.
---

# Code Security — Security Analysis Skill

## How to use this skill

This skill provides a structured framework for security analysis during code review. It can be run independently or as a component of the `code-review` skill. The severity system is built on four levels: 🔴 — exploitable vulnerability, blocks merge; 🟡 — potential threat that must be addressed; 🔵 — recommendation for defense-in-depth; ℹ️ — informational note with no security impact. When this skill runs together with code-review, these four levels map directly onto code-review's identical 🔴/🟡/🔵/ℹ️ scale (see §9 for exact calibration): a 🔴 here is a 🔴 there — Request Changes guaranteed, no exceptions. A 🟡 here follows code-review's normal 🟡 rule (Request Changes unless explained). 🔵 and ℹ️ findings are reported but never block merge on their own — do not escalate a defense-in-depth recommendation to 🔴 just because it's security-flavored.

**Code examples** in this skill are written in **Go**. When reviewing code in a different language or framework, mentally map each example to the equivalent pattern in the codebase's stack — the vulnerability class and the fix principle are identical; only the package names and syntax differ. For example: `database/sql` parameterized queries → `pg`/`psycopg2` in Node.js/Python / `PDO` in PHP; `bcrypt` → `argon2` in Node.js / `passlib` in Python; `golang-jwt` → `jsonwebtoken` in Node.js / `PyJWT` in Python; `golang.org/x/time/rate` → `express-rate-limit` in Node.js / `django-ratelimit` in Python.

---

## 1. OWASP Top 10

### A01: Broken Access Control (CWE-284, CWE-285, CWE-639)

**Impact:** an attacker gains unauthorized access to other users' resources and can modify or delete their data.

#### What to look for
- New endpoints without authorization middleware attached.
- IDOR — an object is loaded by ID from the request without filtering by the current user.
- File path construction via concatenation with user input (path traversal).
- Missing ownership check: one user can view or edit another user's data.
- Ability to escalate privileges (privilege escalation) by changing a role or directly calling an admin function.
- Forced browsing — access to protected URLs by entering the address directly.

#### Vulnerability example

```go
// UNSAFE: order is loaded without checking ownership
func getOrder(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    order, err := db.QueryRow("SELECT * FROM orders WHERE id = $1", id)
    // ...
}
```

#### Safe pattern

```go
// SAFE: scope to authenticated user
func getOrder(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    userID := r.Context().Value(ctxKeyUserID).(int64)
    order, err := db.QueryRow(
        "SELECT * FROM orders WHERE id = $1 AND user_id = $2",
        id, userID,
    )
    if err != nil {
        http.Error(w, "not found", http.StatusNotFound)
        return
    }
    // ...
}
```

---

### A02: Cryptographic Failures (CWE-326, CWE-327, CWE-330)

**Impact:** leakage of passwords, tokens, and personal data; violation of GDPR and PCI DSS requirements.

#### What to look for
- Secrets, API keys, and passwords written directly in code or configuration files.
- Using MD5 or SHA-1 for password hashing or signature generation.
- Storing personal data (PII) in plaintext without encryption.
- HTTP requests instead of HTTPS for communication with external services.
- ECB mode in symmetric encryption.
- Using `Math.random()` instead of `crypto.randomBytes()` for security-sensitive values.
- Sensitive data appearing in logs without masking.

#### Vulnerability example

```go
// UNSAFE: SHA-256 for passwords
h := sha256.Sum256([]byte(password))
hash := hex.EncodeToString(h[:])
```

#### Safe pattern

```go
// SAFE: bcrypt
import "golang.org/x/crypto/bcrypt"

hash, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
valid := bcrypt.CompareHashAndPassword(hash, []byte(password)) == nil
```

#### Vulnerability example — TLS

```go
// UNSAFE: TLS verification disabled
tr := &http.Transport{
    TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
}
client := &http.Client{Transport: tr}
```

#### Safe pattern — TLS

```go
// SAFE: default client (verification enabled)
client := &http.Client{}
resp, err := client.Get("https://api.example.com")
```

---

### A03: Injection (CWE-89, CWE-79, CWE-78)

**Impact:** arbitrary code execution on the server, reading or destroying the database, session theft via XSS.

#### What to look for
- User input embedded in SQL queries, shell commands, LDAP, or XPath expressions.
- Raw database queries without parameterization.
- Rendering user input in HTML without escaping.
- Calls to `eval()`, `new Function()`, `execSync()`, or `child_process.exec()` with user-supplied data.
- Template injection — user input enters server-side templates.
- CRLF injection — user input inserted into HTTP headers without filtering.

#### Vulnerability example — SQL injection

```go
// UNSAFE: string formatting in query
rows, _ := db.Query(fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email))
```

#### Safe pattern — SQL injection

```go
// SAFE: parameterized query
rows, err := db.Query("SELECT * FROM users WHERE email = $1", email)
```

#### Vulnerability example — OS command injection

```go
// UNSAFE: shell=true equivalent — passing user input to sh -c
import "os/exec"

cmd := exec.Command("sh", "-c", "convert "+userFilename+" output.png")
```

#### Safe pattern — OS command injection

```go
// SAFE: args as slice, no shell
import "os/exec"

cmd := exec.Command("convert", userFilename, "output.png")
err := cmd.Run()
```

#### Vulnerability example — XSS

```jsx
// UNSAFE: user content is inserted as raw HTML
function Comment({ content }) {
    return <div dangerouslySetInnerHTML={{ __html: content }} />;
}
```

#### Safe pattern — XSS

```jsx
// SAFE: JSX automatically escapes content
function Comment({ content }) {
    return <div>{content}</div>;
}
```

---

### A04: Insecure Design (CWE-73, CWE-284)

**Impact:** business logic abuse, DDoS due to missing rate limiting, bypassing restrictions through repeated requests.

#### What to look for
- Public endpoints without rate limiting.
- Business operations that can be replayed for gain.
- Unlimited authentication attempts, OTP entry, or password reset.
- TOCTOU (Time-of-check to time-of-use) in financial or inventory operations.
- Missing idempotency key for operations that should not be duplicated.

#### Vulnerability example — rate limiting

```go
// UNSAFE: no rate limiting
http.HandleFunc("/login", loginHandler)
```

#### Safe pattern — rate limiting

```go
// SAFE: rate limiter keyed per IP (5 requests per minute, per client)
// a single package-level *rate.Limiter is shared across ALL clients —
// one abusive IP exhausts the quota for everyone else. Key by client instead.
import "golang.org/x/time/rate"

type ipLimiter struct {
    mu       sync.Mutex
    limiters map[string]*rate.Limiter
}

func (l *ipLimiter) forIP(ip string) *rate.Limiter {
    l.mu.Lock()
    defer l.mu.Unlock()
    lim, ok := l.limiters[ip]
    if !ok {
        lim = rate.NewLimiter(rate.Every(time.Minute/5), 5)
        l.limiters[ip] = lim // evict stale entries on a timer in production
    }
    return lim
}

func loginHandler(limiters *ipLimiter) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        if !limiters.forIP(clientIP(r)).Allow() {
            http.Error(w, "too many requests", http.StatusTooManyRequests)
            return
        }
        user, err := authenticate(r.FormValue("email"), r.FormValue("password"))
        if err != nil {
            http.Error(w, "invalid credentials", http.StatusUnauthorized)
            return
        }
        writeToken(w, user)
    }
}
```

#### Vulnerability example — race condition (TOCTOU)

```go
// UNSAFE: read then write — two concurrent requests deduct twice
var balance float64
db.QueryRow("SELECT balance FROM accounts WHERE user_id = $1", userID).Scan(&balance)
if balance >= amount {
    db.Exec("UPDATE accounts SET balance = balance - $1 WHERE user_id = $2", amount, userID)
}
```

#### Safe pattern — race condition

```go
// SAFE: atomic update with balance check in one statement
result, err := db.Exec(
    "UPDATE accounts SET balance = balance - $1 WHERE user_id = $2 AND balance >= $1",
    amount, userID,
)
if n, _ := result.RowsAffected(); n == 0 {
    return errors.New("insufficient funds")
}
```

---

### A05: Security Misconfiguration (CWE-16, CWE-732)

**Impact:** exposure of internal system structure, access to debug interfaces, bypassing CORS policy.

#### What to look for
- `DEBUG=True` in production configuration.
- Detailed error messages with stack traces in API responses.
- CORS with wildcard `*` on endpoints that require authentication.
- Missing security headers: CSP, X-Frame-Options, Strict-Transport-Security.
- Open administrative or debug endpoints without protection (`/metrics`, `/debug`, `/admin`).
- Overly broad permissions on files or directories.

#### Vulnerability example

```go
// UNSAFE: wildcard with credentials
import "github.com/rs/cors"

c := cors.New(cors.Options{
    AllowedOrigins:   []string{"*"},
    AllowCredentials: true,
})
```

#### Safe pattern

```go
// SAFE: explicit allowlist
import "github.com/rs/cors"

c := cors.New(cors.Options{
    AllowedOrigins:   []string{"https://app.example.com", "https://admin.example.com"},
    AllowCredentials: true,
})
handler := c.Handler(mux)
```

---

### A06: Vulnerable and Outdated Components (CWE-1035)

**Impact:** RCE through known CVEs, supply chain attacks, complete compromise of the production environment.

#### What to look for
- New dependencies added without prior vulnerability checks.
- Dependency versions pinned too loosely (`>=2.0`, `*`, `latest`).
- Libraries that are archived or have not been updated for over a year.
- Docker base image with the `:latest` tag without SHA digest.
- Lock files (`package-lock.json`, `poetry.lock`, `go.sum`) added to `.gitignore`.

#### Vulnerability example

```
// UNSAFE: no version pinning (pseudo-version or branch reference)
require github.com/some/package main
```

```dockerfile
# UNSAFE: latest changes without warning
FROM node:latest
```

#### Safe pattern

```
// SAFE: exact semantic version
require github.com/some/package v1.2.3
```

```dockerfile
# SAFE: specific version with SHA digest
FROM node:20-slim@sha256:a1b2c3d4e5f6...
```

---

### A07: Identification and Authentication Failures (CWE-287, CWE-384)

**Impact:** account takeover, authentication bypass, session hijacking.

#### What to look for
- Login, registration, or password reset endpoints without brute force protection.
- Session ID that is not regenerated after successful login (session fixation).
- JWT without `exp`, `iss`, `aud` validation; `alg: none` attack not blocked.
- JWT stored in localStorage (accessible via XSS) instead of httpOnly cookie.
- Missing revocation mechanism for compromised tokens.
- Cookies without `Secure`, `HttpOnly`, `SameSite` flags.

#### Vulnerability example

```go
// UNSAFE: algorithm not verified — vulnerable to alg:none
import jwtlib "github.com/golang-jwt/jwt/v5"

token, _ := jwtlib.Parse(tokenString, func(t *jwtlib.Token) (interface{}, error) {
    return secretKey, nil
})
```

#### Safe pattern

```go
// SAFE: explicit algorithm check + full claims validation
import jwtlib "github.com/golang-jwt/jwt/v5"

token, err := jwtlib.Parse(tokenString,
    func(t *jwtlib.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwtlib.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return publicKey, nil
    },
    jwtlib.WithAudience("my-app"),
    jwtlib.WithIssuer("auth.example.com"),
    jwtlib.WithExpirationRequired(),
)
```

---

### A08: Software and Data Integrity Failures (CWE-502, CWE-494)

**Impact:** arbitrary code execution via deserialization, data substitution in the supply chain, CI/CD pipeline compromise.

#### What to look for
- Deserialization of user-controlled data (`eval()`, `yaml.load` without safe loader).
- Calling `eval()` or `new Function()` on strings received from external sources.
- Loading external resources without integrity checks.
- CI/CD steps that reference external sources without version pinning.

#### Vulnerability example — deserialization

**Go-specific note:** unlike Python `pickle`, Ruby `YAML.load`, or Java deserialization — which have gadget-chain/polymorphic-class-loading paths to arbitrary code execution — Go's static typing means `Unmarshal`/`Decode` can only populate already-declared struct fields or generic maps/interfaces. There's no mechanism for instantiating arbitrary types from untrusted bytes. The real risk from untrusted `gob`/`yaml` input in Go is **resource exhaustion (deeply nested or huge payloads) and type confusion when decoding into `interface{}`**, not RCE — still worth flagging, just not as this specific impact.

```go
// UNSAFE: gob decoding of user-controlled bytes into interface{} — unbounded
// resource use and type confusion downstream, not arbitrary code execution
dec := gob.NewDecoder(bytes.NewReader(userBytes))
var result interface{}
if err := dec.Decode(&result); err != nil {
    return err
}

// UNSAFE: yaml.v2 Unmarshal into interface{} — same class of risk
import "gopkg.in/yaml.v2"
if err := yaml.Unmarshal(userInput, &result); err != nil {
    return err
}
```

#### Safe pattern — deserialization

```go
// SAFE: JSON for untrusted input; yaml.v3 with known target type
import "encoding/json"

var known KnownStruct
if err := json.Unmarshal(userInput, &known); err != nil {
    return err
}
```

#### Vulnerability example — webhook without signature verification

```go
// UNSAFE: no signature verification
func handlePayment(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    processPayment(body)
}
```

#### Safe pattern — webhook with HMAC

```go
// SAFE: HMAC-SHA256 verification with timing-safe compare
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func handlePayment(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    signature := r.Header.Get("X-Signature-256")
    mac := hmac.New(sha256.New, []byte(webhookSecret))
    mac.Write(body)
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))
    if !hmac.Equal([]byte(expected), []byte(signature)) {
        http.Error(w, "forbidden", http.StatusForbidden)
        return
    }
    processPayment(body)
}
```

---

### A09: Security Logging and Monitoring Failures (CWE-778, CWE-223)

**Impact:** inability to detect an attack in time, missing audit trail, compliance violations.

#### What to look for
- Critical security events (login, permission changes, data deletion) not recorded in logs.
- PII, passwords, or tokens appearing in logs in plaintext.
- Empty `catch` blocks that silently swallow errors.
- Missing correlation ID for end-to-end request tracing across services.

#### Vulnerability example

```go
// UNSAFE: token leaks into logs
slog.Info("auth request", "user", email, "token", apiToken)
```

#### Safe pattern

```go
// SAFE: mask sensitive value
slog.Info("auth request", "user", email, "has_token", apiToken != "")
```

---

### A10: Server-Side Request Forgery (CWE-918)

**Impact:** access to internal services, obtaining cloud metadata (AWS/GCP credentials), port scanning of the internal network.

#### What to look for
- User-controlled URL in a server-side HTTP request.
- Features like "import by URL", "link preview", "configure webhook URL".
- Missing allowlist of permitted domains or IP addresses.
- DNS rebinding risk: the domain is checked before the request, but the DNS response may change.

#### Vulnerability example

```go
// UNSAFE: user-controlled URL fetched directly
func fetchURL(w http.ResponseWriter, r *http.Request) {
    resp, err := http.Get(r.URL.Query().Get("url"))
    // ...
}
```

#### Safe pattern

```go
// SAFE: validate scheme + resolve + block private ranges
import (
    "net"
    "net/url"
)

var privateRanges = []string{
    "10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
    "127.0.0.0/8", "169.254.0.0/16",
}

func isSafeURL(rawURL string) (bool, error) {
    parsed, err := url.Parse(rawURL)
    if err != nil {
        return false, err
    }
    if parsed.Scheme != "http" && parsed.Scheme != "https" {
        return false, nil
    }
    addrs, err := net.LookupHost(parsed.Hostname())
    if err != nil {
        return false, err
    }
    for _, addr := range addrs {
        ip := net.ParseIP(addr)
        for _, cidr := range privateRanges {
            _, network, _ := net.ParseCIDR(cidr)
            if network.Contains(ip) {
                return false, nil
            }
        }
    }
    return true, nil
}
```

---

## 2. Additional Vulnerabilities (High-Value CWEs)

### XXE — XML External Entity (CWE-611)

**Impact:** reading arbitrary files on the server, SSRF via XML parser, denial of service (Billion Laughs).

#### What to look for
- XML parsers with external entity resolution enabled.
- User-supplied XML processed on the server side.
- Using `DocumentBuilderFactory` without `FEATURE_SECURE_PROCESSING`.

#### Vulnerability example

```go
// WARNING: third-party libraries like libxml bindings may resolve entities
// Always check the library's default behavior when using non-stdlib XML parsers.
```

#### Safe pattern

```go
// SAFE by default: Go's encoding/xml does not resolve external entities
import "encoding/xml"

var result MyStruct
if err := xml.Unmarshal(userXML, &result); err != nil {
    return err
}
```

---

### Open Redirect (CWE-601)

**Impact:** redirecting the user to a phishing page, theft of OAuth tokens via a forged redirect URI.

#### What to look for
- Target URL for redirect taken from a query parameter without validation.
- `Location` header constructed from user input.
- Missing redirect URL check against an allowlist of permitted addresses.

#### Vulnerability example

```go
// UNSAFE: redirect to user-supplied URL
func redirect(w http.ResponseWriter, r *http.Request) {
    next := r.URL.Query().Get("next")
    http.Redirect(w, r, next, http.StatusFound)
}
```

#### Safe pattern

```go
// SAFE: allowlist validation
var allowedRedirects = map[string]bool{
    "/dashboard": true,
    "/profile":   true,
    "/settings":  true,
}

func redirect(w http.ResponseWriter, r *http.Request) {
    next := r.URL.Query().Get("next")
    if !allowedRedirects[next] {
        next = "/dashboard"
    }
    http.Redirect(w, r, next, http.StatusFound)
}
```

---

### CSRF — Cross-Site Request Forgery (CWE-352)

**Impact:** performing actions on behalf of an authenticated user without their knowledge — changing password, transferring funds.

#### What to look for
- POST/PUT/DELETE endpoints with cookie-based authentication but without a CSRF token.
- Session cookie without `SameSite=Lax` or `SameSite=Strict` flag.
- State-changing operations performed via GET requests.

#### Vulnerability example

```html
<!-- UNSAFE: form without CSRF token -->
<form method="POST" action="/transfer">
    <input name="amount" value="1000" />
    <input name="to_account" value="attacker-iban" />
    <button type="submit">Send</button>
</form>
```

#### Safe pattern

```html
<!-- SAFE: CSRF token included in the form -->
<form method="POST" action="/transfer">
    <input type="hidden" name="gorilla.csrf.Token" value="{{ .CSRFToken }}">
    <input name="amount" value="1000" />
    <input name="to_account" value="attacker-iban" />
    <button type="submit">Send</button>
</form>
```

```go
// Server-side CSRF token verification
import "github.com/gorilla/csrf"

CSRF := csrf.Protect([]byte(csrfAuthKey), csrf.Secure(true))
http.Handle("/transfer", CSRF(http.HandlerFunc(transferHandler)))
```

---

### Unsafe Merge / Mass Assignment (CWE-915)

**Impact:** user-controlled keys overwrite protected fields (`admin`, `role`, `debug`) during a merge, deserialization, or struct-binding step, leading to privilege escalation or config tampering.

**Note:** this is distinct from JavaScript's "Prototype Pollution" (CWE-1321), which exploits the `Object.prototype` chain — Go has no prototype chain, so that specific mechanism doesn't apply here. The Go risk is the same *class* of bug (untrusted input controls which fields get written) via a different mechanism: unchecked map merges or over-permissive struct binding. If reviewing a JS/Node codebase, watch for `lodash.merge`/`lodash.defaultsDeep` or `Object.assign` on untrusted data instead — those are the prototype-pollution-specific vectors.

#### What to look for
- Deep merge or extend functions that accept keys from user input with no allowlist.
- Request bodies decoded straight into a struct that also carries privileged fields (`IsAdmin`, `Role`, `Balance`) instead of a narrow DTO.
- ORM/struct-binding libraries configured to bind all fields from a request without an explicit allowlist.

#### Vulnerability example

```go
// UNSAFE: blindly merging user-controlled map into config
func mergeConfig(base, override map[string]interface{}) map[string]interface{} {
    for k, v := range override {
        base[k] = v  // attacker can overwrite "admin", "role", "debug" keys
    }
    return base
}
```

#### Safe pattern

```go
// SAFE: allowlist of permitted override keys
var allowedConfigKeys = map[string]bool{
    "theme": true, "language": true, "notifications": true,
}

func mergeConfig(base, override map[string]interface{}) map[string]interface{} {
    for k, v := range override {
        if allowedConfigKeys[k] {
            base[k] = v
        }
    }
    return base
}
```

---

### Unrestricted File Upload (CWE-434)

**Impact:** executing malicious code on the server, overwriting system files, DoS via uploading huge files.

#### What to look for
- Uploaded files stored with the original name in a web-accessible directory.
- File type checked only by the `Content-Type` header (easily spoofed).
- Missing file size limit.

#### 5 levels of upload validation

1. **Size** — limit maximum size to a reasonable value.
2. **Extension** — allowlist of permitted extensions (`.jpg`, `.png`, `.pdf`).
3. **MIME by content** — determine type by magic bytes, not by `Content-Type`.
4. **Safe name** — remove path components, use a safe filename.
5. **Storage** — outside the webroot, with a UUID name.

#### Vulnerability example

```go
// UNSAFE: store original filename in web-accessible directory
func uploadHandler(w http.ResponseWriter, r *http.Request) {
    file, header, _ := r.FormFile("upload")
    dst, _ := os.Create("public/uploads/" + header.Filename)
    io.Copy(dst, file)
}
```

#### Safe pattern

```go
// SAFE: 5-level validation
var allowedExts = map[string]bool{".jpg": true, ".jpeg": true, ".png": true, ".pdf": true}
const maxUploadSize = 5 << 20 // 5 MB

func uploadHandler(w http.ResponseWriter, r *http.Request) {
    r.Body = http.MaxBytesReader(w, r.Body, maxUploadSize)
    file, header, err := r.FormFile("upload")
    if err != nil {
        http.Error(w, "file too large or missing", http.StatusBadRequest)
        return
    }
    defer file.Close()
    ext := strings.ToLower(filepath.Ext(header.Filename))
    if !allowedExts[ext] {
        http.Error(w, "file type not allowed", http.StatusBadRequest)
        return
    }
    // Read magic bytes to verify actual content type
    buf := make([]byte, 512)
    file.Read(buf)
    contentType := http.DetectContentType(buf)
    if !strings.HasPrefix(contentType, "image/") {
        http.Error(w, "invalid file content", http.StatusBadRequest)
        return
    }
    // Store with random name outside webroot
    name := make([]byte, 16)
    rand.Read(name)
    safeName := hex.EncodeToString(name) + ext
    dst, _ := os.Create(filepath.Join("/var/uploads", safeName))
    file.Seek(0, 0)
    io.Copy(dst, file)
}
```

---

### Path Traversal (CWE-22)

**Impact:** reading arbitrary files on the server (`/etc/passwd`, configs with secrets), potential writing to system files.

#### What to look for
- Filename from user input substituted into a path via concatenation or template literal.
- Missing path normalization (`path.resolve`) and check that the result stays within the base directory.

#### Vulnerability example

```go
// UNSAFE: user controls the filename
func serveFile(w http.ResponseWriter, r *http.Request) {
    filename := r.URL.Query().Get("file")
    http.ServeFile(w, r, "/uploads/"+filename) // ../../etc/passwd
}
```

#### Safe pattern

```go
// SAFE: resolve and verify path stays within base directory
import "path/filepath"

func serveFile(w http.ResponseWriter, r *http.Request) {
    base := "/uploads"
    filename := r.URL.Query().Get("file")
    fullPath := filepath.Join(base, filename)
    cleanPath := filepath.Clean(fullPath)
    if !strings.HasPrefix(cleanPath, base+string(filepath.Separator)) {
        http.Error(w, "invalid path", http.StatusBadRequest)
        return
    }
    http.ServeFile(w, r, cleanPath)
}
```

---

## 3. Secrets and Credentials

Secret leakage is one of the most frequent and at the same time easiest-to-prevent vulnerabilities. A single hardcoded key can compromise the entire infrastructure.

### What to look for

- **Hardcoded secrets** — API keys, passwords, private keys directly in source code or configuration files (including tests).
- **`.env` files in the repository** — check `.gitignore`; even `.env.example` sometimes contains real values.
- **Tokens in logs** — any token, password, secret in log entries.
- **Secrets in error responses** — connection strings or API keys in stack traces.
- **Predictable token generation** — `Math.random()` for security-sensitive values.
- **Missing rotation** — unlimited lifetime for keys and tokens.
- **Secrets in Docker layers** — `COPY .env`, `ARG PASSWORD=...` are preserved even after deletion.
- **Secrets in URLs** — `?token=abc123` ends up in access logs, Referer header, browser history.

### Vulnerability example — hardcoded credentials

```go
// UNSAFE
const dbPassword = "super_secret_password_123"
const stripeKey = "sk_live_abc123xyz"
```

### Safe pattern — environment variables

```go
// SAFE: load from environment
dbPassword := os.Getenv("DB_PASSWORD")
if dbPassword == "" {
    log.Fatal("DB_PASSWORD environment variable is required")
}
```

### Vulnerability example — weak token generation

```go
// UNSAFE: math/rand is not cryptographically secure
import "math/rand"

token := fmt.Sprintf("%x", rand.Int63())
```

### Safe pattern — cryptographically secure tokens

```go
// SAFE: crypto/rand
import (
    "crypto/rand"
    "encoding/hex"
)

b := make([]byte, 32)
if _, err := rand.Read(b); err != nil {
    return "", err
}
token := hex.EncodeToString(b)
```

---

## 4. Input Validation

Incorrect validation is the root cause of most injection attacks, data corruption, and DoS vulnerabilities.

### Trust boundaries

Everything coming from outside the system is untrusted:
- HTTP request body, query parameters, headers, cookies.
- Files uploaded by users.
- Responses from external APIs (a third party may be compromised).
- Data from the database (may have been poisoned earlier — stored XSS).
- Environment variables in shared hosting.

### Principles

1. **Validate at the system boundary** — check data as early as possible, at the entry point.
2. **Allowlist instead of blocklist** — explicitly allow specific values rather than blocking "bad" ones.
3. **Type coercion** — `parseInt(req.query.id)`, `Number(idStr)`.
4. **Separate sanitization step** — each output context (HTML, SQL, shell, URL) requires its own escaping method.

### Vulnerability example — blocklist instead of allowlist

```go
// UNSAFE: blocklist can always be bypassed — forgot .py, .sh, .jsp
var blockedExtensions = []string{".exe", ".bat", ".cmd"}

func validateUpload(filename string) error {
    ext := strings.ToLower(filepath.Ext(filename))
    for _, blocked := range blockedExtensions {
        if ext == blocked {
            return errors.New("blocked file type")
        }
    }
    return nil
}
```

### Safe pattern — allowlist

```go
// SAFE: only explicitly listed types are allowed
var allowedExtensions = map[string]bool{
    ".jpg": true, ".jpeg": true, ".png": true, ".gif": true, ".pdf": true,
}

func validateUpload(filename string) error {
    ext := strings.ToLower(filepath.Ext(filename))
    if !allowedExtensions[ext] {
        return errors.New("file type not allowed")
    }
    return nil
}
```

### ReDoS — Regular Expression Denial of Service

**Go-specific note:** the standard library's `regexp` package is RE2-based and guarantees linear-time matching — it cannot exhibit catastrophic backtracking regardless of pattern shape, so nested quantifiers like `^(a+)+$` are **not** exploitable through `regexp`. This class of bug applies to backtracking engines: JavaScript, Python `re`, Java, PHP, and Go's own `dlclark/regexp2` if it's pulled in for lookaheads/backreferences that RE2 doesn't support.

```go
// UNSAFE only with a backtracking engine (e.g. github.com/dlclark/regexp2),
// NOT with the standard library's RE2-based regexp — flag this only if regexp2
// or a similar backtracking package is in use
var pattern = regexp2.MustCompile(`^(a+)+$`, regexp2.None)

// Defense in depth regardless of engine: bound input length before matching
const maxInputLength = 1000

func validateInput(userInput string) error {
    if len(userInput) > maxInputLength {
        return errors.New("input too long")
    }
    // ... apply regex to the bounded input
    return nil
}
```

---

## 5. Authentication and Authorization

**Authentication** establishes the user's identity — who is accessing the system.
**Authorization** determines permitted actions — what this user is allowed to do.

The most common mistake: an endpoint verifies that the user is logged in, but **does not check whether they have the right to perform the requested action**.

### Vulnerability example — authorization missing

```go
// UNSAFE: any authenticated user can delete any project
func deleteProjectHandler(w http.ResponseWriter, r *http.Request) {
    user, err := authenticate(r)
    if err != nil {
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
    }
    projectID := chi.URLParam(r, "id")
    // No check: does the user have the right to delete this project?
    deleteProject(projectID)
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]bool{"success": true})
}
```

### Safe pattern — authentication + authorization

```go
// SAFE: both authentication and authorization are checked
func deleteProjectHandler(w http.ResponseWriter, r *http.Request) {
    user, err := authenticate(r)
    if err != nil {
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
    }
    projectID := chi.URLParam(r, "id")
    if !authorize(user, "delete", projectID) {
        http.Error(w, "forbidden", http.StatusForbidden)
        return
    }
    deleteProject(projectID)
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]bool{"success": true})
}
```

### JWT — common mistakes

| Mistake | Impact |
|---------|--------|
| `exp` not validated | Token remains valid indefinitely |
| `iss` / `aud` not validated | Token issued by another service is accepted |
| `alg: none` not blocked | Attacker can strip the signature |
| Secret too short | HS256 can be brute-forced in an hour |
| JWT stored in localStorage | XSS attack grants full access to the token |
| No revocation mechanism | Compromised token cannot be revoked |

### Full JWT validation

```go
// SAFE: full validation with all required checks
import jwtlib "github.com/golang-jwt/jwt/v5"

token, err := jwtlib.Parse(tokenString,
    func(t *jwtlib.Token) (interface{}, error) {
        if _, ok := t.Method.(*jwtlib.SigningMethodRSA); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
        }
        return publicKey, nil
    },
    jwtlib.WithAudience("my-app"),
    jwtlib.WithIssuer("auth.example.com"),
    jwtlib.WithExpirationRequired(),
)
```

### Session management

Three mandatory steps for secure session management:

```go
// After successful login — regenerate session ID (prevents session fixation)
session.Regenerate(r, w)

// On logout — completely destroy the session
session.Destroy(r, w)

// Cookie flags:
// Secure: true — HTTPS only
// HttpOnly: true — inaccessible to JavaScript
// SameSite: http.SameSiteLaxMode — CSRF protection
// MaxAge: reasonable lifetime
```

### OAuth/OIDC — common mistakes

- **Missing `state` parameter** — allows CSRF attack in the OAuth flow.
- **Redirect URI without strict matching** — opens the path to open redirect and token theft.
- **Access token stored without encryption** — accessible if the storage is compromised.
- **`nonce` not validated** in OIDC id_token — allows replay attack.
- **Implicit flow for confidential clients** — should use Authorization Code + PKCE.

---

## 6. Data Protection

Inadequate data protection leads to PII leaks, GDPR/PCI DSS violations, financial penalties, and loss of user trust.

### What to look for

- PII (email, phone, address, IP) without encryption at rest.
- Sensitive data in URL query parameters (ends up in access logs, Referer header, browser history).
- Sensitive data in error responses or debug output.
- Missing TLS for requests to external services.
- PII in logs without masking.
- Violation of the data minimization principle — more information is collected than needed.
- Sensitive data in cache (Redis, Memcached) without TTL.

### Vulnerability example — PII in URL

```go
// UNSAFE: SSN in query parameter — will end up in access logs and Referer
func verifyIdentity(w http.ResponseWriter, r *http.Request) {
    // ?ssn=123-45-6789
    ssn := r.URL.Query().Get("ssn")
    json.NewEncoder(w).Encode(verifySsn(ssn))
}
```

### Safe pattern — PII in request body

```go
// SAFE: sensitive data transmitted in the POST request body
func verifyIdentity(w http.ResponseWriter, r *http.Request) {
    var body struct {
        SSN string `json:"ssn"`
    }
    json.NewDecoder(r.Body).Decode(&body)
    json.NewEncoder(w).Encode(verifySsn(body.SSN))
}
```

### Vulnerability example — excessive data in API response

```go
// UNSAFE: entire object returned — password_hash and internal_notes in response
func getUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    row := db.QueryRow("SELECT * FROM users WHERE id = $1", id)
    var user User
    row.Scan(&user)
    json.NewEncoder(w).Encode(user)
}
```

### Safe pattern — explicit response schema

```go
// SAFE: response struct defines only allowed fields
type UserResponse struct {
    ID    int64  `json:"id"`
    Email string `json:"email"`
    Name  string `json:"name"`
    // password_hash, internal_notes — deliberately excluded
}

func getUser(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    row := db.QueryRow("SELECT id, email, name FROM users WHERE id = $1", id)
    var user UserResponse
    row.Scan(&user.ID, &user.Email, &user.Name)
    json.NewEncoder(w).Encode(user)
}
```

---

## 7. Infrastructure and Deployment

### Docker

#### What to look for

- Base image with the `:latest` tag or without version pinning.
- Container running as `root` or missing non-root user.
- Secrets in `COPY`, `ARG`, `ENV` instructions — they remain in image layers even after deletion.
- Unnecessary tools (curl, wget, net-tools) in the production image.
- `.dockerignore` does not exclude `.env`, `.git`, `node_modules`.

#### Key Dockerfile rules

```dockerfile
# UNSAFE                   →  SAFE
FROM node:latest           →  FROM node:20-slim@sha256:...
COPY .env /app/.env        →  # Secrets via runtime env or mounted secrets
ENV DB_PASS=secret         →  # Never store secrets in ENV/ARG — they are visible in layers
# (no USER)                →  USER appuser  # non-root is mandatory
```

### Kubernetes

#### What to look for

- Pods without `securityContext` (`runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `drop: ["ALL"]`).
- Missing `NetworkPolicy` — by default all pods can communicate with any others.
- Secrets placed in `ConfigMap` instead of `Secret` (or hardcoded in manifests).
- RBAC with wildcard (`*`) permissions.
- `hostNetwork: true` or `privileged: true` without justified necessity.
- Missing resource limits (CPU/memory) — allows DoS through resource exhaustion.
- Images without SHA pinning (`image: app:latest` instead of `app:1.2.3@sha256:...`).

### CI/CD Pipeline Security

#### What to look for

- Secrets leaking into pipeline logs (`echo $SECRET`, verbose mode).
- Dependencies without version pinning in CI scripts.
- Pipeline scripts that execute user-controlled input (PR title in a shell command).
- Missing branch protection rules.
- External actions/plugins without version pinning (`uses: action@main` instead of `@v2.1.0`).

#### CI pipeline injection

```yaml
# UNSAFE: PR title goes directly into shell — code injection possible
run: echo "PR: ${{ github.event.pull_request.title }}"

# SAFE: pass through environment variable
env:
  PR_TITLE: ${{ github.event.pull_request.title }}
run: echo "PR: $PR_TITLE"
```

### TLS/mTLS

- Minimum allowed version — TLS 1.2; recommended — TLS 1.3.
- For internal service-to-service communication, use mTLS.
- Certificate pinning for critical external APIs.
- Mandatory HTTP to HTTPS redirect.

---

## 8. Severity Calibration

Correct severity assessment helps the PR author understand the fix priority and not miss a critical vulnerability.

### 🔴 Critical — blocks merge

- SQL/command/code injection allowing arbitrary query or command execution.
- RCE via deserialization (`eval`, `yaml.load`) or `new Function()` with user input.
- Protected endpoint without authentication check.
- IDOR — resource access without ownership verification.
- Hardcoded secrets or credentials in source code.
- Broken access control or privilege escalation.
- SSRF with access to internal network or cloud metadata.
- Unsafe file upload with server-side execution capability.
- PII, passwords, or tokens in logs without masking.
- XSS in user-facing output.
- XXE with external entity processing enabled.
- Path traversal in file operations.

### 🟡 Warning — needs to be fixed

- Missing rate limiting on authentication or OTP endpoints.
- Missing CSRF protection on state-changing endpoint.
- Security headers missing on HTML responses.
- Open redirect via user-controlled URL.
- Account enumeration through different error messages.
- Weak but not broken algorithm (SHA-1 for non-passwords).
- New dependency with a known CVE.
- Debug mode or verbose errors in production.
- Missing re-authentication for sensitive actions.
- Unsafe merge / mass assignment via deep merge with user input (prototype pollution in JS).
- Missing TLS verification on external calls.
- JWT without `exp` or `alg` validation.

### 🔵 Recommendation — does not block

- Verbose error messages in non-production environment.
- Missing audit log for non-critical actions.
- Non-cryptographic random in non-security context (feature flags, A/B tests).
- Cookie without `SameSite` when other CSRF protection is present.

### ℹ️ Informational

- Dependency not pinned to an exact version (range is acceptable).
- Missing documentation for an accepted security decision.

---

When in doubt — choose the higher severity. It is better to flag a potential issue at 🟡 than to miss a real vulnerability.

### Severity summary table

| Vulnerability | Severity |
|---|---|
| SQL/command/code injection | 🔴 |
| RCE via deserialization or eval | 🔴 |
| Missing authentication on protected endpoint | 🔴 |
| IDOR — resource access without ownership check | 🔴 |
| Hardcoded secrets or credentials | 🔴 |
| Broken access control / privilege escalation | 🔴 |
| SSRF with access to internal network | 🔴 |
| Unsafe file upload with server-side execution | 🔴 |
| PII/passwords/tokens in logs | 🔴 |
| XSS in user-facing output | 🔴 |
| XXE — external entity processing enabled | 🔴 |
| Path traversal in file operations | 🔴 |
| Missing rate limiting on auth/OTP endpoints | 🟡 |
| Missing CSRF protection on state-changing endpoint | 🟡 |
| Security headers missing on HTML responses | 🟡 |
| Open redirect via user-controlled URL | 🟡 |
| Account enumeration via different error responses | 🟡 |
| Weak algorithm (SHA-1 for non-password) | 🟡 |
| New dependency with known CVE | 🟡 |
| Debug mode / verbose errors in production | 🟡 |
| Missing re-authentication for sensitive actions | 🟡 |
| Unsafe merge / mass assignment (prototype pollution in JS) | 🟡 |
| Missing TLS verification on external calls | 🟡 |
| JWT without exp or alg validation | 🟡 |
| Verbose errors in non-production | 🔵 |
| Missing audit log for non-critical actions | 🔵 |
| Non-cryptographic random in non-security context | 🔵 |

---

## 9. Security Review Checklist

Go through this checklist for **every PR** that modifies server-side code.

### Access Control

- [ ] Every new endpoint has authorization middleware.
- [ ] Data queries are scoped to the current user (ownership check).
- [ ] Admin functionality is protected by role verification.
- [ ] File paths do not contain user-controlled components without validation.
- [ ] No IDOR: IDs from the request are always verified for ownership by the current user.

### Injection Prevention

- [ ] All SQL queries are parameterized (no concatenation of user input).
- [ ] No `eval()`, `new Function()`, `execSync()` calls with user input.
- [ ] HTML output is escaped (user content is not rendered as raw HTML).
- [ ] HTTP headers do not contain raw user input.
- [ ] OS commands use an argument array (no shell string).

### Secrets & Crypto

- [ ] No hardcoded credentials, API keys, or tokens in code.
- [ ] `.env` files are included in `.gitignore`.
- [ ] Passwords are hashed with bcrypt or argon2 (not MD5/SHA).
- [ ] Tokens are generated via `crypto.randomBytes`.
- [ ] TLS verification is not disabled.

### Authentication & Sessions

- [ ] JWT is validated with an explicit algorithm, `exp`, `iss`, `aud`.
- [ ] Session ID is regenerated after login.
- [ ] Session is destroyed on logout.
- [ ] Cookies have `Secure`, `HttpOnly`, `SameSite` flags.
- [ ] Rate limiting is configured for login/register/password-reset.

### Data Protection

- [ ] API responses do not contain unnecessary fields (password_hash, internal data).
- [ ] PII does not appear in URL query parameters.
- [ ] Logs do not contain passwords, tokens, or PII without masking.
- [ ] Error responses do not expose internal paths, stack traces, or DB schema.

### Input Validation

- [ ] All user inputs are validated on the server (not only on the client).
- [ ] File uploads are checked: type (magic bytes), size, extension (allowlist).
- [ ] Regex patterns do not have nested quantifiers (ReDoS).
- [ ] URL inputs are checked for SSRF (blocked private networks).

### Infrastructure

- [ ] Docker images use a non-root user and pinned versions.
- [ ] Secrets are absent from Docker layers, ConfigMaps, and CI logs.
- [ ] CI/CD pipeline does not execute user-controlled input (PR title, branch name) in a shell.
- [ ] Kubernetes pods have `securityContext`, resource limits, and NetworkPolicy.

### Deserialization & Integrity

- [ ] No `eval()`, `yaml.load()` on user input.
- [ ] Webhook endpoints verify HMAC signatures.
- [ ] External dependencies are pinned to specific versions.
- [ ] CI actions are pinned to SHA (not `@main` or `@latest`).

### Logging & Monitoring

- [ ] Security-critical actions are logged (login, permission changes, data access).
- [ ] Logs do not contain plaintext secrets or full PII.
- [ ] Exception handlers do not silently swallow errors.
- [ ] Correlation ID is present for end-to-end request tracing.

### Additional Vulnerabilities

- [ ] XML parsers do not allow external entities (use safe parsers like `fast-xml-parser`).
- [ ] Redirect URLs are verified against an allowlist (not arbitrary user input).
- [ ] State-changing endpoints (POST/PUT/DELETE) have CSRF protection or SameSite cookie.
- [ ] Deep merge / extend operations filter `__proto__` and `constructor` keys.
- [ ] File uploads are validated by content (magic bytes), not only by extension or Content-Type.
