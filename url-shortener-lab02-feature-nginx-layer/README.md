# URL Shortener with Nginx Reverse Proxy

A containerized URL shortening service built with Node.js, Express, MongoDB, and Nginx as a reverse proxy. The application provides URL shortening and redirection functionality with visit tracking and health monitoring. This document outlines the system architecture, design decisions, deployment instructions, and testing procedures.

---

## Prologue

This project demonstrates a modern, containerized web application following microservices principles. It consists of three primary components:

- **Nginx** as a reverse proxy and load balancer.
- **Node.js/Express** application handling business logic and API endpoints.
- **MongoDB** for persistent storage of URL mappings and visit analytics.

The system is designed with scalability, observability, and reliability in mind. It includes health checks, structured logging, input validation, and persistent data volumes. The following sections detail the architecture, data flow, deployment steps, and testing strategies.

All diagrams in this document use a greyscale palette to maintain a professional, distraction‑free appearance while clearly conveying system interactions.

---

## Architecture

### System Overview

The diagram below illustrates the high-level interaction between components.

```mermaid
graph TD
    Client[Client Browser] -->|HTTP :80| Nginx[Nginx Reverse Proxy]
    Nginx -->|Forward Requests| App[Node.js App :3000]
    App -->|Database Queries| MongoDB[MongoDB :27017]

    subgraph Docker[Docker Environment]
        Nginx
        App
        MongoDB
    end

    style Client fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Nginx fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style App fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style MongoDB fill:#555,stroke:#000,stroke-width:2px,color:#fff
    style Docker fill:#666,stroke:#111,stroke-width:2px,color:#fff
```

### Complete System Overview

A more detailed view includes the client layer, proxy layer, application layer, data layer, and infrastructure.

```mermaid
graph TB
    subgraph "Client Layer"
        Users[Users]
        Browser[Web Browser]
        API[API Clients]
    end

    subgraph "Proxy Layer"
        Nginx[Nginx Reverse Proxy<br/>Port 80]
    end

    subgraph "Application Layer"
        App[Node.js Express App<br/>Port 3000]
        Routes[Routes: /urls, /:id, /health]
        Logic[Business Logic]
    end

    subgraph "Data Layer"
        MongoDB[MongoDB Database<br/>Port 27017]
        Schema[URL Schema]
    end

    subgraph "Infrastructure"
        Docker[Docker Containers]
        Network[Docker Network]
        Volumes[Persistent Volumes]
    end

    Users --> Browser
    Users --> API
    Browser --> Nginx
    API --> Nginx

    Nginx --> App
    App --> Routes
    Routes --> Logic
    Logic --> MongoDB
    MongoDB --> Schema

    App -.- Docker
    Nginx -.- Docker
    MongoDB -.- Docker

    Docker --> Network
    Docker --> Volumes

    style Users fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Nginx fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style App fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style MongoDB fill:#555,stroke:#000,stroke-width:2px,color:#fff
    style Docker fill:#666,stroke:#111,stroke-width:2px,color:#fff
```

### Application Layer Architecture

Inside the Node.js application, the request flow follows a typical MVC pattern.

```mermaid
graph TD
    Client[Client] --> Router[Express Router]
    Router --> Controller[Controllers]
    Controller --> Service[Business Logic]
    Service --> Model[Data Model]
    Model --> DB[MongoDB]

    subgraph Routes[API Routes]
        HealthRoute[GET /health]
        CreateRoute[POST /urls]
        RedirectRoute[GET /:id]
    end

    subgraph Logic[Core Functions]
        Generate[Generate Short ID]
        Validate[Validate URL]
        Track[Track Visits]
    end

    Router --> Routes
    Service --> Logic

    style Client fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Router fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style Controller fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style Service fill:#555,stroke:#000,stroke-width:2px,color:#fff
    style Model fill:#666,stroke:#000,stroke-width:2px,color:#fff
    style DB fill:#777,stroke:#000,stroke-width:2px,color:#fff
```

