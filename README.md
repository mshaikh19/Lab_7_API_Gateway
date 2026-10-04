# Lab 7: API Gateway, Service Discovery & Cloud Deployment

**Student ID:** 202512074  
**Course:** IT645 — Web Services & SOA  
**Lab:** 7 — API Gateway, Service Discovery & Cloud Deployment

---

## Overview

Lab 7 extends the three-microservice system from Lab 6 by adding:

1. **Part A — API Gateway**: A single public entry point that routes all client requests to the correct backend service, adds request logging, and handles errors from unreachable services.
2. **Part B — Configuration-Based Service Discovery**: Service URLs are externalized into environment variables so the gateway's routing table can change without any code modifications.
3. **Part C — Cloud Deployment**: The full system (gateway + three services) is deployed as Docker containers on a cloud platform and tested over a public URL.

---

## Architecture Diagram

```
                        ┌─────────────────────┐
                        │   Client / Postman   │
                        └──────────┬──────────┘
                                   │  HTTP requests to :3000
                                   ▼
                   ╔═══════════════════════════════╗
                   ║     API Gateway  :3000        ║  ← PUBLIC (Internet)
                   ║  /users  /products  /orders   ║
                   ║  Routing • Logging • Errors   ║
                   ╚══════╤════════╤══════╤════════╝
                          │        │      │
         ┌────────────────┘        │      └─────────────────┐
         │                         │                         │
╔════════╧══════════════════════════╧═════════════════════════╧═══════╗
║              Docker Network: campus-network (Internal Only)         ║
║                                                                      ║
║  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           ║
║  │ User Service │    │Product Service│   │ Order Service│           ║
║  │  :3001       │◄───│  :3002       │◄───│  :3003       │           ║
║  │ (no ext port)│    │ (no ext port)│    │ (no ext port)│           ║
║  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘           ║
║         │                   │                   │                   ║
║         ▼                   ▼                   ▼                   ║
║  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐           ║
║  │   User DB    │    │  Product DB  │    │   Order DB   │           ║
║  │  (MongoDB)   │    │  (MongoDB)   │    │  (MongoDB)   │           ║
║  └──────────────┘    └──────────────┘    └──────────────┘           ║
╚══════════════════════════════════════════════════════════════════════╝

                  MongoDB Atlas (cloud) or local containers
```

**Key design decision**: Only the **API Gateway's port 3000** is exposed externally. The three microservice containers have **no `ports:` mapping** in `compose.yaml` — they are reachable only from within the Docker network via the gateway or via direct inter-service calls.

---

## Part A — API Gateway

### What the Gateway Does

| Concern | Implementation |
|---|---|
| **Routing** | `/users/*` → User Service, `/products/*` → Product Service, `/orders/*` → Order Service |
| **Health check** | `GET /health` returns gateway status + service registry (no proxying) |
| **Request logging** | `morgan` logs method, path, target service, response status, and response time |
| **Error handling** | Proxy `error` event catches `ECONNREFUSED`/`ENOTFOUND` → returns `503 Service Unavailable`; other proxy errors → `502 Bad Gateway` |
| **No business logic** | The gateway only routes and handles cross-cutting concerns |

### Gateway Endpoints

| Gateway Path | Routed To | Method(s) |
|---|---|---|
| `GET /health` | Gateway itself | GET |
| `/users` | User Service `:3001` | GET, POST |
| `/users/:id` | User Service `:3001` | GET, PUT, DELETE |
| `/products` | Product Service `:3002` | GET, POST |
| `/products/:id` | Product Service `:3002` | GET, PUT, DELETE |
| `/orders` | Order Service `:3003` | GET, POST |
| `/orders/:id` | Order Service `:3003` | GET |

### Sample Gateway Log Output

```
GET /users → User Service | status=200 | 12.3 ms
POST /orders → Order Service | status=201 | 45.1 ms
GET /users → User Service | status=503 | 3002.5 ms   ← unreachable service
```

### Discussion: Why Use an API Gateway?

