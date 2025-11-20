# Quick Start: Multi-User Authentication Setup

Get gitlab-mcp running with multi-user support in under 5 minutes!

## Prerequisites

- Docker installed (or Node.js 18+ for local development)
- GitLab account(s) with Personal Access Tokens
- MCP-compatible client (Cursor IDE, Claude Desktop, or custom client)

## Step 1: Start the Server (Choose One)

### Option A: Docker (Recommended for Production)

```bash
docker run -d \
  --name gitlab-mcp \
  -e STREAMABLE_HTTP=true \
  -e REMOTE_AUTHORIZATION=true \
  -e GITLAB_API_URL="https://gitlab.com/api/v4" \
  -e SESSION_TIMEOUT_SECONDS=3600 \
  -e MAX_SESSIONS=1000 \
  -p 3002:3002 \
  iwakitakuma/gitlab-mcp
```

### Option B: npx (Quickest for Testing)

```bash
# Set environment variables
export STREAMABLE_HTTP=true
export REMOTE_AUTHORIZATION=true
export GITLAB_API_URL="https://gitlab.com/api/v4"

# Run server
npx -y @zereight/mcp-gitlab
```

### Option C: Docker Compose (Best for Development)

Create `docker-compose.yml`:

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
```

Then run:

```bash
docker-compose up -d
```

## Step 2: Verify Server is Running

```bash
# Check health
curl http://localhost:3002/health

# Expected output:
# {"status":"healthy","activeSessions":0,"maxSessions":1000,"uptime":...}
```

## Step 3: Configure Your Client

### For Cursor IDE

Edit your Cursor settings (`.cursor/mcp_settings.json` or Settings → MCP):

```json
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",
      "url": "http://localhost:3002/mcp",
      "headers": {
        "Authorization": "Bearer glpat-your-token-here"
      }
    }
  }
}
```

### For Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%/Claude/claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",
      "url": "http://localhost:3002/mcp",
      "headers": {
        "Authorization": "Bearer glpat-your-token-here"
      }
    }
  }
}
```

### For Custom Client (Python)

```python
import requests

response = requests.post(
    "http://localhost:3002/mcp",
    headers={
        "Authorization": "Bearer glpat-your-token-here",
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

### For Custom Client (JavaScript/TypeScript)

```typescript
const response = await fetch("http://localhost:3002/mcp", {
  method: "POST",
  headers: {
    "Authorization": "Bearer glpat-your-token-here",
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    jsonrpc: "2.0",
    method: "tools/list",
    id: 1
  })
});

const result = await response.json();
console.log(result);
```

## Step 4: Get Your GitLab Token

If you don't have a GitLab Personal Access Token yet:

1. Go to GitLab: **Settings → Access Tokens** (or visit https://gitlab.com/-/user_settings/personal_access_tokens)
2. Create a new token with required scopes:
   - `api` - Full API access (for read/write)
   - Or `read_api` - Read-only access (safer)
3. Copy the token (starts with `glpat-`)
4. Use it in your client configuration

## Step 5: Test the Connection

### Using curl

```bash
# Test with your token
curl -X POST http://localhost:3002/mcp \
  -H "Authorization: Bearer glpat-your-token-here" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "tools/list",
    "id": 1
  }'
