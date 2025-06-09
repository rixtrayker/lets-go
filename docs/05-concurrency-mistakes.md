# Common Concurrency Mistakes in Go

Understanding what can go wrong with concurrent code is crucial for writing robust Go applications.

## Common Concurrency Mistakes

### 1. Goroutine Leaks 💧

Goroutines that never terminate consume memory and resources.

#### The Problem

```go
// ❌ DANGEROUS: Goroutine leak
func leakyFunction() {
    ch := make(chan int)
    
    go func() {
        // This goroutine will run forever
        for {
            ch <- rand.Int()
        }
    }()
    
    // Function returns but goroutine keeps running
    return
}
```

#### The Solution

```go
// ✅ SAFE: Proper cleanup with context
func safeFunction(ctx context.Context) {
    ch := make(chan int)
    
    go func() {
        for {
            select {
            case ch <- rand.Int():
                // Send data
            case <-ctx.Done():
                return // Goroutine exits when context is cancelled
            }
        }
    }()
    
    // Use ch...
}

// ✅ SAFE: Use buffered channel for fire-and-forget
func safeFire andForget() {
    ch := make(chan int, 1) // Buffered channel
    
    go func() {
        result := expensiveComputation()
        select {
        case ch <- result:
            // Sent successfully
        default:
            // Channel full, but that's OK for fire-and-forget
        }
    }()
}
```

### 2. Race Conditions 🏁

Multiple goroutines accessing shared data without proper synchronization.

#### The Problem

```go
// ❌ DANGEROUS: Race condition
type Counter struct {
    count int
}

func (c *Counter) Increment() {
    c.count++ // Not atomic!
}

func (c *Counter) Value() int {
    return c.count // Not atomic!
}

// This will produce unpredictable results
func racyCounter() {
    counter := &Counter{}
    
    for i := 0; i < 1000; i++ {
        go counter.Increment()
    }
    
    time.Sleep(time.Second)
    fmt.Println(counter.Value()) // Unpredictable result
}
```

#### The Solution

```go
// ✅ SAFE: Using mutex
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

// ✅ SAFE: Using atomic operations
type AtomicCounter struct {
    count int64
}

func (c *AtomicCounter) Increment() {
    atomic.AddInt64(&c.count, 1)
}

func (c *AtomicCounter) Value() int64 {
    return atomic.LoadInt64(&c.count)
}
```

### 3. Channel Deadlocks 🔒

Goroutines waiting forever on channels.

#### The Problem

```go
// ❌ DEADLOCK: Unbuffered channel with no receiver
func deadlock1() {
    ch := make(chan int)
    ch <- 42 // Blocks forever - no receiver
}

// ❌ DEADLOCK: Circular dependency
func deadlock2() {
    ch1 := make(chan int)
    ch2 := make(chan int)
    
    go func() {
        <-ch1 // Waiting for ch1
        ch2 <- 1
    }()
    
    go func() {
        <-ch2 // Waiting for ch2
        ch1 <- 1
    }()
    
    // Both goroutines wait forever
}
```

#### The Solution

```go
// ✅ SAFE: Use buffered channels
func safeChannel1() {
    ch := make(chan int, 1) // Buffered
    ch <- 42 // Doesn't block
    value := <-ch
    fmt.Println(value)
}

// ✅ SAFE: Use select with timeout
func safeChannel2() {
    ch := make(chan int)
    
    go func() {
        time.Sleep(2 * time.Second)
        ch <- 42
    }()
    
    select {
    case value := <-ch:
        fmt.Println("Received:", value)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout!")
    }
}
```

### 4. Shared Memory Without Synchronization 📤

Accessing shared variables without proper synchronization.

#### The Problem

```go
// ❌ DANGEROUS: Shared map without synchronization
var sharedMap = make(map[string]int)

func unsafeMapAccess() {
    for i := 0; i < 100; i++ {
        go func(i int) {
            key := fmt.Sprintf("key%d", i)
            sharedMap[key] = i // Race condition!
        }(i)
    }
}
```

#### The Solution

