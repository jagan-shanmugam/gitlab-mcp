# Multi-User Authentication with gitlab-mcp

This document explains how to configure and use the **Remote Authorization** feature in gitlab-mcp, which enables multi-user support with per-session authentication via HTTP headers.

## Overview

The gitlab-mcp server supports multi-user authentication when running in **Streamable HTTP transport mode** with **Remote Authorization** enabled. In this mode:

- Each client session can use a different GitLab Personal Access Token (PAT) or OAuth token
- Tokens are passed via HTTP headers on each request
- Session-based authentication with automatic token management
- Built-in rate limiting and capacity controls
- Automatic cleanup when sessions close or timeout

## Architecture

### Session-Based Authentication Flow

```
1. Client initiates connection with Authorization header
   └─> Server creates new session with unique session ID
   └─> Server stores token associated with session ID
   └─> Server sets inactivity timeout for auth token

2. Client makes subsequent requests with session ID header
   └─> Server retrieves token from session store
   └─> Server uses session-specific token for GitLab API calls
   └─> Server resets inactivity timeout

3. Session cleanup (automatic)
   └─> On timeout: Auth token removed, session remains active
   └─> On connection close: Both session and auth token removed
```

### Key Features

- **Session Isolation**: Each session's authentication is completely isolated from other sessions
- **Token Validation**: Basic format validation for GitLab PAT tokens (minimum 20 characters, alphanumeric)
- **Automatic Cleanup**: Auth tokens automatically expire after inactivity
- **Rate Limiting**: Per-session rate limiting to prevent abuse
- **Capacity Management**: Maximum concurrent session limits
- **Metrics & Monitoring**: Built-in endpoints for health checks and metrics

## Configuration

### Required Environment Variables

```bash
# Enable Streamable HTTP transport (REQUIRED for multi-user auth)
STREAMABLE_HTTP=true

# Enable remote authorization (REQUIRED for multi-user auth)
REMOTE_AUTHORIZATION=true

# GitLab API URL
GITLAB_API_URL=https://gitlab.com/api/v4

# OPTIONAL: GITLAB_PERSONAL_ACCESS_TOKEN is NOT required in this mode
# (tokens come from client headers instead)
```

### Optional Configuration

```bash
# Session auth timeout (default: 3600 seconds = 1 hour)
# After this period of inactivity, the auth token expires
SESSION_TIMEOUT_SECONDS=3600

# Maximum concurrent sessions (default: 1000)
MAX_SESSIONS=1000

# Rate limit per session (default: 60 requests/minute)
MAX_REQUESTS_PER_MINUTE=60

# Server port (default: 3002)
PORT=3002

# Server host (default: 0.0.0.0)
HOST=0.0.0.0
```

## Deployment Examples

### Option 1: Docker

```bash
# Start server with remote authorization
docker run -d \
  -e STREAMABLE_HTTP=true \
  -e REMOTE_AUTHORIZATION=true \
  -e GITLAB_API_URL="https://gitlab.com/api/v4" \
  -e SESSION_TIMEOUT_SECONDS=3600 \
  -e MAX_SESSIONS=1000 \
  -e MAX_REQUESTS_PER_MINUTE=60 \
  -p 3002:3002 \
  iwakitakuma/gitlab-mcp
```

### Option 2: npx (Local Development)

```bash
# Set environment variables
export STREAMABLE_HTTP=true
export REMOTE_AUTHORIZATION=true
export GITLAB_API_URL="https://gitlab.com/api/v4"

# Run server
npx -y @zereight/mcp-gitlab
```

### Option 3: Docker Compose