### Data Flow Overview

The data flows through three main operations: URL creation, redirection, and health checks.

```mermaid
graph LR
    Client[Client] -->|Request| Nginx[Nginx :80]
    Nginx -->|Proxy| App[Node.js :3000]
    App -->|Query| DB[MongoDB :27017]

    subgraph Flow[Data Flow Types]
        Create[Create Short URL]
        Redirect[Redirect to Long URL]
        Health[Health Check]
    end

    subgraph Network[Docker Network]
        Bridge[url-shortener-network]
    end

    App --> Flow
    DB --> Flow

    Nginx -.- Network
    App -.- Network
    DB -.- Network

    style Client fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Nginx fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style App fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style DB fill:#555,stroke:#000,stroke-width:2px,color:#fff
    style Flow fill:#666,stroke:#111,stroke-width:2px,color:#fff
```

---

## Technology Stack

| Component       | Technology                | Role                                      |
|-----------------|---------------------------|-------------------------------------------|
| Proxy Layer     | Nginx                     | Reverse proxy, load balancing, SSL termination |
| Backend         | Node.js / Express         | Business logic, REST API                   |
| Database        | MongoDB                   | Persistent storage of URL mappings         |
| Containerization| Docker & Docker Compose   | Orchestration, isolation, portability      |
| Logging         | Winston                   | Structured JSON logs                        |
| Monitoring      | Docker Health Checks      | Container health verification               |

---

## Features

- **URL Shortening** – Generates 7‑character unique IDs (62⁷ possible combinations) for long URLs.
- **Redirection with Tracking** – 301 redirects to original URLs while incrementing a visit counter for analytics.
- **Nginx Reverse Proxy** – Handles incoming requests, forwards to the Node.js app, and can be extended for SSL and load balancing.
- **Container Orchestration** – Multi‑container setup with Docker Compose for easy deployment.
- **Health Monitoring** – Docker health checks and a `/health` endpoint to verify service availability.
- **Structured Logging** – Winston produces JSON logs suitable for aggregation tools (e.g., ELK stack).
- **Data Persistence** – MongoDB data stored in Docker volumes to survive container restarts.
- **Input Validation** – URL format validation and proper error responses.

---

## Quick Start

### Deployment Process

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Docker as Docker

    Dev->>Docker: docker-compose up --build

    Note over Docker: Building Images
    Docker->>Docker: Build Nginx
    Docker->>Docker: Build Node App
    Docker->>Docker: Pull MongoDB

    Note over Docker: Setup Infrastructure
    Docker->>Docker: Create Network
    Docker->>Docker: Create Volumes

    Note over Docker: Start Services
    Docker->>Docker: Start MongoDB (27017)
    Docker->>Docker: Start App (3000)
    Docker->>Docker: Start Nginx (80)

    Note over Docker: Health Checks
    Docker->>Docker: Verify /health endpoint

    Docker-->>Dev: All services running
```

### Deployment State Management

```mermaid
stateDiagram-v2
    [*] --> Starting : docker-compose up

    Starting --> Building : Building Images
    Building --> Creating : Creating Containers
    Creating --> Starting_Services : Starting Services

    Starting_Services --> MongoDB_Ready : MongoDB (27017)
    Starting_Services --> App_Ready : Node.js (3000)
    Starting_Services --> Nginx_Ready : Nginx (80)

    MongoDB_Ready --> All_Ready
    App_Ready --> All_Ready
    Nginx_Ready --> All_Ready

    All_Ready --> Healthy : Health Checks Pass

    Healthy --> Running : System Operational

    Running --> Healthy : Monitoring OK
    Running --> Degraded : Issues Detected

    Degraded --> Healthy : Auto Recovery
    Degraded --> Failed : Max Retries

    Failed --> Starting : Restart

    Running --> Stopping : docker-compose down
    Stopping --> [*] : Clean Shutdown