```

**Expected output:** List of available GitLab tools

### Using Your MCP Client

In Cursor or Claude Desktop, try asking:

```
List my GitLab projects
```

or

```
Show me open merge requests in my projects
```

## Multi-User Setup

### Scenario: Multiple developers sharing one server

Each developer configures their own client with their own token:

**Developer 1 (Alice):**
```json
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",
      "url": "http://shared-server:3002/mcp",
      "headers": {
        "Authorization": "Bearer glpat-alice-token"
      }
    }
  }
}
```

**Developer 2 (Bob):**
```json
{
  "mcpServers": {
    "gitlab": {
      "type": "streamable-http",
      "url": "http://shared-server:3002/mcp",
      "headers": {
        "Authorization": "Bearer glpat-bob-token"
      }
    }
  }
}
```

Each will access GitLab with their own permissions. Sessions are completely isolated!

## Monitoring

### Check Active Sessions

```bash
curl http://localhost:3002/metrics | jq
```

### View Server Health

```bash
curl http://localhost:3002/health
```

### View Logs (Docker)

```bash
docker logs -f gitlab-mcp
```

## Common Issues & Solutions

### ❌ "Missing Authorization or Private-Token header"

**Problem:** Client not sending auth headers

**Solution:** Verify your client config includes the `headers` section with `Authorization: Bearer <token>`

### ❌ "REMOTE_AUTHORIZATION requires STREAMABLE_HTTP"

**Problem:** `STREAMABLE_HTTP` not enabled

**Solution:** Set `STREAMABLE_HTTP=true` in environment variables

### ❌ Connection refused

**Problem:** Server not running or wrong port

**Solution:**
- Check server is running: `docker ps` or check process
- Verify port mapping: `-p 3002:3002`
- Check firewall rules

### ❌ "Rate limit exceeded"

**Problem:** Too many requests from your session

**Solution:**
- Wait 1 minute for rate limit reset
- Or increase `MAX_REQUESTS_PER_MINUTE` (default: 60)

### ❌ GitLab API errors (403, 404)

**Problem:** Token lacks required permissions

**Solution:**
- Verify token has `api` or `read_api` scope
- Check token is not expired
- Test token directly: `curl -H "Authorization: Bearer glpat-xxx" https://gitlab.com/api/v4/user`

## Security Checklist

For production deployments:

- [ ] Use HTTPS (deploy behind reverse proxy with TLS)
- [ ] Restrict network access (firewall, VPN, or internal network only)
- [ ] Use tokens with minimal required scopes
- [ ] Set reasonable `SESSION_TIMEOUT_SECONDS` (1-4 hours recommended)
- [ ] Monitor `/metrics` endpoint regularly
- [ ] Keep `MAX_SESSIONS` appropriate for your user base
- [ ] Review rate limits (`MAX_REQUESTS_PER_MINUTE`)
- [ ] Regularly update to latest version

## Next Steps

- Read the [full multi-user authentication guide](./MULTI_USER_AUTHENTICATION.md)
- Explore [available GitLab tools](./README.md#tools-)
- Configure [optional features](./README.md#environment-variables) (wiki, pipelines, milestones)
- Set up [monitoring and alerting](./MULTI_USER_AUTHENTICATION.md#monitoring--observability)

## Getting Help

- 📖 [Full Documentation](./README.md)
- 🔐 [Multi-User Auth Details](./MULTI_USER_AUTHENTICATION.md)
- 🐛 [Report Issues](https://github.com/zereight/gitlab-mcp/issues)
- 💬 [Discussions](https://github.com/zereight/gitlab-mcp/discussions)

## Example Use Cases

### Use Case 1: Team IDE Integration

**Scenario:** Development team using Cursor IDE with shared MCP server

**Setup:**
1. Deploy server on internal network
2. Each developer adds server URL to their Cursor settings
3. Each developer uses their own GitLab PAT
4. Team shares access to server but maintains individual permissions

**Benefits:**
- Single server maintenance
- Individual GitLab permissions respected
- Session isolation prevents conflicts

### Use Case 2: Multi-Tenant SaaS

**Scenario:** SaaS application offering GitLab integration

**Setup:**
1. Deploy high-capacity server (`MAX_SESSIONS=5000`)
2. Each user authenticates with OAuth
3. OAuth tokens passed via headers
4. Monitor `/metrics` for capacity planning

**Benefits:**
- Scales to many concurrent users
- OAuth provides better UX than PAT management
- Built-in rate limiting prevents abuse

### Use Case 3: CI/CD Pipeline

**Scenario:** Automated agents using GitLab MCP in CI/CD

**Setup:**
1. Deploy server with `GITLAB_READ_ONLY_MODE=true`
2. Create dedicated PAT for CI/CD with `read_api` scope
3. Agents connect with PAT in environment variables
4. Monitor for rate limiting issues

**Benefits:**
- Read-only mode prevents accidental modifications
- Centralized API access logging
- Easy to rotate tokens

---

**You're all set!** 🎉 Your gitlab-mcp server is now running with multi-user support.
