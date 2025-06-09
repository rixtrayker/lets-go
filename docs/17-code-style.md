# Code Style & Linting in Go

## Table of Contents
- [Code Style \& Linting in Go](#code-style--linting-in-go)
  - [Table of Contents](#table-of-contents)
  - [The Go Philosophy of Code Style](#the-go-philosophy-of-code-style)
  - [Official Formatting Tool: `gofmt`](#official-formatting-tool-gofmt)
    - [What is `gofmt`?](#what-is-gofmt)
    - [`goimports`](#goimports)
  - [Effective Go: The Style Guide](#effective-go-the-style-guide)
    - [Key Highlights from Effective Go](#key-highlights-from-effective-go)
  - [Code Comments: The Go Way](#code-comments-the-go-way)
  - [Linting: Beyond Formatting](#linting-beyond-formatting)
    - [What is a Linter?](#what-is-a-linter)
    - [The Standard Linter: `golangci-lint`](#the-standard-linter-golangci-lint)
  - [An Opinionated `golangci.yml`](#an-opinionated-golangciyml)
  - [Common Style Gotchas](#common-style-gotchas)
  - [Best Practices for Style and Linting](#best-practices-for-style-and-linting)

## The Go Philosophy of Code Style

Go takes a strong, opinionated stance on code formatting. The goal is to eliminate debates about style, allowing developers to focus on writing code. There is one official style, and powerful tools enforce it automatically.

**Key Principles:**
-   **One True Style:** There is a canonical format for Go code.
-   **Automation:** Tooling should handle formatting, not humans.
-   **Readability:** The chosen style prioritizes code that is easy to read and understand.
-   **Simplicity:** The rules are straightforward and universally applied.

## Official Formatting Tool: `gofmt`

### What is `gofmt`?

`gofmt` (or `go fmt`) is a command-line tool, built into the Go toolchain, that automatically formats Go source code according to the standard style.

It handles all aspects of formatting:
-   Indentation (tabs, not spaces)
-   Spacing around operators
-   Alignment of struct fields
-   Sorting of imports
-   And much more.

The Go community does not argue about brace style or indentation. They just run `gofmt`.

**How to Use It:**
```bash
# Format a single file and write the result to stdout
gofmt my_file.go

# Format a single file and overwrite it in place
gofmt -w my_file.go

# Format all .go files in the current directory and subdirectories
go fmt ./...
```
It is standard practice to configure your code editor to run `gofmt` (or `goimports`) on every file save.

### `goimports`

`goimports` is a tool that does everything `gofmt` does, but with an additional feature: it automatically adds missing `import` statements and removes unused ones.

It is the de facto standard in the Go community and should be used in place of `gofmt`.

**How to Install and Use:**
```bash
# Install the tool
go install golang.org/x/tools/cmd/goimports@latest

# Use it just like gofmt
goimports -w my_file.go
```

## Effective Go: The Style Guide

While `gofmt` handles the *formatting*, "Effective Go" is the official document that describes the idiomatic *style* and conventions for writing Go code. It's a must-read for any Go developer.

[Read Effective Go Here](https://go.dev/doc/effective_go)

It covers naming, control structures, concurrency patterns, error handling, and more.

### Key Highlights from Effective Go

-   **`MixedCaps` for Names:** Use `MixedCaps` or `mixedCaps` for multi-word names, not underscores. Whether the first letter is capitalized determines if the name is exported (public) or unexported (private).
-   **Interfaces:** Interface names should typically end with `er` (e.g., `Reader`, `Writer`, `Formatter`).
-   **Package Names:** Should be short, concise, and all lowercase. Avoid underscores or `mixedCaps`.
-   **Keep It Short:** Short, descriptive variable names are preferred. For example, use `i` for a loop index, not `index`. Use `r` for a reader, not `myReader`.
-   **Line Length:** Go has no official line length limit. Let `gofmt` handle it. Readable code is more important than fitting within an arbitrary character count.
-   **Naked Returns:** Avoid using "naked" returns (a `return` statement without arguments in a function with named return values) except in the shortest of functions. They can harm readability.

## Code Comments: The Go Way

Go has a specific commenting style that is enforced by convention and tooling.

- **Package Comments:** Every package should have a package comment, a comment preceding the `package` clause. For multi-file packages, the package comment only needs to be in one file.
  ```go
  // Package sort provides primitives for sorting slices and user-defined collections.
  package sort
  ```
- **Documenting Exports:** Every exported name—variable, constant, function, or struct—should have a doc comment. The comment should be a full sentence and start with the name of the thing it's describing.
  ```go
  // Regexp is a compiled representation of a regular expression.
  // A Regexp is safe for concurrent use by multiple goroutines.
  type Regexp struct {
      ...
  }
  ```
- **Use `//` not `/* */`:** Block comments (`/* */`) are rarely used in Go. Stick to `//` for all comments.

## Linting: Beyond Formatting

Formatting ensures your code looks right. Linting ensures your code *is* right.

### What is a Linter?

A linter is a static analysis tool that checks your code for programmatic and stylistic errors, potential bugs, and non-idiomatic constructs that `gofmt` wouldn't catch.

**Examples of what a linter can find:**
-   Unused variables or parameters.
-   Code that will cause a panic (e.g., a `nil` pointer deference).
-   Shadowed variables.
-   Sub-optimal or non-idiomatic constructs.
-   Code that is unnecessarily complex.

### The Standard Linter: `golangci-lint`

`golangci-lint` is an extremely popular and powerful linter for Go. It's a "meta-linter" that runs many different individual linters at once, is very fast, and is highly configurable. It is the unofficial standard for linting in the Go community.

**How to Install and Use:**
```bash
# Install on macOS
brew install golangci-lint

# Or on other platforms
go install github.com/golangci/golangci-lint/cmd/golangci-lint@v1.56.2
```

**Configuration:**
Create a `.golangci.yml` file in the root of your project.

`.golangci.yml`:
```yaml
run:
  # The default timeout is 1m, which may be too short for a large codebase.
  timeout: 3m

linters:
  # Enable all default linters and some extras.
  # Run `golangci-lint help linters` to see all available options.
  enable:
    - gofmt
    - goimports
    - revive
    - errcheck
    - staticcheck
    - unused
    - govet
    # Add more linters you find useful.

linters-settings:
  # Example: configure the line-length linter (if you choose to enable it).
  lll:
    line-length: 120

issues:
  # Don't fail the build for new issues in generated code.
  exclude-rules:
    - path: _test\.go
      linters:
        - funlen # Don't complain about long test functions.
    - path: .*/zz_generated.*\.go
      linters:
        - all
```

**Running the Linter:**
```bash
golangci-lint run ./...
```
This should be a standard part of your CI/CD pipeline.

## An Opinionated `golangci.yml`

A good starting point for a project configuration file (`.golangci.yml`) is to enable the default linters plus a few other high-value ones.

```yaml
run:
  timeout: 3m

linters:
  enable:
    # --- Default linters ---
    - errcheck      # Checks for unchecked errors. A must-have.
    - govet         # Analyzes code for suspicious constructs.
    - staticcheck   # Huge collection of powerful static analysis checks.
    - unused        # Checks for unused code.
    # --- Recommended extras ---
    - goimports     # Enforces `goimports` formatting.
    - revive        # A faster, more configurable replacement for golint.
    - ineffassign   # Detects when assignments to variables are not used.
    - dogsled       # Finds redundant assignments.
    - goconst       # Finds repeated strings that could be constants.
    - gocritic      # Provides various checks for style, performance, etc.
    - unconvert     # Removes unnecessary type conversions.

linters-settings:
  gocritic:
    # Enable all diagnostic checks. These are usually the most valuable.
    enabled-tags:
      - diagnostic

issues:
  # Don't fail the build for new issues in generated code or tests.
  exclude-rules:
    - path: _test\.go
      linters:
        - funlen    # Don't complain about long test functions.
        - goconst   # Test data often has repeated strings.
    - path: .*/zz_generated.*\.go
      linters:
        - all
```

**Running the Linter:**
```bash
# Install or update
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
# Run it
golangci-lint run ./...
```

## Common Style Gotchas

- **Error Strings:** Error strings should not be capitalized or end with punctuation. They are often joined with other context.
  ```go
  // YES
  return fmt.Errorf("something bad happened")
  
  // NO
  return fmt.Errorf("Something bad happened.")
  ```
- **Handler Naming:** HTTP handler types should be named `someHandler`, not `SomeHandler`. They are often not exported. The function that returns the handler (`NewSomeHandler`) is the exported part.
- **Variable Declaration:** Use `:=` for initialization when possible to keep code concise. Use `var` only when you need to declare a variable without initializing it to its zero value.

## Best Practices for Style and Linting

1.  **Format on Save:** Configure your editor to run `goimports` every time you save a file. This is the single most important habit to adopt.
2.  **Use `golangci-lint`:** Integrate it into your CI pipeline to catch issues before they make it into your main branch.
3.  **Read "Effective Go":** Understand the "why" behind Go's idioms. It will make you a better Go developer.
4.  **Don't Argue About Style:** If you have a style question, the answer is whatever `gofmt` produces.
5.  **Listen to the Linter:** If the linter flags something, try to understand why. It's usually pointing out a real potential issue or a piece of code that could be clearer.

---

Next: [The Go Community](18-community-resources.md) 