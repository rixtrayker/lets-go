# Concurrency Fundamentals in Go

Go's concurrency model is built around goroutines and channels, following the principle:

> *"Don't communicate by sharing memory; share memory by communicating."*

## Table of Contents
- [Goroutines: Lightweight Threads](#goroutines-lightweight-threads)
- [Channels: Communication](#channels-communication)
- [Common Patterns](#common-patterns)
- [Select Statement](#select-statement)
- [Go's Scheduler (GMP Model)](#gos-scheduler-gmp-model)
- [Synchronization Primitives](#synchronization-primitives)
- [Best Practices](#best-practices)
- [Advanced Concurrency Patterns](#advanced-concurrency-patterns)
- [Debugging Concurrent Code](#debugging-concurrent-code)
- [Performance Considerations](#performance-considerations)
- [Common Interview Questions](#common-interview-questions)

## Goroutines: Lightweight Threads

Goroutines are lightweight threads managed by the Go runtime.

### Creating Goroutines

```go
// Basic goroutine
go func() {
    fmt.Println("Hello from goroutine!")
}()

// Goroutine with parameters
go func(name string) {
    fmt.Printf("Hello %s!\n", name)
}("Alice")

// Waiting for goroutines
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    doWork()
}()
wg.Wait()
```

### Goroutine Characteristics

- **Lightweight**: Only ~2KB initial stack size
- **Growable**: Stack grows and shrinks as needed  
- **Multiplexed**: Many goroutines run on few OS threads
- **Cooperative**: Scheduled by Go runtime

## Channels: Communication

Channels are Go's pipes for goroutine communication.

### Channel Types

```go
ch := make(chan int)        // Unbuffered (synchronous)
buffered := make(chan int, 5) // Buffered (asynchronous)

// Channel directions
func sender(ch chan<- int) { ch <- 42 }    // Send-only
func receiver(ch <-chan int) { v := <-ch } // Receive-only
```

### Channel Operations

```go
// Sending and receiving
ch <- 42        // Send
value := <-ch   // Receive

// Closing channels
close(ch)

// Range over channel
for value := range ch {
    fmt.Println(value)
}

// Check if closed
value, ok := <-ch
if !ok {
    // Channel is closed
}
```

## Common Patterns

### Worker Pool

```go
func workerPool(jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- processJob(job)
            }
        }()
    }
    
    go func() {
        wg.Wait()
        close(results)
    }()
}
```

### Pipeline

```go
// Generate -> Square -> Print
func pipeline() {
    numbers := generate(1, 2, 3, 4, 5)
    squares := square(numbers)
    print(squares)
}

func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}
```

## Select Statement

Select lets a goroutine wait on multiple channel operations.

```go
select {
case msg1 := <-ch1:
    fmt.Println("Received from ch1:", msg1)
case msg2 := <-ch2:
    fmt.Println("Received from ch2:", msg2)
case <-time.After(1 * time.Second):
    fmt.Println("Timeout!")
default:
    fmt.Println("No channels ready")
}
```

## Go's Scheduler (GMP Model)

- **G (Goroutine)**: Lightweight thread
- **M (Machine)**: OS thread
- **P (Processor)**: Scheduling context

```
P0    P1    P2    P3
G     G     G     G
G G   G G   G     G G
│     │     │     │
M0    M1    M2    M3
```

### Scheduling Points

Goroutines yield at:
- Channel operations
- System calls
- `runtime.Gosched()`
- Function calls (occasionally)
- Memory allocations

## Synchronization Primitives

### Mutex

```go
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

### Atomic Operations

```go
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}

func load() int64 {
    return atomic.LoadInt64(&counter)
}
```

## Best Practices

### When to Use What

**Channels**: Communication, coordination, pipelines
**Mutexes**: Protecting shared state, performance-critical sections
**Atomic**: Simple counters, flags

### Common Patterns

```go
// ✅ Close channels in sender
func producer(ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// ✅ Use buffered channels to prevent leaks
ch := make(chan int, 1)
go func() {
    ch <- expensiveWork()
}()

select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(timeout):
    fmt.Println("timeout")
}

// ✅ Limit concurrent goroutines
semaphore := make(chan struct{}, maxConcurrency)
for _, item := range items {
    semaphore <- struct{}{}
    go func(item Item) {
        defer func() { <-semaphore }()
        processItem(item)
    }(item)
}
```

## Advanced Concurrency Patterns

### Context Package

The `context` package provides a way to carry deadlines, cancellation signals, and request-scoped values across API boundaries.

```go
func worker(ctx context.Context, jobs <-chan Job) {
    for {
        select {
        case job := <-jobs:
            // Process job
        case <-ctx.Done():
            return // Context cancelled or timed out
        }
    }
}

// Usage
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
go worker(ctx, jobs)
```

### Rate Limiting

```go
type RateLimiter struct {
    ticker *time.Ticker
    done   chan struct{}
}

func NewRateLimiter(rate time.Duration) *RateLimiter {
    return &RateLimiter{
        ticker: time.NewTicker(rate),
        done:   make(chan struct{}),
    }
}

func (r *RateLimiter) Wait() {
    select {
    case <-r.ticker.C:
        return
    case <-r.done:
        return
    }
}
```

## Debugging Concurrent Code

### Race Detection

```bash
go run -race main.go
```

### Deadlock Detection

```go
// Add this to your main function
go func() {
    time.Sleep(5 * time.Second)
    fmt.Println("Checking for deadlocks...")
    buf := make([]byte, 1<<16)
    runtime.Stack(buf, true)
    fmt.Printf("%s", buf)
}()
```

### Debugging Tools

1. **pprof**: For profiling goroutines
```go
import _ "net/http/pprof"

go func() {
    log.Println(http.ListenAndServe("localhost:6060", nil))
}()
```

2. **trace**: For detailed execution tracing
```go
f, err := os.Create("trace.out")
if err != nil {
    log.Fatal(err)
}
defer f.Close()
err = trace.Start(f)
if err != nil {
    log.Fatal(err)
}
defer trace.Stop()
```

## Performance Considerations

### Goroutine Pooling

```go
type Pool struct {
    jobs    chan func()
    workers int
}

func NewPool(workers int) *Pool {
    p := &Pool{
        jobs:    make(chan func()),
        workers: workers,
    }
    p.start()
    return p
}

func (p *Pool) start() {
    for i := 0; i < p.workers; i++ {
        go func() {
            for job := range p.jobs {
                job()
            }
        }()
    }
}
```

### Memory Management

1. **Avoid Goroutine Leaks**
```go
// Bad
go func() {
    // No way to stop this goroutine
    for {
        // ...
    }
}()

// Good
go func() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            // ...
        case <-done:
            return
        }
    }
}()
```

2. **Channel Buffer Sizing**
```go
// Size buffer based on expected throughput
ch := make(chan int, 1000)  // Buffer for 1000 items
```

## Common Interview Questions

### 1. Goroutines vs Threads
Q: What's the difference between a goroutine and an OS thread?
A: Goroutines are lightweight (2KB initial stack), managed by Go runtime, and multiplexed onto OS threads.

### 2. Channel Types
Q: What are the different types of channels in Go?
A: Unbuffered (synchronous) and buffered (asynchronous) channels.

### 3. Select Statement
Q: How does the select statement work?
A: It allows a goroutine to wait on multiple channel operations, executing the first one that's ready.

### 4. Context Package
Q: What is the context package used for?
A: It provides a way to carry deadlines, cancellation signals, and request-scoped values across API boundaries.

### 5. Race Conditions
Q: How do you prevent race conditions in Go?
A: Use channels for communication, mutexes for shared state, or atomic operations for simple cases.

---

Next: [Common Concurrency Mistakes](05-concurrency-mistakes.md) 