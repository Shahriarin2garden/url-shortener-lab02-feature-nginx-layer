# URL Shortener with Nginx Reverse Proxy

A containerized URL shortening service built with Node.js, Express, MongoDB, and Nginx as a reverse proxy. The application provides URL shortening and redirection functionality with visit tracking and health monitoring.

## Architecture

### System Overview
```
┌─────────────────┐    ┌──────────────┐    ┌─────────────────┐
│   Client        │───▶│   Nginx      │───▶│   Node.js App   │
│   (Browser)     │    │   :80        │    │   :3000         │
└─────────────────┘    └──────────────┘    └─────────────────┘
                                                     │
                                                     ▼
                                           ┌─────────────────┐
                                           │   MongoDB       │
                                           │   :27017        │
                                           └─────────────────┘
```

### Container Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Compose Environment                  │
│                   url-shortener-network (bridge)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  nginx:alpine   │  │ node:18-alpine  │  │  mongo:latest   │ │
│  │                 │  │                 │  │                 │ │
│  │ ┌─────────────┐ │  │ ┌─────────────┐ │  │ ┌─────────────┐ │ │
│  │ │   Reverse   │ │  │ │  Express.js │ │  │ │  MongoDB    │ │ │
│  │ │   Proxy     │◄┼──┼►│   Server    │◄┼──┼►│  Database   │ │ │
│  │ │   Port 80   │ │  │ │  Port 3000  │ │  │ │ Port 27017  │ │ │
│  │ └─────────────┘ │  │ └─────────────┘ │  │ └─────────────┘ │ │
│  │                 │  │                 │  │                 │ │
│  │ upstream:       │  │ Health Check:   │  │ Database:       │ │
│  │ server app:3000 │  │ /health         │  │ url_shortener   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│         │                      │                      │        │
│         ▼                      ▼                      ▼        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  nginx_logs     │  │   Live Reload   │  │  mongodb_data   │ │
│  │   Volume        │  │    Volume       │  │    Volume       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘

