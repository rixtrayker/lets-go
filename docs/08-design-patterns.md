# Design Patterns in Go

## Table of Contents
- [Design Patterns in Go](#design-patterns-in-go)
  - [Table of Contents](#table-of-contents)
  - [The Go Philosophy on Design Patterns](#the-go-philosophy-on-design-patterns)
  - [Creational Patterns](#creational-patterns)
    - [The Factory Pattern](#the-factory-pattern)
    - [The Builder Pattern](#the-builder-pattern)
    - [The Singleton Pattern](#the-singleton-pattern)
  - [Structural Patterns](#structural-patterns)
    - [The Decorator Pattern](#the-decorator-pattern)
    - [The Adapter Pattern](#the-adapter-pattern)
  - [Behavioral Patterns](#behavioral-patterns)
    - [The Strategy Pattern](#the-strategy-pattern)
    - [The Observer Pattern](#the-observer-pattern)
  - [Go's Native Patterns: Concurrency](#gos-native-patterns-concurrency)
    - [Worker Pool](#worker-pool)
    - [Fan-out / Fan-in](#fan-out--fan-in)
    - [Pipeline](#pipeline)
  - [Best Practices for Patterns in Go](#best-practices-for-patterns-in-go)
  - [What's Next](#whats-next)

## The Go Philosophy on Design Patterns

Go's approach to design patterns is pragmatic and minimalistic. The language's built-in features, like interfaces and goroutines, often provide simpler solutions than the traditional "Gang of Four" (GoF) patterns.

**Key Principles:**
- **Simplicity:** If a simple function or struct will do, don't force a complex pattern.
- **Composition over Inheritance:** Go uses struct embedding and interfaces for composition.
- **Concurrency:** Many problems are solved elegantly using Go's native concurrency primitives.
- **Interfaces:** Small, focused interfaces are preferred for creating flexible abstractions.

## Creational Patterns

### The Factory Pattern

**Purpose:** To create objects without specifying the exact class of object that will be created.

**When to Use:** When you have multiple implementations of an interface and need to decide at runtime which one to create.

```go
// The interface
type Notifier interface {
    Send(message string) error
}

// Concrete implementations
type EmailNotifier struct{}
func (e EmailNotifier) Send(message string) error { /* ... */ return nil }

type SMSNotifier struct{}
func (s SMSNotifier) Send(message string) error { /* ... */ return nil }

// The Factory Function
func NewNotifier(notifierType string) (Notifier, error) {
    switch notifierType {
    case "email":
        return EmailNotifier{}, nil
    case "sms":
        return SMSNotifier{}, nil
    default:
        return nil, fmt.Errorf("unknown notifier type: %s", notifierType)
    }
}

func main() {
    notifier, err := NewNotifier("email")
    if err != nil {
        log.Fatal(err)
    }
    notifier.Send("Hello, Go!")
}
```

### The Builder Pattern

**Purpose:** To construct a complex object step by step.

**When to Use:** When an object has many configuration options, some of which may be optional. It provides a more readable alternative to a constructor with many parameters.

```go
type ServerConfig struct {
    Host    string
    Port    int
    Timeout time.Duration
    MaxConn int
    IsTLS   bool
}

type ServerBuilder struct {
    config ServerConfig
}

func NewServerBuilder(host string, port int) *ServerBuilder {
    return &ServerBuilder{
        config: ServerConfig{
            Host: host,
            Port: port,
            // Set default values
            Timeout: 30 * time.Second,
            MaxConn: 100,
            IsTLS:   false,
        },
    }
}

func (b *ServerBuilder) WithTimeout(timeout time.Duration) *ServerBuilder {
    b.config.Timeout = timeout
    return b
}

func (b *ServerBuilder) WithMaxConn(maxConn int) *ServerBuilder {
    b.config.MaxConn = maxConn
    return b
}

func (b *ServerBuilder) WithTLS() *ServerBuilder {
    b.config.IsTLS = true
    return b
}

func (b *ServerBuilder) Build() *Server {
    // Here you would create the actual server with the config
    return &Server{config: b.config}
}

func main() {
    server := NewServerBuilder("localhost", 8080).
        WithTimeout(60 * time.Second).
        WithTLS().
        Build()
    // Use the configured server
}
```

### The Singleton Pattern

**Purpose:** To ensure that a class has only one instance and provide a global point of access to it.

**When to Use:** For managing a shared resource, like a database connection pool or a global configuration. Use with caution as it can introduce global state.

```go
// The shared resource
type DatabaseConnection struct {
    // ... connection details
}

var (
    dbInstance *DatabaseConnection
    once       sync.Once
)

// The Singleton accessor function
func GetDatabaseInstance() *DatabaseConnection {
    once.Do(func() {
        // This function will only be executed once
        dbInstance = &DatabaseConnection{
            // ... expensive initialization logic ...
        }
    })
    return dbInstance
}
```

## Structural Patterns

### The Decorator Pattern

**Purpose:** To add new functionality to an object dynamically without altering its structure.

**When to Use:** When you want to add behavior (like logging, caching, or monitoring) to an existing component in a composable way. Interfaces are key to this pattern in Go.

```go
// The core component interface
type Doer interface {
    Do(task string) error
}

// The concrete component
type MyComponent struct{}
func (c *MyComponent) Do(task string) error {
    fmt.Printf("Doing task: %s\n", task)
    return nil
}

// The Decorator struct
type LoggingDecorator struct {
    wrapped Doer
    logger  *log.Logger
}

// The decorator also implements the same interface
func (d *LoggingDecorator) Do(task string) error {
    d.logger.Printf("Starting task: %s", task)
    err := d.wrapped.Do(task)
    if err != nil {
        d.logger.Printf("Finished task with error: %v", err)
    } else {
        d.logger.Printf("Finished task successfully")
    }
    return err
}

func main() {
    component := &MyComponent{}
    
    // Wrap the component with the decorator
    decorated := &LoggingDecorator{
        wrapped: component,
        logger:  log.New(os.Stdout, "DECORATOR: ", 0),
    }

    // Use them interchangeably
    component.Do("task1")
    decorated.Do("task2")
}
```

### The Adapter Pattern

**Purpose:** To allow incompatible interfaces to work together.

**When to Use:** When you have an existing component that you want to use, but its interface doesn't match the one required by your system.

```go
// The target interface our system expects
type MessageSender interface {
    Send(message string)
}

// An existing, "legacy" or third-party component with an incompatible interface
type LegacyPrinter struct{}
func (p *LegacyPrinter) PrintMessage(msg []byte) {
    fmt.Println(string(msg))
}

// The Adapter struct that wraps the legacy component
type PrinterAdapter struct {
    legacyPrinter *LegacyPrinter
}

// The adapter implements the target interface
func (a *PrinterAdapter) Send(message string) {
    // It translates the call to the legacy component's method
    a.legacyPrinter.PrintMessage([]byte(message))
}

func main() {
    legacyPrinter := &LegacyPrinter{}
    
    // Wrap the legacy printer in our adapter
    adapter := &PrinterAdapter{legacyPrinter: legacyPrinter}
    
    // Now we can use it as a MessageSender
    var sender MessageSender = adapter
    sender.Send("Hello, Adapter!")
}
```

## Behavioral Patterns

### The Strategy Pattern

**Purpose:** To define a family of algorithms, encapsulate each one, and make them interchangeable.

**When to Use:** When you need to select an algorithm or behavior at runtime.

```go
// The strategy interface
type EvictionStrategy interface {
    Evict(cache map[string]string)
}

// Concrete strategies
type FIFOStrategy struct{}
func (s *FIFOStrategy) Evict(cache map[string]string) { fmt.Println("Evicting using FIFO") }

type LRUStrategy struct{}
func (s *LRUStrategy) Evict(cache map[string]string) { fmt.Println("Evicting using LRU") }

// The context that uses a strategy
type Cache struct {
    storage  map[string]string
    strategy EvictionStrategy
}

func NewCache(strategy EvictionStrategy) *Cache {
    return &Cache{
        storage:  make(map[string]string),
        strategy: strategy,
    }
}

func (c *Cache) Add(key, value string) {
    if len(c.storage) > 10 { // Example capacity
        c.strategy.Evict(c.storage)
    }
    c.storage[key] = value
}

func main() {
    lruCache := NewCache(&LRUStrategy{})
    lruCache.Add("key1", "value1")
    
    fifoCache := NewCache(&FIFOStrategy{})
    fifoCache.Add("key2", "value2")
}
```

### The Observer Pattern

**Purpose:** To define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically.

**When to Use:** When a change in one object requires changing others, and you don't know how many objects need to be changed. Often implemented with channels in Go.

```go
// The Subject (the thing being observed)
type Subject struct {
    observers []chan string
}

func (s *Subject) Register(observer chan string) {
    s.observers = append(s.observers, observer)
}

func (s *Subject) Notify(event string) {
    // Notify all observers concurrently
    for _, observer := range s.observers {
        go func(obs chan string) {
            obs <- event
        }(observer)
    }
}

// The Observer (the thing that listens)
func createObserver(id int) chan string {
    observerChan := make(chan string)
    go func() {
        for event := range observerChan {
            fmt.Printf("Observer %d received event: %s\n", id, event)
        }
    }()
    return observerChan
}

func main() {
    subject := &Subject{}

    // Create and register observers
    obs1 := createObserver(1)
    obs2 := createObserver(2)
    subject.Register(obs1)
    subject.Register(obs2)
    
    // Notify observers of an event
    subject.Notify("state changed")
    time.Sleep(10 * time.Millisecond) // Allow time for goroutines to process
}
```

## Go's Native Patterns: Concurrency

Go's concurrency features provide native solutions to many common problems.

### Worker Pool

**Purpose:** To limit the number of concurrent goroutines processing work, preventing resource exhaustion.

See `04-concurrency-fundamentals.md` for a detailed example.

### Fan-out / Fan-in

**Purpose:** To parallelize a stage in a processing pipeline. Fan-out distributes work to multiple goroutines, and fan-in collects the results.

See `04-concurrency-fundamentals.md` for a detailed example.

### Pipeline

**Purpose:** To create a series of processing stages connected by channels, where each stage is a goroutine.

See `04-concurrency-fundamentals.md` for a detailed example.

## Best Practices for Patterns in Go

1.  **Start Simple:** Don't apply a pattern just for the sake of it. A simple function is often the best solution.
2.  **Leverage Interfaces:** Interfaces are the key to many structural and behavioral patterns in Go, providing the necessary decoupling.
3.  **Think Concurrently:** Before reaching for a traditional pattern, consider if channels and goroutines can solve the problem more idiomatically. The Observer pattern, for example, is often better implemented with channels.
4.  **Avoid a `/patterns` Package:** Don't centralize your pattern implementations. Integrate them naturally where they are needed in your application's architecture.
5.  **Understand the Trade-offs:** Patterns can add complexity. Ensure the flexibility you gain is worth the added code.

## What's Next

To further explore design patterns in Go, check out these resources:

1. **Official Documentation**
   - [Go Blog: Design Patterns in Go](https://go.dev/blog/pipelines)
   - [Effective Go](https://go.dev/doc/effective_go)

2. **Books**
   - "Go Design Patterns" by Mario Castro Contreras
   - "Go Programming Language" by Alan Donovan and Brian Kernighan

3. **Articles & Blog Posts**
   - [Go Patterns](https://github.com/tmrts/go-patterns)
   - [GoF Patterns in Go](https://refactoring.guru/design-patterns/go)

4. **Practice & Tools**
   - [Go by Example](https://gobyexample.com/)
   - [Go Playground](https://play.golang.org/)

5. **Related Topics to Explore**
   - Idiomatic Go Patterns
   - Concurrency Patterns
   - SOLID Principles in Go

---

Next: [Error Handling](09-error-handling.md) 