> **Answer (Part A discussion question):**
>
> Without a gateway, clients must know the address of every individual service — and there could be dozens in a real system. An API Gateway provides **a single stable entry point** that hides all internal service locations from the outside world. This means:
>
> 1. **Single entry point** — clients call one URL; internal topology can change freely.
> 2. **Internal structure is hidden** — the number of services, their ports, and their hosts are invisible to callers.
> 3. **Centralized cross-cutting concerns** — logging, authentication, rate-limiting, and error handling are implemented once in the gateway rather than duplicated in every service. In this lab, request logging and 502/503 error handling live entirely in the gateway; none of the three backend services need to know they are behind a proxy.

---

## Part B — Service Discovery (Configuration-Based)

### How It Works

Service locations are **never hard-coded** inside route-handling code. Instead, the gateway reads them from environment variables at startup:

```javascript
// api-gateway/server.js — service registry built from env vars
const SERVICE_REGISTRY = {
  user:    process.env.USER_SERVICE_URL    || 'http://user-service:3001',
  product: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3002',
  order:   process.env.ORDER_SERVICE_URL   || 'http://order-service:3003',
};
```

The routing table is built from this registry:

```javascript
app.use('/users',    buildProxy('user',    'User Service'));
app.use('/products', buildProxy('product', 'Product Service'));
app.use('/orders',   buildProxy('order',   'Order Service'));
```

### Service Registry (compose.yaml environment block)

```yaml
api-gateway:
  environment:
    - USER_SERVICE_URL=http://user-service:3001
    - PRODUCT_SERVICE_URL=http://product-service:3002
    - ORDER_SERVICE_URL=http://order-service:3003
```

### Proof of Config-Only Change

To move, say, the User Service to port 4001 **without touching any code**:
1. Edit `compose.yaml`: change `USER_SERVICE_URL=http://user-service:4001`
2. Run `docker compose up -d --build api-gateway`
3. `GET http://localhost:3000/users` still works — gateway re-points automatically.

### Static vs. Dynamic Service Discovery

| Feature | Static Config (this lab) | Dynamic Registry (Consul, Eureka, K8s DNS) |
|---|---|---|
| **Service locations** | Fixed at deploy time in env vars / config file | Registered at runtime by services themselves |
| **Failure detection** | None — gateway retries until timeout | Health-check driven; failed instances removed automatically |
| **Scaling** | Manual config update per instance | New instances register, load balancer updated automatically |
| **Complexity** | Very low — just env vars | Requires a running registry cluster |
| **Best for** | Small, stable deployments (this lab) | Large fleets with frequent scale events |

> **What dynamic discovery adds:** A static config file cannot detect that a service has crashed or that a new replica has been added. Dynamic registries (e.g., Consul) allow services to *self-register* on startup and *deregister* on failure, enabling the gateway to route only to healthy, currently running instances — essential in production autoscaling environments.

---

## Part C — Cloud Deployment

### Platform & Deployment