External Access: localhost:80 ──► nginx ──► app:3000 ──► mongodb:27017
```

### Application Layer Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                      Node.js Application                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                Application Bootstrap                    │    │
│  │  ┌─────────────────────────────────────────────────┐   │    │
│  │  │ • Web server initialization                     │   │    │
│  │  │ • Structured logging setup (JSON format)       │   │    │
│  │  │ • Database connection management                │   │    │
│  │  │ • Health monitoring endpoint                    │   │    │
│  │  │ • Global error handling                         │   │    │
│  │  └─────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Request Router                          │    │
│  │  ┌─────────────────────────────────────────────────┐   │    │
│  │  │ • POST /urls ──► URL Creation Handler           │   │    │
│  │  │ • GET /:shortUrlId ──► Redirection Handler      │   │    │
│  │  └─────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Request Handlers                           │    │
│  │  ┌─────────────────────────────────────────────────┐   │    │
│  │  │ • HTTP request/response processing              │   │    │
│  │  │ • Input validation (URL format checking)       │   │    │
│  │  │ • Error response formatting                     │   │    │
│  │  │ • Status code management (400, 404, 500)       │   │    │
│  │  └─────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │               Business Logic Layer                      │    │
│  │  ┌─────────────────────────────────────────────────┐   │    │
│  │  │ • Short URL ID generation (7 chars A-Z,a-z,0-9)│   │    │
│  │  │ • URL shortening operations                     │   │    │
│  │  │ • URL resolution & analytics tracking          │   │    │
│  │  │ • Visit counter management                      │   │    │
│  │  └─────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │               Data Access Layer                         │    │
│  │  ┌─────────────────────────────────────────────────┐   │    │
│  │  │ • Document schema definition                    │   │    │
│  │  │ • Data validation rules                         │   │    │
│  │  │ • Database query operations                     │   │    │
│  │  │ • Index management (unique shortUrlId)          │   │    │
│  │  └─────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### Request Flow Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                    1. URL Creation Flow                        │
└─────────────────────────────────────────────────────────────────┘

Client ──POST /urls──► Nginx ──proxy_pass──► Express.js App
  │                     │                         │
  │ {                   │ upstream backend {      │ ┌─────────────┐
  │   "longUrl":        │   server app:3000;     │ │ Router      │
  │   "https://..."     │ }                       ▼ │ POST /urls  │
  │ }                   │                       ┌───┴─────────────┤
  │                     │                       │ Request Handler │
  │                     │                       │ • Input Validation│
  │                     │                       └───┬─────────────┤
  │                     │                           │ • URL Format│
  │                     │                           │   Check     │
  │                     │                           │ • Delegate  │
  │                     │                           │   to Logic  │
  │                     │                           └─────────────┘
  │                     │                                 │
  │                     │                                 ▼
  │                     │                         ┌─────────────────┐
  │                     │                         │ Business Logic  │
  │                     │                         │ • ID Generation │
  │                     │                         ├─────────────────┤
  │                     │                         │ • Generate 7    │
  │                     │                         │   character ID  │
  │                     │                         │ • Create URL    │
  │                     │                         │   mapping       │
  │                     │                         │ • Persist data  │
  │                     │                         └─────────────────┘
  │                     │                                 │
  │                     │                                 ▼
  │                     │                         ┌─────────────────┐
  │                     │                         │ Data Layer      │
  │                     │                         │ • Document      │
  │                     │                         │   Creation      │
  │                     │                         │ • Schema        │
  │                     │                         │   Validation    │
  │                     │                         │ • Database      │
  │                     │                         │   Persistence   │
  │                     │                         └─────────────────┘
  │                     │                                 │
  ◄─────────────────────◄─────────────────────────────────┘
  {"shortUrl": "http://localhost/aBc123D"}

┌─────────────────────────────────────────────────────────────────┐
│                   2. URL Redirection Flow                      │
└─────────────────────────────────────────────────────────────────┘

Client ──GET /aBc123D──► Nginx ──proxy_pass──► Express.js App
  │                        │                         │
  │                        │ upstream backend {      │ ┌─────────────┐
  │                        │   server app:3000;     │ │ Router      │
  │                        │ }                       ▼ │ GET /:id    │
  │                        │                       ┌───┴─────────────┤
  │                        │                       │ Request Handler │
  │                        │                       │ • Extract ID    │
  │                        │                       └───┬─────────────┤
  │                        │                           │ • Parameter │
  │                        │                           │   Parsing   │
  │                        │                           │ • Delegate  │
  │                        │                           │   to Logic  │
  │                        │                           └─────────────┘
  │                        │                                 │
  │                        │                                 ▼
  │                        │                         ┌─────────────────┐
  │                        │                         │ Business Logic  │
  │                        │                         │ • URL Resolution│
  │                        │                         ├─────────────────┤
  │                        │                         │ • Find URL by   │
  │                        │                         │   identifier    │
  │                        │                         │ • Update visit  │
  │                        │                         │   analytics     │
  │                        │                         │ • Return target │
  │                        │                         └─────────────────┘
  │                        │                                 │
  │                        │                                 ▼
  │                        │                         ┌─────────────────┐
  │                        │                         │ Data Layer      │
  │                        │                         │ • Query by ID   │
  │                        │                         │ • Increment     │
  │                        │                         │   counter       │
  │                        │                         │ • Update record │
  │                        │                         └─────────────────┘
  │                        │                                 │
  ◄────────────────────────◄─────────────────────────────────┘
  301 Redirect
  Location: https://original-url.com
```

