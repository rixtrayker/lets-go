# Dependency Management in Go

## Table of Contents
- [Dependency Management in Go](#dependency-management-in-go)
  - [Table of Contents](#table-of-contents)
  - [Go Modules: The Official Standard](#go-modules-the-official-standard)
  - [Key Files: `go.mod` and `go.sum`](#key-files-gomod-and-gosum)
    - [The `go.mod` File](#the-gomod-file)
    - [The `go.sum` File](#the-gosum-file)
  - [How Go Modules Work: Minimal Version Selection](#how-go-modules-work-minimal-version-selection)
  - [Common `go mod` Commands](#common-go-mod-commands)
    - [`go mod init`](#go-mod-init)
    - [`go get`](#go-get)
    - [`go mod tidy`](#go-mod-tidy)
    - [`go mod why`](#go-mod-why)
    - [`go mod vendor`](#go-mod-vendor)
  - [The `replace` Directive: Forks and Local Development](#the-replace-directive-forks-and-local-development)
  - [Best Practices for Dependency Management](#best-practices-for-dependency-management)

## Go Modules: The Official Standard

Since Go 1.11, **Go Modules** have been the official and standard way to manage dependencies in a Go project. They solve the problems of versioning and package distribution that existed in earlier dependency management systems (like `dep` or `govendor`).

**Key Features:**
-   **Decentralized:** Dependencies are fetched directly from their source repositories (e.g., GitHub).
-   **Versioning:** Go Modules use semantic versioning (SemVer) to specify dependency versions.
-   **Reproducible Builds:** The `go.mod` and `go.sum` files ensure that you get the exact same dependency versions every time you build your project.
-   **Minimal Version Selection:** A deterministic algorithm for resolving dependency versions.
-   **Integrated Tooling:** The `go` command has built-in support for working with modules.

## Key Files: `go.mod` and `go.sum`

When you initialize a module in your project, two critical files are created.

### The `go.mod` File

This is the manifest for your module. It's the most important dependency management file.

**Example `go.mod`:**
```
module github.com/my-org/my-project

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/google/uuid v1.6.0
    golang.org/x/sync v0.6.0
)

require (
    github.com/bytedance/sonic v1.11.3 // indirect
    github.com/twitchyliquid64/golang-asm v0.15.1 // indirect
)
```

**Sections Explained:**
-   `module`: Defines the module path for your project. This is how other projects will import it.
-   `go`: Specifies the minimum version of Go required to compile your module.
-   `require`: Lists the direct dependencies of your project, along with their specific versions.
-   `indirect`: The second `require` block lists indirect dependencies—dependencies of your dependencies. These are managed automatically by the Go toolchain.

### The `go.sum` File

This is a lock file that contains the expected cryptographic checksums of the source code for every dependency (direct and indirect).

**Purpose:** To ensure that the dependencies you download have not been tampered with and are the exact same versions used to build the project previously.

**Example entry in `go.sum`:**
```
github.com/gin-gonic/gin v1.9.1 h1:GINi/3fVp5q9EuD2+V2aB2Ua6CQsB94Aro0Hi3L/l5k=
github.com/gin-gonic/gin v1.9.1/go.mod h1:A8B8r2CLA2k2tt+vJfg+4Gg2bKEe+xKa6iQ3A3eMneo=
```

-   Each dependency version has two entries: one for the `.zip` file of its source and one for its `go.mod` file.
-   You should **always commit your `go.sum` file** to your version control system. This is what guarantees reproducible builds for everyone on your team.

## How Go Modules Work: Minimal Version Selection

Unlike other package managers that might get the latest compatible version of a dependency, Go uses a simpler, more predictable algorithm called **Minimal Version Selection (MVS)**.

**How it works:**
- If your project requires `A v1.1` and one of your dependencies requires `A v1.2`, Go will select `A v1.2` because it's the *highest* of the required versions.
- It will **not** automatically upgrade to `A v1.3` if it exists. It selects the *minimal* version that satisfies all requirements.

This makes builds more stable and deterministic. Your build won't suddenly break because a dependency of a dependency released a new, buggy version. You only get new versions when you explicitly ask for them via `go get`.

## Common `go mod` Commands

You interact with Go Modules through the `go` command.

### `go mod init`

Initializes a new module in the current directory. You must provide the module path.

```bash
# Start a new project
go mod init github.com/my-org/my-new-project
```

### `go get`

Adds or changes a dependency.

```bash
# Add a new dependency (or update to the latest version)
go get github.com/google/uuid

# Get a specific version
go get github.com/gin-gonic/gin@v1.8.0

# Get the latest commit from a branch
go get github.com/gin-gonic/gin@master
```
After running `go get`, the `go.mod` and `go.sum` files are updated automatically.

### `go mod tidy`

This is one of the most useful commands. It "tidies up" your `go.mod` and `go.sum` files.

-   **Removes** any dependencies that you are no longer using in your code.
-   **Adds** any dependencies that are used in your code but are missing from `go.mod`.

You should run `go mod tidy` before committing your code to ensure your module files are clean and up-to-date.

```bash
go mod tidy
```

### `go mod why`

A diagnostic tool that explains why a particular package is included in the build.

```bash
go mod why github.com/twitchyliquid64/golang-asm
```

### `go mod vendor`

This command creates a `vendor` directory in your project and copies all your dependencies into it. The Go compiler will then use the packages in `vendor` instead of fetching them from the network.

**When to Use:**
-   In environments with no network access.
-   For projects that require strict auditing of all source code.
-   To guarantee that dependencies are never removed from source control (e.g., if a GitHub repo is deleted).

```bash
# Create the vendor directory
go mod vendor

# To build using the vendored packages
go build -mod=vendor
```

For most projects, vendoring is not necessary. The module cache and `go.sum` provide sufficient reliability.

## The `replace` Directive: Forks and Local Development

The `replace` directive in `go.mod` allows you to override the location of a dependency. This is extremely useful in two scenarios:

1.  **Using a Forked Repo:** If you need to use a forked version of a library with your custom changes.
2.  **Local Development:** If you are developing two modules simultaneously and need one to use the local, on-disk version of the other, not the one from its remote repository.

**Example `go.mod` with `replace`:**
```go
module example.com/my-app

go 1.22

require (
    github.com/some/dependency v1.2.3
)

**Special Rule for Major Versions > 1:** If a library releases a `v2` or higher, its module path must be updated to include the major version. For example, `github.com/some/lib/v2`. This allows a project to import `v1` and `v2` of the same library side-by-side if needed.

## Working with Private Repositories

To use private repositories (e.g., on GitHub) as dependencies, you need to configure `git` to use SSH and tell Go how to access them using the `GOPRIVATE` environment variable.

```bash
# Tell git to use SSH for GitHub instead of HTTPS
git config --global url."git@github.com:".insteadOf "https://github.com/"

# Tell Go that any modules under your organization's path are private
export GOPRIVATE="github.com/my-private-org/*"
```
Now, `go get` will use your SSH credentials to fetch private modules.

## Best Practices for Dependency Management

1.  **Always Commit `go.mod` and `go.sum`:** This is non-negotiable. It ensures reproducible and secure builds.
2.  **Use `go mod tidy`:** Run it before you commit to keep your module files clean.
3.  **Specify Versions:** When adding a new dependency, use a specific, tagged version (`go get lib@v1.2.3`) instead of just `latest`. This prevents surprises.
4.  **Update Dependencies Deliberately:** Run `go list -u -m all` to see available updates. Don't just `go get -u ./...`. Update one dependency at a time and run your tests to ensure nothing broke.
5.  **Use a `tools.go` File:** For build-time tools (like linters or code generators), create a file like `tools.go` to track them as versioned dependencies without polluting your main module's dependency graph.
   `tools.go`:
   ```go
   //go:build tools
   
   package tools
   
   import (
       _ "golang.org/x/tools/cmd/stringer"
       _ "github.com/golangci/golangci-lint/cmd/golangci-lint"
   )
   ```
   Now, `go mod tidy` won't remove them.
6.  **Scan for Vulnerabilities:** Regularly run `govulncheck ./...` as part of your CI pipeline to find known security vulnerabilities in your dependency tree.

---

Next: [Build & Deploy](15-build-deploy.md) 