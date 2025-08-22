# URL Shortener Lab02 - Feature Nginx Layer

A containerized URL shortening service built with Node.js, Express, MongoDB, and Nginx as a reverse proxy layer.

## 📋 Presentation Summary

### Project Overview
- **Name**: URL Shortener with Nginx Layer
- **Type**: Containerized Web Service
- **Architecture**: 3-Tier (Nginx + Node.js + MongoDB)
- **Deployment**: Docker Compose

### Key Technologies
- **Frontend**: REST API endpoints
- **Backend**: Node.js + Express.js
- **Database**: MongoDB with Mongoose ODM
- **Proxy**: Nginx reverse proxy
- **Containerization**: Docker & Docker Compose
- **Logging**: Winston structured logging

### Core Features
✅ **URL Shortening**: Convert long URLs to 7-character short codes  
✅ **URL Redirection**: 301 redirects with visit tracking  
✅ **Health Monitoring**: Built-in health checks and uptime tracking  
✅ **Error Handling**: Comprehensive validation and error responses  
✅ **Containerized**: Full Docker environment with networking  
✅ **Scalable**: Nginx reverse proxy for load distribution  

### Technical Specifications
| Component | Technology | Port | Purpose |
|-----------|------------|------|---------|
| Nginx | Nginx Alpine | 80 | Reverse Proxy & Load Balancer |
| API Server | Node.js 18 | 3000 (internal) | REST API & Business Logic |
| Database | MongoDB Latest | 27017 | Data Persistence |

### API Endpoints
| Method | Endpoint | Purpose | Response |
|--------|----------|---------|----------|
| `POST` | `/urls` | Create short URL | `{"shortUrl": "http://localhost/abc123"}` |
| `GET` | `/:id` | Redirect to original | `301 Redirect` |
| `GET` | `/health` | Service status | `{"status": "healthy", "uptime": 120}` |

### Database Schema
```javascript
{
  shortUrlId: String,    // 7-char unique ID (e.g., "aBc123D")
  longUrl: String,       // Original URL (validated)
  createdAt: Date,       // Auto timestamp
  visits: Number         // Visit counter (auto-increment)
}
```

### Deployment Architecture
```
Internet (Port 80) 
    ↓
Nginx Container (Reverse Proxy)
    ↓
Node.js Container (Port 3000)
    ↓
MongoDB Container (Port 27017)
```

### Quick Demo Commands
```bash
# 1. Start the application
docker-compose up --build -d

# 2. Create a short URL
curl -X POST http://localhost/urls 
  -H "Content-Type: application/json" 
  -d '{"longUrl": "https://www.google.com"}'

# 3. Test redirect
curl -L http://localhost/aBc123D

# 4. Check health
curl http://localhost/health
```

### Performance & Monitoring
- **Health Checks**: 30-second intervals with 3 retries
- **Logging**: JSON structured logs with timestamps
- **Error Tracking**: 400/404/500 HTTP status codes
- **Visit Analytics**: Automatic visit counter per URL
- **Uptime Monitoring**: Built-in uptime tracking

### Development Workflow
1. **Local Development**: `npm run dev` with nodemon
2. **Containerized Testing**: `docker-compose up --build`
3. **Monitoring**: `docker-compose logs -f`
4. **Health Checks**: Automated container health monitoring

### Project Benefits
🚀 **Scalability**: Nginx proxy enables horizontal scaling  
🔒 **Reliability**: Health checks and error handling  
📦 **Portability**: Complete Docker environment  
🔧 **Maintainability**: Modular MVC architecture  
📊 **Analytics**: Built-in visit tracking  
⚡ **Performance**: Lightweight Alpine containers  

### Use Cases
- **Link Shortening Services**: Social media, marketing campaigns
- **Analytics Tracking**: Monitor link engagement
- **Branded URLs**: Custom domain short links
- **QR Code Generation**: Short URLs for QR codes
- **Email Marketing**: Track campaign performance

## Architecture

### High-Level Architecture
```
┌─────────────┐    ┌──────────────────┐    ┌─────────────────────┐    ┌─────────────────┐
│   Client    │───▶│  Nginx Container │───▶│  Node.js Container  │───▶│ MongoDB Container│
│ (Browser/   │    │   (Port 80)      │    │    (Port 3000)      │    │  (Port 27017)   │
│  cURL/API)  │    │  Reverse Proxy   │    │   Express Server    │    │   Database      │
└─────────────┘    └──────────────────┘    └─────────────────────┘    └─────────────────┘
```

