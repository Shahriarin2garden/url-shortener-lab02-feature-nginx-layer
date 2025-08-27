# URL Shortener Lab02 - Feature Nginx Layer

A production-ready, containerized URL shortening service built with Node.js, Express, MongoDB, and Nginx as a reverse proxy layer. This project demonstrates modern microservices architecture with containerization, load balancing, and comprehensive monitoring capabilities.

## 🏗️ Architecture Overview

### System Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        Docker Compose Environment                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                           Network Layer                                        │ │
│  │                      url-shortener-network                                     │ │
│  │                         (Bridge Driver)                                       │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                     │
│  ┌──────────────────┐  ┌───────────────────────────┐  ┌──────────────────────────┐ │
│  │  NGINX LAYER     │  │     APPLICATION LAYER     │  │      DATA LAYER         │ │
│  │  (Port 80)       │  │      (Port 3000)          │  │     (Port 27017)        │ │
│  │                  │  │                           │  │                          │ │
│  │ ┌──────────────┐ │  │  ┌─────────────────────┐  │  │ ┌──────────────────────┐ │
│  │ │ nginx:alpine │ │  │  │   node:18-alpine    │  │  │ │    mongo:latest      │ │
│  │ │              │ │  │  │                     │  │  │ │                      │ │
│  │ │ Reverse      │ │  │  │ Express.js Server   │  │  │ │ MongoDB Database     │ │
│  │ │ Proxy        │◄─┼──┼──┤ RESTful API        │◄─┼──┼─┤ Document Store       │ │
│  │ │ Load         │  │  │  │ Business Logic     │  │  │ │ Persistent Storage   │ │
│  │ │ Balancer     │  │  │  │ Health Monitoring  │  │  │ │                      │ │
│  │ └──────────────┘ │  │  └─────────────────────┘  │  │ └──────────────────────┘ │
│  └──────────────────┘  └───────────────────────────┘  └──────────────────────────┘ │
│           │                          │                            │                │
│           │                          │                            │                │
│  ┌────────▼──────────┐    ┌──────────▼────────────┐    ┌─────────▼──────────────┐ │
│  │ nginx_logs Volume │    │  Application Volume   │    │  mongodb_data Volume  │ │
│  │ /var/log/nginx    │    │  /app (live reload)   │    │  /data/db             │ │
│  └───────────────────┘    └───────────────────────┘    └────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘

External Access:
┌─────────────────┐     HTTP/HTTPS     ┌──────────────────┐
│   Client/User   │ ──────────────────► │  localhost:80    │
│  (Browser/API)  │                    │  (Nginx Entry)   │
└─────────────────┘                    └──────────────────┘
```

### Component Interaction Flow
```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            Request/Response Flow                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘

1. URL Creation Flow:
Client ──POST /urls──► Reverse Proxy ──forward──► API Server ──► Document Database
   │                     │                          │                    │
   │                     │         ┌────────────────▼────────────────────▼──────────┐
   │                     │         │              Processing Chain                  │
   │                     │         │                                                │
   │                     │         │  Route Handler ──► HTTP Controller            │
   │                     │         │       │                │                      │
   │                     │         │       │                ▼                      │
   │                     │         │       │         Business Logic                │
   │                     │         │       │         - ID Generation               │
   │                     │         │       │         - URL Shortening              │
   │                     │         │       │                │                      │
   │                     │         │       │                ▼                      │
   │                     │         │       │         Data Model (ODM)             │
   │                     │         │       │         - Schema validation           │
   │                     │         │       │         - Database persistence        │
   │                     │         └───────┼────────────────┼──────────────────────┘
   │                     │                 │                │
   ◄─────────────────────◄─────────────────┘                │
   JSON Response                                             │
   {"shortUrl": "http://localhost/aBc123D"}                  │
                                                             │
2. URL Redirection Flow:                                     │
Client ──GET /aBc123D──► Reverse Proxy ──forward──► API Server ──► Document Database
   │                       │                         │                │
   │                       │                         │                │
   │                       │         ┌───────────────▼────────────────▼─────────┐
   │                       │         │           Redirection Chain             │
   │                       │         │                                         │
   │                       │         │  Route Handler ──► HTTP Controller     │
   │                       │         │       │                │               │
   │                       │         │       │                ▼               │
   │                       │         │       │         Business Logic        │
   │                       │         │       │         - URL Resolution      │
   │                       │         │       │         - Analytics Update    │
   │                       │         │       │                │               │
   │                       │         │       │                ▼               │
   │                       │         │       │         Data Model            │
   │                       │         │       │         - Document Query      │
   │                       │         │       │         - Counter Increment   │
   │                       │         └───────┼────────────────┼───────────────┘
   │                       │                 │                │
   ◄─────────────────────◄─────────────────┘                │
   301 Redirect to original URL                              │
                                                             │
