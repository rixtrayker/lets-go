# Building and Deploying Go Applications

## Table of Contents
- [The `go build` Command](#the-go-build-command)
  - [Basic Usage](#basic-usage)
  - [Cross-Compilation](#cross-compilation)
  - [Optimizing for Size and Performance](#optimizing-for-size-and-performance)
- [Deployment Strategies](#deployment-strategies)
  - [Strategy 1: The Single Binary](#strategy-1-the-single-binary)
  - [Strategy 2: The Minimal Docker Image](#strategy-2-the-minimal-docker-image)
- [Working with Configuration](#working-with-configuration)
  - [Configuration via Environment Variables](#configuration-via-environment-variables)
  - [Configuration via Files](#configuration-via-files)
- [Graceful Shutdown](#graceful-shutdown)
- [Deployment Best Practices](#deployment-best-practices)

## The `go build` Command

The `go build` command compiles your Go packages and their dependencies into an executable binary.

### Basic Usage

When run inside a project directory, `go build` will create an executable in that same directory.

```bash
# Inside /my-project/cmd/api
go build

# This creates an executable named 'api' in the current directory.
# You can run it with:
./api
```

You can also specify the path to the package you want to build.
```bash
# From the project root
go build ./cmd/api
```

To name the output file, use the `-o` flag.
```bash
go build -o my-awesome-api ./cmd/api
```

### Cross-Compilation

One of Go's killer features is its ability to easily cross-compile for different operating systems and architectures from any machine. This is done by setting the `GOOS` (target Operating System) and `GOARCH` (target Architecture) environment variables.

**Example: Build a Linux binary from a Mac.**
```bash
GOOS=linux GOARCH=amd64 go build -o my-api-linux ./cmd/api
```

**Common Targets:**
-   `linux/amd64`: Standard Linux servers.
-   `linux/arm64`: ARM-based Linux servers (e.g., AWS Graviton).
-   `darwin/amd64`: Intel-based Macs.
-   `darwin/arm64`: Apple Silicon Macs.
-   `windows/amd64`: Windows servers.

You can see a full list of supported targets with `go tool dist list`.

### Optimizing for Size and Performance

You can use build flags to create smaller binaries, which is particularly useful for containerized deployments.

-   `-ldflags="-s -w"`: This is the most common optimization.
    -   `-s`: Strips the symbol table.
    -   `-w`: Strips the DWARF debugging information.

```bash
# Build a small Linux binary
GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o my-api-small ./cmd/api
```
This can reduce the binary size by 50% or more with minimal impact on performance.

## Deployment Strategies

### Strategy 1: The Single Binary

The simplest way to deploy a Go application is to copy the compiled binary to a server and run it.

**Steps:**
1.  **Build:** `GOOS=linux GOARCH=amd64 go build -o my-app ./cmd/app`
2.  **Copy:** Use `scp` or another tool to copy `my-app` and any necessary config files to your server.
3.  **Run:** On the server, run `./my-app`. It's recommended to run it as a `systemd` service so it restarts on failure or server reboot.

**Pros:** Simple, fast, minimal dependencies on the server (just the OS).
**Cons:** Managing server setup and updates can be manual.

### Strategy 2: The Minimal Docker Image

This is the most common and recommended strategy for modern cloud-native applications. Go is perfect for Docker because its statically linked binaries don't require any OS-level dependencies.

We can use a **multi-stage Dockerfile** to create a tiny, secure container image.

`Dockerfile`:
```dockerfile
# ---- Build Stage ----
# Use the official Go image as the builder.
FROM golang:1.21-alpine AS builder

# Set the working directory inside the container.
WORKDIR /app

# Copy the Go module files and download dependencies.
COPY go.mod go.sum ./
RUN go mod download

# Copy the rest of the source code.
COPY . .

# Build the application, stripping debug info for a smaller binary.
# CGO_ENABLED=0 is important for creating a static binary.
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /main ./cmd/api


# ---- Final Stage ----
# Use a minimal, non-root base image. `scratch` is the smallest possible.
FROM scratch

# Copy the compiled binary from the build stage.
COPY --from=builder /main /main

# Copy any necessary config files or static assets.
# COPY --from=builder /app/configs /configs

# Expose the port the application runs on.
EXPOSE 8080

# Set the binary as the entrypoint.
ENTRYPOINT ["/main"]
```

**Why this is great:**
-   **Tiny Size:** The final image is often just 5-10 MB, as it only contains your binary.
-   **Secure:** It contains no shell, no package manager, and no other tools, drastically reducing the attack surface.
-   **Portable:** This image can be deployed to any container orchestrator like Kubernetes, ECS, or Docker Swarm.

## Working with Configuration

Never hardcode configuration like database passwords or API keys in your code.

### Configuration via Environment Variables

This is the standard 12-Factor App methodology and works perfectly with containerized deployments.

```go
package config

import "os"

type Config struct {
    Port     string
    DB_DSN   string
}

func Load() *Config {
    return &Config{
        Port:   getEnv("PORT", "8080"),
        DB_DSN: getEnv("DB_DSN", "user:pass@tcp(localhost:5432)/db"),
    }
}

func getEnv(key, fallback string) string {
    if value, ok := os.LookupEnv(key); ok {
        return value
    }
    return fallback
}
```

### Configuration via Files

Libraries like `Viper` can read configuration from files (YAML, TOML, JSON), environment variables, and remote config systems, providing a powerful and flexible solution.

## Graceful Shutdown

When a service is asked to shut down (e.g., during a new deployment), it should stop accepting new requests but finish processing any in-flight requests before exiting.

This is typically done by listening for the `SIGINT` (Ctrl+C) and `SIGTERM` (sent by orchestrators) signals.

```go
func main() {
    // ... setup http server `srv` ...

    // Channel to listen for OS signals
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

    // Start server in a goroutine
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()

    // Block until a signal is received
    <-quit
    log.Println("Shutting down server...")

    // Create a context with a timeout to allow for cleanup
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    // Attempt a graceful shutdown
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }

    log.Println("Server exiting")
}
```

## Deployment Best Practices

1.  **Use Multi-Stage Dockerfiles:** Create small, secure, and efficient container images.
2.  **Configure with Environment Variables:** Follow 12-Factor App principles for portability.
3.  **Implement Graceful Shutdown:** Ensure zero-downtime deployments and prevent data corruption.
4.  **Add Health Check Endpoints:** Expose a `/healthz` endpoint that orchestrators can use to check if your application is alive and ready to serve traffic.
5.  **Cross-Compile with Confidence:** Use `GOOS` and `GOARCH` to build for your target environment easily.

---

Next: [Security Best Practices](16-security.md) 