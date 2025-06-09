# Reflection & Generics in Go

## Table of Contents
- [Reflection \& Generics in Go](#reflection--generics-in-go)
  - [Table of Contents](#table-of-contents)
  - [The High-Level Philosophy](#the-high-level-philosophy)
  - [The Reflection Toolbox (`reflect` package)](#the-reflection-toolbox-reflect-package)
    - [Good Use Cases for Reflection](#good-use-cases-for-reflection)
    - [The Dangers of Reflection](#the-dangers-of-reflection)
    - [Reflection Examples](#reflection-examples)
  - [The Generics Toolbox (Type Parameters)](#the-generics-toolbox-type-parameters)
    - [Good Use Cases for Generics](#good-use-cases-for-generics)
    - [The Dangers of Generics](#the-dangers-of-generics)
    - [Generics Examples](#generics-examples)
      - [Generic Function](#generic-function)
      - [Generic Data Structure](#generic-data-structure)
      - [Constraints](#constraints)
  - [Head-to-Head: Generics vs. Reflection](#head-to-head-generics-vs-reflection)
  - [When to Use Which](#when-to-use-which)
  - [Key Takeaways](#key-takeaways)
  - [What's Next](#whats-next)

## The High-Level Philosophy

Both reflection and generics address the problem of writing code that can work with multiple different types. However, they do so in fundamentally different ways.

-   **Generics (Type Parameters):** This is the **compile-time** way to write type-agnostic code. It provides full type safety. You tell the compiler what types your function will work with, and it generates specialized code for you. **This should be your default choice.**

-   **Reflection (`reflect` package):** This is the **run-time** way to work with types. It allows you to inspect and manipulate objects of unknown types. It sacrifices compile-time safety and performance for ultimate flexibility. **Use this as a last resort.**

> **Golden Rule:** If you can solve your problem with generics, use generics. If you can't, consider if there's a simpler design using interfaces. Only use reflection when you have no other choice.

## The Reflection Toolbox (`reflect` package)

Reflection allows a program to inspect its own structure at runtime. In Go, this means inspecting an `interface{}` value to discover its underlying type and value.

### Good Use Cases for Reflection

Reflection is powerful and necessary for certain types of problems where the type isn't known at compile time.

1.  **JSON Marshalling/Unmarshalling:** The `encoding/json` package must be able to serialize and deserialize arbitrary structs without knowing their structure beforehand.
2.  **Database ORMs:** An ORM needs to map database rows to struct fields dynamically.
3.  **Validation Libraries:** A validation library can inspect struct tags and fields to apply validation rules.
4.  **Dependency Injection Frameworks:** Frameworks can inspect types to automatically wire up dependencies.

### The Dangers of Reflection

1.  **No Compile-Time Type Safety:** `reflect.Value` operations can fail at runtime, causing a panic.
2.  **Poor Performance:** Reflection is significantly slower than direct code execution due to the overhead of type lookups and dynamic dispatch.
3.  **Reduced Readability:** Reflection-heavy code can be difficult to understand and debug. The flow of control is less explicit.
4.  **Panics:** Incorrect use of reflection (e.g., trying to set an unaddressable value) will cause a panic.

### Reflection Examples

The two most important types in the `reflect` package are `reflect.Type` and `reflect.Value`.

```go
func inspect(v any) {
    // reflect.TypeOf gets the dynamic type
    t := reflect.TypeOf(v)
    // reflect.ValueOf gets the dynamic value
    val := reflect.ValueOf(v)

    fmt.Printf("Type: %s, Kind: %s\n", t.Name(), t.Kind())

    if t.Kind() == reflect.Struct {
        fmt.Println("Fields:")
        for i := 0; i < t.NumField(); i++ {
            field := t.Field(i)
            value := val.Field(i)
            fmt.Printf("  - %s (%s): %v\n", field.Name, field.Type, value.Interface())
        }
    }
}

type User struct {
    ID   int
    Name string
}

func main() {
    u := User{ID: 1, Name: "Alice"}
    inspect(u)
    // Output:
    // Type: User, Kind: struct
    // Fields:
    //   - ID (int): 1
    //   - Name (string): Alice
}
```

## The Generics Toolbox (Type Parameters)

Generics, introduced in Go 1.18, allow you to write functions and data structures that work with a set of types.

### Good Use Cases for Generics

Generics are best suited for code that implements a common algorithm for multiple types, where the algorithm is the same regardless of the type.

1.  **General-Purpose Data Structures:** Writing a linked list, a binary tree, or a heap that can store any type.
2.  **Utility Functions:**
    -   `Map`, `Filter`, `Reduce` functions for slices.
    -   A `Keys` function to get all keys from any map type.
    -   A `Min` or `Max` function that works for any ordered type.
3.  **Type-Safe Abstractions:** Creating abstractions that need to work with channels of different types, etc.

### The Dangers of Generics

1.  **Over-Generalization:** Don't use generics just because you can. Sometimes a simple interface is a clearer and better solution.
2.  **Interface Pollution:** If a generic function only needs to call a single method on the type, an interface is often more idiomatic and flexible.
3.  **Slower Compile Times:** Heavily generic code can increase compile times as the compiler must instantiate the function for each type it's used with.

### Generics Examples

#### Generic Function

This `Map` function works on a slice of any type `T` and returns a slice of any type `R`.

```go
// [T any, R any] are the type parameters.
func Map[T any, R any](input []T, f func(T) R) []R {
    result := make([]R, len(input))
    for i, v := range input {
        result[i] = f(v)
    }
    return result
}

func main() {
    ints := []int{1, 2, 3}
    strs := Map(ints, func(i int) string {
        return strconv.Itoa(i)
    })
    // strs is []string{"1", "2", "3"}
}
```

#### Generic Data Structure

A `Stack` that can hold values of any type.

```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T // The zero value for type T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func main() {
    intStack := &Stack[int]{}
    intStack.Push(10)
    
    stringStack := &Stack[string]{}
    stringStack.Push("hello")
}
```

#### Constraints

Constraints are interfaces that specify what methods or properties a generic type must have.

```go
// The `Number` constraint allows any type that is an integer or a float.
type Number interface {
    ~int | ~int64 | ~float32 | ~float64
}

// This function only works with types that satisfy the Number constraint.
func Sum[T Number](values []T) T {
    var total T
    for _, v := range values {
        total += v
    }
    return total
}
```

## Head-to-Head: Generics vs. Reflection

| Feature             | Generics                                | Reflection                              |
| ------------------- | --------------------------------------- | --------------------------------------- |
| **Type Safety**     | ✅ **Compile-time** checked             | ❌ **Run-time** checked (panics on error) |
| **Performance**     | ✅ **Fast** (comparable to native code)   | ❌ **Slow** (10-100x slower)            |
| **Readability**     | ✅ **Good** (types are explicit)        | ❌ **Poor** (logic is less obvious)     |
| **Use Case**        | Algorithms for known type sets          | Algorithms for unknown types            |
| **When to use?**    | **First choice** for type-agnostic code | **Last resort** when flexibility is paramount |

## When to Use Which

1.  **Do you need to operate on a collection (slice, map) of some type `T`?**
    *   → Use **Generics**. This is the prime use case.

2.  **Is the code an algorithm that is identical for multiple numeric types?**
    *   → Use **Generics** with a constraint like `constraints.Ordered`.

3.  **Are you writing code that inspects or modifies an arbitrary struct at runtime (e.g., a JSON marshaller)?**
    *   → Use **Reflection**. Generics cannot help you here.

4.  **Do you just need to call a specific method on different types?**
    *   → Use an **Interface**. This is often simpler and more idiomatic than a generic function with a constraint.

    ```go
    // Interface is better here
    type Stringer interface {
        String() string
    }
    func Print(s Stringer) { fmt.Println(s.String()) }

    // Generics are overkill
    func PrintGeneric[T Stringer](v T) { fmt.Println(v.String()) }
    ```

## Key Takeaways

-   **Generics are for writing type-safe code that operates on multiple, known types at compile time.** They are your first choice for general-purpose data structures and functions.
-   **Reflection is for working with types that are unknown until run time.** It is powerful but unsafe and slow. Its use should be confined to frameworks, marshalling, and other meta-programming tasks.
-   **Interfaces are for abstracting over behavior.** If your logic only depends on a few methods, an interface is often the clearest and most flexible solution.

## What's Next

To further your understanding of reflection and generics in Go, here are some recommended resources:

1. **Official Documentation**
   - [Go Blog: Generics in Go](https://go.dev/blog/intro-generics)
   - [Go Blog: The Laws of Reflection](https://go.dev/blog/laws-of-reflection)
   - [Go 1.18 Release Notes (Generics)](https://go.dev/doc/go1.18#generics)

2. **Books**
   - "Go Programming Language" by Alan Donovan and Brian Kernighan
   - "Mastering Go" by Mihalis Tsoukalos

3. **Articles & Blog Posts**
   - [Generics in Go: The Ultimate Guide](https://blog.devgenius.io/generics-in-go-the-ultimate-guide-2e5e7b3c8b3c)
   - [Reflection in Go](https://blog.golang.org/laws-of-reflection)

4. **Practice & Tools**
   - [Go by Example: Generics](https://gobyexample.com/generics)
   - [Go Playground](https://play.golang.org/)

5. **Related Topics to Explore**
   - Type Constraints
   - Interface vs. Generics
   - Performance Implications of Reflection and Generics

---

Next: [The Unsafe Package](11-unsafe-cgo.md) 