3. Health Check Flow:                                        │
Client ──GET /health──► Reverse Proxy ──forward──► API Server
   │                      │                        │
   │                      │            ┌───────────▼─────────────┐
   │                      │            │     Health Monitor      │
   │                      │            │                         │
   │                      │            │ - Process uptime        │
   │                      │            │ - Current timestamp     │
   │                      │            │ - Service status        │
   │                      │            └───────────┼─────────────┘
   │                      │                        │
   ◄──────────────────────◄────────────────────────┘
   JSON Response
   {"status": "healthy", "timestamp": "...", "uptime": 123.45}
```

### Technical Specifications

| Component | Technology | Version | Port | Memory | CPU | Purpose |
|-----------|------------|---------|------|--------|-----|---------|
| **Nginx** | nginx:alpine | Latest | 80 (external) | ~10MB | Low | Reverse Proxy, Load Balancer, Static Content |
| **API Server** | node:18-alpine | 18.x | 3000 (internal) | ~50MB | Medium | REST API, Business Logic, Request Processing |
| **Database** | mongo:latest | 7.x | 27017 (internal/external) | ~100MB | Medium | Document Storage, Data Persistence |

### Core Features Matrix

| Feature | Implementation | Status | Details |
|---------|---------------|--------|---------|
| **URL Shortening** | 7-character alphanumeric IDs | ✅ Active | Random ID generation with 62-character charset |
| **URL Redirection** | 301 redirects with analytics | ✅ Active | Automatic visit tracking with database updates |
| **Health Monitoring** | Multi-layer health checks | ✅ Active | Container health checks + health endpoint |
| **Error Handling** | Comprehensive validation | ✅ Active | URL validation, 404/500 error responses |
| **Containerization** | Docker + Docker Compose | ✅ Active | Multi-container orchestration with networking |
| **Reverse Proxy** | Load balancing | ✅ Active | Upstream backend configuration |
| **Logging** | Structured logging | ✅ Active | JSON formatted logs with timestamps |
| **Data Persistence** | Document database with volumes | ✅ Active | Named volumes for data and logs |

### API Specification

| Method | Endpoint | Request Body | Response | Status Codes | Purpose |
|--------|----------|-------------|----------|-------------|---------|
| `POST` | `/urls` | `{"longUrl": "https://example.com"}` | `{"shortUrl": "http://localhost/aBc123D"}` | 201, 400, 500 | Create short URL |
| `GET` | `/:shortUrlId` | None | 301 Redirect to original URL | 301, 404, 500 | Redirect to original |
| `GET` | `/health` | None | `{"status": "healthy", "timestamp": "...", "uptime": 123.45}` | 200 | Service health check |

### Database Schema

```
Document Structure:
{
  shortUrlId: String,    // 7-character unique identifier (indexed)
  longUrl: String,       // Original URL (validated)
  createdAt: Date,       // Auto-generated timestamp
  visits: Number         // Analytics counter (auto-incremented)
}

Collection: urls
Indexes: _id (default), shortUrlId (unique B-tree)
```

### Environment Configuration

```
Application Configuration:
- PORT: 3000 (internal container port)
- MONGO_URI: Internal container DNS resolution
- BASE_URL: Base URL for short link generation