```yaml
version: '3.8'
services:
  gitlab-mcp:
    image: iwakitakuma/gitlab-mcp
    ports:
      - "3002:3002"
    environment:
      - STREAMABLE_HTTP=true
      - REMOTE_AUTHORIZATION=true
      - GITLAB_API_URL=https://gitlab.com/api/v4
      - SESSION_TIMEOUT_SECONDS=3600
      - MAX_SESSIONS=1000
      - MAX_REQUESTS_PER_MINUTE=60
      - GITLAB_READ_ONLY_MODE=false
      - USE_GITLAB_WIKI=true
      - USE_MILESTONE=true
      - USE_PIPELINE=true
    restart: unless-stopped
```

## Client Configuration

### Authentication Headers

Clients must include authentication in **one** of these two formats:

#### Option 1: Bearer Token (Recommended)

```http
Authorization: Bearer glpat-xxxxxxxxxxxxxxxxxxxx
```

#### Option 2: Private Token (Legacy GitLab)

```http
Private-Token: glpat-xxxxxxxxxxxxxxxxxxxx
```

### MCP Client Configuration Examples

#### Cursor IDE

```json
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",
      "url": "http://localhost:3002/mcp",
      "headers": {
        "Authorization": "Bearer glpat-your-personal-access-token"
      }
    }
  }
}
```

#### Claude Desktop

```json
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",
      "url": "http://localhost:3002/mcp",
      "headers": {
        "Authorization": "Bearer glpat-your-personal-access-token"
      }
    }
  }
}
```

#### Custom HTTP Client (Python Example)

```python
import requests

# Server endpoint
MCP_SERVER_URL = "http://localhost:3002/mcp"

# User-specific GitLab token
GITLAB_TOKEN = "glpat-xxxxxxxxxxxxxxxxxxxx"

# Make MCP request with authentication
response = requests.post(
    MCP_SERVER_URL,
    headers={
        "Authorization": f"Bearer {GITLAB_TOKEN}",
        "Content-Type": "application/json"
    },
    json={
        "jsonrpc": "2.0",
        "method": "tools/list",
        "id": 1
    }
)

print(response.json())
```

#### Custom HTTP Client (cURL Example)

```bash
# Initialize session and list tools
curl -X POST http://localhost:3002/mcp \
  -H "Authorization: Bearer glpat-xxxxxxxxxxxxxxxxxxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```

## Token Support

### Supported Token Types

1. **GitLab Personal Access Token (PAT)**
   - Format: `glpat-xxxxxxxxxxxxxxxxxxxx`
   - Minimum length: 20 characters
   - Recommended: Create with specific scopes (api, read_api, etc.)

2. **OAuth Access Token**
   - Format: Standard OAuth 2.0 bearer token
   - Works with both `Authorization: Bearer <token>` header
   - Automatically refreshed if using the built-in OAuth flow

### Token Validation

The server performs basic validation on incoming tokens:

- **Length check**: Minimum 20 characters
- **Format check**: Alphanumeric with allowed characters: `a-z, A-Z, 0-9, _, ., -`
- **Session association**: Tokens are bound to specific session IDs

Invalid tokens result in `HTTP 401 Unauthorized` responses.

## Session Management

### Session Lifecycle

1. **Session Creation**
   - Client sends initial request with authentication header
   - Server generates unique session ID via `mcp-session-id` header
   - Authentication token stored in session-specific storage
   - Inactivity timeout timer started

2. **Active Session**
   - Client includes `mcp-session-id` header in subsequent requests
   - Server retrieves auth token from session store
   - Each request resets the inactivity timeout
   - Token can be rotated by sending new auth header

3. **Session Termination**
   - **Automatic**: Inactivity timeout expires (default: 1 hour)
   - **Automatic**: Transport connection closes
   - **Manual**: Client sends DELETE request to `/mcp` endpoint
   - Auth token immediately removed from memory

### Session Timeout Behavior

When `SESSION_TIMEOUT_SECONDS` expires:

- ✅ **Auth token removed** - Frees memory and enhances security
- ✅ **Transport session remains active** - No connection disruption
- ⚠️ **Next request requires auth** - Client must send auth header again
- ✅ **New timeout set** - After successful re-authentication