```

1. **Clone the repository**
   ```bash
   git clone https://github.com/Shahriarin2garden/url-shortener-lab02-feature-nginx-layer.git
   cd url-shortener-lab02-feature-nginx-layer
   ```

2. **Start all services**
   ```bash
   docker-compose up --build -d
   ```

3. **Verify services are running**
   ```bash
   docker-compose ps
   ```

4. **Test health endpoint**
   ```bash
   curl http://localhost/health
   ```

---

## API Endpoints

### Health Check
- **Endpoint**: `GET /health`
- **Response**: Service status, uptime, and timestamp
  ```json
  {
    "status": "healthy",
    "timestamp": "2025-08-28T10:30:00.000Z",
    "uptime": 120.45
  }
  ```

### Create Short URL
- **Endpoint**: `POST /urls`
- **Headers**: `Content-Type: application/json`
- **Body**: `{"longUrl": "https://example.com"}`
- **Response**: `{"shortUrl": "http://localhost/aBc123D"}`

### Redirect to Original URL
- **Endpoint**: `GET /:shortUrlId`
- **Response**: 301 redirect to original URL with visit counter increment

---

## API Workflows

### URL Creation Workflow

```mermaid
sequenceDiagram
    participant C as Client
    participant N as Nginx
    participant A as App
    participant DB as MongoDB

    C->>N: POST /urls<br/>{"longUrl": "https://example.com"}
    N->>A: Forward request

    Note over A: Validate URL format
    A->>A: Check https?:// pattern

    Note over A: Generate Short ID
    A->>A: Create 7-char random ID

    Note over A: Save to Database
    A->>DB: Save URL mapping
    DB-->>A: Success

    Note over A: Return Response
    A-->>N: {"shortUrl": "http://localhost/abc123"}
    N-->>C: Return short URL

    Note over A,DB: Error Handling
    alt Invalid URL
        A-->>C: 400 Bad Request
    else Database Error
        DB-->>A: Error
        A-->>C: 500 Internal Error
    end
```

### URL Redirection Workflow

```mermaid
sequenceDiagram
    participant C as Client
    participant N as Nginx
    participant A as App
    participant DB as MongoDB

    C->>N: GET /abc123
    N->>A: Forward request

    Note over A: Find URL
    A->>DB: Query by shortUrlId

    alt URL Found
        DB-->>A: Return URL data

        Note over A: Update Analytics
        A->>DB: Increment visit count

        Note over A: Redirect
        A-->>N: 301 Redirect<br/>Location: https://example.com
        N-->>C: 301 Redirect

        Note over C: Browser follows redirect
        C->>C: Navigate to original URL

    else URL Not Found
        DB-->>A: No results
        A-->>N: 404 Not Found
        N-->>C: {"error": "Not Found"}
    end
```

---

## Database Schema

```mermaid
erDiagram
    URL {
        string shortUrlId "Unique 7-char ID"
        string longUrl "Original URL"
        date createdAt "Timestamp"
        number visits "Visit counter"
    }

    URL ||--|| UNIQUE_INDEX : "indexed on"

    UNIQUE_INDEX {
        string field "shortUrlId"
        boolean unique "true"
    }
```

**Schema Details:**
```javascript
{
  shortUrlId: String,    // 7-character unique identifier (indexed)
  longUrl: String,       // Original URL (required, validated)
  createdAt: Date,       // Auto-generated timestamp
  visits: Number         // Visit counter (incremented on each redirect)
}
```

---

## Infrastructure

### Nginx Configuration

The Nginx server acts as a reverse proxy, forwarding all requests to the Node.js application.

```mermaid
graph TD
    Client[Client Request :80] --> Nginx[Nginx Server]

    subgraph Config[Nginx Configuration]
        Events[Events<br/>worker_connections: 1024]
        Upstream[Upstream Backend<br/>server app:3000]
        Location[Location /<br/>proxy_pass to backend]
    end

    Nginx --> Config
    Config --> App[Node.js App :3000]

    style Client fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Nginx fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style Config fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style App fill:#555,stroke:#000,stroke-width:2px,color:#fff