Docker Environment Variables:
- PORT=3000
- MONGO_URI=mongodb://mongodb:27017/url_shortener  
- BASE_URL=http://localhost
```

## 🏗️ Detailed Architecture Analysis

### Container Architecture Deep Dive

#### 1. **Reverse Proxy Layer (Lightweight Alpine)**
```
┌─────────────────────────────────────────────────────────────┐
│                    Reverse Proxy Container                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              HTTP Configuration                     │    │
│  │                                                     │    │
│  │  Event-driven Architecture:                        │    │
│  │    worker_connections: 1024                        │    │
│  │                                                     │    │
│  │  HTTP Module:                                       │    │
│  │    upstream backend {                              │    │
│  │      server api_server:3000                        │    │
│  │    }                                               │    │
│  │                                                     │    │
│  │    server block {                                   │    │
│  │      listen: 80                                     │    │
│  │      proxy_pass: upstream backend                   │    │
│  │    }                                               │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  External Port 80 ◄──────────────► Internal Communication  │
│  Volume: proxy_logs:/var/log/proxy                          │
└─────────────────────────────────────────────────────────────┘
```

**Key Features:**
- **Load Balancing**: Single upstream server (scalable to multiple)
- **Reverse Proxy**: All requests forwarded to API server container
- **Alpine Linux**: Minimal 5MB base image
- **Log Persistence**: Proxy logs stored in named volume

#### 2. **Application Container (Lightweight Node.js Runtime)**
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Web API Application Container                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │                        Web Framework Application                        │    │
│  │                                                                         │    │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │    │
│  │  │   Server Boot   │  │   Middleware     │  │     Routes           │   │    │
│  │  │                 │  │                  │  │                      │   │    │
│  │  │ ┌─────────────┐ │  │ ┌──────────────┐ │  │ ┌──────────────────┐ │   │    │
│  │  │ │HTTP Server  │ │  │ │JSON Parser   │ │  │ │POST /urls        │ │   │    │
│  │  │ │Port: 3000   │ │  │ │Request Logger│ │  │ │GET /:shortUrlId  │ │   │    │
│  │  │ │DB Connection│ │  │ │Error Handler │ │  │ │GET /health       │ │   │    │
│  │  │ └─────────────┘ │  │ └──────────────┘ │  │ └──────────────────┘ │   │    │
│  │  └─────────────────┘  └──────────────────┘  └──────────────────────┘   │    │
│  │                                 │                       │               │    │
│  │                                 ▼                       ▼               │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │    │
│  │  │                     HTTP Controllers Layer                         │   │    │
│  │  │                                                                     │   │    │
│  │  │  Request Handlers:                                                  │   │    │
│  │  │  ┌─────────────────────┐  ┌──────────────────────────────────────┐ │   │    │
│  │  │  │ createShortUrl()    │  │ redirectToLongUrl()                  │ │   │    │
│  │  │  │ - Validate input    │  │ - Extract ID from params             │ │   │    │
│  │  │  │ - Call business     │  │ - Call business layer                │ │   │    │
│  │  │  │ - Return response   │  │ - Send 301 redirect                  │ │   │    │
│  │  │  │ - Handle errors     │  │ - Handle 404/500 errors              │ │   │    │
│  │  │  └─────────────────────┘  └──────────────────────────────────────┘ │   │    │
│  │  └─────────────────────────────────────────────────────────────────────┘   │    │
│  │                                 │                                           │    │
│  │                                 ▼                                           │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │    │
│  │  │                      Business Logic Layer                          │   │    │
│  │  │                                                                     │   │    │
│  │  │  Core Functions:                                                    │   │    │
│  │  │  ┌────────────────────────────────────────────────────────────────┐ │   │    │
│  │  │  │ generateShortUrlId():                                          │ │   │    │
│  │  │  │ - 62-character alphabet (A-Z, a-z, 0-9)                       │ │   │    │
│  │  │  │ - 7-character random string generation                         │ │   │    │
│  │  │  │ - 62^7 = 3.5+ trillion possible combinations                   │ │   │    │
│  │  │  └────────────────────────────────────────────────────────────────┘ │   │    │
│  │  │  ┌────────────────────────────────────────────────────────────────┐ │   │    │
│  │  │  │ shortenUrl(longUrl):                                           │ │   │    │
│  │  │  │ - Generate unique shortUrlId                                   │ │   │    │
│  │  │  │ - Create new document                                          │ │   │    │
│  │  │  │ - Save to database                                             │ │   │    │
│  │  │  │ - Return formatted shortUrl                                    │ │   │    │
│  │  │  └────────────────────────────────────────────────────────────────┘ │   │    │
│  │  │  ┌────────────────────────────────────────────────────────────────┐ │   │    │
│  │  │  │ getLongUrl(shortUrlId):                                        │ │   │    │
│  │  │  │ - Query database for shortUrlId                                │ │   │    │
│  │  │  │ - Increment visit counter                                      │ │   │    │
│  │  │  │ - Update document in database                                  │ │   │    │
│  │  │  │ - Return original longUrl                                      │ │   │    │
│  │  │  └────────────────────────────────────────────────────────────────┘ │   │    │
│  │  └─────────────────────────────────────────────────────────────────────┘   │    │
│  │                                 │                                           │    │
│  │                                 ▼                                           │    │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │    │
│  │  │                       Data Model Layer                             │   │    │
│  │  │                                                                     │   │    │
│  │  │  Database Schema (Object Document Mapper):                         │   │    │
│  │  │  ┌─────────────────────────────────────────────────────────────┐   │   │    │
│  │  │  │ Document Schema Definition:                                 │   │   │    │
│  │  │  │   shortUrlId: String (unique index)                        │   │   │    │
│  │  │  │   longUrl: String (required)                               │   │   │    │
│  │  │  │   createdAt: Date (auto-timestamp)                         │   │   │    │
│  │  │  │   visits: Number (analytics counter)                       │   │   │    │
│  │  │  └─────────────────────────────────────────────────────────────┘   │   │    │
│  │  └─────────────────────────────────────────────────────────────────────┘   │    │
│  │                                                                         │    │
│  │  Structured Logger: JSON formatted logs with timestamps                 │    │
│  │  Health Check: /health endpoint with uptime and timestamp              │    │
│  │  Error Handling: Global error middleware with 500 responses            │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                 │
│  Internal Port 3000 ◄──────────► Database Connection                           │
│  Volume: /app (development live reload), /app/node_modules                     │
│  Environment: PORT=3000, MONGO_URI, BASE_URL                                   │
│  Health Check: curl -f http://localhost:3000/health every 30s                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 3. **Database Container (Document Store)**
```
┌─────────────────────────────────────────────────────────────┐
│                    Document Database Container              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                Database Structure                   │    │
│  │                                                     │    │
│  │  Database: url_shortener                           │    │
│  │  ┌─────────────────────────────────────────────┐   │    │
│  │  │           Collection: urls                  │   │    │
│  │  │                                             │   │    │
│  │  │  Documents Structure:                       │   │    │
│  │  │  {                                          │   │    │
│  │  │    _id: ObjectId("..."),                    │   │    │
│  │  │    shortUrlId: "aBc123D",     ◄── Unique    │   │    │
│  │  │    longUrl: "https://...",                  │   │    │
│  │  │    createdAt: ISODate("..."),               │   │    │
│  │  │    visits: 42,                              │   │    │
│  │  │    __v: 0                                   │   │    │
│  │  │  }                                          │   │    │
│  │  │                                             │   │    │
│  │  │  Indexes:                                   │   │    │
│  │  │  - _id (default)                            │   │    │
│  │  │  - shortUrlId (unique)                      │   │    │
│  │  └─────────────────────────────────────────────┘   │    │
│  │                                                     │    │
│  │  Connection String:                                 │    │
│  │  mongodb://database_server:27017/url_shortener     │    │
│  │                   ▲         ▲                       │    │
│  │              DNS Name    Database                   │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  Port 27017 (Internal + External for development)          │
│  Volume: database_data:/data/db (persistent storage)       │
│  Auto-restart: unless-stopped                              │
└─────────────────────────────────────────────────────────────┘
```

### Network Communication Detailed Analysis

#### Container Network: `url-shortener-network`
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Bridge Network (Internal Container Network)                 │
│                                                                                 │
│  Container Internal DNS Resolution:                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                      │
│  │reverse_proxy│────▶│ api_server  │────▶│  database   │                      │
│  │  proxy:80   │     │ server:3000 │     │  db:27017   │                      │
│  └─────────────┘     └─────────────┘     └─────────────┘                      │
│                                                                                 │
│  External Access:                                                               │
│  ┌─────────────────┐                                                           │
│  │  Host:80  ────────────────────────────────────────────┐                     │
│  │ (localhost)     │                                     │                     │
│  └─────────────────┘                                     │                     │
│                                                          │                     │
│  Port Mapping:                                           │                     │
│  - reverse_proxy: 80:80 (host:container)                 ▼                     │
│  - database: 27017:27017 (development access)   ┌─────────────┐                │
│  - api_server: No external port (internal only) │reverse_proxy│                │
│                                                  │   :80       │                │
│                                                  └─────────────┘                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Analysis

#### 1. URL Shortening Request Flow
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            POST /urls Request                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

Step 1: Client Request
┌─────────────────┐
│ POST /urls      │
│ Content-Type:   │
│ application/json│
│ Body: {         │
│   "longUrl":    │
│   "https://..." │
│ }               │
└─────────▼───────┘
         │
Step 2: Reverse Proxy    ┌─────────────────────────────────────────┐
         │               │ HTTP Configuration                      │
         ▼               │ upstream backend { server api:3000; }   │
┌─────────────────┐      │ location / {                            │
│ reverse_proxy   │      │   proxy_pass http://backend;            │
│ proxy_pass      │      │ }                                       │
│ http://backend  │      └─────────────────────────────────────────┘
└─────────▼───────┘
         │
Step 3: Route Resolution
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ HTTP Route Mapping                                              │
│ router.post('/urls', createShortUrl);                           │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 4: Controller Processing
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ HTTP Request Handler                                            │
│                                                                 │
│ URL Creation Process:                                           │
│   - Extract longUrl from request body                          │
│   - Validate URL format (must start with http:// or https://)  │
│   - Call business logic layer                                  │
│   - Return shortUrl response                                   │
│   - Handle 400/500 error cases                                 │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 5: Business Logic Processing
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Core Business Operations                                        │
│                                                                 │
│ URL Shortening Algorithm:                                       │
│   - Generate 7-character random ID                             │
│   - Create new document object                                 │
│   - Persist to database                                        │
│   - Format and return complete short URL                       │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 6: Database Operation
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Database Persistence                                            │
│                                                                 │
│ Document Creation:                                              │
│   shortUrlId: "aBc123D"                                         │
│   longUrl: "https://example.com"                               │
│   createdAt: new Date()                                        │
│   visits: 0                                                    │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 7: Response Chain
         ▼
Client ◄── Reverse Proxy ◄── API Server ◄── {"shortUrl": "http://localhost/aBc123D"}
```

