# What is Docker Compose?

Docker Compose is a tool used to define and manage multi-container Docker applications using a YAML configuration file called:

compose.yaml

OR

docker-compose.yml

Instead of running multiple docker run commands manually, Docker Compose allows you to manage everything from a single file.

## Why Docker Compose is Used

### Suppose your application contains:

Frontend
Backend API
Database
Redis Cache
Nginx Reverse Proxy

Managing all these containers manually becomes difficult.

### Docker Compose helps by:

Creating containers automatically
Creating networks automatically
Managing volumes
Managing environment variables
Starting all services with one command
  
### Docker Compose Architecture
  
                Docker Compose
                        |
        -----------------------------------
        |                |                |
        v                v                v

    Frontend         Backend API        Database
    Container         Container         Container

All containers communicate through an internal Docker network.

### Docker Compose Workflow
  
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
  
### Difference Between         Dockerfile and Docker Compose
Dockerfile	                     Docker Compose
Builds image 	                   Runs multi-container apps
Defines image instructions	     Defines application stack
Used with docker build	         Used with docker compose up
  
Creates one image	Manages many containers
  
### Important Components of Docker Compose

#### 1. Services

Services define containers.

Example:

services:
  web:
    image: nginx

Here:

web = service name
nginx = image used to create container

#### 2. Networks

Docker Compose automatically creates a network.

All containers can communicate using service names.

Example:

backend can access db using:

db:3306

No need to know IP addresses.

#### 3. Volumes

Volumes store persistent data.

Useful for:

MySQL data
PostgreSQL data
MongoDB data

Example:

volumes:
  - mysql-data:/var/lib/mysql
Volume Architecture
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

Data survives even after container deletion.

#### 4. Port Mapping

Syntax:

ports:
  - "HOST_PORT:CONTAINER_PORT"

Example:

ports:
  - "8080:80"

Meaning:

Host Port 8080 ---> Container Port 80

Basic Docker Compose Example

compose.yaml

services:
  web:
    image: nginx
    ports:
      - "8080:80"
      
Run Application
docker compose up

Open Browser
http://localhost:8080

#### Architecture

Browser
   |
localhost:8080
   |
Docker Host
   |
Nginx Container:80
Build Images Using Dockerfile

Docker Compose can build images automatically.

#### Dockerfile

FROM node:18

WORKDIR /app

COPY . .

RUN npm install

CMD ["node", "server.js"]


compose.yaml
services:
  app:
    build: .
    ports:
      - "3000:3000"
      
Build Process
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

### Important Concept

Docker Compose can:

Build images
Create containers
Start containers

automatically.

#### Internal Working

When you run:

docker compose up

Docker Compose internally performs:

1. Create Network
2. Create Volumes
3. Build Images
4. Create Containers
5. Start Containers

### Docker Compose Commands

Start Containers:
docker compose up

Run in Background:
docker compose up -d

-d means detached mode.

Stop Containers:
docker compose down

View Running Containers:
docker compose ps

View Logs: 
docker compose logs

Live logs:

docker compose logs -f

Rebuild Images:
docker compose up --build

Build Only:
docker compose build

Docker Compose Networking

All services communicate using service names.

Example:

services:
  backend:
    image: node

  db:
    image: mysql

Backend accesses database using:

db

NOT:

localhost
Internal Docker Network
backend <----> db

Docker automatically provides DNS resolution.

#### Multi-Container Example
compose.yaml

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
      
#### Architecture

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
                 
Real Example: Nginx + Multiple Node.js Containers + Redis

compose.yaml
services:
  redis:
    image: 'redislabs/redismod'
    ports:
      - '6379:6379'

  web1:
    restart: on-failure
    build: ./web
    hostname: web1
    ports:
      - '81:5000'

  web2:
    restart: on-failure
    build: ./web
    hostname: web2
    ports:
      - '82:5000'

  nginx:
    build: ./nginx
    ports:
      - '80:80'
    depends_on:
      - web1
      - web2
      
#### Architecture Diagram
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

#### How It Works

##### Step 1

Docker Compose builds:

web image
nginx image

##### Step 2
Docker Compose creates:

web1 container
web2 container
nginx container
redis container

##### Step 3

Nginx acts as:

reverse proxy
load balancer

It forwards requests to:

web1
web2
Scaling Concept

Both web containers are created from the same image.

Same Image
     |
-------------
|           |
v           v

web1      web2

This provides:

load balancing
scalability
high availability
depends_on

##### Example:

depends_on:
  - web1
  - web2

Meaning:

Start web1 and web2 before nginx
Restart Policy

###### Example:

restart: on-failure

If container crashes:

Docker automatically restarts it.
Environment Variables

###### Example:

environment:
  MYSQL_ROOT_PASSWORD: root

Inside container:

echo $MYSQL_ROOT_PASSWORD