```

### Health Monitoring System

Docker executes periodic health checks against the application’s `/health` endpoint.

```mermaid
sequenceDiagram
    participant Docker as Docker
    participant App as App Container
    participant Health as /health Endpoint

    Note over Docker,Health: Health Check (Every 30s)

    loop Every 30 seconds
        Docker->>App: Execute health check
        App->>Health: curl -f localhost:3000/health

        alt Healthy
            Health-->>App: 200 OK<br/>{"status": "healthy", "uptime": 123}
            App-->>Docker: Health check PASSED
        else Unhealthy
            Health-->>App: 500 Error
            App-->>Docker: Health check FAILED
            Docker->>Docker: Retry (3x max)
        end
    end

    Note over Docker: Health Check Config<br/>interval: 30s, timeout: 10s, retries: 3
```

---

## Error Handling

The application implements consistent error handling across all endpoints.

```mermaid
graph TD
    Request[Incoming Request] --> Validate{Valid Input?}

    Validate -->|Yes| Process[Process Request]
    Validate -->|No| BadRequest[400 Bad Request]

    Process --> Database[Database Query]
    Database --> Found{Found?}

    Found -->|Yes| Success[200/301 Success]
    Found -->|No| NotFound[404 Not Found]

    Database -->|Error| ServerError[500 Server Error]

    subgraph Errors[Error Responses]
        BadRequest
        NotFound
        ServerError
    end

    BadRequest --> Logger[Winston Logger]
    NotFound --> Logger
    ServerError --> Logger

    style Request fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Success fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style Errors fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style Logger fill:#555,stroke:#000,stroke-width:2px,color:#fff
```

---

## Environment Variables

| Variable     | Description                         | Default                                  |
|--------------|-------------------------------------|------------------------------------------|
| `PORT`       | Application port                    | `3000`                                   |
| `MONGO_URI`  | MongoDB connection string           | `mongodb://mongodb:27017/url_shortener`  |
| `BASE_URL`   | Base URL for short links            | `http://localhost`                       |

---

## Testing

### Testing Workflow

```mermaid
graph TD
    Start[Start Testing] --> Setup[Setup Environment]
    Setup --> Health[Health Check]

    Health --> Create[Create URLs]
    Create --> Redirect[Test Redirects]
    Redirect --> Analytics[Visit Tracking]

    Analytics --> Errors[Error Handling]
    Errors --> Performance[Performance Tests]
    Performance --> Complete[Testing Complete]

    subgraph Tests[Test Categories]
        Unit[Unit Tests]
        Integration[Integration Tests]
        E2E[End-to-End Tests]
    end

    Create --> Tests
    Redirect --> Tests
    Analytics --> Tests

    style Start fill:#222,stroke:#000,stroke-width:2px,color:#fff
    style Health fill:#333,stroke:#000,stroke-width:2px,color:#fff
    style Create fill:#444,stroke:#000,stroke-width:2px,color:#fff
    style Redirect fill:#555,stroke:#000,stroke-width:2px,color:#fff
    style Complete fill:#666,stroke:#000,stroke-width:2px,color:#fff
```

### Prerequisites for Testing

Before running tests, ensure the application is running:

```bash
# Start the application
docker-compose up --build -d

# Verify all containers are healthy
docker-compose ps
```

### 1. Health Check Testing

```bash
curl -X GET http://localhost/health
```

**Expected Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-08-28T10:30:00.123Z",
  "uptime": 45.67
}
```

### 2. URL Shortening Testing

```bash
curl -X POST http://localhost/urls \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "https://www.google.com"}'
```

**Expected Response:**
```json
{
  "shortUrl": "http://localhost/aBc123D"
}
```

### 3. URL Redirection Testing

```bash
# Test redirect (will follow the redirect)
curl -L http://localhost/aBc123D