This design balances security (token expiration) with usability (persistent connections).

### Manual Session Deletion

```bash
# Delete specific session
curl -X DELETE http://localhost:3002/mcp \
  -H "mcp-session-id: <session-id>"
```

## Security Considerations

### Token Storage

- **In-Memory Only**: Auth tokens stored in memory (not persisted to disk)
- **Session-Scoped**: Tokens isolated per session ID
- **Automatic Cleanup**: Removed on session close or timeout
- **No Logging**: Tokens never logged to console or files

### Rate Limiting

Each session is independently rate-limited:

```
Default: 60 requests per minute per session
Configurable via: MAX_REQUESTS_PER_MINUTE
HTTP Response: 429 Too Many Requests (when exceeded)
```

Rate limiting prevents:
- API abuse from individual sessions
- Accidental DOS from misbehaving clients
- Excessive load on GitLab API

### Capacity Management

Server-wide concurrent session limits:

```
Default: 1000 concurrent sessions
Configurable via: MAX_SESSIONS
HTTP Response: 503 Service Unavailable (when capacity reached)
```

Protects against:
- Memory exhaustion
- Connection pool saturation
- Server overload

### Best Practices

1. **Use HTTPS in Production**
   ```bash
   # Deploy behind reverse proxy with TLS
   # Example: nginx, Caddy, Traefik
   ```

2. **Restrict Network Access**
   ```bash
   # Firewall rules to limit access
   # Internal network only or VPN-protected
   ```

3. **Token Scope Limitation**
   ```bash
   # Create GitLab PATs with minimal required scopes
   # Example: read_api for read-only operations
   ```

4. **Monitor Metrics**
   ```bash
   # Regular monitoring of /metrics endpoint
   curl http://localhost:3002/metrics
   ```

5. **Set Appropriate Timeouts**
   ```bash
   # Balance security vs usability
   # Shorter timeout = more secure, less convenient
   # Recommended: 1-4 hours for interactive use
   SESSION_TIMEOUT_SECONDS=3600
   ```

## Monitoring & Observability

### Health Check Endpoint

```bash
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "activeSessions": 42,
  "maxSessions": 1000,
  "uptime": 86400.5
}
```

**Status Codes:**
- `200`: Server healthy (below capacity)
- `503`: Server degraded (at or near capacity)

### Metrics Endpoint

```bash
GET /metrics
```

**Response:**
```json
{
  "activeSessions": 42,
  "totalSessions": 156,
  "expiredSessions": 12,
  "authFailures": 3,
  "requestsProcessed": 8234,
  "rejectedByRateLimit": 45,
  "rejectedByCapacity": 2,
  "authenticatedSessions": 42,
  "uptime": 86400.5,
  "memoryUsage": {
    "rss": 123456789,
    "heapTotal": 87654321,
    "heapUsed": 65432109,
    "external": 1234567
  },
  "config": {
    "maxSessions": 1000,
    "maxRequestsPerMinute": 60,
    "sessionTimeoutSeconds": 3600,
    "remoteAuthEnabled": true
  }
}
```

**Key Metrics:**

- `activeSessions`: Current active sessions
- `totalSessions`: Cumulative session count since startup
- `expiredSessions`: Sessions expired due to inactivity
- `authFailures`: Failed authentication attempts
- `rejectedByRateLimit`: Requests blocked by rate limiting
- `rejectedByCapacity`: Requests rejected due to max sessions

### Monitoring with Prometheus

Example scrape configuration:

```yaml
scrape_configs:
  - job_name: 'gitlab-mcp'
    static_configs:
      - targets: ['localhost:3002']
    metrics_path: '/metrics'
    scrape_interval: 30s
```

## Troubleshooting

### Common Issues

#### 1. "Missing Authorization or Private-Token header"

