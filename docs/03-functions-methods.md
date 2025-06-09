# Functions & Methods in Go

## Table of Contents
- [Function Basics](#function-basics)
- [Value vs Pointer Receivers](#value-vs-pointer-receivers)
- [The Append Trap](#the-append-trap)
- [Method Design Patterns](#method-design-patterns)
- [Function Parameters Best Practices](#function-parameters-best-practices)
- [Error Handling in Functions](#error-handling-in-functions)
- [Advanced Function Patterns](#advanced-function-patterns)

## Function Basics

### Function Declaration

```go
// Basic function
func add(a, b int) int {
    return a + b
}

// Multiple return values
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Named return values
func calculate(a, b int) (sum, product int) {
    sum = a + b
    product = a * b
    return // naked return
}

// Variadic functions
func sum(numbers ...int) int {
    total := 0
    for _, n := range numbers {
        total += n
    }
    return total
}
```

### Function Types and Variables

```go
// Function types
type Operation func(int, int) int

// Function as variable
var add Operation = func(a, b int) int {
    return a + b
}

// Higher-order functions
func applyOperation(a, b int, op Operation) int {
    return op(a, b)
}

// Closures
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
```

## Value vs Pointer Receivers

### The Decision Framework

Choose receiver type based on these rules:

1. **Does the method modify the receiver?** → Use pointer
2. **Is the receiver large?** → Use pointer for performance
3. **Do other methods use pointer receivers?** → Use pointer for consistency

### Value Receivers

```go
type Point struct {
    X, Y float64
}

// Value receiver - good for:
// - Small structs
// - Read-only operations
// - When you want immutability
func (p Point) Distance() float64 {
    return math.Sqrt(p.X*p.X + p.Y*p.Y)
}

func (p Point) String() string {
    return fmt.Sprintf("(%f, %f)", p.X, p.Y)
}

// Usage
point := Point{3, 4}
fmt.Println(point.Distance()) // 5
```

### Pointer Receivers

```go
type Counter struct {
    count int
}

// Pointer receiver - necessary for:
// - Modifying the receiver
// - Large structs (>100 bytes)
// - Consistency
func (c *Counter) Increment() {
    c.count++
}

func (c *Counter) Reset() {
    c.count = 0
}

// Even read-only methods use pointer for consistency
func (c *Counter) Value() int {
    return c.count
}

// Usage
counter := &Counter{}
counter.Increment()
fmt.Println(counter.Value()) // 1
```

### Large Struct Example

```go
type LargeStruct struct {
    data [1024]byte
    metadata map[string]string
    // ... many fields
}

// ✅ Use pointer receiver to avoid copying 1KB+ on each call
func (ls *LargeStruct) Process() {
    // Process the data without copying the entire struct
}

func (ls *LargeStruct) Size() int {
    return len(ls.data)
}

// ❌ Don't use value receiver for large structs
func (ls LargeStruct) SlowProcess() {
    // This copies 1KB+ on every call!
}
```

## The Append Trap

### The Problem

The `append` function can modify the underlying array of the original slice if it has spare capacity:

```go
// ❌ DANGEROUS: This function can have unexpected side effects
func appendDangerous(slice []int, value int) []int {
    return append(slice, value)
}

func main() {
    original := make([]int, 3, 5) // len=3, cap=5
    original[0] = 1
    original[1] = 2  
    original[2] = 3
    // original is [1, 2, 3] with capacity 5
    
    // This modifies the original slice's underlying array!
    newSlice := appendDangerous(original, 4)
    
    // If we later append to original, we might overwrite newSlice[3]
    original = append(original, 999)
    
    fmt.Println(newSlice)  // Might print [1, 2, 3, 999] instead of [1, 2, 3, 4]
}
```

### Safe Append Patterns

#### Pattern 1: Copy Before Append

```go
// ✅ SAFE: Copy the slice before appending
func appendSafe(slice []int, value int) []int {
    // Create a copy to avoid modifying the original
    result := make([]int, len(slice), len(slice)+1)
    copy(result, slice)
    return append(result, value)
}
```

#### Pattern 2: Use Full Slice Expression

```go
// ✅ SAFE: Limit capacity to prevent unexpected sharing
func appendWithLimitedCap(slice []int, value int) []int {
    // slice[:len(slice):len(slice)] limits capacity to length
    return append(slice[:len(slice):len(slice)], value)
}
```

#### Pattern 3: Always Reassign Result

```go
// ✅ SAFE: Always reassign when appending
func processSlice(items []string) []string {
    // Always assign the result back
    items = append(items, "new item")
    return items
}

// Usage
func main() {
    mySlice := []string{"a", "b", "c"}
    mySlice = processSlice(mySlice) // Always reassign
}
```

### Real-World Append Safety

```go
type Server struct {
    handlers []Handler
}

// ❌ DANGEROUS: Might modify caller's slice
func (s *Server) AddHandler(h Handler) {
    s.handlers = append(s.handlers, h)
}

// ✅ SAFE: Make defensive copy if needed
func (s *Server) AddHandlerSafe(h Handler) {
    // Option 1: Always copy
    newHandlers := make([]Handler, len(s.handlers), len(s.handlers)+1)
    copy(newHandlers, s.handlers)
    s.handlers = append(newHandlers, h)
    
    // Option 2: Use full slice expression
    s.handlers = append(s.handlers[:len(s.handlers):len(s.handlers)], h)
}

// ✅ SAFEST: Return new slice, don't modify receiver
func (s *Server) WithHandler(h Handler) *Server {
    newHandlers := make([]Handler, len(s.handlers)+1)
    copy(newHandlers, s.handlers)
    newHandlers[len(s.handlers)] = h
    
    return &Server{handlers: newHandlers}
}
```

## Method Design Patterns

### Builder Pattern with Methods

```go
type HTTPRequest struct {
    method  string
    url     string
    headers map[string]string
    body    []byte
}

// Builder methods return pointer to enable chaining
func NewHTTPRequest() *HTTPRequest {
    return &HTTPRequest{
        headers: make(map[string]string),
    }
}

func (r *HTTPRequest) Method(method string) *HTTPRequest {
    r.method = method
    return r
}

func (r *HTTPRequest) URL(url string) *HTTPRequest {
    r.url = url
    return r
}

func (r *HTTPRequest) Header(key, value string) *HTTPRequest {
    r.headers[key] = value
    return r
}

func (r *HTTPRequest) Body(body []byte) *HTTPRequest {
    r.body = body
    return r
}

// Usage
request := NewHTTPRequest().
    Method("POST").
    URL("https://api.example.com").
    Header("Content-Type", "application/json").
    Body([]byte(`{"key": "value"}`))
```

### Option Pattern

```go
type ServerConfig struct {
    Host    string
    Port    int
    Timeout time.Duration
    Debug   bool
}

type ServerOption func(*ServerConfig)

func WithHost(host string) ServerOption {
    return func(cfg *ServerConfig) {
        cfg.Host = host
    }
}

func WithPort(port int) ServerOption {
    return func(cfg *ServerConfig) {
        cfg.Port = port
    }
}

func WithTimeout(timeout time.Duration) ServerOption {
    return func(cfg *ServerConfig) {
        cfg.Timeout = timeout
    }
}

func NewServer(options ...ServerOption) *Server {
    // Default configuration
    cfg := &ServerConfig{
        Host:    "localhost",
        Port:    8080,
        Timeout: 30 * time.Second,
        Debug:   false,
    }
    
    // Apply options
    for _, option := range options {
        option(cfg)
    }
    
    return &Server{config: cfg}
}

// Usage
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9090),
    WithTimeout(60*time.Second),
)
```

## Function Parameters Best Practices

### Parameter Ordering

```go
// ✅ GOOD: Context first, options last
func ProcessUser(ctx context.Context, userID int, options ProcessOptions) error

// ✅ GOOD: Required parameters first, optional last
func CreateFile(path string, content []byte, options ...FileOption) error

// ❌ BAD: Options in the middle
func CreateFile(path string, options FileOption, content []byte) error
```

### Limit Parameter Count

```go
// ❌ BAD: Too many parameters
func CreateUser(name, email, phone, address, city, state, zip, country string, age int, active bool) *User

// ✅ GOOD: Use a struct for related parameters
type UserInfo struct {
    Name    string
    Email   string
    Phone   string
    Address Address
    Age     int
    Active  bool
}

func CreateUser(info UserInfo) *User
```

### Use Interfaces for Flexibility

```go
// ✅ GOOD: Accept interfaces, return concrete types
func WriteData(w io.Writer, data []byte) error {
    _, err := w.Write(data)
    return err
}

// This works with files, buffers, network connections, etc.
file, _ := os.Create("data.txt")
WriteData(file, []byte("hello"))

var buf bytes.Buffer
WriteData(&buf, []byte("hello"))
```

## Error Handling in Functions

### Error Patterns

```go
// Basic error handling
func readFile(filename string) ([]byte, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read file %s: %w", filename, err)
    }
    return data, nil
}

// Multiple operations with early return
func processUser(id int) (*User, error) {
    user, err := fetchUser(id)
    if err != nil {
        return nil, fmt.Errorf("fetch user: %w", err)
    }
    
    if err := validateUser(user); err != nil {
        return nil, fmt.Errorf("validate user: %w", err)
    }
    
    if err := enrichUser(user); err != nil {
        return nil, fmt.Errorf("enrich user: %w", err)
    }
    
    return user, nil
}

// Sentinel errors for expected conditions
var (
    ErrUserNotFound = errors.New("user not found")
    ErrInvalidInput = errors.New("invalid input")
)

func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, ErrInvalidInput
    }
    
    user, exists := userCache[id]
    if !exists {
        return nil, ErrUserNotFound
    }
    
    return user, nil
}

// Usage with errors.Is
user, err := findUser(123)
if err != nil {
    if errors.Is(err, ErrUserNotFound) {
        // Handle not found case
        return createDefaultUser()
    }
    return nil, err
}
```

### Custom Error Types

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation error on field %s: %s", e.Field, e.Message)
}

func validateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return ValidationError{
            Field:   "email",
            Message: "must contain @ symbol",
        }
    }
    return nil
}

// Usage with type assertion
if err := validateEmail("invalid-email"); err != nil {
    var validationErr ValidationError
    if errors.As(err, &validationErr) {
        fmt.Printf("Validation failed on field: %s\n", validationErr.Field)
    }
}
```

## Advanced Function Patterns

### Functional Options

```go
type Client struct {
    timeout time.Duration
    retries int
    baseURL string
}

type ClientOption func(*Client)

func WithTimeout(timeout time.Duration) ClientOption {
    return func(c *Client) {
        c.timeout = timeout
    }
}

func WithRetries(retries int) ClientOption {
    return func(c *Client) {
        c.retries = retries
    }
}

func NewClient(baseURL string, opts ...ClientOption) *Client {
    client := &Client{
        baseURL: baseURL,
        timeout: 30 * time.Second, // defaults
        retries: 3,
    }
    
    for _, opt := range opts {
        opt(client)
    }
    
    return client
}

// Usage
client := NewClient("https://api.example.com",
    WithTimeout(60*time.Second),
    WithRetries(5),
)
```

### Method Chaining

```go
type QueryBuilder struct {
    query strings.Builder
    args  []interface{}
}

func NewQuery() *QueryBuilder {
    return &QueryBuilder{}
}

func (q *QueryBuilder) Select(fields string) *QueryBuilder {
    q.query.WriteString("SELECT " + fields + " ")
    return q
}

func (q *QueryBuilder) From(table string) *QueryBuilder {
    q.query.WriteString("FROM " + table + " ")
    return q
}

func (q *QueryBuilder) Where(condition string, args ...interface{}) *QueryBuilder {
    q.query.WriteString("WHERE " + condition + " ")
    q.args = append(q.args, args...)
    return q
}

func (q *QueryBuilder) Build() (string, []interface{}) {
    return q.query.String(), q.args
}

// Usage
query, args := NewQuery().
    Select("id, name, email").
    From("users").
    Where("age > ? AND active = ?", 18, true).
    Build()
```

### Middleware Pattern

```go
type Handler func(http.ResponseWriter, *http.Request)

type Middleware func(Handler) Handler

func withLogging(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next(w, r)
        log.Printf("%s %s - %v", r.Method, r.URL.Path, time.Since(start))
    }
}

func withAuth(next Handler) Handler {
    return func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }
        next(w, r)
    }
}

// Chain middlewares
func chainMiddleware(h Handler, middlewares ...Middleware) Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

// Usage
handler := chainMiddleware(
    myHandler,
    withLogging,
    withAuth,
)
```

## Best Practices Summary

### Do's ✅

1. **Use pointer receivers for**:
   - Methods that modify the receiver
   - Large structs (>100 bytes)
   - Consistency across all methods

2. **Handle the append trap**:
   - Always reassign when appending
   - Copy slices when needed
   - Use full slice expressions to limit capacity

3. **Design functions well**:
   - Accept interfaces, return concrete types
   - Keep parameter count low (use structs for many params)
   - Put context first, options last

4. **Error handling**:
   - Wrap errors with context
   - Use sentinel errors for expected conditions
   - Return errors as the last return value

### Don'ts ❌

1. **Don't mix receiver types** - be consistent
2. **Don't ignore the append trap** - it causes subtle bugs  
3. **Don't use too many parameters** - use structs or options
4. **Don't panic in library code** - return errors instead

### Performance Tips 🚀

1. **Use pointer receivers for large structs** to avoid copying
2. **Pre-allocate slices with known capacity** using `make([]T, 0, capacity)`
3. **Avoid string concatenation in loops** - use `strings.Builder`
4. **Consider sync.Pool for frequently allocated objects**

## Key Takeaways

1. **Receiver choice matters**: Use pointers for modification, large structs, or consistency
2. **The append trap is real**: Always be careful when appending to slices
3. **Good function design**: Accept interfaces, limit parameters, handle errors properly
4. **Patterns help**: Use builder, options, and middleware patterns for flexibility

## Next Steps

- Read [Concurrency Fundamentals](04-concurrency-fundamentals.md) to learn about goroutines and channels
- Study [Interfaces & Composition](07-interfaces-composition.md) for advanced interface patterns
- Learn [Error Handling](09-error-handling.md) for robust error management

---

*"Clear is better than clever."* - Go Proverb 