# BigScoots v2 API Gateway

> **Cloudflare Worker API Gateway** - Single entry point for BigScoots v2 API with dual authentication (JWT + HMAC)

[![Deploy Status](https://github.com/harisejaz-septem/bigscoots-worker/workflows/Deploy/badge.svg)](https://github.com/harisejaz-septem/bigscoots-worker/actions)

## 🚀 Quick Start

### Gateway URL
```
Production: https://v2-cloudflare.bigscoots.dev
```

### Features
- ✅ **JWT Authentication** (Auth0) - For end users
- ✅ **HMAC Authentication** - For enterprise/API clients  
- ✅ **Public Routes** - No authentication required
- ✅ **Request Routing** - Routes to backend microservices
- ✅ **Identity Injection** - Standardized headers for backends
- ✅ **Replay Protection** - Durable Objects nonce tracking
- ✅ **Global Edge Deployment** - 300+ Cloudflare locations

## 📋 Table of Contents

- [Architecture](#architecture)
- [Authentication](#authentication)
- [Development](#development)
- [Deployment](#deployment)
- [Configuration](#configuration)
- [Testing](#testing)
- [Documentation](#documentation)

## 🏗️ Architecture

```
┌─────────────┐
│   Client    │
│ (Web/Mobile)│
└──────┬──────┘
       │ HTTPS Request
       ↓
┌──────────────────────────────┐
│  API Gateway (Cloudflare)    │
│  v2-cloudflare.bigscoots.dev │
│                              │
│  1. Authenticate (JWT/HMAC)  │
│  2. Validate Permissions     │
│  3. Inject Identity Headers  │
│  4. Route to Backend         │
└──────────────┬───────────────┘
               │
       ┌───────┴────────┐
       ↓                ↓
┌─────────────┐  ┌──────────────┐
│User Service │  │ Site Service │
│             │  │              │
│ v2-user     │  │  v2-sites    │
│.bigscoots   │  │  .bigscoots  │
│.dev         │  │  .dev        │
└─────────────┘  └──────────────┘
```

### Backend Service Mapping

| Path Prefix | Backend Service | Description |
|-------------|----------------|-------------|
| `/user-mgmt/*` | `https://v2-user.bigscoots.dev` | User management, authentication |
| `/site-mgmt/*` | `https://v2-sites.bigscoots.dev` | Site management (primary) |
| `/sites/*` | `https://v2-sites.bigscoots.dev` | Site operations (legacy support) |
| `/authentication/*` | `https://v2-sites.bigscoots.dev` | Authentication operations |
| `/dashboard/*` | `https://v2-sites.bigscoots.dev` | Dashboard data |
| `/management/*` | `https://v2-sites.bigscoots.dev` | Management operations |
| `/service/*` | `https://v2-sites.bigscoots.dev` | Service operations (legacy support) |
| `/plans/*` | `https://v2-sites.bigscoots.dev` | Plan management |

## 🔐 Authentication

### JWT Authentication (End Users)

**Request:**
```http
GET /user-mgmt/profile
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
Content-Type: application/json
```

**Gateway injects:**
```http
X-Auth-Type: jwt
X-User-Id: auth0|68cbea0228b131ec7437b5cc
X-Client-Id: auth0|68cbea0228b131ec7437b5cc
X-Org-Id: null
X-Scopes: ["openid","email","profile"]
X-Role: customer
X-Email: user@example.com
```

### HMAC Authentication (Enterprise Clients)

**Request:**
```http
POST /sites/service-123
X-Key-Id: live_org_test123
X-Timestamp: 1728200000
X-Nonce: 7d6b6a1c-6f55-4e8a-bf4a-58c5a70f1d2e
X-Signature: AbCdEf123...
X-Content-SHA256: 3f786850e387550...
Host: v2-cloudflare.bigscoots.dev
Content-Type: application/json
```

**Gateway injects:**
```http
X-Auth-Type: hmac
X-Client-Id: live_org_test123
X-Org-Id: org_abc123
X-Scopes: ["sites:read","sites:write"]
```

### Public Routes

No authentication required:
- `/user-mgmt/auth/login`
- `/user-mgmt/auth/refresh`
- `/user-mgmt/auth/verify-oob-code`
- `/user-mgmt/auth/social-login`
- `/authentication/get-token`

## 💻 Development

### Prerequisites

- Node.js 18+ 
- npm or yarn
- Cloudflare account with Workers access
- Wrangler CLI installed globally (optional)

### Setup

```bash
# Install dependencies
npm install

# Generate TypeScript types from wrangler.jsonc
npm run cf-typegen

# Start local development server
npm run dev
```

### Local Testing

```bash
# Run unit tests
npm test

# Test HMAC authentication
npm run test:hmac
```

### Environment Variables

Configured in `wrangler.jsonc`:

```jsonc
{
  "vars": {
    "AUTH0_ISSUER": "https://auth.scoots-test.com/",
    "AUTH0_AUDIENCE": "https://dev-luzn47z5slx3svx7.us.auth0.com/api/v2/",
    "JWKS_URL": "https://auth.scoots-test.com/.well-known/jwks.json",
    "USER_SERVICE_URL": "https://v2-user.bigscoots.dev",
    "SITE_SERVICE_URL": "https://v2-sites.bigscoots.dev"
  }
}
```

## 🚢 Deployment

### Manual Deployment

```bash
# Deploy to production
npm run deploy

# Deploy to preview environment
wrangler deploy --env preview
```

### Automated Deployment (GitHub Actions)

Deployments are automated via GitHub Actions:
- **Push to `main`** → Deploys to production
- **Pull Requests** → Deploys to preview environment

See [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml) for details.

### Deployment Checklist

See [DEPLOYMENT.md](DEPLOYMENT.md) for complete deployment checklist.

## ⚙️ Configuration

### Wrangler Configuration

Key settings in `wrangler.jsonc`:

- **Worker Name**: `bigscoots-v2-gateway-test`
- **Route**: `v2-cloudflare.bigscoots.dev/*`
- **KV Namespace**: API key storage
- **Durable Objects**: Nonce replay protection
- **Compatibility Date**: `2025-09-22`

### KV Storage Schema

API keys stored as:
```
Key: "api_key:{keyId}"
Value: {
  "secret": "base64randomsecret",
  "orgId": "enterprise-1",
  "scopes": ["users:read", "sites:write"],
  "rateLimit": {
    "minute": 60,
    "hour": 1000,
    "day": 20000
  }
}
```

## 🧪 Testing

### Test Routes

Built-in test endpoints:
- `GET /hi` - Simple health check
- `GET /json` - JSON response test

### Example Requests

**JWT Test:**
```bash
curl -H "Authorization: Bearer <token>" \
     https://v2-cloudflare.bigscoots.dev/user-mgmt/profile
```

**HMAC Test:**
```bash
# See test/hmac-gateway-test.js for complete example
npm run test:hmac
```

## 📚 Documentation

Comprehensive documentation available in [`notes/`](notes/):

- **[Usage Tutorial](notes/usage-tutorial.md)** - Complete API integration guide
- **[JWT Code Walkthrough](notes/jwt-code-walkthrough.md)** - JWT implementation details
- **[HMAC Implementation](notes/hmac-implementation-update.md)** - HMAC authentication guide
- **[Durable Objects Explained](notes/durable-objects-explained.md)** - Nonce replay protection
- **[Cloudflare Edge Network](notes/cloudflare-edge-anycast-magic.md)** - Deployment architecture

## 🔒 Security

### Implemented Features

- ✅ JWT signature verification (RS256)
- ✅ HMAC-SHA256 request signing
- ✅ Timestamp validation (±300s window)
- ✅ Nonce replay protection (Durable Objects)
- ✅ Body hash verification
- ✅ Identity header injection
- ✅ Auth header stripping before forwarding

### Backend Security Requirements

Backend services **MUST** verify:
- `X-Auth-Type` header is present
- `X-Client-Id` header is present
- `X-Scopes` header is present
- No direct authentication headers (`Authorization`, `X-Key-Id`, etc.)

## 📊 Monitoring

### Cloudflare Dashboard

- **Workers Dashboard**: https://dash.cloudflare.com
- **Analytics**: Request metrics, error rates, performance
- **Logs**: Real-time request/response logging

### Observability

Enabled in `wrangler.jsonc`:
```jsonc
{
  "observability": {
    "enabled": true
  }
}
```

## 🛠️ Troubleshooting

### Common Issues

**401 Unauthorized (JWT)**
- Check token expiration
- Verify issuer/audience match
- Ensure token signature is valid

**401 Signature Verification Failed (HMAC)**
- Verify canonical string format
- Check headers are lowercase and sorted
- Ensure body hash matches `X-Content-SHA256`

**404 Not Found**
- Verify path prefix matches route config
- Check backend service is accessible
- Ensure route is defined in `route-config.ts`

See [Usage Tutorial - Troubleshooting](notes/usage-tutorial.md#troubleshooting) for detailed solutions.

## 📝 License

Private - BigScoots Internal Use Only

## 👥 Contributing

1. Create feature branch from `main`
2. Make changes and test locally
3. Submit PR with description
4. CI/CD will deploy to preview environment
5. After approval, merge to `main` for production deployment

## 🔗 Related Repositories

- **User Service**: `v2-user.bigscoots.dev`
- **Site Service**: `v2-sites.bigscoots.dev`

---

**Last Updated**: February 2026  
**Maintainer**: BigScoots Engineering Team