```go
// ✅ SAFE: Using sync.Map
var safeMap sync.Map

func safeMapAccess() {
    for i := 0; i < 100; i++ {
        go func(i int) {
            key := fmt.Sprintf("key%d", i)
            safeMap.Store(key, i)
        }(i)
    }
}

// ✅ SAFE: Using mutex with regular map
type SafeStringMap struct {
    mu   sync.RWMutex
    data map[string]int
}

func (sm *SafeStringMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.data[key] = value
}

func (sm *SafeStringMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    value, ok := sm.data[key]
    return value, ok
}
```

## Uncommon Concurrency Mistakes

### 1. Closing Channels from Multiple Goroutines 🚪

#### The Problem

```go
// ❌ DANGEROUS: Multiple goroutines closing the same channel
func badChannelClose() {
    ch := make(chan int, 10)
    
    // Multiple producers
    for i := 0; i < 3; i++ {
        go func() {
            ch <- 42
            close(ch) // Panic: close of closed channel
        }()
    }
}
```

#### The Solution

```go
// ✅ SAFE: Only one goroutine closes the channel
func goodChannelClose() {
    ch := make(chan int, 10)
    var wg sync.WaitGroup
    
    // Multiple producers
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            ch <- 42
        }()
    }
    
    // One goroutine handles closing
    go func() {
        wg.Wait()
        close(ch) // Only closed once
    }()
    
    // Consumer
    for value := range ch {
        fmt.Println(value)
    }
}
```

### 2. Forgetting to Handle Context Cancellation 🛑

#### The Problem

```go
// ❌ BAD: Ignoring context cancellation
func ignoringContext(ctx context.Context) {
    for {
        // Long-running work
        time.Sleep(time.Second)
        doWork()
        // Context cancellation is ignored!
    }
}
```

#### The Solution

```go
// ✅ GOOD: Properly handling context
func respectingContext(ctx context.Context) {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            doWork()
        case <-ctx.Done():
            fmt.Println("Context cancelled:", ctx.Err())
            return
        }
    }
}
```

### 3. WaitGroup Mistakes 📊

#### The Problem

```go
// ❌ WRONG: WaitGroup counter mismatch
func wrongWaitGroup() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            // Missing wg.Done() - deadlock!
            doWork()
        }()
    }
    
    wg.Wait() // Waits forever
}

// ❌ WRONG: Adding to WaitGroup inside goroutine
func wrongWaitGroup2() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        go func() {
            wg.Add(1) // Race condition!
            defer wg.Done()
            doWork()
        }()
    }
    
    wg.Wait() // Might not wait for all goroutines
}
```

#### The Solution

```go
// ✅ CORRECT: Proper WaitGroup usage
func correctWaitGroup() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1) // Add before starting goroutine
        go func() {
            defer wg.Done() // Always call Done()
            doWork()
        }()
    }
    
    wg.Wait()
}
```

### 4. Mutex Copy 📋

#### The Problem

```go
// ❌ DANGEROUS: Copying mutex
type BadCounter struct {
    mu    sync.Mutex
    count int
}

func (c BadCounter) BadIncrement() { // Value receiver copies mutex!
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func badMutexCopy() {
    counter := BadCounter{}
    
    // This creates multiple copies of the mutex
    for i := 0; i < 10; i++ {
        go counter.BadIncrement() // Each call gets a copy!
    }
}
```

#### The Solution

```go
// ✅ SAFE: Always use pointer receivers with mutexes
type GoodCounter struct {
    mu    sync.Mutex
    count int
}

func (c *GoodCounter) Increment() { // Pointer receiver
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

## Debugging Concurrent Code

### Race Detection

```bash
# Run with race detector
go run -race main.go
go test -race ./...
go build -race
```

### Finding Goroutine Leaks

```go
func findGoroutineLeaks() {
    fmt.Println("Goroutines before:", runtime.NumGoroutine())
    
    // Your concurrent code here
    
    time.Sleep(time.Second) // Let goroutines finish
    runtime.GC()
    runtime.GC() // Force GC twice
    
    fmt.Println("Goroutines after:", runtime.NumGoroutine())
    // Should be the same number (or close to it)
}
```

### Profiling Goroutines

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Your application code
}

// Then visit: http://localhost:6060/debug/pprof/goroutine?debug=1
```

