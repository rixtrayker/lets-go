# Go Security Best Practices

## Table of Contents
- [Go's Security Advantages](#gos-security-advantages)
- [Web Application Security](#web-application-security)
  - [Preventing SQL Injection](#preventing-sql-injection)
  - [Cross-Site Scripting (XSS)](#cross-site-scripting-xss)
  - [Cross-Site Request Forgery (CSRF)](#cross-site-request-forgery-csrf)
  - [Directory Traversal](#directory-traversal)
- [Dependency Management Security](#dependency-management-security)
- [Secure Containerization](#secure-containerization)
- [Secrets Management](#secrets-management)
- [General Security Best Practices](#general-security-best-practices)

## Go's Security Advantages

Go was designed with features that help developers write secure and robust software:

-   **Memory Safety:** Go is a memory-safe language. It handles its own memory allocation and garbage collection, which eliminates entire classes of vulnerabilities like buffer overflows and use-after-free bugs that are common in languages like C/C++.
-   **Strong Typing:** Go's static type system prevents type confusion bugs at compile time.
-   **Explicit Error Handling:** The `if err != nil` pattern forces developers to handle potential failures, reducing the chance of unhandled errors leading to security vulnerabilities.
-   **Statically Linked Binaries:** Go compiles to a single binary with no external dependencies (unless using cgo), making it harder for attackers to exploit system libraries.

## Web Application Security

### Preventing SQL Injection

SQL Injection occurs when user input is insecurely added to a database query, allowing an attacker to execute arbitrary SQL commands.

**How to Prevent It:** **Never** use `fmt.Sprintf` or string concatenation to build queries with user-provided data. **Always** use parameterized queries.

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
rows, err := db.Query(query, userEmail)
```

### Cross-Site Scripting (XSS)

XSS occurs when an application includes untrusted data in a web page without proper validation or escaping, allowing an attacker to execute malicious scripts in the user's browser.

Go's `html/template` package is context-aware and automatically escapes data to prevent XSS by default.

**Wrong (Vulnerable):**
Using `text/template` or manual string writing for HTML output.
```go
// text/template does NOT escape HTML.
t, _ := template.New("foo").Parse(`Hello, {{.Name}}`)
// If name is "<script>alert('pwned')</script>", the script will execute.
t.Execute(out, data) 
```

**Right (Secure):**
Always use `html/template` to render HTML.
```go
// html/template automatically escapes HTML characters.
t, _ := template.New("foo").Parse(`Hello, {{.Name}}`)
// The output will be "Hello, &lt;script&gt;alert(&#39;pwned&#39;)&lt;/script&gt;"
// The browser will render this as text, not execute it as a script.
t.Execute(out, data)
```

### Cross-Site Request Forgery (CSRF)

CSRF tricks a logged-in user into submitting a malicious request to a web application they are authenticated with.

**How to Prevent It:**
1.  **Use a CSRF token:** On state-changing requests (POST, PUT, DELETE), the server provides a secret, unique token to the client. The client must include this token in the next request. The server validates the token before processing the request.
2.  **Use a library:** Libraries like `gorilla/csrf` provide middleware to handle this automatically.

**Example with `gorilla/csrf`:**
```go
// Set up CSRF protection middleware. The key should be a 32-byte random secret.
csrfMiddleware := csrf.Protect([]byte("32-byte-long-auth-key"))

r := mux.NewRouter()
r.HandleFunc("/signup", ShowSignupForm)
// Protect the POST handler with the middleware.
r.Handle("/signup", csrfMiddleware(http.HandlerFunc(SubmitSignupForm)))

// In your template for the form:
// <input type="hidden" name="gorilla.csrf.Token" value="{{.CSRFToken}}">
```

### Directory Traversal

This attack uses file paths (`../`) to access files and directories stored outside the intended web root folder.

**How to Prevent It:**
-   Don't pass user input directly to file system calls.
-   Sanitize and validate any user input that is used to construct a file path.
-   Use `http.FileServer` correctly, as it has built-in protections.

```go
// Safe way to serve files from a directory.
// http.Dir strips leading slashes and prevents directory traversal.
fs := http.FileServer(http.Dir("./static"))
http.Handle("/static/", http.StripPrefix("/static/", fs))
```

## Dependency Management Security

Your application is only as secure as its weakest dependency.

-   **Scan Your Dependencies:** Use tools like `govulncheck` (official Go tool) or `Snyk` to scan your `go.mod` for dependencies with known vulnerabilities.
    ```bash
    # Install the tool
    go install golang.org/x/vuln/cmd/govulncheck@latest
    
    # Run the scan in your project directory
    govulncheck ./...
    ```
-   **Keep Dependencies Updated:** Regularly update your dependencies with `go get -u` and `go mod tidy` to patch security issues.
-   **Commit `go.sum`:** This file ensures your builds are reproducible and that the code you download matches the expected checksum, preventing tampered dependencies.

## Secure Containerization

When deploying with Docker, follow these best practices:
-   **Use Minimal Base Images:** Start your final container stage `FROM scratch` or `FROM gcr.io/distroless/static-debian11`. These images contain only your application and its essential libraries, with no shell or other programs for an attacker to use.
-   **Run as a Non-Root User:** Don't run your container as the `root` user.
    ```dockerfile
    # In your final stage
    FROM scratch
    # Create a non-root user (requires a base image that's not scratch)
    # USER nonroot:nonroot 
    COPY --from=builder /main /main
    ENTRYPOINT ["/main"]
    ```

## Secrets Management

Never hardcode secrets (API keys, database passwords, certs) in your source code.
-   **Use Environment Variables:** Load secrets from the environment. This is portable and works well with container orchestrators.
-   **Use a Secrets Management System:** For production, use a dedicated service like HashiCorp Vault, AWS Secrets Manager, or Google Secret Manager. Your application can be configured with credentials to fetch its secrets from these services on startup.

## General Security Best Practices

1.  **Use HTTPS:** Always use TLS to encrypt traffic between your clients and your server. Use a library like `acme/autocert` for free, automatic certificates from Let's Encrypt.
2.  **Validate All Input:** Treat all data from users or external systems as untrusted. Validate it for type, length, format, and range.
3.  **Use Secure Headers:** Add security-related headers to your HTTP responses, such as `Content-Security-Policy`, `Strict-Transport-Security`, and `X-Frame-Options`.
4.  **Limit Request Rates:** Use a middleware to limit the number of requests a client can make in a given time period to prevent brute-force and denial-of-service attacks.
5.  **Log Security Events:** Log important events like failed login attempts, access denied errors, and changes to sensitive data.

---

Next: [Code Style & Linting](17-code-style.md) 