#### 2. URL Redirection Request Flow
```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        GET /:shortUrlId Request                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

Step 1: Client Request
┌─────────────────┐
│ GET /aBc123D    │
└─────────▼───────┘
         │
Step 2: Reverse Proxy (same as above)
         ▼
Step 3: Route Resolution
┌─────────────────────────────────────────────────────────────────┐
│ Dynamic Route Matching                                          │
│ router.get('/:shortUrlId', redirectToLongUrl);                  │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 4: Controller Processing
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Redirection Handler                                             │
│                                                                 │
│ URL Resolution Process:                                         │
│   - Extract shortUrlId from URL parameters                     │
│   - Call business logic to resolve URL                         │
│   - Send 301 permanent redirect response                       │
│   - Handle 404/500 error cases                                 │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 5: Business Logic Processing
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ URL Resolution & Analytics                                      │
│                                                                 │
│ Lookup and Analytics Process:                                  │
│   - Query database by shortUrlId                               │
│   - Increment visit counter                                    │
│   - Update document in database                                │
│   - Return original longUrl                                    │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 6: Database Operations
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ Database Lookup & Update                                        │
│                                                                 │
│ Operations:                                                     │
│   1. Query: findOne({ shortUrlId: "aBc123D" })                 │
│   2. Update: increment visits counter                          │
│   3. Save updated document                                     │
└─────────────────────────▼───────────────────────────────────────┘
         │
Step 7: HTTP Redirect Response
         ▼
Client ◄── 301 Redirect
           Location: https://example.com
```