**Error:**
```json
{
  "error": "Missing Authorization or Private-Token header",
  "message": "Remote authorization is enabled. Please provide Authorization or Private-Token header."
}
```

**Solution:**
- Ensure client sends either `Authorization: Bearer <token>` or `Private-Token: <token>` header
- Verify token is at least 20 characters
- Check token contains only valid characters

#### 2. "Rate limit exceeded"

**Error:**
```json
{
  "error": "Rate limit exceeded",
  "message": "Maximum 60 requests per minute allowed"
}
```

**Solution:**
- Reduce request frequency
- Increase `MAX_REQUESTS_PER_MINUTE` if legitimate traffic
- Check for client-side retry loops

#### 3. "Server capacity reached"

**Error:**
```json
{
  "error": "Server capacity reached",
  "message": "Maximum 1000 concurrent sessions allowed. Please try again later."
}
```

**Solution:**
- Increase `MAX_SESSIONS` if needed
- Check for session leaks (connections not properly closed)
- Scale horizontally with multiple server instances

#### 4. "REMOTE_AUTHORIZATION requires STREAMABLE_HTTP"

**Error:**
```
REMOTE_AUTHORIZATION=true requires STREAMABLE_HTTP=true
Set STREAMABLE_HTTP=true to enable remote authorization
```

**Solution:**
- Set `STREAMABLE_HTTP=true` environment variable
- Remote authorization only works with Streamable HTTP transport

#### 5. Token Expired During Session

**Behavior:**
- Session remains active but requests fail with auth errors

**Solution:**
- Client sends auth header again on next request
- Server automatically restores authentication
- Consider increasing `SESSION_TIMEOUT_SECONDS`

### Debugging Tips

1. **Enable Debug Logging**
   ```bash
   # Check server logs for detailed session information
   docker logs -f <container-id>
   ```

2. **Monitor Metrics**
   ```bash
   # Check metrics endpoint for authentication issues
   curl http://localhost:3002/metrics | jq
   ```

3. **Validate Token Manually**
   ```bash
   # Test token directly against GitLab API
   curl -H "Authorization: Bearer glpat-xxx" \
        https://gitlab.com/api/v4/user
   ```

4. **Check Session State**
   ```bash
   # Use metrics endpoint to see authenticated sessions
   curl http://localhost:3002/metrics | jq '.authenticatedSessions'
   ```

## Migration Guide

### From Single-User to Multi-User

**Before (Single-User):**
```bash
# Environment variables
GITLAB_PERSONAL_ACCESS_TOKEN=glpat-xxxxxxxxxxxxxxxxxxxx
GITLAB_API_URL=https://gitlab.com/api/v4

# Client config - no headers needed
{
  "mcpServers": {
    "gitlab": {
      "command": "npx",
      "args": ["-y", "@zereight/mcp-gitlab"]
    }
  }
}
```

**After (Multi-User):**
```bash
# Environment variables
STREAMABLE_HTTP=true                    # NEW
REMOTE_AUTHORIZATION=true               # NEW
GITLAB_API_URL=https://gitlab.com/api/v4
# GITLAB_PERSONAL_ACCESS_TOKEN - REMOVED (now from headers)

# Client config - add headers
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",         # NEW
      "url": "http://localhost:3002/mcp", # NEW
      "headers": {                        # NEW
        "Authorization": "Bearer glpat-xxxxxxxxxxxxxxxxxxxx"
      }
    }
  }
}
```

### Breaking Changes

⚠️ **Important**: Remote authorization mode has specific requirements:

1. **Transport Mode**: Must use `STREAMABLE_HTTP=true` (SSE is not supported)
2. **Authentication Source**: Tokens come from headers, not environment variables
3. **Client Configuration**: Clients must be configured to send auth headers
4. **Network Accessibility**: Server must be network-accessible (not just STDIO)

## Performance Considerations

### Memory Usage

