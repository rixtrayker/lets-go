# Testing in Go

## Table of Contents
- [Testing in Go](#testing-in-go)
  - [Table of Contents](#table-of-contents)
  - [The Go Philosophy of Testing](#the-go-philosophy-of-testing)
  - [Unit Tests](#unit-tests)
    - [The Basics: `_test.go` files and `TestXxx`](#the-basics-_testgo-files-and-testxxx)
    - [Table-Driven Tests](#table-driven-tests)
    - [Test Coverage](#test-coverage)
  - [Test Doubles: Mocks \& Stubs](#test-doubles-mocks--stubs)
    - [Manual Mocks with Interfaces](#manual-mocks-with-interfaces)
    - [Mocking Libraries](#mocking-libraries)
  - [Integration Tests](#integration-tests)
  - [Concurrency Testing](#concurrency-testing)
    - [The Race Detector](#the-race-detector)
    - [Using `t.Parallel()` for Faster Tests](#using-tparallel-for-faster-tests)
    - [The `errgroup` Package](#the-errgroup-package)
  - [Best Practices for Testing](#best-practices-for-testing)
  - [What's Next](#whats-next)

## The Go Philosophy of Testing

Testing is a first-class citizen in Go. The language's standard library provides a lightweight, built-in testing framework that is simple to use and encourages writing tests alongside your code.

**Key Principles:**
-   **Simplicity:** The `testing` package provides the core tools. No complex setup or third-party assertion libraries are needed.
-   **Convention over Configuration:** `go test` automatically finds and runs your tests based on file and function naming conventions.
-   **Tests as Code:** Tests are written in Go, just like your application code.
-   **Parallelism:** Tests are designed to be run in parallel easily.

## Unit Tests

Unit tests focus on a single unit of code, like a function or a method, in isolation.

### The Basics: `_test.go` files and `TestXxx`

-   Tests for `mypackage/code.go` live in `mypackage/code_test.go`.
-   Test functions are named `func TestXxx(t *testing.T)`. `Xxx` should start with a capital letter.
-   You run tests with the `go test` command.

`math.go`:
```go
package math

func Add(a, b int) int {
    return a + b
}
```

`math_test.go`:
```go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5

    if result != expected {
        // t.Errorf fails the test but continues execution.
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}
```

### Table-Driven Tests

This is the idiomatic way to test multiple cases for the same function. It keeps your test logic concise and makes it easy to add new test cases.

`math_test.go`:
```go
package math

import "testing"

func TestAddTable(t *testing.T) {
    testCases := []struct {
        name   string
        a, b   int
        want int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed numbers", 2, -3, -1},
        {"zero", 0, 0, 0},
    }

    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            got := Add(tc.a, tc.b)
            if got != tc.want {
                t.Errorf("Add(%d, %d) = %d; want %d", tc.a, tc.b, got, tc.want)
            }
        })
    }
}
```

### Test Coverage

You can measure your test coverage to see what percentage of your code is executed by your tests.

```bash
# Run tests and generate a coverage profile
go test -coverprofile=coverage.out

# View the coverage profile in your browser
go tool cover -html=coverage.out
```

This will open an HTML report showing which lines of your code were covered (green), uncovered (red), or partially covered. The goal is not 100% coverage, but to use coverage as a tool to find untested parts of your code.

## Test Doubles: Mocks & Stubs

When your code has external dependencies (like a database or a third-party API), you need to replace them with "test doubles" to keep your unit tests isolated and fast.

### Manual Mocks with Interfaces

The best way to enable mocking is to use interfaces. Your code should depend on interfaces, not concrete types.

`notifier.go`:
```go
package app

// The interface our application code depends on.
type Notifier interface {
    Send(message string) error
}

type Service struct {
    notifier Notifier
}

func (s *Service) DoSomething() error {
    return s.notifier.Send("work done")
}
```

`notifier_test.go`:
```go
package app

import "testing"

// This is our mock (or stub). It implements the Notifier interface.
type mockNotifier struct {
    // We can add fields to control its behavior or inspect its usage.
    sendCalled bool
    shouldFail bool
}

func (m *mockNotifier) Send(message string) error {
    m.sendCalled = true
    if m.shouldFail {
        return errors.New("failed to send")
    }
    return nil
}

func TestService_DoSomething(t *testing.T) {
    mock := &mockNotifier{}
    service := Service{notifier: mock}

    err := service.DoSomething()

    if err != nil {
        t.Errorf("Expected no error, got %v", err)
    }
    if !mock.sendCalled {
        t.Error("Expected Send to be called, but it was not")
    }
}
```

### Mocking Libraries

For complex interfaces, writing manual mocks can be tedious. Libraries can generate mock implementations for you.
- **`gomock`**: The official mocking framework from Google.
- **`testify/mock`**: A popular alternative from the `testify` suite.

## Integration Tests

Integration tests verify that different parts of your system work together correctly. These tests are slower and more complex than unit tests because they often require real dependencies like a database or a network service.

**How to structure them:**
- Use a `_test` package name (e.g., `package myapp_test`) to test from a client's perspective.
- Use Go's build tags to separate them from unit tests.

`db_integration_test.go`:
```go
//go:build integration

package myapp_test

import (
    "testing"
    // ... imports for your database driver and app code
)

func TestUserCreation_Integration(t *testing.T) {
    // Setup: Connect to a real (test) database.
    db := setupTestDB(t) 
    
    // Run the test logic that interacts with the database.
    // ...
    
    // Teardown: Clean up the database.
    teardownTestDB(t, db)
}
```

Run only the integration tests:
```bash
go test -v -tags=integration
```

## Concurrency Testing

Go's strong support for concurrency also extends to testing.

### The Race Detector

The race detector is a powerful tool built into the Go toolchain that can detect race conditions in your code at runtime. It's essential for any concurrent Go program.

**Always run your tests with the `-race` flag.**

```bash
go test -race ./...
```

If the race detector finds a data race, it will print a detailed report and fail the test.

### Using `t.Parallel()` for Faster Tests

You can mark tests that are safe to run concurrently with `t.Parallel()`. This can significantly speed up your test suite.

```go
func TestSomething(t *testing.T) {
    t.Parallel() // This test can run alongside other parallel tests.
    // ...
}

func TestAnotherThing(t *testing.T) {
    t.Parallel()
    // ...
}
```

**Important:** When using `t.Parallel()` inside a table-driven test, you must be careful about variable scoping.

```go
// ... (inside TestAddTable)
for _, tc := range testCases {
    tc := tc // Capture range variable for each parallel test.
    t.Run(tc.name, func(t *testing.T) {
        t.Parallel() // Now each sub-test can run in parallel.
        got := Add(tc.a, tc.b)
        // ...
    })
}
```

### The `errgroup` Package

The `golang.org/x/sync/errgroup` package is excellent for managing and collecting errors from a group of goroutines, which is a common pattern in concurrent tests and applications.

## Best Practices for Testing

1.  **Test Behavior, Not Implementation:** Focus on what your code *does*, not how it does it. This makes your tests more resilient to refactoring.
2.  **Use Table-Driven Tests:** They are the idiomatic standard for testing multiple scenarios.
3.  **Depend on Interfaces:** This is the single most important technique for writing testable code.
4.  **Always Use the Race Detector:** Run `go test -race ./...` as part of your standard workflow.
5.  **Write Clear Failure Messages:** A good error message tells you what went wrong, what you got, and what you expected.
6.  **Separate Unit and Integration Tests:** Use build tags to keep your fast unit tests separate from your slow integration tests.
7.  **Strive for Readability:** A test should be easy to understand. It's documentation for your code's behavior.

## What's Next

To further your understanding of testing in Go, here are some recommended resources:

1. **Official Documentation**
   - [Testing in Go](https://go.dev/doc/testing)
   - [Go Blog: Table-Driven Tests](https://go.dev/blog/table-driven-tests)
   - [Go Blog: Subtests and Sub-benchmarks](https://go.dev/blog/subtests)

2. **Books**
   - "Go Programming Language" by Alan Donovan and Brian Kernighan
   - "Go Testing Recipes" by Lee Campbell

3. **Articles & Blog Posts**
   - [Learn Go with Tests](https://quii.gitbook.io/learn-go-with-tests/)
   - [Test Coverage in Go](https://blog.golang.org/cover)

4. **Practice & Tools**
   - [Go by Example: Testing](https://gobyexample.com/testing-and-benchmarking)
   - [Go Playground](https://play.golang.org/)
   - [Testify](https://github.com/stretchr/testify)
   - [Gomock](https://github.com/golang/mock)

5. **Related Topics to Explore**
   - Mocking and Stubbing
   - Integration Testing
   - Benchmarking
   - Test Automation in CI/CD

---

Next: [Project Structure](13-project-structure.md) 