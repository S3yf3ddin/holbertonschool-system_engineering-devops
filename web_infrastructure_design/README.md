# 🌐 Web Infrastructure Design

> A series of progressive infrastructure design tasks that build from a minimal single-server stack to a fully redundant, secured, and monitored production environment for `www.foobar.com`.

---

## 📋 Task Overview

| File | Task | Key Addition |
|------|------|--------------|
| [0-simple_web_stack](./0-simple_web_stack) | Simple Web Stack | One server, all components on a single machine |
| [1-distributed_web_infrastructur](./1-distributed_web_infrastructur) | Distributed Web Infrastructure | Load balancer + 2 app servers + DB replication |
| [2-secured_and_monitored_web_infrastructure](./2-secured_and_monitored_web_infrastructure) | Secured & Monitored Infrastructure | Firewalls + SSL/TLS + monitoring agents |
| [3-scale_up](./3-scale_up) | Scale Up | HA load balancer cluster + separated component tiers |

---

## 📄 Task Details

---

### 0️⃣ — Simple Web Stack
**File:** [`0-simple_web_stack`](./0-simple_web_stack)

A one-server infrastructure that hosts the complete web application for `www.foobar.com` on IP `8.8.8.8`.

#### 🔧 Components
| Component | Role |
|-----------|------|
| DNS (A record) | Maps `www.foobar.com` → `8.8.8.8` |
| Nginx (Web Server) | Receives HTTP/HTTPS requests, serves static files, proxies to app server |
| Application Server | Runs business logic, processes requests, generates dynamic content |
| Application Files | The codebase (scripts, logic) |
| MySQL Database | Stores and retrieves persistent data |

#### 🔄 Request Flow
```
User Browser
     │
     ▼  DNS query: www.foobar.com → 8.8.8.8
     │
     ▼  HTTP/HTTPS request
┌─────────────────────┐
│       Server        │
│  ┌───────────────┐  │
│  │  Nginx        │  │
│  └──────┬────────┘  │
│         ▼           │
│  ┌───────────────┐  │
│  │  App Server   │  │
│  └──────┬────────┘  │
│         ▼           │
│  ┌───────────────┐  │
│  │  MySQL DB     │  │
│  └───────────────┘  │
└─────────────────────┘
     │
     ▼  HTTP Response → User Browser
```

#### ⚠️ Issues
| Issue | Description |
|-------|-------------|
| **SPOF** | If the single server fails, the entire website goes offline |
| **Maintenance Downtime** | Deploying new code requires restarting the web/app server, causing downtime |
| **Scalability** | A single server cannot handle high traffic — becomes a bottleneck |

---

### 1️⃣ — Distributed Web Infrastructure
**File:** [`1-distributed_web_infrastructur`](./1-distributed_web_infrastructur)

A three-server setup that introduces a load balancer and database replication to eliminate the single application-server bottleneck.

#### 🔧 Components Added
| Component | Why Added |
|-----------|-----------|
| HAProxy (Load Balancer) | Distributes traffic across two app servers; provides redundancy |
| 2nd Application Server | Redundancy and increased processing capacity |
| MySQL Primary–Replica cluster | Separates read/write traffic; provides a hot standby |

#### ⚖️ Load Balancer — Round Robin Algorithm
Requests are forwarded to each server in turn: `R1 → Server A`, `R2 → Server B`, `R3 → Server A`, …  
Ensures roughly equal distribution without considering current server load.

#### 🔄 Active-Active vs. Active-Passive
| Mode | Description |
|------|-------------|
| **Active-Active** ✅ (this design) | Both app servers handle live traffic simultaneously — maximises throughput |
| **Active-Passive** | One server is on standby and only takes over on failure — used for failover, not load distribution |

#### 🗄️ Primary–Replica Database
| Node | Responsibility |
|------|---------------|
| **Primary (Master)** | All `INSERT`, `UPDATE`, `DELETE` write operations |
| **Replica (Slave)** | `SELECT` read operations; continuously replicates from Primary |

#### ⚠️ Issues
| Issue | Description |
|-------|-------------|
| **SPOF — Load Balancer** | HAProxy is still a single point of failure; if it crashes, the whole site goes down |
| **No Firewall** | Servers are exposed to attacks on any open port |
| **No HTTPS** | Traffic travels in plain text — vulnerable to MITM attacks |
| **No Monitoring** | No visibility into server health or application errors |

---