Each session stores:
- Session ID: ~36 bytes (UUID)
- Auth token: ~20-100 bytes
- Last used timestamp: 8 bytes
- Rate limit counters: ~20 bytes per session

**Estimated memory per session**: ~100-200 bytes

**For 1000 concurrent sessions**: ~100-200 KB (negligible)

### Connection Pooling

The server reuses HTTP connections to GitLab API:
- Agent pooling for HTTP/HTTPS
- Persistent connections
- Automatic connection management

### Scaling Recommendations

| Concurrent Sessions | Recommended RAM | Recommended CPU |
|-------------------|-----------------|-----------------|
| 100               | 512 MB         | 1 core          |
| 1000              | 1 GB           | 2 cores         |
| 5000              | 2 GB           | 4 cores         |
| 10000             | 4 GB           | 8 cores         |

## Advanced Configuration

### Custom Token Validation

To implement custom token validation logic, modify the `validateToken` function in `index.ts`:

```typescript
const validateToken = (token: string): boolean => {
  // Custom validation logic
  if (token.length < 20) return false;
  if (!/^[a-zA-Z0-9_\.-]+$/.test(token)) return false;

  // Add custom checks here
  // Example: Check token prefix
  if (!token.startsWith('glpat-')) return false;

  return true;
};
```

### Reverse Proxy Configuration

#### nginx

```nginx
server {
    listen 443 ssl;
    server_name gitlab-mcp.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location /mcp {
        proxy_pass http://localhost:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        # Preserve auth headers
        proxy_pass_request_headers on;
    }
}
```

#### Caddy

```
gitlab-mcp.example.com {
    reverse_proxy localhost:3002 {
        # Preserve all headers including auth
        header_up Host {host}
        header_up X-Real-IP {remote}
    }
}
```

## FAQ

### Q: Can I use both REMOTE_AUTHORIZATION and GITLAB_PERSONAL_ACCESS_TOKEN?

**A:** No. When `REMOTE_AUTHORIZATION=true`, the `GITLAB_PERSONAL_ACCESS_TOKEN` environment variable is ignored. All authentication must come from HTTP headers.

### Q: What happens if a client doesn't send auth headers?

**A:** The request is rejected with `HTTP 401 Unauthorized`. Remote authorization requires auth headers on every new session.

### Q: Can I rotate tokens without disconnecting?

**A:** Yes! Send a new auth header in any request, and the server will update the stored token for that session.

### Q: How many tokens can I use simultaneously?

**A:** Limited only by `MAX_SESSIONS`. Each session can have a different token, supporting true multi-user scenarios.

### Q: Does this work with self-hosted GitLab?

**A:** Yes! Set `GITLAB_API_URL` to your GitLab instance URL. Both SaaS and self-hosted GitLab are fully supported.

### Q: Can I use OAuth tokens instead of PATs?

**A:** Yes! OAuth access tokens work with the `Authorization: Bearer <token>` header format.

### Q: What happens when SESSION_TIMEOUT_SECONDS expires?

**A:** The auth token is removed from memory, but the transport session remains active. The client must send auth headers again on the next request.

### Q: Is there a way to disable the timeout?

**A:** No, but you can set a very long timeout (max: 86400 seconds = 24 hours). However, automatic expiration is recommended for security.

### Q: Can I see which users/tokens are connected?

**A:** The server doesn't identify users (privacy by design). The `/metrics` endpoint shows session counts but not user identities.

### Q: Does this support SSO or SAML?

**A:** Not directly. You need to obtain a GitLab PAT or OAuth token through your SSO provider, then pass it via headers.

## References

- [GitLab Personal Access Tokens Documentation](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html)
- [GitLab OAuth 2.0 Documentation](https://docs.gitlab.com/ee/api/oauth2.html)
- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [gitlab-mcp Repository](https://github.com/zereight/gitlab-mcp)

## License

This documentation is part of the gitlab-mcp project and is available under the MIT License.