**Recommended Platform:** [Render](https://render.com) (free tier)

#### Step-by-Step Deployment on Render

1. **Push code to GitHub** (create a repo with this project structure).

2. **Create four Render Web Services** (one per container):
   - `campusconnect-gateway` → root dir: `api-gateway/`, Dockerfile detected automatically.
   - `campusconnect-user` → root dir: `user-service/`
   - `campusconnect-product` → root dir: `product-service/`
   - `campusconnect-order` → root dir: `order-service/`

3. **Set environment variables on Render** for each service:

   | Service | Variable | Value |
   |---|---|---|
   | `api-gateway` | `USER_SERVICE_URL` | `https://campusconnect-user.onrender.com` |
   | `api-gateway` | `PRODUCT_SERVICE_URL` | `https://campusconnect-product.onrender.com` |
   | `api-gateway` | `ORDER_SERVICE_URL` | `https://campusconnect-order.onrender.com` |
   | all services | `MONGO_URI` | Your MongoDB Atlas connection string |
   | `user-service` | `PORT` | `3001` |
   | `product-service` | `PORT` | `3002` |
   | `order-service` | `PORT` | `3003` |

4. **MongoDB Atlas** (from Lab 4): Enable Atlas access from `0.0.0.0/0` (Network Access) so Render IPs are allowed.

5. **Public Gateway URL**: Once deployed, Render assigns a URL such as:
   ```
   https://campusconnect-gateway.onrender.com
   ```

6. **Test via Postman**: In the Postman collection, update the `gateway_url` variable to the Render URL and run the "☁️ Cloud Deployment Tests" folder.

### Cloud Configuration Note

All service URLs and MongoDB connection strings are injected as environment variables on the cloud platform — **no code or Dockerfile changes needed** — following the same config-driven approach as Part B.

---

## How to Run Locally

### Start All Services

```bash
docker compose up -d --build
```

*Builds and starts: api-gateway, user-service, product-service, order-service + 3 MongoDB containers.*

### Verify All Containers Are Running

```bash
docker compose ps
```

Expected output shows 7 containers (gateway + 3 services + 3 databases), all `running`.

### Test the Gateway

```bash
# Health check
curl http://localhost:3000/health

# List users via gateway
curl http://localhost:3000/users

# List products via gateway
curl http://localhost:3000/products

# List orders via gateway
curl http://localhost:3000/orders
```

### Test 503 Error Handling (Unreachable Service)

```bash
# Stop User Service
docker compose stop user-service

# Request via gateway → should return 503
curl http://localhost:3000/users

# Restart User Service
docker compose start user-service
```

### Stop All Services

```bash
docker compose down
```

---

## Project Structure

```
202512074_Assignment_7/
├── api-gateway/                # NEW in Lab 7
│   ├── server.js               # Gateway: routing, logging, error handling
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── user-service/               # Unchanged from Lab 6
│   ├── server.js
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── product-service/            # Unchanged from Lab 6
│   ├── server.js
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── order-service/              # Unchanged from Lab 6
│   ├── server.js
│   ├── package.json
│   ├── Dockerfile
│   └── .dockerignore
├── compose.yaml                # Updated: gateway added, service ports hidden
├── .env.example                # Service registry / config reference
├── CampusConnect-Lab7-Gateway.postman_collection.json
└── README.md
```

---

## Postman Testing

Import `CampusConnect-Lab7-Gateway.postman_collection.json`. The collection has five folders:

| Folder | Purpose |
|---|---|
| 🏥 Gateway Health | Verify gateway is running |
| 👤 Users (via Gateway) | Full CRUD routed through gateway |
| 📦 Products (via Gateway) | Full CRUD routed through gateway |
| 🛒 Orders (via Gateway) | Create and list orders via gateway |
| ⚠️ Error Handling Tests | 404 (missing resource), 503 (stopped service), unknown route |
| ☁️ Cloud Deployment Tests | Same tests against cloud public URL |

**For local testing:** `gateway_url` = `http://localhost:3000`  
**For cloud testing:** Change `gateway_url` to your cloud platform's public gateway URL.

---

## Troubleshooting

| Issue | Fix |
|---|---|
| Gateway returns 503 | Ensure the target service container is running: `docker compose ps` |
| Gateway won't start | Check that ports 3000 is not already in use: `netstat -an \| findstr 3000` |
| MongoDB connection failed | Services fall back to in-memory store; check `MONGO_URI` env var is correct |
| Cloud services can't reach each other | Ensure the cloud service URLs in env vars use the correct hostname (Render internal URLs differ from public URLs) |
| Render service sleeps (free tier) | Render free tier spins down after inactivity; first request may take ~30s to wake up |

---

## Reflection (Short Written Reflection)

> Lab 6 required clients to know three different ports (3001, 3002, 3003) and manage three separate base URLs. Every concern — error handling, logging — had to be considered per service.  
>
> Lab 7 collapses this to a single address. The client calls one URL and the gateway handles the rest invisibly. This dramatically simplifies client code and centralizes operational concerns: all logs flow through one place, all errors are formatted consistently, and adding authentication or rate-limiting would require a change in only the gateway, not in each of the three services.  
>
> From an operations standpoint, hiding internal service ports also reduces the attack surface — callers cannot bypass the gateway's error handling or logging by directly targeting backend services. Cloud deployment made this concrete: only the gateway has a public URL; the microservices are completely unreachable from the internet.