### Detailed Component Architecture
```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           Docker Compose Environment                                │
│  ┌────────────────┐  ┌─────────────────────────────────────┐  ┌─────────────────┐  │
│  │ nginx:80       │  │           app:3000                  │  │   mongodb:27017 │  │
│  │ ┌────────────┐ │  │  ┌─────────────────────────────────┐│  │ ┌─────────────┐ │  │
│  │ │nginx.conf  │ │  │  │            app.js               ││  │ │  Database   │ │  │
│  │ │upstream    │ │  │  │  ┌─────────────────────────────┐││  │ │             │ │  │
│  │ │backend     │ │  │  │  │        Express App          │││  │ │ ┌─────────┐ │ │  │
│  │ │server app: │ │  │  │  │                             │││  │ │ │   Url   │ │ │  │
│  │ │3000        │ │  │  │  │ ┌─────────────────────────┐ │││  │ │ │Collection│ │ │  │
│  │ └────────────┘ │  │  │  │ │      urlRoutes.js       │ │││  │ │ └─────────┘ │ │  │
│  │                │  │  │  │ │  POST /urls             │ │││  │ └─────────────┘ │  │
│  │  proxy_pass    │  │  │  │ │  GET /:shortUrlId       │ │││  └─────────────────┘  │
│  │  http://backend│◄─┼──┼──┼─┤  GET /health            │ │││                       │
│  └────────────────┘  │  │  │ └─────────────────────────┘ │││                       │
│                      │  │  │                             │││                       │
│                      │  │  │ ┌─────────────────────────┐ │││                       │
│                      │  │  │ │    urlController.js     │ │││                       │
│                      │  │  │ │  createShortUrl()       │ │││                       │
│                      │  │  │ │  redirectToLongUrl()    │ │││                       │
│                      │  │  │ └─────────────────────────┘ │││                       │
│                      │  │  │           │                 │││                       │
│                      │  │  │           ▼                 │││                       │
│                      │  │  │ ┌─────────────────────────┐ │││                       │
│                      │  │  │ │     urlService.js       │ │││                       │
│                      │  │  │ │  shortenUrl()           │ │││                       │
│                      │  │  │ │  getLongUrl()           │ │││                       │
│                      │  │  │ │  generateShortUrlId()   │ │││                       │
│                      │  │  │ └─────────────────────────┘ │││                       │
│                      │  │  │           │                 │││                       │
│                      │  │  │           ▼                 │││                       │
│                      │  │  │ ┌─────────────────────────┐ │││                       │
│                      │  │  │ │     urlModel.js         │ │││───────────────────────┤
│                      │  │  │ │  Mongoose Schema        │ │││     mongodb://        │
│                      │  │  │ │  - shortUrlId: String   │ │││   mongodb:27017/      │
│                      │  │  │ │  - longUrl: String      │ │││   url_shortener       │
│                      │  │  │ │  - createdAt: Date      │ │││                       │
│                      │  │  │ │  - visits: Number       │ │││                       │
│                      │  │  │ └─────────────────────────┘ │││                       │
│                      │  │  │                             │││                       │
│                      │  │  │ ┌─────────────────────────┐ │││                       │
│                      │  │  │ │  Winston Logger         │ │││                       │
│                      │  │  │ │  - HTTP requests        │ │││                       │
│                      │  │  │ │  - Errors               │ │││                       │
│                      │  │  │ │  - JSON format          │ │││                       │
│                      │  │  │ └─────────────────────────┘ │││                       │
│                      │  │  └─────────────────────────────┘││                       │
│                      │  └─────────────────────────────────┘│                       │
│                      └─────────────────────────────────────┘                       │
│                                                                                     │
│  Networks: url-shortener-network (bridge)                                          │
│  Volumes: mongodb_data, nginx_logs                                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### Request Flow

#### 1. URL Shortening Flow
```
Client ──POST /urls──▶ Nginx ──proxy_pass──▶ Express App
                                                │
                                                ▼
                                            urlRoutes.js
                                                │
                                                ▼
                                          urlController.js
                                           createShortUrl()
                                                │
                                                ▼
                                           urlService.js
                                           shortenUrl()
                                                │
                                                ▼
                                         generateShortUrlId()
                                                │
                                                ▼
                                           urlModel.js ──save()──▶ MongoDB
                                                │
                                                ▼
                                         Return shortUrl
                                                │
                               JSON Response ◀──┘
```

#### 2. URL Redirection Flow
```
Client ──GET /:id──▶ Nginx ──proxy_pass──▶ Express App
                                               │
                                               ▼
                                           urlRoutes.js
                                               │
                                               ▼
                                        urlController.js
                                       redirectToLongUrl()
                                               │
                                               ▼
                                          urlService.js
                                          getLongUrl()
                                               │
                                               ▼
                                      urlModel.js ──findOne()──▶ MongoDB
                                               │                      │
                                               ▼                      │
                                        Increment visits ────────────┘
                                               │
                                               ▼
                                       301 Redirect Response