## 📋 Implementation Details

### Container Configuration Analysis

#### **Multi-Container Orchestration**
```yaml
Services Architecture:
  reverse_proxy:
    - Custom build with configuration
    - External HTTP access port 80
    - Depends on API server availability
    - Internal container networking
    - Persistent log storage

  api_server:
    - Application runtime build
    - Internal port 3000 only
    - Environment configuration
    - Database dependency
    - Live code reload capability
    - Container health monitoring

  database:
    - Official document database image
    - Internal + external port access
    - Persistent data storage
    - Internal container networking

Volume Management:
  - database_data: Persistent database storage
  - proxy_logs: Reverse proxy log storage

Network Architecture:
  - Internal bridge network
  - Container DNS resolution
  - Isolated from host network
```

#### **Application Container Build Process**
```dockerfile
Build Strategy:
FROM lightweight_runtime:18-alpine
WORKDIR /app
COPY package*.json ./          # Dependency layer caching
RUN install_dependencies       # Install application dependencies  
COPY . .                       # Copy application source code
EXPOSE 3000                    # Document internal port
CMD ["start_command"]          # Application startup command
```

#### **Reverse Proxy Configuration**
```
HTTP Server Configuration:
- Event-driven architecture with worker connections: 1024
- Upstream backend server configuration
- Location-based request forwarding
- HTTP port 80 listener
- Proxy pass to internal backend
- Log directory creation
- Foreground daemon execution
```

### Application Architecture Layers

#### **Main Application Entry Point**
- Web framework initialization
- Structured logging configuration with JSON formatting and timestamps
- Middleware stack: JSON parsing, request logging, error handling
- Health check endpoint: Returns service status, timestamp, and uptime
- Route mounting and global error handling
- Database connection with error handling and logging
- Server startup on configured port

#### **Environment Configuration Management**
- Environment variable loading from system
- Application port configuration with defaults
- Database connection string configuration
- Base URL configuration for short link generation
- Production/development environment support

#### **Data Model Layer**
- Document schema definition with field types and constraints
- Unique indexing strategy for fast lookups
- Auto-timestamping for creation tracking
- Analytics counter for visit tracking
- Input validation and data persistence

#### **Business Logic Layer**
- 7-character ID generation using 62-character alphabet (A-Z, a-z, 0-9)
- 62^7 = 3.5+ trillion possible combinations for collision avoidance
- URL shortening: ID generation, document creation, database persistence
- URL resolution: Database lookup, analytics update, original URL return
- Error handling for not found cases

#### **HTTP Request Handlers**
- POST /urls: Input validation, business logic calls, JSON responses, error handling
- GET /:shortUrlId: Parameter extraction, URL resolution, 301 redirects, 404 handling
- Standardized HTTP status codes: 200, 201, 301, 400, 404, 500

#### **Route Definitions**
- URL creation endpoint mapping
- Dynamic route for redirection with parameter capture
- Route pattern matching for 7-character alphanumeric strings

