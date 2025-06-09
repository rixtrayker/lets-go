# Concurrency Fundamentals in Go

Go's concurrency model is built around goroutines and channels, following the principle:

> *"Don't communicate by sharing memory; share memory by communicating."*

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

## Key Takeaways

1. **Goroutines are cheap** - use them liberally but responsibly
2. **Channels for communication** - safer than shared memory
3. **Close channels in sender** - never in receiver
4. **Use select for timeouts** and non-blocking operations
5. **Choose right tool**: channels vs mutexes vs atomic

---

Next: [Common Concurrency Mistakes](05-concurrency-mistakes.md) 