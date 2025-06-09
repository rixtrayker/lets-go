# Go Project Structure

## Table of Contents
- [The Philosophy of Project Layout](#the-philosophy-of-project-layout)
- [A Simple, Flat Structure (The Default)](#a-simple-flat-structure-the-default)
- [The Standard Go Project Layout](#the-standard-go-project-layout)
  - [Top-Level Directories](#top-level-directories)
  - [The `/cmd` Directory](#the-cmd-directory)
  - [The `/internal` Directory](#the-internal-directory)
  - [The `/pkg` Directory](#the-pkg-directory)
- [Domain-Driven Layout](#domain-driven-layout)
- [Putting It All Together: A Sample Web Service](#putting-it-all-together-a-sample-web-service)
- [Project Structure Best Practices](#project-structure-best-practices)

## The Philosophy of Project Layout

Unlike some frameworks that enforce a strict directory structure, Go is more flexible. However, community-developed patterns have emerged to help manage complexity as projects grow.

**Key Principles:**
-   **Start Simple:** Don't create a complex directory structure until you need it. A flat layout is fine for small projects.
-   **Clarity and Intent:** Your project structure should make it easy to find code and understand the purpose of each package.
-   **Enforce Separation:** Use packages to create clear boundaries between different parts of your application. The `internal` directory is a key tool for this.

## A Simple, Flat Structure (The Default)

For a small application or a single-purpose library, you don't need a complex hierarchy.

```
/my-project
├── go.mod
├── go.sum
├── main.go         // Your main application logic.
├── handlers.go     // HTTP handlers, if it's a web service.
├── handlers_test.go
├── models.go       // Your data structures.
└── models_test.go
```

**When to Use:**
-   Small CLIs or services.
-   When you're starting a new project.
-   When you have only one main executable and a handful of supporting files.

**When to Move On:**
-   When you have multiple binaries to build from the same repository.
-   When you want to share some code as a public library (`/pkg`) but keep other parts private (`/internal`).
-   When the number of files in the root becomes overwhelming.

## The Standard Go Project Layout

This is a well-known, but often overused, community standard. It provides a comprehensive structure for larger applications. **Do not use all of these directories.** Pick and choose only the ones you need.

Reference: [Standard Go Project Layout](https://github.com/golang-standards/project-layout)

### Top-Level Directories

-   `/cmd`: Main applications for your project.
-   `/internal`: Private application and library code.
-   `/pkg`: Public library code that's okay to be used by others.
-   `/api`: OpenAPI/Swagger specs, JSON schema files.
-   `/web`: Web assets like templates and static files.
-   `/configs`: Configuration files.
-   `/scripts`: Scripts to perform various build, install, analysis, etc. operations.

Of these, `/cmd`, `/internal`, and `/pkg` are the most important and have special meaning.

### The `/cmd` Directory

**Purpose:** To house the `main` package for each binary you want to build.

You should not put a lot of code here. The `main.go` file in a `cmd` directory should be a small entrypoint that imports and calls code from `/internal` and `/pkg` to do the actual work.

```
/cmd
├── /my-api-server
│   └── main.go      // main package for the API server
└── /my-cli-tool
    └── main.go      // main package for the CLI tool
```

### The `/internal` Directory

**Purpose:** To store all the code that is **not meant to be imported** by other projects.

This is a special directory enforced by the Go compiler. If another project tries to import a package from your `/internal` directory, the build will fail. This is Go's primary mechanism for enforcing code privacy.

The bulk of your application logic—HTTP handlers, business logic, data access—lives here. You can structure it however you like.

```
/internal
├── /api           // Your HTTP handlers and routing.
├── /database      // Your database connection and query logic.
├── /domain        // Core business logic and types.
│   └── /user
│       ├── user.go
│       └── repository.go
├── /config        // Application configuration loading.
└── /auth          // Authentication and authorization logic.
```

### The `/pkg` Directory

**Purpose:** To store packages that are safe to be imported and used by external applications.

**Warning:** This is one of the most misunderstood directories. Before putting a package here, ask yourself:
1.  Is this code generic and useful to other projects?
2.  Do I want to support and maintain this as a public library?

If the answer is no, it belongs in `/internal`. Many projects don't need a `/pkg` directory at all. Don't use it as a dumping ground for all your code.

## Domain-Driven Layout

As your `internal` directory grows, a good way to structure it is by business domain or feature. This keeps related code together and makes the system easier to understand.

```
/internal/
├── platform/         # Framework-level code.
│   ├── database/
│   └── server/
├── user/             # "User" domain.
│   ├── user.go       # The User model.
│   ├── repository.go # The interface for user storage.
│   ├── service.go    # Business logic for users.
│   └── handler.go    # HTTP handlers for users.
├── payment/          # "Payment" domain.
│   ├── payment.go
│   ├── repository.go
│   └── handler.go
└── auth/             # "Auth" domain.
    └── ...
```

This approach groups code by what it *does* rather than by what it *is* (e.g., putting all handlers in one `handlers` package).

## Putting It All Together: A Sample Web Service

Here is a sensible structure for a medium-sized web service.

```
/my-awesome-api
├── go.mod
├── go.sum
├── README.md
├── configs/
│   ├── config.yml
│   └── config.dev.yml
├── cmd/
│   └── api/
│       └── main.go           # Wires everything together and starts the server.
└── internal/
    ├── server/
    │   ├── server.go         # The http.Server setup.
    │   └── routes.go         # All the API routes, pointing to handlers.
    ├── auth/
    │   └── middleware.go     # Auth middleware.
    ├── user/
    │   ├── user.go           # The User model and business logic.
    │   ├── handler.go        # HTTP handlers for /users endpoints.
    │   └── store.go          # The user storage implementation.
    └── platform/
        └── database/
            └── postgres.go   # Logic for connecting to the database.
```

## Project Structure Best Practices

1.  **Start Simple and Evolve:** Begin with a flat structure. Only add directories like `/cmd` and `/internal` when you have a clear need for them.
2.  **Use `/internal` for Your Business Logic:** Protect your implementation details. Most of your code should probably live here.
3.  **Be Deliberate About `/pkg`:** Don't create a `/pkg` directory unless you intend to share that code as a library for others to import.
4.  **Keep `main` Packages Small:** The code in `/cmd/app/main.go` should be minimal. It's for setup and wiring, not business logic.
5.  **Group by Domain/Feature:** As your project grows, structuring by feature (e.g., `/internal/user`) is often clearer than structuring by layer (e.g., `/internal/models`, `/internal/handlers`).

---

Next: [Dependency Management](14-dependency-management.md) 