#### **Reverse Proxy Configuration**
- Event-driven worker configuration with 1024 connections
- Upstream backend server definition
- Simple reverse proxy with all requests forwarded to API server
- No SSL/HTTPS configuration (development setup)
- No caching configuration - all requests processed by application


## Features

- ✅ URL shortening with 7-character alphanumeric IDs
- ✅ URL redirection with visit tracking
- ✅ Nginx reverse proxy configuration
- ✅ Complete Docker containerization
- ✅ Health check endpoints with uptime monitoring
- ✅ Structured logging with Winston
- ✅ MongoDB persistence with Mongoose ODM
- ✅ Environment-based configuration
- ✅ Input validation for URLs

## Quick Start

1. **Clone and navigate to the project**
```bash
git clone https://github.com/poridhioss/url-shortener-lab02.git
cd url-shortener-lab02
```

2. **Start all services with Docker Compose**
```bash
docker-compose up --build -d
```

3. **Verify all containers are running**
```bash
docker-compose ps
```

4. **Test the health endpoint**
```bash
curl http://localhost/health
```

## API Endpoints

### Health Check
```bash
GET /health
```
Returns service status, timestamp, and uptime.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-08-21T10:30:00.000Z",
  "uptime": 120.45
}
```

### Create Short URL
```bash
POST /urls
Content-Type: application/json

{
  "longUrl": "https://www.example.com"
}
```

**Response:**
```json
{
  "shortUrl": "http://localhost/a1B2c3D"
}
```

### Redirect to Original URL
```bash
GET /:shortUrlId
```
Redirects to the original URL and increments visit counter.

## Service Configuration

### Container Services

- **reverse_proxy**: HTTP reverse proxy (Port 80)
  - Custom build with configuration
  - Proxies requests to the API server
  - Logs stored in named volume

- **api_server**: Web API application (Port 3000, internal)
  - Application runtime build
  - Health checks every 30 seconds
  - Auto-restart on failure

- **database**: Document database (Port 27017)
  - Official database image
  - Data persisted in named volume
  - Connected via internal network

### Environment Variables

The application uses the following environment variables:

- `PORT`: Application port (default: 3000)
- `MONGO_URI`: Database connection string (internal container DNS)
- `BASE_URL`: Base URL for generated short URLs (http://localhost)

### Reverse Proxy Configuration

Internal proxy configuration:
- Upstream backend pointing to api_server:3000
- Simple reverse proxy configuration
- 1024 worker connections

## Application Architecture

```
Containerized URL Shortener Service/
├── Container Orchestration Config     # Multi-container orchestration
├── Application Container Build        # API server container
├── Dependency Management              # Application dependencies and scripts
├── Package Lock                       # Locked dependency versions
├── Version Control Config             # Git ignore rules
├── Reverse Proxy/
│   ├── Container Build Config         # Proxy container configuration
│   └── HTTP Configuration             # Reverse proxy config
└── Source Code/
    ├── Application Entry Point        # Web server and app initialization
    ├── Configuration/
    │   └── Environment Config         # Environment configuration management
    ├── HTTP Controllers/
    │   └── Request Handlers           # HTTP request handlers
    ├── Data Models/
    │   └── Schema Definition          # Database schema definition
    ├── API Routes/
    │   └── Endpoint Definitions       # API route definitions
    └── Business Logic/
        └── Core Operations            # Business logic and URL operations
```

## Database Schema

### URL Document Model
```
Document Structure:
{
  shortUrlId: String,    // 7-character unique identifier (indexed)
  longUrl: String,       // Original URL (validated)
  createdAt: Date,       // Auto-generated timestamp
  visits: Number         // Analytics counter (auto-incremented)
}