## Best Practices for Concurrent Code

### 1. Design for Synchronization

```go
// ✅ GOOD: Channel-based synchronization
func channelBased() {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)
    
    // Start workers
    for i := 0; i < 3; i++ {
        go worker(jobs, results)
    }
    
    // Send jobs
    go func() {
        defer close(jobs)
        for _, job := range getAllJobs() {
            jobs <- job
        }
    }()
    
    // Collect results
    for result := range results {
        handleResult(result)
    }
}

// ✅ GOOD: Mutex for shared state
func mutexBased() {
    var mu sync.RWMutex
    cache := make(map[string]string)
    
    getValue := func(key string) (string, bool) {
        mu.RLock()
        defer mu.RUnlock()
        value, ok := cache[key]
        return value, ok
    }
    
    setValue := func(key, value string) {
        mu.Lock()
        defer mu.Unlock()
        cache[key] = value
    }
}
```

### 2. Graceful Shutdown

```go
func gracefulShutdown() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Start workers
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            worker(ctx)
        }()
    }
    
    // Wait for interrupt signal
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan
    
    // Cancel context and wait for workers
    cancel()
    wg.Wait()
    
    fmt.Println("Shutdown complete")
}

func worker(ctx context.Context) {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            doWork()
        case <-ctx.Done():
            return
        }
    }
}
```

### 3. Error Handling in Goroutines

```go
type Result struct {
    Value string
    Error error
}

func errorHandling() {
    jobs := make(chan string, 10)
    results := make(chan Result, 10)
    
    // Worker that can fail
    go func() {
        defer close(results)
        for job := range jobs {
            value, err := processJob(job)
            results <- Result{Value: value, Error: err}
        }
    }()
    
    // Send jobs
    go func() {
        defer close(jobs)
        for i := 0; i < 5; i++ {
            jobs <- fmt.Sprintf("job-%d", i)
        }
    }()
    
    // Handle results and errors
    for result := range results {
        if result.Error != nil {
            fmt.Printf("Error: %v\n", result.Error)
            continue
        }
        fmt.Printf("Success: %s\n", result.Value)
    }
}
```

## Key Takeaways

1. **Always clean up goroutines** - use context for cancellation
2. **Protect shared state** - use mutexes or channels
3. **Avoid copying mutexes** - always use pointer receivers
4. **Close channels in sender** - never in receiver
5. **Use race detector** - `go run -race`
6. **Handle context cancellation** - respect context.Done()
7. **Match WaitGroup Add/Done** - add before goroutine, done in defer
8. **Use buffered channels** - to prevent goroutine leaks

## Debugging Tips

- **Use `go run -race`** to detect race conditions
- **Monitor goroutine count** with `runtime.NumGoroutine()`
- **Use pprof** to profile goroutines: `/debug/pprof/goroutine`
- **Add logging** to trace goroutine lifecycle
- **Test with high concurrency** to expose race conditions

## What's Next

To further explore concurrency mistakes and advanced concurrency in Go, check out these resources:

1. **Official Documentation**
   - [Go Blog: Concurrency Patterns](https://go.dev/blog/pipelines)
   - [Go Memory Model](https://go.dev/ref/mem)
   - [Go Race Detector](https://go.dev/doc/articles/race_detector)

2. **Books**
   - "Concurrency in Go" by Katherine Cox-Buday
   - "Go Programming Language" by Alan Donovan and Brian Kernighan

3. **Articles & Blog Posts**
   - [Common Go Concurrency Mistakes](https://medium.com/@cep21/common-go-concurrency-mistakes-and-how-to-avoid-them-4378592c4c44)
   - [Go: Goroutine Leaks](https://blog.golang.org/goroutine-leaks)

4. **Practice & Tools**
   - [Go by Example: Goroutines](https://gobyexample.com/goroutines)
   - [Go by Example: Channels](https://gobyexample.com/channels)
   - [Go Playground](https://play.golang.org/)

5. **Related Topics to Explore**
   - Deadlocks and Race Conditions
   - Profiling and Debugging Concurrent Programs
   - Advanced Channel Patterns

---

Next: [Performance & Profiling](06-performance-profiling.md) 