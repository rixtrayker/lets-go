# Go Security Best Practices

## Table of Contents
- [Go Security Best Practices](#go-security-best-practices)
  - [Table of Contents](#table-of-contents)
  - [Go's Security Advantages](#gos-security-advantages)
  - [Web Application Security](#web-application-security)
    - [Preventing SQL Injection](#preventing-sql-injection)
    - [Cross-Site Scripting (XSS)](#cross-site-scripting-xss)
    - [Cross-Site Request Forgery (CSRF)](#cross-site-request-forgery-csrf)
    - [Secure HTTP Headers](#secure-http-headers)
  - [Application-Level Vulnerabilities](#application-level-vulnerabilities)
    - [Integer Overflows](#integer-overflows)
    - [Time-of-Check to Time-of-Use (TOCTOU)](#time-of-check-to-time-of-use-toctou)
  - [Dependency Management Security](#dependency-management-security)
  - [Securely Using `cgo`](#securely-using-cgo)
  - [Secure Containerization](#secure-containerization)
  - [Secrets Management](#secrets-management)
  - [General Security Best Practices](#general-security-best-practices)

## Go's Security Advantages

Go was designed with features that help developers write secure and robust software:

-   **Memory Safety:** Go is a memory-safe language. It handles its own memory allocation and garbage collection, which eliminates entire classes of vulnerabilities like buffer overflows and use-after-free bugs that are common in languages like C/C++.
-   **Strong Typing:** Go's static type system prevents type confusion bugs at compile time.
-   **Explicit Error Handling:** The `if err != nil` pattern forces developers to handle potential failures, reducing the chance of unhandled errors leading to security vulnerabilities.
-   **Statically Linked Binaries:** Go compiles to a single binary with no external dependencies (unless using `cgo`), making it harder for attackers to exploit system libraries and simplifying containerization.

## Web Application Security

### Preventing SQL Injection

SQL Injection occurs when user input is insecurely added to a database query, allowing an attacker to execute arbitrary SQL commands.

**How to Prevent It:** **Never** use `fmt.Sprintf` or string concatenation to build queries with user-provided data. **Always** use parameterized queries provided by the `database/sql` package.

**Wrong (Vulnerable):**
```go
// NEVER DO THIS
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", userEmail)
rows, err := db.Query(query)
```

**Right (Secure):**
```go
// Use placeholders ('$' for Postgres, '?' for MySQL)
query := "SELECT * FROM users WHERE email = $1"
// Pass user input as arguments to the query function.
// The database driver will safely sanitize the input.
rows, err := db.QueryContext(ctx, query, userEmail)
```

### Cross-Site Scripting (XSS)

XSS occurs when an application includes untrusted data in a web page without proper validation or escaping, allowing an attacker to execute malicious scripts in the user's browser.

Go's `html/template` package is context-aware and automatically escapes data to prevent XSS by default.

**Wrong (Vulnerable):**
Using `text/template` for HTML output or manually writing to the `http.ResponseWriter`.
```go
// text/template does NOT escape HTML.
t, _ := template.New("foo").Parse(`Hello, {{.Name}}`)
// If name is "<script>alert('pwned')</script>", the script will execute.
t.Execute(out, data) 
```

**Right (Secure):**
Always use `html/template` to render HTML. If you must include unescaped data (e.g., from a trusted source), use the specific `template.HTML`, `template.JS`, or `template.URL` types to declare it safe.
```go
// html/template automatically escapes HTML characters.
t, _ := template.New("foo").Parse(`Hello, {{.Name}}`)
// The output will be "Hello, &lt;script&gt;alert(&#39;pwned&#39;)&lt;/script&gt;"
t.Execute(out, data)
```

### Cross-Site Request Forgery (CSRF)

CSRF tricks a logged-in user into submitting a malicious request to a web application they are authenticated with.

**How to Prevent It:**
1.  **Use a CSRF token:** On state-changing requests (POST, PUT, DELETE), the server provides a secret, unique token to the client. The client must include this token in the next request (e.g., in a hidden form field or header). The server validates the token before processing the request.
2.  **Use a library:** Libraries like `gorilla/csrf` or the built-in features of web frameworks provide middleware to handle this automatically.

**Example with `gorilla/csrf`:**
```go
// The key should be a 32-byte random secret, loaded securely.
csrfMiddleware := csrf.Protect([]byte("32-byte-long-auth-key"), csrf.Secure(true))
r := mux.NewRouter()
// Protect all state-changing routes with the middleware.
api := r.PathPrefix("/api").Subrouter()
api.Use(csrfMiddleware)
api.HandleFunc("/user", UpdateUser).Methods("POST")
// In your template for the form:
// <input type="hidden" name="gorilla.csrf.Token" value="{{.CSRFToken}}">
```

### Secure HTTP Headers
Use middleware to add security headers to all responses.
- `Content-Security-Policy`: Prevents XSS by specifying which sources of content are allowed.
- `Strict-Transport-Security`: Enforces the use of HTTPS.
- `X-Content-Type-Options: nosniff`: Prevents the browser from MIME-sniffing a response away from the declared content type.
- `X-Frame-Options: deny`: Prevents clickjacking by disabling rendering in frames.

## Application-Level Vulnerabilities

### Integer Overflows
While Go has some protections, integer overflows can still occur, especially during type conversions. This can lead to unexpected behavior, such as turning a large positive number into a negative one, which might bypass security checks.

```go
// User provides a large value for `count`
var count int16 = 32767 
// Attacker adds 1, causing an overflow
total := count + 1 // total is now -32768
if total > 0 { // This check is now bypassed
    // allocate resources...
}
```
**Prevention:** Use packages like `math/bits` for overflow-aware arithmetic or validate inputs against `math.MaxInt` constants.

### Time-of-Check to Time-of-Use (TOCTOU)
This is a race condition where a resource's state is checked but then changes before it is used. A common example is checking if a file exists before writing to it. An attacker could replace the file with a symlink to a sensitive location between the check and the write.

```go
// VULNERABLE
if _, err := os.Stat(filePath); os.IsNotExist(err) {
    // Check passes. Attacker now replaces filePath with a symlink to /etc/passwd.
    os.WriteFile(filePath, data, 0644) // Writes to /etc/passwd!
}
```
**Prevention:** Don't check and then act. Act and then check the result. Use file-opening flags like `O_CREATE|O_EXCL` to ensure you are the one creating the file.

## Dependency Management Security

Your application is only as secure as its weakest dependency.

-   **Scan Your Dependencies:** Use `govulncheck` to scan your `go.mod` for dependencies with known vulnerabilities.
    ```bash
    go install golang.org/x/vuln/cmd/govulncheck@latest
    govulncheck ./...
    ```
-   **Commit `go.sum`:** This file ensures your builds are reproducible and that the code you download matches the expected checksum, preventing tampered dependencies.

## Securely Using `cgo`
`cgo` is an escape hatch from Go's memory safety. When calling C code:
- **Memory Management:** Any memory allocated by C code must be manually freed using `C.free`. Use `defer` to ensure this happens.
- **Input Sanitization:** C code is susceptible to buffer overflows. Treat any C library as a potential source of vulnerabilities and rigorously sanitize any data passed to it.

## Secure Containerization

When deploying with Docker, follow these best practices:
-   **Use Multi-Stage Builds:** Use one stage for building with the full Go SDK and a final, minimal stage for the production image.
-   **Use Minimal Base Images:** Start your final container stage `FROM scratch` or `FROM gcr.io/distroless/static-debian11`. These images contain no shell or other programs for an attacker to use.
-   **Run as a Non-Root User:** Don't run your container as the `root` user.

**Secure Dockerfile Example:**
```dockerfile
# ---- Builder Stage ----
FROM golang:1.22-alpine as builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .

# Build a static, CGO_ENABLED=0 binary
RUN CGO_ENABLED=0 go build -ldflags="-w -s" -o /app/main .

# ---- Final Stage ----
FROM gcr.io/distroless/static-debian11

# Copy the static binary from the builder stage
COPY --from=builder /app/main /main

# (Optional) For images not from scratch, create a non-root user
# USER nonroot:nonroot

ENTRYPOINT ["/main"]
```

## Secrets Management

Never hardcode secrets (API keys, database passwords) in your source code.
-   **Use Environment Variables:** Load secrets from the environment. This is portable and works well with container orchestrators.
-   **Use a Secrets Management System:** For production, use a dedicated service like HashiCorp Vault, AWS Secrets Manager, or Google Secret Manager.

**Example: Loading from Environment**
```go
import (
    "os"
    "database/sql"
)

func connectToDB() (*sql.DB, error) {
    dbPassword := os.Getenv("DB_PASSWORD")
    if dbPassword == "" {
        return nil, errors.New("DB_PASSWORD not set")
    }
    connStr := fmt.Sprintf("user=... password=%s ...", dbPassword)
    return sql.Open("postgres", connStr)
}
```

## General Security Best Practices

1.  **Use HTTPS:** Always use TLS to encrypt traffic. Use a library like `acme/autocert` for free, automatic certificates from Let's Encrypt.
2.  **Validate All Input:** Treat all data from users or external systems as untrusted. Validate it for type, length, format, and range.
3.  **Limit Request Rates:** Use a middleware to limit the number of requests a client can make in a given time period to prevent brute-force and denial-of-service attacks.
4.  **Log Security Events:** Log important events like failed login attempts, access denied errors, and changes to sensitive data.

---

Next: [Code Style & Linting](17-code-style.md) 