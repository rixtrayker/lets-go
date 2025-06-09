# Interfaces & Composition in Go

## Table of Contents
- [Interfaces \& Composition in Go](#interfaces--composition-in-go)
  - [Table of Contents](#table-of-contents)
  - [The Philosophy of Interfaces in Go](#the-philosophy-of-interfaces-in-go)
  - [Defining and Satisfying Interfaces](#defining-and-satisfying-interfaces)
  - [Where to Define Interfaces: The Three Scenarios](#where-to-define-interfaces-the-three-scenarios)
    - [Scenario 1: The Consumer-Side Interface (Default Choice)](#scenario-1-the-consumer-side-interface-default-choice)
    - [Scenario 2: The Shared Domain Interface](#scenario-2-the-shared-domain-interface)
    - [Scenario 3: The Public Library Interface (Rarest Case)](#scenario-3-the-public-library-interface-rarest-case)
  - [A Decision Tree for Interface Placement](#a-decision-tree-for-interface-placement)
  - [Composition: The Heart of Go's Design](#composition-the-heart-of-gos-design)
    - [Struct Embedding](#struct-embedding)
    - [Composing with Interfaces](#composing-with-interfaces)
  - [Interface Best Practices](#interface-best-practices)
  - [The Empty Interface: `any`](#the-empty-interface-any)
  - [Key Takeaways](#key-takeaways)

## The Philosophy of Interfaces in Go

In Go, interfaces are a way to specify behavior. They allow you to write functions that are more flexible and decoupled because they operate on behavior, not on a specific data type. Go's philosophy is that you should define interfaces where they are *used*, not where they are *implemented*. This is known as **consumer-driven interfaces** or **dependency inversion**.

> *"The bigger the interface, the weaker the abstraction."* - Rob Pike

## Defining and Satisfying Interfaces

An interface is a collection of method signatures. A type satisfies an interface by implementing all of its methods. This satisfaction is **implicit**—no `implements` keyword is needed.

```go
// 1. Define the interface
type Writer interface {
    Write(p []byte) (n int, err error)
}

// 2. Define a type that implements the interface's methods
type ConsoleWriter struct{}

func (cw ConsoleWriter) Write(p []byte) (n int, err error) {
    return fmt.Print(string(p))
}

// 3. Use the interface to write flexible functions
func writeMessage(w Writer, message string) {
    w.Write([]byte(message))
}

func main() {
    // ConsoleWriter implicitly satisfies the Writer interface
    var w Writer = ConsoleWriter{}
    writeMessage(w, "Hello, Go!")

    // The standard library's bytes.Buffer also satisfies the interface
    var buf bytes.Buffer
    writeMessage(&buf, "Hello, Buffer!")
}
```

## Where to Define Interfaces: The Three Scenarios

This is one of the most critical design decisions in a Go project. Placing interfaces correctly leads to clean, decoupled architecture. Placing them incorrectly leads to a coupled monolith.

### Scenario 1: The Consumer-Side Interface (Default Choice)

**Rule:** Define the interface in the package that **consumes** it.

This is the most common and idiomatic pattern. It keeps your packages self-contained and free of external dependencies.

**The Situation:** Your `api` package needs to send notifications, but it shouldn't know or care *how* they are sent (email, SMS, etc.).

**The Solution:** The `api` package defines its own `Notifier` interface.

**Directory Structure:**
```
/myproject
├── /api                  # The consumer package
│   ├── notifier.go       # Defines the Notifier interface
│   └── handlers.go       # Uses the Notifier interface
├── /email                # An implementation package
│   └── client.go         # Implements the api.Notifier interface
└── /cmd/server
    └── main.go           # Assembles everything
```

**Code Example:**

`myproject/api/notifier.go`:
```go
package api

// The API package defines the contract it needs.
// This is its "port" to the outside world.
type Notifier interface {
    Send(userID, message string) error
}
```

`myproject/api/handlers.go`:
```go
package api

type UserHandler struct {
    notifier Notifier // Depends on the local interface
}

func (h *UserHandler) RegisterUser( /* ... */ ) {
    // ... logic to create user ...
    h.notifier.Send("user-123", "Welcome aboard!")
}
```

`myproject/email/client.go`:
```go
package email

// Notice the import! The implementing package imports the consuming package.
// The dependency is inverted from traditional languages.
import "myproject/api"

type Client struct { /* ... */ }

// email.Client implicitly satisfies api.Notifier
func (c *Client) Send(userID, message string) error {
    // ... logic to send an email ...
    return nil
}
```

`myproject/cmd/server/main.go` (The "Wiring"):
```go
package main

import (
    "myproject/api"
    "myproject/email"
)

func main() {
    emailNotifier := &email.Client{}                 // Create the concrete type
    userHandler := api.NewUserHandler(emailNotifier) // Inject it
    // ... start server with userHandler ...
}
```

**Verdict:** This is peak Go architecture. The `api` package is completely self-contained and has zero knowledge of `email`. It is trivially testable by creating a simple mock `Notifier` in `api_test.go`. **Always strive for this first.**

### Scenario 2: The Shared Domain Interface

**Rule:** Define an interface in a central package when it represents a core, shared business concept.

**The Anti-Pattern to Avoid:** The first impulse might be to create a generic `/pkg/interfaces` directory. **Do not do this.** This creates a central "bag of contracts" that everything depends on, leading back to high coupling.

**The Correct Solution:** Create a package that represents this shared **domain**. This package contains both the core data structures (the "nouns") and the key interfaces that operate on them (the "verbs").

**The Situation:** Your `api` service and a separate `reporting` service both need to fetch and manage `User` entities.

**The Solution:** Create a central `user` domain package.

**Directory Structure:**
```
/myproject
├── /user                 # The central domain package
│   ├── user.go           # Defines the `User` struct
│   └── repository.go     # Defines the `Repository` interface
├── /postgres             # A data layer implementation
│   └── user_repo.go      # Implements the `user.Repository` interface
├── /api                  # A consumer of the domain
│   └── handlers.go       # Depends on and uses `user.Repository`
└── /reporting            # Another consumer of the domain
    └── jobs.go           # Also depends on and uses `user.Repository`
```

**Code Example:**

`myproject/user/repository.go`:
```go
package user

// The core data model of this domain.
type User struct {
    ID   int
    Name string
}

// The core behavioral contract for this domain.
// Both high-level (api) and low-level (postgres) packages will depend on this.
type Repository interface {
    FindByID(id int) (*User, error)
    Update(u *User) error
}
```

**Verdict:** Use this pattern when you have a well-defined **core business domain** that is fundamental to multiple parts of your application. The key is that this central package should be very stable and change infrequently. It represents the core concepts of your system.

### Scenario 3: The Public Library Interface (Rarest Case)

**Rule:** Define an interface in its own standalone package when creating a universal plug-in architecture for public consumption.

**The Situation:** You are writing a major framework or library and want to define a contract for the entire Go ecosystem to use.

**The Canonical Examples:**
* **`io.Reader` and `io.Writer`**: The most successful interfaces in Go. They allow any code to work with any data stream.
* **`database/sql.Driver`**: Allows any database vendor to create a driver that works with Go's standard database tools.
* **`net/http.Handler`**: Allows any web framework to create components that plug into Go's standard HTTP server.

**Verdict:** You should only do this if you are a **library author** creating a public, stable, and widely-used extension point. **You are probably not writing the next `database/sql`.** For internal application code, always prefer Scenario 1 or 2.

## A Decision Tree for Interface Placement

1.  **Is this interface only needed by one consumer package (e.g., my `api` needs a `Notifier`)?**
    *   **YES** → **Define the interface in that consumer package (`api`)**. This is the default, most decoupled choice. (Scenario 1)

2.  **Is the *exact same interface and its associated data types* needed by multiple, distinct consumer packages (e.g., `api` and a `worker` both need to manage `Users`)?**
    *   **YES** → **Define the interface and structs in a central *domain* package** (e.g., a `user` package). (Scenario 2)
    *   **NO** → **STOP.** Are you sure it's the *exact* same interface? Perhaps the `api` needs a `UserFinder` and the `worker` needs a `UserUpdater`. In that case, they are two different contracts. Define them separately in their respective consumer packages (back to Scenario 1). **Don't prematurely create a shared interface.**

3.  **Am I writing a public library to create a universal plug-in system for the world?**
    *   **YES** → A standalone interface package might be appropriate. (Scenario 3)
    *   **NO** → You almost certainly want Scenario 1 or 2.

## Composition: The Heart of Go's Design

Go favors composition over inheritance. You build complex types by assembling simpler ones.

### Struct Embedding

Struct embedding allows you to "borrow" the fields and methods of another type.

```go
type Logger struct {
    Prefix string
}

func (l *Logger) Log(message string) {
    fmt.Printf("%s: %s\n", l.Prefix, message)
}

type Server struct {
    Logger // Embed the Logger
    Host   string
    Port   int
}

func main() {
    server := &Server{
        Logger: Logger{Prefix: "SERVER"},
        Host:   "localhost",
        Port:   8080,
    }

    // We can call the embedded type's methods directly.
    server.Log("Starting up...") // "SERVER: Starting up..."
}
```

### Composing with Interfaces

Interfaces are a powerful way to compose behaviors.

```go
type Reader interface { /* ... */ }
type Writer interface { /* ... */ }
type Closer interface { /* ... */ }

// ReadWriter is composed of Reader and Writer
type ReadWriter interface {
    Reader
    Writer
}

// ReadWriteCloser is composed of all three
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

## Interface Best Practices

1.  **Keep Interfaces Small.** Single-method interfaces are very common in Go (`io.Reader`, `io.Writer`, `http.Handler`). They are easy to implement and highly reusable.
2.  **Accept Interfaces, Return Structs.** This is a common Go proverb. It means your functions should be liberal in what they accept (any type that satisfies the behavior) but conservative in what they return (a specific, concrete type).
3.  **Don't Create Interfaces Prematurely.** Don't define an interface until you have at least two different concrete types that will use it. Start with a concrete type, and only introduce an interface when you need the abstraction.
4.  **The Name Should Describe the Behavior.** Good interface names are often suffixed with "-er" (e.g., `Reader`, `Writer`, `Logger`, `Notifier`).

## The Empty Interface: `any`

The empty interface, `interface{}` (or its alias `any` in modern Go), can hold a value of any type. It represents an unknown type.

```go
// `any` can hold any value
var anything any
anything = 42
anything = "hello"

// Use a type switch to determine the concrete type
func process(v any) {
    switch val := v.(type) {
    case int:
        fmt.Printf("It's an integer: %d\n", val)
    case string:
        fmt.Printf("It's a string: %s\n", val)
    default:
        fmt.Printf("It's an unknown type: %T\n", val)
    }
}
```

**Warning:** Use `any` sparingly. It bypasses Go's static type safety. It's useful for things like JSON marshalling or data structures that must hold heterogeneous types, but it often indicates that a better, more type-safe design is possible, perhaps with generics.

## Key Takeaways

1.  **Interfaces describe behavior, not data.**
2.  **Define interfaces in the consuming package**, not the implementing one. This is the cornerstone of dependency inversion in Go.
3.  **Favor small, focused interfaces** over large, monolithic ones.
4.  **Use struct embedding for composition**, not inheritance.
5.  **Only introduce an interface when you need an abstraction** for multiple concrete types.

---

Next: [Design Patterns in Go](08-design-patterns.md) 