Collection: urls
Indexes: _id (default), shortUrlId (unique B-tree)
```

## Development

### Local Development (without containers)
- Install application dependencies
- Set environment variables for database connection and base URL
- Start development server with auto-reload capability
- Or start production server

### Container Development
- Build and start all services with orchestration
- Start in background mode
- View logs from all services or specific services
- Stop all services
- Stop and remove persistent volumes

## Testing Examples

### Create and Use Short URL
- Create a short URL via POST request with JSON payload
- Expected response contains the generated short URL
- Test the redirect functionality with follow redirects
- Test redirect headers only for validation

### Health and Monitoring
- Check service health endpoint
- Check container status
- View real-time logs from application

## Error Handling

The application includes comprehensive error handling:

- **400 Bad Request**: Invalid URL format (must start with http:// or https://)
- **404 Not Found**: Short URL ID not found in database
- **500 Internal Server Error**: Database connection issues or server errors

## Dependencies

### Production Dependencies
- **Web Framework**: Modern web application framework
- **Database ODM**: Document database object modeling
- **Environment Config**: Environment variable management
- **Structured Logging**: Production-grade logging library

### Development Dependencies
- **Development Server**: Auto-reload development server

### Container Dependencies
- **Application Runtime**: Lightweight runtime environment
- **Reverse Proxy**: Lightweight HTTP reverse proxy
- **Document Database**: Latest document database

## Monitoring and Logs

### Application Logs
The application uses Winston for structured logging:
- JSON formatted logs with timestamps
- HTTP request logging middleware
- Error stack trace logging

### Container Health Checks
The Node.js app includes a health check:
- Endpoint: `GET /health`
- Interval: 30 seconds
- Timeout: 10 seconds
- Retries: 3
- Start period: 40 seconds

### Volume Management
- `mongodb_data`: Persistent MongoDB data
- `nginx_logs`: Nginx access and error logs

## Network Configuration

All services communicate via the `url-shortener-network` bridge network:
- Internal DNS resolution between containers
- Isolated from host network by default
- Only Nginx port 80 exposed to host

## 🔧 Advanced Features & Monitoring

### Health Monitoring System

#### **Container Health Checks**
- Health check configuration with intervals and timeouts
- Health endpoint testing with startup grace periods
- Health status monitoring and failure detection
- Automatic container restart on health failures

#### **Health Endpoint Response**
- Service status indicator
- Current timestamp for verification
- Process uptime tracking in seconds

#### **Health Check Operations**
- Container health status verification
- Manual health endpoint testing
- Container log analysis for health check results

### Logging & Monitoring

#### **Structured Logger Configuration**
- JSON formatted logging with timestamps
- Configurable log levels and output formats
- Multiple transport options for log destinations
- Console logging for development and file logging for production

#### **Log Output Examples**
- Request logging with HTTP method and URL
- Database connection status logging
- Error logging with detailed timestamps
- Structured JSON format for log aggregation

#### **Monitoring Operations**
- View all container logs in real-time
- View specific service logs individually
- Filter logs by level (error, info, debug)
- Follow logs with timestamps for debugging

### Performance Analytics

#### **URL Visit Tracking**
- Automatic visit counter increment on each redirect
- Atomic counter updates to prevent race conditions
- Database document updates with visit statistics
- Analytics data persistence for reporting

#### **Database Query Optimization**
- Optimized database queries with proper indexing
- B-tree index on shortUrlId for O(log n) lookup time
- Database connection pooling for performance
- Aggregation pipelines for analytics queries

#### **Performance Monitoring Queries**
- Analytics aggregation for total URLs and visits
- Most visited URLs tracking
- Time-based URL creation analytics
- Database performance statistics

### Security Considerations

#### **URL Validation**
- Enhanced URL validation with proper format checking
- Protocol validation (HTTP/HTTPS only)
- Input sanitization and trimming
- XSS prevention through JSON-only API responses

#### **Rate Limiting (Future Enhancement)**
- Request rate limiting per IP address
- Configurable time windows and request limits
- Custom error messages for rate limit exceeded
- Middleware integration for URL creation endpoints

#### **Input Sanitization**
- URL sanitization before database storage
- Input trimming and normalization
- XSS prevention through JSON-only responses
- No HTML rendering - API-only service

### Error Handling & Resilience

#### **Global Error Handler**
- Comprehensive error handling middleware
- Error logging with context (URL, method, timestamp)
- Error stack trace logging for debugging
- Client-safe error responses without internal details

#### **Database Connection Resilience**
- Database connection with retry logic and timeouts
- Connection pool configuration for production
- Graceful shutdown on connection failures
- Buffer management and timeout configuration

#### **HTTP Status Codes**
- Standardized HTTP response codes for all endpoints
- 200: Service health check successful
- 201: URL created successfully  
- 301: Permanent redirect to original URL
- 400: Invalid URL format
- 404: Short URL not found
- 500: Internal server error

## 🚀 Deployment & Production

### Production Environment Setup

#### **Environment Variables**
- Production-specific port and database configurations
- Secure base URL with HTTPS domain
- Database authentication credentials
- Environment-specific logging and monitoring settings

#### **Production Container Configuration**
- Multi-service production orchestration
- HTTPS SSL certificate configuration
- Database authentication with username/password
- Auto-restart policies for high availability
      - "80:80"                             # HTTP redirect
    environment:
      - SSL_CERT_PATH=/etc/ssl/certs/
    volumes:
      - ./ssl:/etc/ssl/certs                # SSL certificates
      - nginx_logs:/var/log/nginx
    
  app:
    build: .
    environment:
      - NODE_ENV=production
      - PORT=3000
      - MONGO_URI=mongodb://mongodb:27017/url_shortener
    restart: unless-stopped                  # Auto-restart policy
    
  mongodb:
    image: mongo:latest
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGODB_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGODB_PASSWORD}
    volumes:
      - mongodb_data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d
    restart: unless-stopped
```