### Component Specifications
```
┌─────────────┬────────────────┬──────────┬────────────────────┐
│ Component   │ Technology     │ Port     │ Purpose            │
├─────────────┼────────────────┼──────────┼────────────────────┤
│ Nginx       │ nginx:alpine   │ 80       │ Reverse Proxy      │
│ API Server  │ node:18-alpine │ 3000     │ REST API           │
│ Database    │ mongo:latest   │ 27017    │ Data Persistence   │
├─────────────┼────────────────┼──────────┼────────────────────┤
│ Router      │ Express Router │ -        │ Request Routing    │
│ Handler     │ Request Handler│ -        │ HTTP Processing    │
│ Logic       │ Business Logic │ -        │ Core Operations    │
│ Data Layer  │ Data Access    │ -        │ Database Interface │
└─────────────┴────────────────┴──────────┴────────────────────┘
```

## Technology Stack

- **Frontend Layer**: Nginx (reverse proxy)
- **Backend**: Node.js with Express.js
- **Database**: MongoDB
- **Containerization**: Docker & Docker Compose
- **Logging**: Winston (structured JSON logs)


## Features

- ✅ **URL Shortening**: Generate 7-character unique IDs (62^7 combinations)
- ✅ **URL Redirection**: 301 redirects with visit tracking analytics
- ✅ **Nginx Reverse Proxy**: Load balancing and request forwarding
- ✅ **Container Orchestration**: Multi-container Docker setup
- ✅ **Health Monitoring**: Container health checks and `/health` endpoint
- ✅ **Structured Logging**: JSON formatted logs with Winston
- ✅ **Data Persistence**: MongoDB with persistent volumes
- ✅ **Input Validation**: URL format validation and error handling

## Quick Start

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
- **Response**: 301 redirect to original URL + visit counter increment

## Database Schema

```javascript
{
  shortUrlId: String,    // 7-character unique identifier (indexed)
  longUrl: String,       // Original URL (required, validated)
  createdAt: Date,       // Auto-generated timestamp
  visits: Number         // Visit counter (incremented on each redirect)
}
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Application port | `3000` |
| `MONGO_URI` | MongoDB connection string | `mongodb://mongodb:27017/url_shortener` |
| `BASE_URL` | Base URL for short links | `http://localhost` |

## Testing

This section provides comprehensive testing procedures and examples for the URL shortener application.

### Prerequisites for Testing

Before running tests, ensure the application is running:

```bash
# Start the application
docker-compose up --build -d

# Verify all containers are healthy
docker-compose ps
```

### 1. Health Check Testing

#### Test the Health Endpoint
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

**Test Screenshot Placeholder:**
```
[ Health Check Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ curl -X GET http://localhost/health                       │
│ {                                                           │
│   "status": "healthy",                                      │
│   "timestamp": "2025-08-28T10:30:00.123Z",                 │
│   "uptime": 45.67                                           │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
```

### 2. URL Shortening Testing

#### Create a Short URL
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

**Test Screenshot Placeholder:**
```
[ URL Creation Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ curl -X POST http://localhost/urls \                     │
│   -H "Content-Type: application/json" \                    │
│   -d '{"longUrl": "https://www.google.com"}'               │
│ {                                                           │
│   "shortUrl": "http://localhost/aBc123D"                   │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
```

#### Test Multiple URL Creation
```bash
# Create multiple short URLs
curl -X POST http://localhost/urls -H "Content-Type: application/json" -d '{"longUrl": "https://github.com"}'
curl -X POST http://localhost/urls -H "Content-Type: application/json" -d '{"longUrl": "https://stackoverflow.com"}'
curl -X POST http://localhost/urls -H "Content-Type: application/json" -d '{"longUrl": "https://youtube.com"}'
```

### 3. URL Redirection Testing

#### Test Redirect Functionality
```bash
# Test redirect (will follow the redirect)
curl -L http://localhost/aBc123D

# Test redirect headers only (see the redirect response)
curl -I http://localhost/aBc123D
```

**Expected Redirect Response:**
```
HTTP/1.1 301 Moved Permanently
Location: https://www.google.com
```

