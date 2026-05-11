# Multi Stage Docker Build

The main purpose of choosing a golang based applciation to demostrate this example is golang is a statically-typed programming language that does not require a runtime in the traditional sense. Unlike dynamically-typed languages like Python, Ruby, and JavaScript, which rely on a runtime environment to execute their code, Go compiles directly to machine code, which can then be executed directly by the operating system.

So the real advantage of multi stage docker build and distro less images can be understand with a drastic decrease in the Image size.


# A multi-stage Docker build is a way to use multiple FROM statements in one Dockerfile to create a smaller, cleaner, and more secure final image.

It is mainly used when building applications like:

Go
Java
Node.js
React
Python
Rust

where you need build tools during compilation, but not in the final container.

The Problem Without Multi-Stage Builds

Suppose you have a Go application.

# To compile Go code, Docker needs:

Go compiler
Go libraries
Build dependencies

# If you keep all of these inside the final image:

image becomes large
unnecessary tools remain
security risk increases

Example:

FROM golang:1.22

WORKDIR /app

COPY . .

RUN go build -o calculator

CMD ["./calculator"]

This image contains:

your app
Go compiler
Go SDK
build cache
extra files

# Even though the container only needs the final binary.

# Multi-Stage Build Solution

We separate:

Build stage
Runtime stage
How It Works
Stage 1 → Build the application
FROM golang:1.22 AS builder

This stage contains Go tools.

Then:

WORKDIR /app
COPY . .
RUN go build -o calculator

Now Docker creates the binary.

Stage 2 → Create minimal final image
FROM ubuntu:latest

This is a fresh new image.

Now copy ONLY the compiled app:

COPY --from=builder /app/calculator /calculator

Then run it:

CMD ["/calculator"]



#Complete Example

# Stage 1 - Build
FROM golang:1.22 AS builder

WORKDIR /app

COPY . .

RUN go build -o calculator

# Stage 2 - Final Image
FROM ubuntu:latest

COPY --from=builder /app/calculator /calculator

CMD ["/calculator"]
What Happens Internally

Docker first:

Starts Go container
Builds app
Creates binary

Then:
4. Starts new clean Ubuntu container
5. Copies only binary
6. Removes everything else

#Final image does NOT contain:

Go compiler
source code
build tools

Only:

your executable
Benefits of Multi-Stage Builds
1. Smaller Image Size

Example:

Normal Go image → 1GB+
Multi-stage image → 20MB

#Smaller images:

download faster
deploy faster
use less storage
2. Better Security

No compiler/tools inside final container.

Less attack surface.

3. Cleaner Containers

Only production files remain.

4. Faster Deployments

Smaller images push/pull quickly.

Useful in:

Kubernetes
CI/CD
cloud deployments