### Scaling & Load Balancing

#### **Multiple Application Instances**
- Horizontal scaling with multiple API server containers
- Load distribution across multiple service instances
- Container orchestration for service coordination
- Environment configuration for different ports

#### **Reverse Proxy Load Balancer Configuration**
- Multiple upstream servers with weighted load distribution
- Various load balancing algorithms (least connections, IP hash)
- SSL/HTTPS configuration with HTTP/2 support
- Security headers and compression settings
- Connection timeout configurations
- Health check endpoint optimization

### Database Optimization

#### **Production Database Configuration**
- Production-optimized connection pooling
- Connection lifecycle management with idle timeouts
- Server selection and socket timeout configuration
- Buffer management and command optimization

#### **Database Indexing Strategy**
- Compound indexes for complex query performance
- Time-based indexing for analytics queries
- TTL indexes for automatic data cleanup
- Query performance optimization strategies

### Monitoring & Alerting

#### **Application Metrics**
- Custom metrics collection and tracking
- Request counting and response code monitoring
- Error rate tracking and alerting
- Performance metrics endpoint
- Memory and uptime monitoring

#### **Container Health Monitoring**
- Health monitoring scripts for automation
- Health check status validation
- Automated restart on health failures
- Health status logging and alerting

### Backup & Disaster Recovery

#### **Database Backup Strategy**
- Automated database backup creation
- Backup compression and storage management
- Backup rotation and retention policies
- Disaster recovery procedures

#### **Automated Backup Scheduling**
- Cron-based backup automation
- Backup logging and monitoring
- Backup verification and integrity checks

## 📊 Testing & Quality Assurance

### API Testing Examples

#### **URL Creation Tests**
- Successful URL creation validation
- Invalid URL format testing
- Missing URL parameter handling
- Error response validation

#### **URL Redirection Tests**
- Successful redirection testing
- Non-existent URL handling
- Malformed short URL validation
- HTTP status code verification

#### **Health Check Tests**
- Health endpoint response validation
- Status and uptime verification
- Timestamp accuracy testing

### Load Testing

#### **Performance Testing**
- URL creation endpoint load testing
- Redirection endpoint performance testing
- Health endpoint stress testing
- Concurrent user simulation

#### **Stress Testing**
- Multi-phase load testing configuration
- Arrival rate ramping and stress phases
- Performance baseline establishment
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      ### Container Testing

#### **Container Health Verification**
- Container health status verification
- Internal container connectivity testing
- Resource usage monitoring
- Volume integrity verification

## 🔍 Troubleshooting Guide

### Common Issues & Solutions

#### **Container Startup Issues**
- Application container startup failures
- Database connection error resolution
- Port conflict detection and resolution
- Missing environment variable identification

#### **Database Connection Issues**
- Network connectivity verification
- Database availability testing
- Connection string validation

#### **Reverse Proxy Issues**
- Gateway error troubleshooting
- Container communication verification
- Proxy configuration validation
- Upstream connectivity testing

#### **Performance Issues**
- Resource usage monitoring
- Database performance analysis
- Connection pool optimization

### Debug Operations

#### **Container Debugging**
- Container shell access for troubleshooting
- Process monitoring within containers
- Network configuration inspection

#### **Database Debugging**
- Database connection and query testing
- Collection statistics and indexing analysis
- Operation monitoring and query profiling

## 🎯 Best Practices & Recommendations

### Development Best Practices

1. **Environment Separation**: Use different container configurations for development and production
2. **Security**: Never commit sensitive data like passwords or API keys
3. **Logging**: Implement structured logging with appropriate log levels
4. **Error Handling**: Always handle errors gracefully with proper HTTP status codes
5. **Testing**: Implement comprehensive testing including unit, integration, and load tests
6. **Monitoring**: Set up health checks and monitoring from day one
7. **Documentation**: Keep documentation up-to-date with code changes

### Production Deployment Checklist

- [ ] SSL/HTTPS configuration
- [ ] Environment variables properly configured
- [ ] Database authentication enabled
- [ ] Rate limiting implemented
- [ ] Monitoring and alerting set up
- [ ] Backup strategy implemented
- [ ] Load testing completed
- [ ] Security headers configured
- [ ] Log aggregation configured
- [ ] Container resource limits set

### Performance Optimization Tips

1. **Database Indexing**: Ensure proper indexes on frequently queried fields
2. **Connection Pooling**: Configure appropriate connection pool sizes
3. **Caching**: Implement Redis caching for frequently accessed URLs
4. **CDN**: Use CDN for static assets and global distribution
5. **Compression**: Enable compression in reverse proxy
6. **Resource Limits**: Set appropriate CPU and memory limits for containers
7. **Monitoring**: Continuously monitor performance metrics and optimize bottlenecks