```

### Container Communication

The application uses a three-tier containerized architecture:
- **Nginx Layer**: Reverse proxy and load balancer (Port 80)
- **Application Layer**: Node.js Express server (Internal Port 3000)
- **Data Layer**: MongoDB database (Internal Port 27017, External 27017)

All containers communicate through the `url-shortener-network` bridge network with internal DNS resolution.

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

### Docker Compose Services

- **nginx**: Reverse proxy (Port 80)
  - Built from `./nginx/Dockerfile`
  - Proxies requests to the Node.js app
  - Logs stored in named volume

- **app**: Node.js application (Port 3000, internal)
  - Built from `./Dockerfile`
  - Health checks every 30 seconds
  - Auto-restart on failure

- **mongodb**: MongoDB database (Port 27017)
  - Official MongoDB image
  - Data persisted in named volume
  - Connected via internal network

### Environment Variables

The application uses the following environment variables:

- `PORT`: Application port (default: 3000)
- `MONGO_URI`: MongoDB connection string (mongodb://mongodb:27017/url_shortener)
- `BASE_URL`: Base URL for generated short URLs (http://localhost)

### Nginx Configuration

Located in `nginx/nginx.conf`:
- Upstream backend pointing to `app:3000`
- Simple reverse proxy configuration
- 1024 worker connections

## Project Structure

```
url-shortener-lab02-feature-nginx-layer/
├── docker-compose.yml              # Multi-container orchestration
├── Dockerfile                      # Node.js app container
├── package.json                    # Node.js dependencies and scripts
├── package-lock.json              # Locked dependency versions
├── .gitignore                     # Git ignore rules
├── nginx/
│   ├── Dockerfile                 # Nginx container configuration
│   └── nginx.conf                 # Nginx reverse proxy config
└── src/
    ├── app.js                     # Express server and app initialization
    ├── config/
    │   └── index.js               # Environment configuration management
    ├── controllers/
    │   └── urlController.js       # HTTP request handlers
    ├── models/
    │   └── urlModel.js            # MongoDB schema definition
    ├── routes/
    │   └── urlRoutes.js           # API route definitions
    └── services/
        └── urlService.js          # Business logic and URL operations
```

## Database Schema

### URL Model (MongoDB)
```javascript
{
  shortUrlId: String,    // 7-character unique identifier (e.g., "a1B2c3D")
  longUrl: String,       // Original URL (must start with http:// or https://)
  createdAt: Date,       // Auto-generated creation timestamp
  visits: Number         // Visit counter (starts at 0, incremented on each access)
}
```

## Development

### Local Development (without Docker)
```bash
# Install dependencies
npm install

# Set environment variables
export MONGO_URI="mongodb://localhost:27017/url_shortener"
export BASE_URL="http://localhost:3000"

# Start development server with auto-reload
npm run dev

# Or start production server
npm start
```

### Docker Development
```bash
# Build and start all services
docker-compose up --build

# Start in background
docker-compose up -d

# View logs from all services
docker-compose logs -f

# View logs from specific service
docker-compose logs -f app

# Stop all services
docker-compose down

# Stop and remove volumes
docker-compose down -v
```

## Testing Examples

### Create and Use Short URL
```bash
# Create a short URL
curl -X POST http://localhost/urls 
  -H "Content-Type: application/json" 
  -d '{"longUrl": "https://www.google.com"}'

# Expected response:
# {"shortUrl":"http://localhost/aBc1234"}

# Test the redirect (follow redirects)
curl -L http://localhost/aBc1234

# Test redirect headers only
curl -I http://localhost/aBc1234
```

### Health and Monitoring
```bash
# Check service health
curl http://localhost/health

# Check container status
docker-compose ps

# View real-time logs
docker-compose logs -f app
```

## Error Handling

The application includes comprehensive error handling:

- **400 Bad Request**: Invalid URL format (must start with http:// or https://)
- **404 Not Found**: Short URL ID not found in database
- **500 Internal Server Error**: Database connection issues or server errors

## Dependencies

### Production Dependencies
- **express** (v5.1.0): Web application framework
- **mongoose** (v8.14.3): MongoDB object modeling
- **dotenv** (v16.5.0): Environment variable management
- **winston** (v3.17.0): Structured logging

### Development Dependencies
- **nodemon** (v3.1.10): Development server with auto-reload

### Container Dependencies
- **Node.js 18 Alpine**: Lightweight Node.js runtime
- **Nginx Alpine**: Lightweight reverse proxy
- **MongoDB Latest**: Document database

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


