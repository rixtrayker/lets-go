# Data Types & Structures in Go

Understanding Go's type system and when to use each structure is fundamental to writing efficient, maintainable code.

## The Three-Question Framework

Before writing any function signature, ask these questions:

### 1. The Modification Question
**"Do I want this function to change my original data?"**

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

```go
type SmallStruct struct {
    ID   int
    Name string
}

type LargeStruct struct {
    Data [1024 * 1024]byte  // 1MB
}

// Small: pass by value
func processSmall(s SmallStruct) {}

// Large: pass by pointer
func processLarge(s *LargeStruct) {}
```

### 3. The Optionality Question
**"Does this value need to represent 'absence'?"**

```go
// Pointer for nil-ability
func findUser(id int) *User {
    if found {
        return &User{ID: id}
    }
    return nil
}

// Go idiom: value + boolean
func findUserIdiomatic(id int) (User, bool) {
    if found {
        return User{ID: id}, true
    }
    return User{}, false
}
```

## Data Types Overview

| Type | Pass By | Memory | Use Case |
|------|---------|--------|----------|
| **Primitives** | Value | Stack | Simple data |
| **Arrays** | Value | Stack/Heap | Fixed-size collections |
| **Slices** | Value | Heap | Dynamic arrays |
| **Maps** | Value | Heap | Key-value pairs |
| **Structs** | Value/Pointer | Stack/Heap | Custom types |
| **Interfaces** | Value | Stack/Heap | Contracts |

## Slices: Dynamic Arrays

```go
// Declaration
var s []int                    // nil slice
var s2 = []int{}              // Empty slice  
var s3 = []int{1, 2, 3}       // Initialized
var s4 = make([]int, 5)       // Length 5
var s5 = make([]int, 0, 10)   // Length 0, capacity 10

// The append trap - DANGEROUS!
func dangerousAppend(s []int) []int {
    return append(s, 4)  // Might modify original!
}

// Safe approach
func safeAppend(s []int) []int {
    return append(s[:], 4)  // Copy first
}
```

## Maps: Hash Tables

```go
// Declaration
var m map[string]int                    // nil map - DANGEROUS!
var m2 = make(map[string]int)          // Safe empty map
var m3 = map[string]int{"key": 42}     // Initialized

// Safe operations
if value, ok := m3["key"]; ok {
    fmt.Printf("Found: %d\n", value)
}

delete(m3, "key")
```

## Structs: Custom Types

### Value vs Pointer Receivers

```go
type Counter struct {
    count int
}

// Value receiver - read-only
func (c Counter) GetCount() int {
    return c.count
}

// Pointer receiver - can modify
func (c *Counter) Increment() {
    c.count++
}

// Use pointer receivers when:
// 1. Method modifies the struct
// 2. Struct is large (>100 bytes)
// 3. For consistency across methods
```

## Interfaces: Contracts

```go
// Define interfaces where you USE them
package api

type EmailSender interface {
    SendEmail(to, subject, body string) error
}

type UserService struct {
    emailSender EmailSender  // Dependency injection
}

// Implementation in separate package
package email

import "myapp/api"

type Client struct{}

func (c Client) SendEmail(to, subject, body string) error {
    // Implementation
    return nil
}
```

## Special Cases: Reference Types

Slices and maps behave differently - they contain pointers to underlying data:

```go
func modifySlice(s []int) {
    s[0] = 999  // DOES modify original's underlying array
}

func modifyMap(m map[string]int) {
    m["key"] = 999  // DOES modify original map
}

// Always pass slices/maps by value - they're already "reference-like"
func processUsers(users []User) {}        // ✅ Correct
func badProcessUsers(users *[]User) {}    // ❌ Unnecessary
```

## Decision Cheat Sheet

| Scenario | Choice | Example |
|----------|--------|---------|
| Simple calculation | Value | `func add(a, b int) int` |
| Large struct, read-only | Pointer | `func process(u *User)` |
| Need to modify | Pointer | `func update(u *User)` |
| Collection processing | Value | `func filter([]Item) []Item` |
| Optional values | Pointer | `func find(int) *User` |

## Best Practices

1. **Use the three-question framework** for every parameter
2. **Slices/maps**: Always pass by value (already reference-like)
3. **Consistency**: If any method uses pointer receiver, use for all
4. **Zero values**: Design types to be useful without initialization
5. **Small interfaces**: Define where you use them, keep focused

---

Next: [Functions & Methods](03-functions-methods.md)