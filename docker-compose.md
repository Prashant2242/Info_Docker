# Docker Compose

Docker Compose is a tool used to define and manage **multi-container Docker applications** using a YAML configuration file.

Supported file names:

```yaml
compose.yaml
```

or

```yaml
docker-compose.yml
```

Instead of running multiple `docker run` commands manually, Docker Compose allows you to manage the entire application stack from a single file.

---

# Why Docker Compose is Used

Modern applications usually contain multiple services such as:

* Frontend
* Backend API
* Database
* Redis Cache
* Nginx Reverse Proxy

Managing all these containers manually becomes difficult.

Docker Compose solves this problem by automating:

* Container creation
* Networking
* Volume management
* Environment variables
* Service startup order

---

# Docker Compose Architecture

```text
                Docker Compose
                        |
        -----------------------------------
        |                |                |
        v                v                v

    Frontend         Backend API        Database
    Container         Container         Container
```

All containers communicate through an **internal Docker network** automatically created by Docker Compose.

---

# Docker Compose Workflow

```text
compose.yaml
      |
      v
docker compose up
      |
      v
Create Network
      |
      v
Build/Pull Images
      |
      v
Create Containers
      |
      v
Start Containers
```

---

# Dockerfile vs Docker Compose

| Dockerfile                 | Docker Compose                    |
| -------------------------- | --------------------------------- |
| Builds Docker images       | Runs multi-container applications |
| Defines image instructions | Defines entire application stack  |
| Used with `docker build`   | Used with `docker compose up`     |
| Creates one image          | Manages many containers           |

---

# Important Components of Docker Compose

## 1. Services

Services define containers.

### Example

```yaml
services:
  web:
    image: nginx
```

### Explanation

| Term    | Meaning      |
| ------- | ------------ |
| `web`   | Service name |
| `nginx` | Docker image |

---

## 2. Networks

Docker Compose automatically creates a network for all services.

Containers communicate using **service names** instead of IP addresses.

### Example

```text
backend --> db:3306
```

No need to know container IP addresses.

---

## 3. Volumes

Volumes store persistent data.

Useful for:

* MySQL
* PostgreSQL
* MongoDB
* Redis

### Example

```yaml
volumes:
  - mysql-data:/var/lib/mysql
```

### Volume Architecture

```text
+----------------------+
| MySQL Container      |
| /var/lib/mysql       |
+----------------------+
          |
          v
+----------------------+
| Docker Volume        |
| mysql-data           |
+----------------------+
```

Data survives even after container deletion.

---

## 4. Port Mapping

### Syntax

```yaml
ports:
  - "HOST_PORT:CONTAINER_PORT"
```

### Example

```yaml
ports:
  - "8080:80"
```

### Meaning

```text
Host Port 8080 ---> Container Port 80
```

---

# Basic Docker Compose Example

## compose.yaml

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
```

---

## Run Application

```bash
docker compose up
```

---

## Open Browser

```text
http://localhost:8080
```

---

## Architecture

```text
Browser
   |
localhost:8080
   |
Docker Host
   |
Nginx Container:80
```

---

# Building Images Using Dockerfile

Docker Compose can automatically build Docker images using a Dockerfile.

---

## Dockerfile

```dockerfile
FROM node:18

WORKDIR /app

COPY . .

RUN npm install

CMD ["node", "server.js"]
```

---

## compose.yaml

```yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
```

---

## Build Process

```text
Dockerfile
     |
     v
Docker Compose
     |
     v
Build Image
     |
     v
Create Container
```

---

# What Happens Internally?

When you run:

```bash
docker compose up
```

Docker Compose internally performs:

1. Create Network
2. Create Volumes
3. Build Images
4. Create Containers
5. Start Containers

---

# Docker Compose Commands

## Start Containers

```bash
docker compose up
```

---

## Run in Background

```bash
docker compose up -d
```

`-d` means **detached mode**.

---

## Stop Containers

```bash
docker compose down
```

---

## View Running Containers

```bash
docker compose ps
```

---

## View Logs

```bash
docker compose logs
```

### Live Logs

```bash
docker compose logs -f
```

---

## Rebuild Images

```bash
docker compose up --build
```

---

## Build Images Only

```bash
docker compose build
```

---

# Docker Compose Networking

All services communicate using **service names**.

---

## Example

```yaml
services:
  backend:
    image: node

  db:
    image: mysql
```

Backend accesses database using:

```text
db
```

NOT:

```text
localhost
```

---

## Internal Docker Network

```text
backend <----> db
```

Docker automatically provides internal DNS resolution.

---

# Multi-Container Example

## compose.yaml

```yaml
services:
  frontend:
    image: nginx
    ports:
      - "80:80"

  backend:
    build: ./backend
    ports:
      - "5000:5000"

  db:
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
```

---

## Architecture

```text
                User Browser
                      |
                      v
               Frontend (Nginx)
                      |
                      v
                Backend API
                      |
                      v
                 MySQL Database
```

---

# Real-World Example

## Nginx + Multiple Node.js Containers + Redis

### compose.yaml

```yaml
services:
  redis:
    image: redislabs/redismod
    ports:
      - "6379:6379"

  web1:
    restart: on-failure
    build: ./web
    hostname: web1
    ports:
      - "81:5000"

  web2:
    restart: on-failure
    build: ./web
    hostname: web2
    ports:
      - "82:5000"

  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - web1
      - web2
```

---

## Architecture Diagram

```text
                    User Browser
                          |
                          v
                     Nginx:80
                          |
                -------------------
                |                 |
                v                 v

            web1:5000        web2:5000
                |
                v
              Redis
```

---

# How This Setup Works

## Step 1 — Build Images

Docker Compose builds:

* web image
* nginx image

---

## Step 2 — Create Containers

Docker Compose creates:

* web1 container
* web2 container
* nginx container
* redis container

---

## Step 3 — Start Services

Nginx acts as:

* Reverse Proxy
* Load Balancer

It forwards requests to:

* web1
* web2

---

# Scaling Concept

Both containers are created from the same image.

```text
Same Image
     |
-------------
|           |
v           v

web1      web2
```

This provides:

* Load balancing
* Scalability
* High availability

---

# depends_on

## Example

```yaml
depends_on:
  - web1
  - web2
```

### Meaning

Start:

* web1
* web2

before starting:

* nginx

---

# Restart Policy

## Example

```yaml
restart: on-failure
```

If the container crashes:

Docker automatically restarts it.

---

# Environment Variables

## Example

```yaml
environment:
  MYSQL_ROOT_PASSWORD: root
```

Inside the container:

```bash
echo $MYSQL_ROOT_PASSWORD
```

---

# Key Advantages of Docker Compose

* Single command deployment
* Easy multi-container management
* Automatic networking
* Persistent storage support
* Easy scaling
* Better development workflow
* Infrastructure as Code (IaC)

---

# Common Docker Compose Lifecycle

```text
Write compose.yaml
        |
        v
docker compose build
        |
        v
docker compose up
        |
        v
Application Running
        |
        v
docker compose down
```

---

# Summary

Docker Compose helps you:

* Define services
* Manage containers
* Create networks
* Handle volumes
* Configure environment variables
* Run complete applications using one command

It is one of the most important tools for:

* Backend Development
* Microservices
* DevOps
* Local Development
* Testing Environments
* Multi-container Applications