# Test redirect headers only
curl -I http://localhost/aBc123D
```

**Expected Redirect Response:**
```
HTTP/1.1 301 Moved Permanently
Location: https://www.google.com
```

### 4. Error Handling Testing

```bash
# Invalid URL
curl -X POST http://localhost/urls \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "invalid-url"}'
```

**Expected Response:**
```json
{
  "error": "Invalid URL"
}
```

### 5. Container and Service Testing

```bash
# Check container logs
docker-compose logs app
```

### 6. Database Testing

```bash
# Connect to MongoDB container
docker exec -it url-shortener-mongodb-1 mongosh url_shortener

# MongoDB queries
db.urls.find()
```

### 7. Performance Testing (Basic)

```bash
# Concurrent URL creation
for i in {1..10}; do
  curl -X POST http://localhost/urls \
    -H "Content-Type: application/json" \
    -d "{\"longUrl\": \"https://example$i.com\"}" &
done
wait
```

### 8. End-to-End Testing Script

Create a script (`test.sh`) with the following content:

```bash
#!/bin/bash
echo "=== URL Shortener End-to-End Test ==="

echo "1. Testing Health Endpoint..."
curl -s http://localhost/health | jq .

echo -e "\n2. Creating Short URL..."
RESPONSE=$(curl -s -X POST http://localhost/urls \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "https://www.github.com"}')
echo $RESPONSE

SHORT_URL=$(echo $RESPONSE | jq -r '.shortUrl')
SHORT_ID=$(echo $SHORT_URL | sed 's/.*\///')

echo -e "\n3. Testing Redirect..."
curl -I http://localhost/$SHORT_ID

echo -e "\nTest Complete!"
```

Run it:
```bash
chmod +x test.sh
./test.sh
```

### 9. Testing Checklist

- [ ] Health endpoint responds correctly
- [ ] Can create short URLs with valid long URLs
- [ ] Short URLs redirect to correct original URLs
- [ ] Visit counter increments correctly
- [ ] Invalid URLs are rejected with appropriate error
- [ ] Non-existent short URLs return 404
- [ ] All containers are running and healthy
- [ ] Database stores URL records correctly
- [ ] Nginx proxy forwards requests correctly
- [ ] Application logs are generated properly

---

## Troubleshooting

### Common Issues

1. **Port already in use**
   ```bash
   # Check what's using port 80
   lsof -i :80
   # Stop conflicting services or change ports in docker-compose.yml
   ```

2. **Container won't start**
   ```bash
   # Check logs for errors
   docker-compose logs app
   docker-compose logs nginx
   ```

3. **Database connection failed**
   ```bash
   # Verify MongoDB is running
   docker-compose ps mongodb
   # Check database logs
   docker-compose logs mongodb
   ```

4. **404 errors for all requests**
   ```bash
   # Check nginx configuration
   docker-compose exec nginx cat /etc/nginx/nginx.conf
   ```

---

## Development

### Running without Docker
```bash
# Install dependencies
npm install

# Set environment variables
export PORT=3000
export MONGO_URI=mongodb://localhost:27017/url_shortener
export BASE_URL=http://localhost:3000

# Start MongoDB (if not using Docker)
mongod

# Run application
npm start
# or for development with auto-reload
npm run dev
```

### Project Structure
```
url-shortener-lab02-feature-nginx-layer/
├── docker-compose.yml          # Container orchestration
├── Dockerfile                  # Application container build
├── package.json                # Node.js dependencies
├── nginx/
│   ├── Dockerfile              # Nginx container build
│   └── nginx.conf              # Nginx configuration
└── src/
    ├── app.js                  # Application entry point
    ├── config/index.js         # Environment configuration
    ├── controllers/urlController.js  # HTTP request handlers
    ├── models/urlModel.js       # Database schema
    ├── routes/urlRoutes.js      # API route definitions
    └── services/urlService.js   # Business logic
```

---

## License

ISC License
