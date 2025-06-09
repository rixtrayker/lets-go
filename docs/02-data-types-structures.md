# Data Types & Structures in Go

Understanding Go's type system and when to use each data structure is fundamental to writing efficient, bug-free, and maintainable code. This guide provides a deeper, more practical look beyond the basics.

## Table of Contents
- [Data Types \& Structures in Go](#data-types--structures-in-go)
  - [Table of Contents](#table-of-contents)
  - [The Three-Question Framework](#the-three-question-framework)
    - [1. The Modification Question](#1-the-modification-question)
    - [2. The Performance Question](#2-the-performance-question)
    - [3. The Optionality Question](#3-the-optionality-question)
  - [Slices: The Workhorse Collection](#slices-the-workhorse-collection)
    - [The `len` vs. `cap` Trap](#the-len-vs-cap-trap)
  - [Maps: Key-Value Power](#maps-key-value-power)
    - [Iteration Order is Random](#iteration-order-is-random)
  - [Strings, Bytes, and Runes](#strings-bytes-and-runes)
  - [Best Practices \& Tricky Parts](#best-practices--tricky-parts)
  - [Advanced Type Concepts](#advanced-type-concepts)
    - [Type Assertions and Type Switches](#type-assertions-and-type-switches)
    - [Custom Types and Type Aliases](#custom-types-and-type-aliases)
  - [Memory Layout \& Performance](#memory-layout--performance)
    - [Struct Field Ordering](#struct-field-ordering)
    - [Slice Memory Management](#slice-memory-management)
  - [Common Interview Questions](#common-interview-questions)
    - [1. Slice vs Array](#1-slice-vs-array)
    - [2. Map Internals](#2-map-internals)
    - [3. String Immutability](#3-string-immutability)
    - [4. Interface Implementation](#4-interface-implementation)
    - [5. Zero Values](#5-zero-values)

## The Three-Question Framework

Before passing a variable to a function, ask yourself these three questions. The answers will guide you to the correct data passing strategy (value vs. pointer).

### 1. The Modification Question
**"Do I want this function to change my original data?"**

This is the most common reason to use a pointer. If the function's purpose is to mutate the state of the caller's variable, you *must* use a pointer.

-   **Pass by Value (Immutable):** A copy of the data is made. Changes inside the function do not affect the original variable. This is the default and safest option.
-   **Pass by Pointer (Mutable):** A pointer to the original data is passed. Changes made through the pointer directly affect the original variable.

```go
// Pass by VALUE (to prevent changes)
func printUser(u User) {
    u.Name = "Modified"  // Only changes the copy
    fmt.Println(u)
}

// Pass by POINTER (to allow changes)  
func updateUser(u *User) {
    u.Name = "Modified"  // Changes the original
}
```

### 2. The Performance Question
**"Is this data expensive to copy?"**

Copying large structs can impact performance. Passing a pointer (typically 8 bytes) is much cheaper than copying a 1MB struct. However, this is not a simple rule.

-   **Trade-off:** Passing by value often allows the data to be stored on the **stack**, which is very fast. Passing a pointer might cause the data to **escape to the heap**, which involves the garbage collector and can sometimes be *slower* overall for small to medium-sized structs.
-   **Rule of Thumb:** Don't prematurely optimize. Start with passing by value unless the struct is demonstrably large (e.g., > 100 bytes is a common heuristic). Profile your code if you suspect a performance issue.

```go
type LargeStruct struct {
    Data [1024 * 1024]byte  // 1MB
}

// Large: pass by pointer to avoid a costly copy
func processLarge(s *LargeStruct) {}
```

### 3. The Optionality Question
**"Does this value need to represent 'absence'?"**

A pointer's `nil` value is a clear, idiomatic way to signal that a value is not present. This is common for function arguments that are optional or for return values that might not exist.

-   **Pointer for `nil`-ability:** A pointer can be `nil`, making it a good choice for optional values.
-   **Go Idiom (Value + `bool`):** For return values, the "comma, ok" idiom is often preferred as it's more explicit about why the value is absent (e.g., not found vs. an error).

```go
// Pointer for an optional argument
func Greet(user *User) {
    if user == nil {
        fmt.Println("Hello, guest!")
        return
    }
    fmt.Printf("Hello, %s!\n", user.Name)
}

// Go idiom for a return value that may not exist
func findUser(id int) (User, bool) {
    user, found := userDatabase[id] // Assume userDatabase is a map[int]User
    return user, found
}
```

## Slices: The Workhorse Collection

Slices are a wrapper around a Go array, providing a dynamic and powerful way to manage contiguous sequences of elements.

### The `len` vs. `cap` Trap

A slice has both a length (`len`) and a capacity (`cap`). Forgetting this is a common source of bugs.
- `len`: The number of elements the slice currently holds.
- `cap`: The number of elements in the underlying array, starting from the slice's first element.

```go
// A slice with len=3, cap=5
// Underlying array: [10, 20, 30, 40, 50]
// Slice `s`:        |-- 10, 20, 30 --|
s := make([]int, 3, 5)
s[0], s[1], s[2] = 10, 20, 30

// `append` within capacity: MODIFIES the underlying array.
s2 := append(s, 99) 
// Underlying array: [10, 20, 30, 99, 50]
// s:  [10, 20, 30] - len 3, cap 5
// s2: [10, 20, 30, 99] - len 4, cap 5. s2 shares the same underlying array!

// `append` beyond capacity: Allocates a NEW array.
s3 := append(s, 1, 2, 3) 
// New underlying array: [10, 20, 30, 1, 2, 3, ?, ?] (capacity is doubled)
// s3 now points to a completely different block of memory.
```

**Golden Rule:** The return value of `append` may or may not be the same as the input slice. **Always assign the result back to the original slice variable:** `s = append(s, ...)`.

## Maps: Key-Value Power

Maps are hash tables that store key-value pairs. They are unordered collections.

### Iteration Order is Random

**Never rely on the iteration order of a map.** Go deliberately randomizes map iteration to prevent developers from depending on a specific order. If you need a stable order, you must enforce it yourself.

```go
m := map[string]int{"beta": 2, "alpha": 1, "gamma": 3}

// Unordered iteration (order will change on each run)
for k, v := range m {
    fmt.Printf("k: %v, v: %v\n", k, v)
}

// Ordered iteration
var keys []string
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys) // Sort the keys
for _, k := range keys {
    fmt.Printf("k: %v, v: %v\n", k, m[k]) // Access map in key-sorted order
}
```

**Concurrency:** Maps are **not safe** for concurrent read/write operations. If you need to access a map from multiple goroutines, you must use a `sync.Mutex` or `sync.RWMutex` to protect it.

## Strings, Bytes, and Runes

This is a critical, often misunderstood topic in Go.

-   **`string`**: An immutable sequence of bytes.
-   **`[]byte`**: A mutable sequence of bytes.
-   **`rune`**: An alias for `int32`. It represents a single Unicode code point.

A common pitfall is treating a string as an array of characters. In Go, indexing a string accesses individual **bytes**, not characters (runes).

```go
s := "Hello, 世界" // "世界" are 3 bytes each in UTF-8

// Loop 1: Iterating by BYTES (incorrect for Unicode)
for i := 0; i < len(s); i++ {
    fmt.Printf("%c ", s[i]) // Prints garbage for multi-byte runes
}
// Output: H e l l o ,   ä ¸ ­ ç晫 

// Loop 2: Iterating by RUNES (correct way)
for _, r := range s {
    fmt.Printf("%c ", r)
}
// Output: H e l l o ,   世 界 
```
**Golden Rule:** When processing characters in a string, always use a `for...range` loop, which correctly decodes UTF-8 runes. If you need to modify the "string," convert it to a `[]rune` slice, modify it, and then convert it back.

## Best Practices & Tricky Parts

1.  **Zero Values are Your Friend**: Design your structs so their zero value is useful. This avoids the need for constructor functions. For example, a `sync.Mutex` is ready to use immediately after declaration (`var mu sync.Mutex`).

2.  **Struct Embedding for Composition**: Use struct embedding to "borrow" functionality, not for inheritance. This promotes composition over inheritance. Be aware of method promotion and potential ambiguities.

    ```go
    type Job struct {
        Name string
        *log.Logger // Embedded field
    }
    
    // The Logger's methods are "promoted" to the Job type.
    j := Job{Name: "MyJob", Logger: log.New(os.Stdout, "", 0)}
    j.Println("Job starting...") // Calling a method from the embedded Logger
    ```

3.  **Use Struct Tags**: Struct tags are string literals that provide metadata to your struct fields. They are heavily used by packages like `encoding/json`, `database/sql`, and ORMs to control how data is marshaled and unmarshaled.

    ```go
    type User struct {
        Username string `json:"username" db:"user_name"`
        Password string `json:"-"` // The '-' tag tells the json package to ignore this field
    }
    ```

4.  **Slices/Maps are Reference-Like**: Remember that while slices and maps are passed by value, the value itself is a header/pointer to a shared underlying data structure. Modifying the elements of a slice or map within a function *will* affect the caller. You rarely need a pointer to a slice (`*[]T`) or map (`*map[K]V`).

5.  **Define Interfaces Where You Use Them**: Follow the Go proverb: "Accept interfaces, return structs." This keeps your code decoupled. Define small, single-method interfaces in the package that *consumes* the functionality, not the one that provides it.

## Advanced Type Concepts

### Type Assertions and Type Switches

Type assertions and switches are powerful tools for working with interfaces.

```go
// Type assertion
var i interface{} = "hello"
s, ok := i.(string)  // s = "hello", ok = true
n, ok := i.(int)     // n = 0, ok = false

// Type switch
switch v := i.(type) {
case string:
    fmt.Printf("string: %s\n", v)
case int:
    fmt.Printf("int: %d\n", v)
default:
    fmt.Printf("unknown type: %T\n", v)
}
```

### Custom Types and Type Aliases

```go
// Type alias (introduced in Go 1.9)
type MyInt = int  // MyInt is exactly the same as int

// Custom type
type MyCustomInt int  // MyCustomInt is a new type based on int
```

## Memory Layout & Performance

### Struct Field Ordering

The order of fields in a struct can affect memory usage due to padding.

```go
// Bad: 24 bytes due to padding
type Bad struct {
    a bool     // 1 byte
    b int64    // 8 bytes
    c bool     // 1 byte
}

// Good: 16 bytes (no padding)
type Good struct {
    a bool     // 1 byte
    c bool     // 1 byte
    b int64    // 8 bytes
}
```

### Slice Memory Management

Understanding slice memory management is crucial for performance.

```go
// Pre-allocate when you know the size
s := make([]int, 0, 1000)  // len=0, cap=1000

// Avoid unnecessary allocations
func process(items []int) {
    // Bad: Creates new slice
    for _, item := range items[:] {
        // ...
    }
    
    // Good: Uses existing slice
    for _, item := range items {
        // ...
    }
}
```

## Common Interview Questions

### 1. Slice vs Array
Q: What's the difference between a slice and an array in Go?
A: Arrays have fixed size, slices are dynamic. Arrays are value types, slices are reference types.

### 2. Map Internals
Q: How do maps work internally in Go?
A: Maps are hash tables with buckets. They use a hash function to distribute keys across buckets.

### 3. String Immutability
Q: Why are strings immutable in Go?
A: Immutability provides thread safety and allows for efficient string operations and memory sharing.

### 4. Interface Implementation
Q: How does Go handle interface implementation?
A: Go uses implicit interface implementation. A type implements an interface by implementing all its methods.

### 5. Zero Values
Q: What are zero values and why are they important?
A: Zero values are default values for types when no value is specified. They help avoid null pointer issues.

---

Next: [Functions & Methods](03-functions-methods.md)