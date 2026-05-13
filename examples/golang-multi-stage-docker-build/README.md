# Multi Stage Docker Build

The main purpose of choosing a Golang-based application to demonstrate this example is that Go is a statically typed programming language that does not require a runtime in the traditional sense.

Unlike dynamically typed languages like Python, Ruby, and JavaScript, which rely on a runtime environment to execute their code, Go compiles directly into machine code that can be executed directly by the operating system.

Because of this, the real advantage of **Multi-Stage Docker Builds** and **Distroless Images** can be clearly understood through the drastic reduction in image size.

---

## What is a Multi-Stage Docker Build?

A multi-stage Docker build is a way to use multiple `FROM` statements in a single Dockerfile to create a:

- Smaller image
- Cleaner image
- More secure image

It is mainly used when building applications like:

- Go
- Java
- Node.js
- React
- Python
- Rust

where build tools are needed during compilation, but not in the final production container.

---

# The Problem Without Multi-Stage Builds

Suppose you have a Go application.

## To compile Go code, Docker needs:

- Go compiler
- Go libraries
- Build dependencies

## If all of these remain inside the final image:

- Image becomes large
- Unnecessary tools remain
- Security risks increase

### Example

```dockerfile
FROM golang:1.22

WORKDIR /app

COPY . .

RUN go build -o calculator

CMD ["./calculator"]
```

This image contains:

- Your application
- Go compiler
- Go SDK
- Build cache
- Extra files

Even though the container only needs the final binary.

---

# Multi-Stage Build Solution

We separate the process into:

1. Build Stage
2. Runtime Stage

---

# How It Works

## Stage 1 → Build the Application

```dockerfile
FROM golang:1.22 AS builder
```

This stage contains Go tools.

Then:

```dockerfile
WORKDIR /app

COPY . .

RUN go build -o calculator
```

Now Docker creates the binary.

---

## Stage 2 → Create Minimal Final Image

```dockerfile
FROM ubuntu:latest
```

This is a fresh new image.

Now copy **ONLY** the compiled application:

```dockerfile
COPY --from=builder /app/calculator /calculator
```

Then run it:

```dockerfile
CMD ["/calculator"]
```

---

## Complete Example

```dockerfile
# Stage 1 - Build
FROM golang:1.22 AS builder

WORKDIR /app

COPY . .

RUN go build -o calculator

# Stage 2 - Final Image
FROM ubuntu:latest

COPY --from=builder /app/calculator /calculator

CMD ["/calculator"]
```

---

## What Happens Internally

Docker first:

1. Starts a Go container
2. Builds the application
3. Creates the binary

Then:

4. Starts a new clean Ubuntu container
5. Copies only the binary
6. Removes everything else

---

## Final Image Does NOT Contain

- Go compiler
- Source code
- Build tools

## Final Image Contains Only

- Your executable binary

---

# Benefits of Multi-Stage Builds

## 1. Smaller Image Size

### Example

| Type | Approx Size |
|---|---|
| Normal Go Image | 1GB+ |
| Multi-Stage Image | 20MB |

### Smaller Images:

- Download faster
- Deploy faster
- Use less storage

---

## 2. Better Security

No compiler or build tools inside the final container.

This reduces the attack surface.

---

## 3. Cleaner Containers

Only production files remain inside the image.

---

## 4. Faster Deployments

Smaller images push and pull quickly.

Especially useful in:

- Kubernetes
- CI/CD Pipelines
- Cloud Deployments

---

# Summary

Multi-stage Docker builds help create:

- Lightweight containers
- Secure containers
- Production-ready images

They are one of the most important Docker optimization techniques used in modern DevOps workflows.