### 2️⃣ — Secured and Monitored Web Infrastructure
**File:** [`2-secured_and_monitored_web_infrastructure`](./2-secured_and_monitored_web_infrastructure)

Builds on the distributed design by hardening security and adding observability.

#### 🔧 Components Added
| Component | Why Added |
|-----------|-----------|
| 3 × Firewalls (one per server) | Filter traffic; deny all by default, allow only required ports |
| SSL Certificate (at Load Balancer) | Serves traffic over HTTPS — encrypts data between browser and LB |
| 3 × Monitoring Agents | Collect metrics (CPU, memory, QPS, error rates) and ship to central service |

#### 🔥 Firewall Rules
| Server | Allowed Traffic |
|--------|----------------|
| Load Balancer | HTTP (80), HTTPS (443), SSH (22) |
| App Servers | HTTP from LB IP only, SSH from admin network |

#### 🔒 SSL Termination
SSL is terminated **at the Load Balancer**. The LB decrypts incoming HTTPS and forwards plain HTTP internally. This offloads CPU-intensive crypto from app servers.

#### 📊 Monitoring
Agents on each server collect:
- **QPS** (Queries Per Second) on web servers
- **Error rates** (HTTP 4xx/5xx)
- **Hardware utilisation** (CPU, RAM, disk I/O)

All metrics are shipped to a centralised service (e.g., Datadog / Sumo Logic) for dashboards and alerting.

#### ⚠️ Issues
| Issue | Description |
|-------|-------------|
| **Unencrypted internal traffic** | SSL terminates at the LB; traffic between LB and app servers is plain HTTP — a risk if the internal network is breached |
| **Single Write Master** | Only the Primary DB accepts writes; if it goes down, no data can be created or updated until failover |
| **Resource Contention** | Web server, app server, and DB all share the same machine — one hungry component can starve the others |
| **Scaling Difficulty** | Cannot scale DB storage independently without provisioning an entirely new server |

---

### 3️⃣ — Scale Up
**File:** [`3-scale_up`](./3-scale_up)

Eliminates the remaining load-balancer SPOF and separates every component onto its own dedicated server.

#### 🔧 Components Added / Changed
| Change | Reason |
|--------|--------|
| 2nd HAProxy (Active-Passive cluster via Keepalived) | Removes LB as SPOF — backup LB automatically takes over on primary failure |
| Dedicated Web Server tier | Web server gets its own machine (bandwidth/connection optimised) |
| Dedicated Application Server tier | App server gets its own machine (CPU optimised) |
| Dedicated Database tier | DB gets its own machine (I/O and RAM optimised) |

#### 🏗️ Architecture
```
Internet (HTTPS)
      │
      ▼
┌─────────────────────────┐
│  HAProxy LB1 (Active)   │◄──── Keepalived ────►│  HAProxy LB2 (Passive)  │
└────────────┬────────────┘                       └─────────────────────────┘
             │
             ▼
┌────────────────────┐
│  Nginx Web Server  │   ← dedicated server
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│  Application Server│   ← dedicated server
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│   MySQL Database   │   ← dedicated server
└────────────────────┘
```

#### 🔍 Web Server vs. Application Server
| | Web Server (Nginx) | Application Server (Gunicorn/Tomcat) |
|-|--------------------|--------------------------------------|
| **Primary role** | Handle HTTP protocol, serve static files | Execute business logic, generate dynamic content |
| **Content served** | HTML, CSS, JS, images | Dynamic HTML constructed from DB data |
| **Optimised for** | High concurrency of connections | CPU-intensive processing |
| **Also acts as** | Reverse proxy / load balancer | Backend processor |

#### ✅ Benefits of Component Separation
- **Resource Isolation** — DB, app, and web each get dedicated resources; no starvation
- **Independent Scaling** — Add more web servers without touching the DB tier
- **Tighter Security** — DB server can be locked down to accept connections only from the app tier

---

## 💡 Concepts Covered

| Concept | Introduced in |
|---------|--------------|
| DNS A records | Task 0 |
| Web server vs. application server | Task 0, Task 3 |
| Load balancing (Round Robin) | Task 1 |
| Active-Active vs. Active-Passive | Task 1 |
| Primary–Replica DB replication | Task 1 |
| Firewalls & port filtering | Task 2 |
| HTTPS / SSL termination | Task 2 |
| Infrastructure monitoring (agents) | Task 2 |
| HA load balancer cluster (Keepalived) | Task 3 |
| Component tier separation | Task 3 |
