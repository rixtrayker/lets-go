# Performance & Profiling in Go

## Table of Contents
- [Go's Memory Allocation Deep Dive](#gos-memory-allocation-deep-dive)
- [The Go Garbage Collector](#the-go-garbage-collector)
- [Profiling Tools](#profiling-tools)
- [Performance Optimization Techniques](#performance-optimization-techniques)
- [Benchmarking](#benchmarking)
- [Memory Optimization](#memory-optimization)
- [CPU Optimization](#cpu-optimization)
- [Best Practices](#best-practices)

## Go's Memory Allocation Deep Dive

### The Memory Hierarchy

Go's memory allocation system has three main levels that work together for efficient memory management:

#### 1. Arenas
- Large contiguous blocks of memory (64MB on 64-bit systems)
- Managed by the OS, acquired as needed
- Form the foundation of Go's memory management
- Subdivided into spans

#### 2. Spans (`mspan`)
- Subdivisions of arenas that manage objects of specific size classes
- Each span contains objects of the same size
- Contains metadata about allocation state
- Linked together in lists for efficient management

#### 3. Size Classes
- Go has ~70 predefined size classes (8 bytes, 16 bytes, 32 bytes, etc.)
- Objects are rounded up to the nearest size class
- Reduces fragmentation and improves allocation speed
- Enables efficient memory pooling

### The Allocation Process

#### A. Small Object Allocation (< 32KB)

The allocation follows a three-tier hierarchy for performance:

```go
// 1. P-local cache check (mcache)
// Each processor (P) has a local cache of spans
// This is the fastest path - no locks needed

// 2. Central cache check (mcentral) 
// If local cache misses, check the central list
// Requires lightweight locking

// 3. Heap allocation (mheap)
// If central cache misses, allocate from the heap
// Most expensive path, requires global locking
```

#### B. Large Object Allocation (> 32KB)
- Allocated directly from the heap (`mheap`)
- Uses dedicated spans 
- More expensive due to size and direct heap interaction
- Bypasses the size class system

#### C. Stack vs. Heap (Escape Analysis)

The Go compiler performs escape analysis to determine allocation location:

```go
func stackAllocation() {
    var x int = 42  // Allocated on stack - doesn't escape
    fmt.Println(x)
}

func heapAllocation() *int {
    var x int = 42  // Allocated on heap - escapes via return
    return &x
}

// Check escape analysis with:
// go build -gcflags="-m" main.go
```

## The Go Garbage Collector

### The Tri-Color Mark-and-Sweep Algorithm

Go uses a concurrent, tri-color mark-and-sweep garbage collector:

#### The Three Colors
- **White**: Objects not yet examined (potentially garbage)
- **Gray**: Objects examined but not fully processed
- **Black**: Objects and all their references have been examined (definitely reachable)

#### The Process
1. **Mark Phase**: Start from root objects (globals, stack variables)
2. **Color objects**: Initially all white, roots become gray
3. **Process gray objects**: Mark references, turn gray objects black
4. **Sweep Phase**: Reclaim all remaining white objects

```go
// GC can be triggered manually (usually not needed)
runtime.GC()

// Get GC statistics
var stats runtime.MemStats
runtime.ReadMemStats(&stats)
fmt.Printf("GC cycles: %d\n", stats.NumGC)
fmt.Printf("GC pause: %v\n", stats.PauseNs[0])
```

### The Write Barrier

The write barrier ensures concurrent GC correctness:

```go
// When this happens during GC:
obj1.field = obj2

// The write barrier ensures obj2 is marked if obj1 is black
// This prevents the GC from incorrectly collecting obj2
```

### Tuning GC Performance

```go
// Set GC target percentage (default is 100)
// Lower values = more frequent GC, less memory
// Higher values = less frequent GC, more memory
debug.SetGCPercent(50) // GC when heap grows 50%

// Set max heap size (Go 1.19+)
debug.SetMemoryLimit(1 << 30) // 1GB limit
```

## Profiling Tools

### CPU Profiling

```go
import (
    "os"
    "runtime/pprof"
)

func profileCPU() {
    f, err := os.Create("cpu.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()
    
    if err := pprof.StartCPUProfile(f); err != nil {
        panic(err)
    }
    defer pprof.StopCPUProfile()
    
    // Your code here
    doExpensiveWork()
}

// Analyze with: go tool pprof cpu.prof
```

### Memory Profiling

```go
func profileMemory() {
    f, err := os.Create("mem.prof")
    if err != nil {
        panic(err)
    }
    defer f.Close()
    
    // Force GC to get accurate heap profile
    runtime.GC()
    
    if err := pprof.WriteHeapProfile(f); err != nil {
        panic(err)
    }
}

// Analyze with: go tool pprof mem.prof
```

### HTTP Profiling Server

```go
import _ "net/http/pprof"

func main() {
    // Start profiling server
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Your application code
    runApplication()
}

// Access profiles at:
// http://localhost:6060/debug/pprof/profile?seconds=30  (CPU)
// http://localhost:6060/debug/pprof/heap               (Memory)
// http://localhost:6060/debug/pprof/goroutine          (Goroutines)
```

### Analyzing Profiles

```bash
# CPU profiling
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof -http=:8080 cpu.prof  # Web interface

# Memory profiling  
go tool pprof http://localhost:6060/debug/pprof/heap
go tool pprof -alloc_space http://localhost:6060/debug/pprof/heap

# Common pprof commands:
# top10        - Show top 10 functions
# list main    - Show source code for main function
# web          - Open web interface
# png          - Generate PNG graph
```

## Benchmarking

### Writing Benchmarks

```go
func BenchmarkStringConcat(b *testing.B) {
    for i := 0; i < b.N; i++ {
        result := ""
        for j := 0; j < 100; j++ {
            result += "hello"
        }
        _ = result
    }
}

func BenchmarkStringBuilder(b *testing.B) {
    for i := 0; i < b.N; i++ {
        var builder strings.Builder
        for j := 0; j < 100; j++ {
            builder.WriteString("hello")
        }
        _ = builder.String()
    }
}

// Run with: go test -bench=. -benchmem
```

### Memory Allocation Benchmarks

```go
func BenchmarkSliceAppend(b *testing.B) {
    b.ReportAllocs() // Report allocation statistics
    
    for i := 0; i < b.N; i++ {
        var s []int
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}

func BenchmarkSlicePrealloc(b *testing.B) {
    b.ReportAllocs()
    
    for i := 0; i < b.N; i++ {
        s := make([]int, 0, 1000) // Pre-allocate capacity
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}
```

### Benchmark Comparison

```bash
# Run benchmarks and save results
go test -bench=. -count=5 > before.txt

# After optimization
go test -bench=. -count=5 > after.txt

# Compare results
go install golang.org/x/perf/cmd/benchcmp@latest
benchcmp before.txt after.txt
```

## Performance Optimization Techniques

### Memory Optimization

#### Object Pooling

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 1024)
    },
}

func processData(data []byte) []byte {
    buf := bufferPool.Get().([]byte)
    defer func() {
        buf = buf[:0] // Reset length
        bufferPool.Put(buf)
    }()
    
    // Use buf for processing
    buf = append(buf, data...)
    return append([]byte(nil), buf...) // Return copy
}
```

#### Struct Field Ordering

```go
// ❌ Poor memory layout (24 bytes due to padding)
type BadStruct struct {
    a bool   // 1 byte + 7 bytes padding
    b int64  // 8 bytes
    c bool   // 1 byte + 7 bytes padding  
}

// ✅ Good memory layout (16 bytes)
type GoodStruct struct {
    b int64  // 8 bytes
    a bool   // 1 byte
    c bool   // 1 byte + 6 bytes padding
}

// Check struct size
fmt.Println("BadStruct size:", unsafe.Sizeof(BadStruct{}))   // 24
fmt.Println("GoodStruct size:", unsafe.Sizeof(GoodStruct{})) // 16
```

#### Avoiding Allocations

```go
// ❌ Creates many allocations
func badStringProcessing(items []string) string {
    result := ""
    for _, item := range items {
        result += item + ", "
    }
    return result
}

// ✅ Minimal allocations
func goodStringProcessing(items []string) string {
    if len(items) == 0 {
        return ""
    }
    
    var builder strings.Builder
    builder.Grow(len(items) * 10) // Pre-allocate approximate size
    
    for i, item := range items {
        if i > 0 {
            builder.WriteString(", ")
        }
        builder.WriteString(item)
    }
    return builder.String()
}
```

### CPU Optimization

#### Avoiding Reflection in Hot Paths

```go
// ❌ Slow: Using reflection
func badTypeSwitch(v interface{}) string {
    val := reflect.ValueOf(v)
    switch val.Kind() {
    case reflect.String:
        return val.String()
    case reflect.Int:
        return strconv.Itoa(int(val.Int()))
    default:
        return "unknown"
    }
}

// ✅ Fast: Type assertion
func goodTypeSwitch(v interface{}) string {
    switch val := v.(type) {
    case string:
        return val
    case int:
        return strconv.Itoa(val)
    default:
        return "unknown"
    }
}
```

#### Loop Optimizations

```go
// ❌ Inefficient: function call in loop condition
func badLoop(items []string) {
    for i := 0; i < len(items); i++ { // len() called each iteration
        process(items[i])
    }
}

// ✅ Efficient: cache length
func goodLoop(items []string) {
    length := len(items)
    for i := 0; i < length; i++ {
        process(items[i])
    }
}

// ✅ Even better: range loop
func bestLoop(items []string) {
    for _, item := range items {
        process(item)
    }
}
```

#### Map Performance

```go
// ❌ Slow: String concatenation as key
func badMapKey(m map[string]int, prefix, suffix string) int {
    key := prefix + "-" + suffix // Allocation on each call
    return m[key]
}

// ✅ Fast: Pre-built keys or struct keys
type MapKey struct {
    Prefix, Suffix string
}

func goodMapKey(m map[MapKey]int, prefix, suffix string) int {
    key := MapKey{Prefix: prefix, Suffix: suffix} // No allocation
    return m[key]
}
```

## Memory Optimization Patterns

### Slice Growth Patterns

```go
// Understanding slice growth
func demonstrateSliceGrowth() {
    s := make([]int, 0, 1)
    
    for i := 0; i < 20; i++ {
        oldCap := cap(s)
        s = append(s, i)
        newCap := cap(s)
        
        if newCap != oldCap {
            fmt.Printf("Length %d: capacity grew from %d to %d\n", 
                len(s), oldCap, newCap)
        }
    }
}

// Pre-allocation when size is known
func efficientSliceBuilding(size int) []int {
    // Pre-allocate with known capacity
    result := make([]int, 0, size)
    
    for i := 0; i < size; i++ {
        result = append(result, i*i)
    }
    
    return result
}
```

### Memory Monitoring

```go
func PrintMemStats(label string) {
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    
    fmt.Printf("[%s]\n", label)
    fmt.Printf("  Alloc = %v MiB", bToMb(m.Alloc))
    fmt.Printf("  TotalAlloc = %v MiB", bToMb(m.TotalAlloc))
    fmt.Printf("  Sys = %v MiB", bToMb(m.Sys))
    fmt.Printf("  NumGC = %v\n", m.NumGC)
    fmt.Printf("  HeapObjects = %v\n", m.HeapObjects)
}

func bToMb(b uint64) uint64 {
    return b / 1024 / 1024
}

// Usage
func monitoredFunction() {
    PrintMemStats("Before")
    
    // Your code here
    doWork()
    
    runtime.GC()
    PrintMemStats("After GC")
}
```

## Best Practices

### Performance Guidelines

1. **Profile First**: Don't optimize without measuring
2. **Focus on Hot Paths**: Optimize code that runs frequently
3. **Measure Everything**: Use benchmarks to validate improvements
4. **Avoid Premature Optimization**: Write clear code first

### Memory Best Practices

```go
// ✅ Pre-allocate slices with known capacity
slice := make([]Item, 0, expectedSize)

// ✅ Reuse buffers with sync.Pool
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 1024)
    },
}

// ✅ Use string.Builder for string concatenation
var builder strings.Builder
builder.Grow(estimatedSize)

// ✅ Avoid keeping large slices for small data
func processLargeSlice(data []byte) []byte {
    // Don't return data[:10] - keeps entire array alive
    result := make([]byte, 10)
    copy(result, data[:10])
    return result
}

// ✅ Zero large structs when done
type LargeStruct struct {
    data [1024 * 1024]byte
}

func useStruct() {
    var s LargeStruct
    defer func() { s = LargeStruct{} }() // Clear when done
    // Use s...
}
```

### CPU Best Practices

```go
// ✅ Use type assertions instead of reflection
switch v := value.(type) {
case string:
    return v
case int:
    return strconv.Itoa(v)
}

// ✅ Cache expensive computations
type Cache struct {
    mu   sync.RWMutex
    data map[string]Result
}

func (c *Cache) Get(key string) (Result, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    result, ok := c.data[key]
    return result, ok
}

// ✅ Use sync.Once for expensive initialization
var (
    instance *ExpensiveResource
    once     sync.Once
)

func GetInstance() *ExpensiveResource {
    once.Do(func() {
        instance = &ExpensiveResource{
            // Expensive initialization
        }
    })
    return instance
}
```

### Profiling Checklist

1. **CPU Profile**: Find hot functions
   ```bash
   go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
   ```

2. **Memory Profile**: Find allocation hotspots
   ```bash
   go tool pprof http://localhost:6060/debug/pprof/heap
   ```

3. **Goroutine Profile**: Find goroutine leaks
   ```bash
   go tool pprof http://localhost:6060/debug/pprof/goroutine
   ```

4. **Trace Analysis**: Understand scheduling
   ```bash
   curl http://localhost:6060/debug/pprof/trace?seconds=5 > trace.out
   go tool trace trace.out
   ```

## Key Takeaways

1. **Profile before optimizing** - measure to find real bottlenecks
2. **Understand Go's memory model** - allocation patterns matter
3. **Use appropriate data structures** - maps, slices, channels have different costs
4. **Minimize allocations** - reuse objects, pre-allocate slices
5. **Monitor production performance** - use pprof in production
6. **Optimize for your use case** - different applications have different needs

## Performance Troubleshooting

### Common Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| Memory leaks | Growing heap, high GC pressure | Profile heap, check goroutine leaks |
| CPU hotspots | High CPU usage | CPU profile, optimize hot functions |
| GC pressure | Frequent GC, high pause times | Reduce allocations, tune GC |
| Lock contention | Poor scalability | Profile blocking, reduce critical sections |

### Quick Wins

1. **Pre-allocate slices**: `make([]T, 0, capacity)`
2. **Use string.Builder**: For string concatenation
3. **Pool expensive objects**: With `sync.Pool`
4. **Avoid reflection**: In hot paths
5. **Cache computations**: Store expensive results

## What's Next

To deepen your knowledge of performance and profiling in Go, explore these resources:

1. **Official Documentation**
   - [Go Performance Profiling](https://go.dev/doc/diagnostics)
   - [Go Blog: Profiling Go Programs](https://go.dev/blog/pprof)
   - [Go Memory Model](https://go.dev/ref/mem)

2. **Books**
   - "High Performance Go" by Dave Cheney (workshop)
   - "Go Programming Language" by Alan Donovan and Brian Kernighan

3. **Articles & Blog Posts**
   - [Profiling Go Programs](https://blog.golang.org/profiling-go-programs)
   - [Go Performance Optimization](https://www.ardanlabs.com/blog/2014/10/go-performance-optimizations.html)

4. **Practice & Tools**
   - [pprof Tool](https://github.com/google/pprof)
   - [Go by Example: Benchmarking](https://gobyexample.com/testing-and-benchmarking)
   - [Go Playground](https://play.golang.org/)

5. **Related Topics to Explore**
   - Garbage Collection Tuning
   - Memory and CPU Profiling
   - Benchmarking Best Practices

---

Next: [Interfaces & Composition](07-interfaces-composition.md) 