# Error Handling in Go

## Table of Contents
- [Error Handling in Go](#error-handling-in-go)
  - [Table of Contents](#table-of-contents)
  - [The Go Philosophy of Error Handling](#the-go-philosophy-of-error-handling)
  - [The `error` Type](#the-error-type)
  - [Basic Error Handling Patterns](#basic-error-handling-patterns)
    - [Returning Errors from Functions](#returning-errors-from-functions)
  - [Adding Context to Errors: Wrapping](#adding-context-to-errors-wrapping)
    - [The `%w` Verb](#the-w-verb)
    - [Unwrapping Errors with `errors.Is` and `errors.As`](#unwrapping-errors-with-errorsis-and-errorsas)
  - [Sentinel Errors](#sentinel-errors)
  - [Custom Error Types](#custom-error-types)
  - [Handling Errors in Concurrent Code](#handling-errors-in-concurrent-code)
  - [Advanced Error Handling Techniques](#advanced-error-handling-techniques)
    - [Handling Panics](#handling-panics)
  - [Best Practices for Error Handling](#best-practices-for-error-handling)

## The Go Philosophy of Error Handling

Go's approach to error handling is explicit and deliberate. Unlike languages with exceptions, Go treats errors as regular values. This forces developers to confront errors at the point they occur, leading to more robust and reliable code.

**Key Principles:**
- **Errors are values:** The `error` type is a built-in interface.
- **Explicit checking:** Functions that can fail return an `error` as their last return value.
- **No exceptions:** Go avoids the complexities of try/catch/finally blocks.
- **Context is key:** Errors should provide clear context about what went wrong.

## The `error` Type

The `error` type is a simple interface defined in the standard library:

```go
type error interface {
    Error() string
}
```

Any type that implements the `Error() string` method satisfies the `error` interface.

## Basic Error Handling Patterns

The most common pattern is checking the returned `error` value immediately.

```go
file, err := os.Open("my-file.txt")
if err != nil {
    // Handle the error immediately.
    // Log it, return it, or take other action.
    log.Fatalf("failed to open file: %v", err)
}
defer file.Close()

// ... proceed with using the file ...
```

### Returning Errors from Functions

When a function encounters an error from a call it makes, it should typically return the error to its caller, adding context.

```go
func readConfig(path string) (*Config, error) {
    file, err := os.Open(path)
    if err != nil {
        // Don't just return `err`. Add context.
        return nil, fmt.Errorf("readConfig: failed to open config file %s: %w", path, err)
    }
    defer file.Close()

    // ... parse the config ...
    return &Config{}, nil
}
```

## Adding Context to Errors: Wrapping

Error wrapping, introduced in Go 1.13, is the standard way to add context to an error while preserving the original error.

### The `%w` Verb

The `fmt.Errorf` function supports the `%w` verb to create a wrapped error.

```go
func dataAccessLayer(id string) ([]byte, error) {
    err := lowLevelDB.Query(id)
    if err != nil {
        // Wrap the original database error with context.
        return nil, fmt.Errorf("querying for id %s: %w", id, err)
    }
    // ...
}
```

### Unwrapping Errors with `errors.Is` and `errors.As`

Use `errors.Is` to check for a specific error instance (sentinel errors) in an error chain. Use `errors.As` to check if any error in the chain matches a specific type.

```go
// In the database package
var ErrNotFound = errors.New("record not found")

func lowLevelDB.Query(id string) error {
    // ... logic to query ...
    if notFound {
        return ErrNotFound
    }
    return nil
}

// In the calling code
data, err := dataAccessLayer("123")
if err != nil {
    // errors.Is checks if ErrNotFound is in the error chain.
    if errors.Is(err, ErrNotFound) {
        log.Printf("Record not found, using default.")
        // Handle the "not found" case gracefully.
    } else {
        log.Fatalf("Unhandled data access error: %v", err)
    }
}
```

## Sentinel Errors

Sentinel errors are predefined error values that indicate a specific, expected condition.

**When to Use:** For common, predictable error states that callers may need to handle specifically.

```go
// Standard library examples
var (
    EOF = errors.New("EOF") // From the io package
    ErrClosed = errors.New("net: connection is closed") // From the net package
)

// Your own sentinel errors
var (
    ErrUserNotAuthenticated = errors.New("user not authenticated")
    ErrPaymentDeclined      = errors.New("payment was declined")
)

func processPayment() error {
    // ... logic ...
    if declined {
        return ErrPaymentDeclined
    }
    return nil
}

func main() {
    err := processPayment()
    if err != nil {
        if errors.Is(err, ErrPaymentDeclined) {
            // Retry with a different card, show a specific message, etc.
        } else {
            // Handle unexpected errors.
        }
    }
}
```

**Caution:** Overusing sentinel errors can lead to coupling between packages. Custom error types are often a better choice for more complex error information.

## Custom Error Types

For errors that need to carry more information than just a string, create a custom error type.

**When to Use:** When you need to provide structured data about an error, like an HTTP status code or validation details.

```go
// Custom error type
type RequestError struct {
    StatusCode int
    Message    string
    Err        error // Underlying error
}

func (e *RequestError) Error() string {
    return fmt.Sprintf("status %d: %s", e.StatusCode, e.Message)
}

// Implement Unwrap to support errors.Is and errors.As
func (e *RequestError) Unwrap() error {
    return e.Err
}

// Function that returns the custom error
func makeAPIRequest() error {
    // ... http request logic ...
    if resp.StatusCode == 404 {
        return &RequestError{
            StatusCode: 404,
            Message:    "Resource not found",
            Err:        nil, // No underlying error in this case
        }
    }
    // ...
    return nil
}

// Calling code
err := makeAPIRequest()
if err != nil {
    var reqErr *RequestError
    // errors.As checks if the error is a *RequestError and extracts it.
    if errors.As(err, &reqErr) {
        if reqErr.StatusCode == 404 {
            // Handle not found specifically
        } else {
            // Handle other API errors
        }
    } else {
        // Handle non-API errors (e.g., network issues)
    }
}
```

## Handling Errors in Concurrent Code

Error handling with goroutines requires channels.

```go
type Result struct {
    Value int
    Err   error
}

func worker(job int, results chan<- Result) {
    // ... do work ...
    if err != nil {
        results <- Result{Err: fmt.Errorf("worker failed on job %d: %w", job, err)}
        return
    }
    results <- Result{Value: job * 2}
}

func main() {
    numJobs := 5
    jobs := make(chan int, numJobs)
    results := make(chan Result, numJobs)

    // Start workers
    for i := 0; i < 3; i++ {
        go worker(i, results)
    }

    // A WaitGroup to wait for all results to be processed
    var wg sync.WaitGroup
    wg.Add(numJobs)

    // Process results
    go func() {
        for i := 0; i < numJobs; i++ {
            res := <-results
            if res.Err != nil {
                log.Printf("ERROR: %v", res.Err)
            } else {
                fmt.Printf("Success: %d\n", res.Value)
            }
            wg.Done()
        }
    }()

    wg.Wait()
}
```
A more robust solution for complex concurrent error handling is the `errgroup` package. See `12-testing.md`.

## Advanced Error Handling Techniques

### Handling Panics

`panic` should be used for unrecoverable, programmer-level errors. It's rarely used in idiomatic Go. However, you can recover from a panic if necessary.

**When to Use `recover`:** In a web server, to prevent a single request from crashing the entire process.

```go
func recoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("recovered from panic: %v", err)
                // Log stack trace
                debug.PrintStack()
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        next.ServeHTTP(w, r)
    })
}
```

## Best Practices for Error Handling

1.  **Check Every Error:** Never ignore an error with `_`. If you explicitly choose to ignore it, document why.
2.  **Return Errors, Don't Panic:** In library code, always return errors. Don't let implementation details like panics leak to your callers.
3.  **Wrap Errors for Context:** Use `fmt.Errorf` with `%w` to add context while preserving the original error. This provides a clear error chain.
4.  **Use Sentinel Errors for Specific, Known Conditions:** Use `errors.Is` to check for them.
5.  **Use Custom Error Types for Richer Information:** Use `errors.As` to access the specific error type.
6.  **Handle Errors at the Right Level:** Don't log and return an error. The caller should decide whether to log it, retry, or return it further up the stack.
7.  **Keep It Simple:** Often, a simple `if err != nil` check is all you need. Don't over-engineer your error handling.

---

Next: [Reflection & Generics](10-reflection-generics.md) 