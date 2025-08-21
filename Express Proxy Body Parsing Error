# Express Proxy Body Parsing Error

## Problem Summary

When using `http-proxy-middleware` in an Express.js API gateway, POST/PUT/PATCH requests with JSON bodies fail with "request aborted" errors, while GET requests work perfectly. The requests reach the target service but fail during body parsing.

## Error Symptoms

### Gateway Logs
```
Proxying POST /auth/admin/register to http://target-service:8001
```

### Target Service Logs
```
POST /auth/admin/register - - ms - -
BadRequestError: request aborted
    at IncomingMessage.onAborted (/app/node_modules/raw-body/index.js:245:10)
    at IncomingMessage.emit (node:events:507:28)
    ...
```

### Client Behavior
- GET requests work normally
- POST/PUT/PATCH requests hang and eventually timeout
- Request body never reaches the target service controller

## Root Cause

The issue occurs due to **middleware order conflict** between `express.json()` body parsing middleware and `http-proxy-middleware`. Here's what happens:

1. **Gateway** receives POST request with JSON body
2. **`express.json()` middleware** consumes the request stream to parse the body
3. **Proxy middleware** attempts to forward the request, but the body stream is already consumed/corrupted
4. **Target service** receives request with empty/broken body stream
5. **Target service's `express.json()`** fails to parse the corrupted stream
6. **Request gets aborted** during body parsing

## How to Recreate This Error

### 1. Create a Gateway with Incorrect Configuration

```typescript
// ❌ INCORRECT - This will cause the error
const app = express()

app.use(helmet())
app.use(cors())
app.use(express.json()) // ⚠️ Body parsing BEFORE proxy = ERROR

// Proxy setup
app.use("/api/v1/auth", createProxyMiddleware({
    target: 'http://auth-service:3001',
    changeOrigin: true,
    pathRewrite: { "^/api/v1/auth": "/auth" }
}))
```

### 2. Create a Target Service

```typescript
const app = express()

app.use(express.json()) // This will fail to parse corrupted stream
app.use("/auth", authRoutes)

// Route that expects JSON body
app.post("/auth/register", (req, res) => {
    console.log("Body:", req.body) // Will be undefined/empty
    res.json({ message: "Registration successful" })
})
```

### 3. Send POST Request

```bash
curl -X POST http://gateway:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "password123"}'
```

**Result**: Request hangs, then fails with "request aborted" error.

## Solutions

### Solution 1: Move Body Parsing After Proxy Routes (Recommended)

```typescript
// ✅ CORRECT - Body parsing after proxy setup
const app = express()

app.use(helmet())
app.use(cors())
// DON'T add express.json() here for proxied routes

// Set up proxy routes FIRST
app.use("/api/v1/auth", createProxyMiddleware({
    target: 'http://auth-service:3001',
    changeOrigin: true,
    pathRewrite: { "^/api/v1/auth": "/auth" }
}))

// Add body parsing ONLY for non-proxied routes
app.use(express.json())

// Direct gateway routes (health checks, etc.)
app.get("/health", (req, res) => {
    res.json({ status: "ok" })
})
```

### Solution 2: Configure Proxy with Body Handling Options

```typescript
const createProxyWithErrorHandling = (target: string, pathRewrite: Record<string, string>) => {
    return createProxyMiddleware({
        target,
        changeOrigin: true,
        pathRewrite,
        timeout: 30000,
        proxyTimeout: 30000,
        
        // Critical: Handle body parsing correctly
        parseReqBody: false,  // Let proxy handle raw body
        
        on: {
            proxyReq: (proxyReq, req, res) => {
                // Ensure proper headers for POST requests
                if (req.method === 'POST' || req.method === 'PUT' || req.method === 'PATCH') {
                    if (req.headers['content-type']) {
                        proxyReq.setHeader('Content-Type', req.headers['content-type'])
                    }
                    if (req.headers['content-length']) {
                        proxyReq.setHeader('Content-Length', req.headers['content-length'])
                    }
                }
            }
        }
    })
}
```

### Solution 3: Selective Body Parsing (Advanced)

```typescript
// Parse body only for specific routes
app.use("/api/direct", express.json()) // Only for direct routes
app.use("/api/v1/auth", proxyMiddleware)  // No body parsing for proxied routes
```

## Prevention Guidelines

### ✅ Best Practices

1. **Always set up proxy routes BEFORE body parsing middleware**
2. **Use `parseReqBody: false` in proxy configuration**
3. **Add proper timeouts to prevent hanging requests**
4. **Test both GET and POST requests when setting up proxies**
5. **Use middleware logging to debug request flow**

### ❌ Common Mistakes

1. Adding `express.json()` before proxy setup
2. Not configuring proxy timeouts
3. Missing Content-Type/Content-Length headers in proxy config
4. Not testing POST requests during development

## Debugging Checklist

When facing similar issues:

- [ ] Check middleware order (body parsing vs proxy)
- [ ] Verify GET requests work but POST requests fail
- [ ] Look for "request aborted" errors in target service
- [ ] Check if request body is empty/undefined in target service
- [ ] Add logging to track request flow through middleware chain
- [ ] Test with simple curl command to isolate the issue

## Additional Notes

- This error is specific to HTTP proxies with JSON body parsing
- File uploads and form data can have similar issues
- The error only affects requests with bodies (POST, PUT, PATCH)
- Docker networking doesn't cause this specific error (but can compound it)

## Related Issues

- Express body parser consuming request streams
- HTTP proxy middleware compatibility
- Microservices communication patterns
- API gateway body handling
