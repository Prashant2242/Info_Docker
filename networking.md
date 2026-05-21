# 🐳 Docker Networking — Complete Guide

> **Beginner-friendly notes** covering every Docker network type, how containers talk to each other, what's happening under the hood (veth, docker0, eth0), and hands-on commands for every scenario.

---

## 📚 Table of Contents

1. [Why Networking Matters in Docker](#1-why-networking-matters-in-docker)
2. [How Docker Networking Works Under the Hood](#2-how-docker-networking-works-under-the-hood)
3. [Default Networks](#3-default-networks)
4. [Bridge Networking (Default)](#4-bridge-networking-default)
5. [Custom Bridge Network (Isolated)](#5-custom-bridge-network-isolated)
6. [Host Networking](#6-host-networking)
7. [Overlay Networking](#7-overlay-networking)
8. [Macvlan Networking](#8-macvlan-networking)
9. [None Network (Full Isolation)](#9-none-network-full-isolation)
10. [Quick Reference — All Commands](#10-quick-reference--all-commands)
11. [Cheat Sheet — When to Use What](#11-cheat-sheet--when-to-use-what)

---

## 1. Why Networking Matters in Docker

Containers are **isolated by default** — they can't talk to the outside world or to each other unless you explicitly set up networking.

Docker networking solves three problems:

- **Container ↔ Container** communication (e.g. your app talking to your database)
- **Container ↔ Host** communication (e.g. your app accessing host services)
- **Container ↔ Internet** communication (e.g. your app calling an external API)

---

## 2. How Docker Networking Works Under the Hood

Understanding a few Linux concepts makes everything click.

### The key pieces

```
INTERNET
    │
    ▼
[ eth0 ]          ← Host's physical network card (real NIC, e.g. 192.168.1.10)
    │
[ iptables/NAT ]  ← Routes packets, handles port mapping, masquerades container traffic
    │
[ docker0 ]       ← Virtual bridge switch created by Docker (172.17.0.1)
   / \
  /   \
vethA  vethB      ← One virtual cable per container (host side)
  │       │
eth0    eth0      ← Same cables, container side — appear as normal NICs inside
(ctrA) (ctrB)
172.17.0.2  172.17.0.3
```

### Full Architecture Diagram

![Docker Networking Architecture](docker-networking-architecture.svg)

### `eth0` — the physical NIC
Your host machine's real network card. It holds the machine's LAN IP and is the only thing talking to the physical network. Everything else Docker does is virtual on top of this.

### `docker0` — the virtual bridge
When Docker installs, it creates a virtual network switch called `docker0` (default IP: `172.17.0.1`). Think of it as a software Ethernet switch. Every container on the default bridge network plugs into it.

```bash
# See docker0 on your host
ip addr show docker0
# Output: inet 172.17.0.1/16 ...
```

### `veth` pairs — the virtual cable
A **veth (virtual Ethernet) pair** is like a physical Ethernet cable with two ends. Docker creates one pair per container:

- **Host end** (`vethXXXX`) — plugs into the `docker0` bridge, visible on the host
- **Container end** — appears as `eth0` inside the container, gets an IP like `172.17.0.2`

A packet pushed into one end comes out the other — exactly like a real cable.

```bash
# See all veth interfaces on the host (one per running container)
ip link show | grep veth

# See the container's network interfaces from inside
docker exec -it my_container ip addr
```

### How a packet travels: Container A → Container B

```
Container A app sends packet
  → exits via container's eth0 (172.17.0.2)
  → travels through veth cable to host
  → arrives at docker0 bridge (172.17.0.1)
  → docker0 forwards to correct veth peer
  → arrives at Container B's eth0 (172.17.0.3)
  → received by Container B app
```
No NAT involved — pure Layer 2 switching between containers on the same bridge.

### How a packet travels: Container → Internet

```
Container → veth → docker0 → iptables MASQUERADE → host eth0 → Internet
```
Docker adds an `iptables` NAT rule so outbound traffic appears to come from the host's IP.

### Port mapping (`-p host_port:container_port`)

```
Internet → host eth0:8080
  → iptables DNAT rule rewrites destination
  → forwarded to container IP:80
```

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx

# Inspect the iptables rules Docker created
sudo iptables -t nat -L -n
```

---

## 3. Default Networks

Every fresh Docker installation comes with three networks built in.

```bash
docker network ls
```

```
NETWORK ID     NAME      DRIVER    SCOPE
abc123         bridge    bridge    local
def456         host      host      local
ghi789         none      null      local
```

| Network | Driver | Purpose |
|---------|--------|---------|
| `bridge` | bridge | Default — containers can talk via docker0 |
| `host` | host | Container shares the host's network stack |
| `none` | null | No networking at all, fully isolated |

---

## 4. Bridge Networking (Default)

The **default mode**. When you run a container without specifying a network, it joins the `bridge` network automatically.

```
Host (docker0: 172.17.0.1)
├── Container A  eth0: 172.17.0.2
├── Container B  eth0: 172.17.0.3
└── Container C  eth0: 172.17.0.4
```

All containers on the same bridge can talk to each other using their IP addresses. They reach the internet via NAT through the host's `eth0`.

![Bridge Networking](https://user-images.githubusercontent.com/43399466/217745543-f40e5614-ac34-4b78-85a9-91b24512388d.png)

### Run a container on the default bridge

```bash
docker run -d --name web nginx
docker run -d --name db postgres
```

### Inspect the bridge network

```bash
# See which containers are connected and their IPs
docker network inspect bridge
```

### Test connectivity between containers

```bash
# Find container B's IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' db

# Ping from container A to container B's IP
docker exec -it web ping 172.17.0.3
```

> ⚠️ **Limitation of the default bridge:** Containers can only talk using IP addresses — DNS name resolution is **not** available on the default bridge. Use a custom bridge (see Section 5) for name-based communication.

### Port mapping examples

```bash
# Map host port 8080 → container port 80
docker run -d -p 8080:80 --name web nginx

# Map multiple ports
docker run -d -p 8080:80 -p 443:443 --name web nginx

# Map to a specific host interface only
docker run -d -p 127.0.0.1:8080:80 --name web nginx

# Let Docker pick a random available host port
docker run -d -p 80 --name web nginx
docker port web   # shows which port was chosen
```

---

## 5. Custom Bridge Network (Isolated)

You can create your own bridge networks to **isolate groups of containers** from each other. This is the **recommended approach** for production.

### Why custom bridges are better than the default

| Feature | Default bridge | Custom bridge |
|---------|---------------|---------------|
| DNS name resolution | ❌ No | ✅ Yes |
| Isolation from other bridges | ❌ No | ✅ Yes |
| Custom subnet | ❌ No | ✅ Yes |
| Containers connect by name | ❌ No | ✅ Yes |

### Create a custom bridge network

```bash
# Basic — Docker picks the subnet
docker network create my_bridge

# Specify your own subnet and gateway
docker network create \
  --driver bridge \
  --subnet 192.168.100.0/24 \
  --gateway 192.168.100.1 \
  my_bridge
```

### Run containers on the custom network

```bash
docker run -d --network my_bridge --name db training/postgres
docker run -d --network my_bridge --name web nginx
```

Now `web` can reach `db` by name — Docker's embedded DNS resolves `db` → `10.x.x.x` automatically.

```bash
# This works because of built-in DNS on custom bridges
docker exec -it web ping db
docker exec -it web curl http://db:5432
```

### Isolation between networks

```
Default bridge (172.17.0.0/16)        my_bridge (192.168.100.0/24)
├── Container A  172.17.0.2           ├── Container db  192.168.100.2
└── Container B  172.17.0.3           └── Container web 192.168.100.3

↑ These two groups CANNOT talk to each other ↑
```

Containers on `bridge` **cannot** reach containers on `my_bridge`, and vice versa. Different subnets, no routing between them.

![Isolated networks — no communication](https://user-images.githubusercontent.com/43399466/217748680-8beefd0a-8181-4752-a098-a905ebed5d2a.png)

### Connect a container to multiple networks

```bash
# Container 'web' is on default bridge — connect it to my_bridge too
docker network connect my_bridge web

# Now 'web' has two IPs — one in each network
docker inspect web | grep -A 20 '"Networks"'

# Disconnect when done
docker network disconnect my_bridge web
```

This is how you build a **gateway container** that bridges two isolated networks.

![After connecting web to my_bridge — now they can talk](https://user-images.githubusercontent.com/43399466/217748726-7bb347d0-3736-4f89-bdff-31d240b15150.png)

### Inspect and manage networks

```bash
# List all networks
docker network ls

# Detailed info about a specific network (shows connected containers + IPs)
docker network inspect my_bridge

# Remove a network (all containers must be disconnected first)
docker network rm my_bridge

# Remove all unused networks at once
docker network prune
```

---

## 6. Host Networking

The container **shares the host's network namespace entirely** — no veth, no docker0, no NAT. The container uses the host's `eth0` directly.

```
Host eth0: 192.168.1.10
     │
     └── Container C  (same IP, same ports, same everything)
```

### Run a container with host networking

```bash
docker run --network="host" nginx
```

Nginx inside the container listens on port 80 — and that's literally port 80 on the host. No `-p` flag needed (or allowed).

### When to use host networking

✅ When you need **maximum network performance** (no NAT overhead)  
✅ When your app needs to bind to **many ports dynamically**  
✅ For network monitoring/diagnostic tools that need raw host access  

### When NOT to use it

❌ When security matters — the container can access all host network interfaces  
❌ When you need port isolation between containers  
❌ On macOS/Windows — host networking only works on Linux  

> ⚠️ **Security note:** With `--network host`, a compromised container has the same network access as the host itself. Use only when you have a specific reason.

---

## 7. Overlay Networking

Overlay networks enable containers to communicate **across multiple Docker hosts** (different machines). This is used with **Docker Swarm** or **Kubernetes**.

```
Machine 1                    Machine 2
┌─────────────────┐          ┌─────────────────┐
│ Container A     │◄────────►│ Container B     │
│ 10.0.0.2        │  VXLAN   │ 10.0.0.3        │
│  (overlay net)  │  tunnel  │  (overlay net)  │
└─────────────────┘          └─────────────────┘
```

Docker wraps container traffic in VXLAN packets and sends them between hosts — the containers think they're on the same LAN.

### Create an overlay network (requires Swarm mode)

```bash
# First, initialize Docker Swarm
docker swarm init

# Create an overlay network
docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  my_overlay

# Deploy a service across the swarm using the overlay network
docker service create \
  --network my_overlay \
  --name my_service \
  --replicas 3 \
  nginx
```

### When to use overlay

✅ Multi-host container communication  
✅ Docker Swarm deployments  
✅ When you need containers on different machines to act like they're on the same network  

---

## 8. Macvlan Networking

Macvlan lets a container **appear on the network as a physical host** — it gets its own MAC address and IP on the LAN, just like a real machine.

```
Router (192.168.1.1)
├── Host machine     (192.168.1.10)
├── Container A      (192.168.1.20)  ← looks like a real machine to the router
└── Container B      (192.168.1.21)  ← same
```

### Create a Macvlan network

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  my_macvlan

# Run a container with a specific IP on the LAN
docker run -d \
  --network my_macvlan \
  --ip 192.168.1.20 \
  --name my_app \
  nginx
```

### When to use Macvlan

✅ When a container needs a routable IP on the physical LAN  
✅ Legacy applications that expect to be "real" machines  
✅ Network appliance containers (firewalls, load balancers)  

> ⚠️ Many cloud providers and managed networks block Macvlan because it requires promiscuous mode on the NIC.

---

## 9. None Network (Full Isolation)

Completely disables all networking for a container. No `eth0`, no internet, no communication — only a loopback (`lo`) interface.

```bash
docker run --network none --name isolated_container alpine sh
```

### When to use none

✅ Batch processing jobs that don't need network access  
✅ Security-sensitive workloads (cryptography, file processing)  
✅ Testing container behaviour with zero network access  

---

## 10. Quick Reference — All Commands

### Viewing networks

```bash
docker network ls                          # list all networks
docker network inspect <name>              # detailed info + connected containers
docker network inspect bridge              # inspect the default bridge
```

### Creating networks

```bash
# Default bridge (Docker picks subnet)
docker network create my_network

# Bridge with custom subnet
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  my_network

# Overlay (Swarm required)
docker network create --driver overlay my_overlay

# Macvlan
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  my_macvlan
```

### Running containers with specific networks

```bash
docker run -d --network bridge nginx           # default bridge
docker run -d --network my_bridge nginx        # custom bridge
docker run -d --network host nginx             # host network
docker run -d --network none nginx             # no network
docker run -d --network my_overlay nginx       # overlay
```

### Port mapping

```bash
docker run -d -p 8080:80 nginx                 # host:8080 → container:80
docker run -d -p 127.0.0.1:8080:80 nginx       # bind to localhost only
docker run -d -p 80 nginx                      # random host port
docker port <container>                        # see mapped ports
```

### Connecting / disconnecting

```bash
docker network connect my_bridge web           # add container to a network
docker network disconnect my_bridge web        # remove container from network
```

### Cleanup

```bash
docker network rm my_network                   # remove one network
docker network prune                           # remove all unused networks
```

### Debugging inside containers

```bash
docker exec -it <container> ip addr            # see network interfaces + IPs
docker exec -it <container> ip route           # see routing table
docker exec -it <container> ping <host>        # test connectivity
docker exec -it <container> curl http://db     # test DNS + HTTP (custom bridge)
```

### Host-side inspection

```bash
ip link show | grep veth                       # see all veth interfaces
ip addr show docker0                           # docker0 bridge details
sudo iptables -t nat -L -n                     # see Docker's NAT rules
brctl show                                     # see all bridges (install bridge-utils)
```

---

## 11. Cheat Sheet — When to Use What

| Use Case | Recommended Network |
|----------|---------------------|
| Simple single-host app | Default bridge |
| Multi-container app (e.g. web + db) | **Custom bridge** (preferred) |
| Container needs to talk to other containers by name | **Custom bridge** |
| Maximum performance, no NAT | Host |
| Containers across multiple machines | Overlay (Swarm) |
| Container needs a real LAN IP | Macvlan |
| No network needed (batch/security) | None |
| Legacy app expecting a real machine | Macvlan |

### The Golden Rule

> **Always use a custom bridge network** for multi-container applications. It gives you DNS-based container discovery, strong isolation, and full control over subnets — all for free.

```bash
# The pattern you'll use 90% of the time
docker network create app_network
docker run -d --network app_network --name db postgres
docker run -d --network app_network --name app -p 8080:8080 my_app
# 'app' can now reach 'db' at http://db:5432 by name
```

---

## 📎 Further Reading

- [Docker Networking Official Docs](https://docs.docker.com/network/)
- [Docker network drivers overview](https://docs.docker.com/network/drivers/)
- [Understand container communication](https://docs.docker.com/network/bridge/)

---

*Notes compiled from Docker official documentation and hands-on examples. Feel free to open a PR if you spot anything outdated!*