**Test Screenshot Placeholder:**
```
[ URL Redirect Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ curl -I http://localhost/aBc123D                          │
│ HTTP/1.1 301 Moved Permanently                             │
│ Server: nginx/1.25.3                                       │
│ Date: Wed, 28 Aug 2025 10:30:15 GMT                        │
│ Content-Type: text/html; charset=utf-8                     │
│ Content-Length: 58                                          │
│ Connection: keep-alive                                      │
│ Location: https://www.google.com                           │
└─────────────────────────────────────────────────────────────┘
```

#### Test Visit Counter
```bash
# Make multiple requests to increment visit counter
curl -I http://localhost/aBc123D
curl -I http://localhost/aBc123D
curl -I http://localhost/aBc123D
```

### 4. Error Handling Testing

#### Test Invalid URL Format
```bash
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

#### Test Missing URL Parameter
```bash
curl -X POST http://localhost/urls \
  -H "Content-Type: application/json" \
  -d '{}'
```

#### Test Non-existent Short URL
```bash
curl -I http://localhost/nonexistent
```

**Expected Response:**
```json
{
  "error": "Not Found"
}
```

**Error Handling Test Screenshot Placeholder:**
```
[ Error Handling Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ curl -X POST http://localhost/urls \                     │
│   -H "Content-Type: application/json" \                    │
│   -d '{"longUrl": "invalid-url"}'                          │
│ {                                                           │
│   "error": "Invalid URL"                                   │
│ }                                                           │
│                                                             │
│ $ curl -I http://localhost/nonexistent                     │
│ HTTP/1.1 404 Not Found                                     │
│ {                                                           │
│   "error": "Not Found"                                     │
│ }                                                           │
└─────────────────────────────────────────────────────────────┘
```

### 5. Container and Service Testing

#### Test Container Status
```bash
# Check all container status
docker-compose ps

# Check container logs
docker-compose logs nginx
docker-compose logs app
docker-compose logs mongodb
```

**Container Status Screenshot Placeholder:**
```
[ Container Status Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ docker-compose ps                                         │
│ NAME               IMAGE                    STATUS           │
│ url-shortener-nginx-1   url-shortener-nginx   Up 2 minutes │
│ url-shortener-app-1     url-shortener-app     Up 2 minutes │
│ url-shortener-mongodb-1 mongo:latest          Up 2 minutes │
└─────────────────────────────────────────────────────────────┘
```

#### Test Network Connectivity
```bash
# Test internal container communication
docker exec url-shortener-app-1 wget -O- http://mongodb:27017
docker exec url-shortener-nginx-1 wget -O- http://app:3000/health
```

### 6. Database Testing

#### Connect to MongoDB and Verify Data
```bash
# Connect to MongoDB container
docker exec -it url-shortener-mongodb-1 mongosh url_shortener

# MongoDB queries to test
db.urls.find()
db.urls.countDocuments()
db.urls.find({}, {shortUrlId: 1, longUrl: 1, visits: 1, _id: 0})
```

**Database Test Screenshot Placeholder:**
```
[ Database Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ docker exec -it url-shortener-mongodb-1 mongosh url_shortener │
│ url_shortener> db.urls.find()                              │
│ [                                                           │
│   {                                                         │
│     _id: ObjectId('...'),                                  │
│     shortUrlId: 'aBc123D',                                 │
│     longUrl: 'https://www.google.com',                     │
│     createdAt: ISODate('2025-08-28T10:30:00.000Z'),       │
│     visits: 3,                                             │
│     __v: 0                                                 │
│   }                                                         │
│ ]                                                           │
└─────────────────────────────────────────────────────────────┘
```

### 7. Performance Testing

#### Basic Load Testing with curl
```bash
# Test concurrent requests
for i in {1..10}; do
  curl -X POST http://localhost/urls \
    -H "Content-Type: application/json" \
    -d "{\"longUrl\": \"https://example$i.com\"}" &
done
wait
```

#### Stress Test Redirects
```bash
# Stress test redirections (replace aBc123D with actual short URL)
for i in {1..50}; do
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost/aBc123D &
done
wait
```

**Performance Test Screenshot Placeholder:**
```
[ Performance Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ $ for i in {1..10}; do                                      │
│   curl -X POST http://localhost/urls \                     │
│     -H "Content-Type: application/json" \                  │
│     -d "{\"longUrl\": \"https://example$i.com\"}" &        │
│ done                                                        │
│ wait                                                        │
│                                                             │
│ {"shortUrl": "http://localhost/xYz789A"}                   │
│ {"shortUrl": "http://localhost/pQr456B"}                   │
│ {"shortUrl": "http://localhost/mNo123C"}                   │
│ ... (10 URLs created successfully)                         │
└─────────────────────────────────────────────────────────────┘
```

### 8. End-to-End Testing Script

Create a comprehensive test script:

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

echo -e "\n4. Testing Multiple Visits..."
for i in {1..3}; do
  echo "Visit $i:"
  curl -s -I http://localhost/$SHORT_ID | grep "HTTP\|Location"
done

echo -e "\n5. Testing Error Cases..."
curl -s -X POST http://localhost/urls \
  -H "Content-Type: application/json" \
  -d '{"longUrl": "invalid"}' | jq .

echo -e "\nTest Complete!"
```

**End-to-End Test Screenshot Placeholder:**
```
[ End-to-End Test Output Screenshot ]
┌─────────────────────────────────────────────────────────────┐
│ === URL Shortener End-to-End Test ===                      │
│                                                             │
│ 1. Testing Health Endpoint...                              │
│ {                                                           │
│   "status": "healthy",                                      │
│   "timestamp": "2025-08-28T10:30:00.123Z",                 │
│   "uptime": 67.89                                           │
│ }                                                           │
│                                                             │
│ 2. Creating Short URL...                                   │
│ {"shortUrl": "http://localhost/gHi789J"}                   │
│                                                             │
│ 3. Testing Redirect...                                     │
│ HTTP/1.1 301 Moved Permanently                             │
│ Location: https://www.github.com                           │
│                                                             │
│ Test Complete!                                              │
└─────────────────────────────────────────────────────────────┘
```

### 9. Browser Testing

#### Manual Browser Tests
1. **Health Check**: Visit `http://localhost/health` in browser
2. **API Testing**: Use browser developer tools or Postman
3. **Redirect Testing**: Create a short URL and test redirect in browser

#### Browser Test Screenshot Placeholders:
```
[ Browser Health Check Screenshot ]
[ Browser API Test Screenshot ]
[ Browser Redirect Test Screenshot ]
```

### 10. Testing Checklist

Use this checklist to verify all functionality:

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

### Test Results Summary

After completing all tests, document your results:

| Test Category | Status | Notes |
|---------------|--------|-------|
| Health Check | ✅ Pass | Response time: ~50ms |
| URL Creation | ✅ Pass | All formats accepted |
| URL Redirection | ✅ Pass | 301 redirects working |
| Visit Tracking | ✅ Pass | Counters incrementing |
| Error Handling | ✅ Pass | Proper error messages |
| Container Health | ✅ Pass | All services running |
| Database | ✅ Pass | Data persisted correctly |
| Performance | ✅ Pass | Handles concurrent requests |

## Troubleshooting

### Common Issues

1. **Port already in use**
   ```bash
   # Check what's using port 80
   lsof -i :80
   # Stop conflicting services or change ports
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
├── package.json               # Node.js dependencies
├── nginx/
│   ├── Dockerfile            # Nginx container build
│   └── nginx.conf            # Nginx configuration
└── src/
    ├── app.js                # Application entry point
    ├── config/index.js       # Environment configuration
    ├── controllers/urlController.js  # HTTP request handlers
    ├── models/urlModel.js    # Database schema
    ├── routes/urlRoutes.js   # API route definitions
    └── services/urlService.js # Business logic
```

## License

ISC License


