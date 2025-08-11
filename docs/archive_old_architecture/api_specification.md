# API Specification

## Overview
Complete API specification for all services in the Secret Panthers system.

## Base URLs

### Production
- Orchestrator: `https://api.secretpanthers.com`
- Agent: `https://agent-{nft-id}.secretpanthers.com`
- WebSocket: `wss://api.secretpanthers.com`

### Development
- Orchestrator: `http://localhost:3000`
- Agent: `http://localhost:8080`
- WebSocket: `ws://localhost:3000`

## Authentication

### Method
- Bearer token (JWT) in Authorization header
- Token obtained via NFT ownership verification
- Tokens expire after 1 hour
- Refresh tokens valid for 7 days

### Header Format
```
Authorization: Bearer <jwt_token>
```

## Orchestrator API

### Authentication Endpoints

#### POST /auth/challenge
Get a message to sign for authentication.

Request:
- wallet_address: string
- nft_token_id: string

Response:
- message: string (to be signed)
- nonce: string
- expires_at: timestamp

#### POST /auth/verify
Verify signature and create session.

Request:
- wallet_address: string
- nft_token_id: string
- signature: string
- message: string

Response:
- access_token: string
- refresh_token: string
- expires_in: number (seconds)
- session_id: string

#### POST /auth/refresh
Get new access token using refresh token.

Request:
- refresh_token: string

Response:
- access_token: string
- expires_in: number

#### POST /auth/logout
Revoke current session.

Headers:
- Authorization: Bearer token

Response:
- message: string

### Deployment Endpoints

#### POST /deployments
Deploy an agent for an NFT.

Headers:
- Authorization: Bearer token

Request:
- nft_token_id: string
- provider: string (secret_vm|akash|local)
- resources: object (optional override)

Response:
- deployment_id: string
- status: string
- endpoint_url: string (when ready)
- estimated_time: number (seconds)

#### GET /deployments/{nft_token_id}
Get deployment status for an NFT.

Headers:
- Authorization: Bearer token

Response:
- deployment_id: string
- status: string
- provider: string
- endpoint_url: string
- created_at: timestamp
- resources: object
- health: object

#### DELETE /deployments/{nft_token_id}
Stop a deployment.

Headers:
- Authorization: Bearer token

Response:
- message: string
- status: string

#### GET /deployments
List all deployments for authenticated user.

Headers:
- Authorization: Bearer token

Query Parameters:
- status: string (filter by status)
- provider: string (filter by provider)

Response:
- deployments: array
- total: number

### Monitoring Endpoints

#### GET /health
Service health check.

Response:
- status: string (healthy|unhealthy)
- version: string
- uptime: number
- deployments_active: number

#### GET /metrics
Prometheus-formatted metrics.

Response:
- Text format metrics

## Agent API

### Public Endpoints

#### GET /health
Agent health check.

Response:
- status: string
- version: string
- uptime: number

#### GET /info
Agent information.

Response:
- name: string
- nft_token_id: string
- capabilities: array
- model: string
- status: string

### Authenticated Endpoints

#### POST /auth/verify
Verify NFT ownership for agent access.

Request:
- wallet_address: string
- signature: string
- message: string

Response:
- access_token: string
- expires_in: number

#### POST /chat
Send message to agent.

Headers:
- Authorization: Bearer token

Request:
- message: string
- conversation_id: string (optional)
- stream: boolean (optional)

Response:
- response: string
- conversation_id: string
- timestamp: timestamp
- tokens_used: number

#### GET /chat/history
Get conversation history.

Headers:
- Authorization: Bearer token

Query Parameters:
- conversation_id: string
- limit: number (default 50)
- offset: number (default 0)

Response:
- messages: array
- total: number
- conversation_id: string

#### POST /chat/clear
Clear conversation history.

Headers:
- Authorization: Bearer token

Request:
- conversation_id: string

Response:
- message: string

## WebSocket API

### Connection
Connect with authentication token as query parameter:
```
wss://api.secretpanthers.com/ws?token=<jwt_token>
```

### Deployment Updates

Subscribe to deployment updates:
```
{
  "type": "subscribe",
  "channel": "deployment",
  "nft_token_id": "123"
}
```

Receive updates:
```
{
  "type": "deployment_status",
  "deployment_id": "uuid",
  "status": "deploying|running|stopped",
  "progress": 50,
  "message": "string"
}
```

### Chat Streaming

Enable streaming for chat:
```
{
  "type": "chat_stream",
  "conversation_id": "uuid"
}
```

Receive streamed responses:
```
{
  "type": "chat_chunk",
  "content": "partial response...",
  "done": false
}
```

## Error Responses

### Standard Error Format
All errors follow this format:
```
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {
      // Optional additional information
    }
  }
}
```

### Error Codes

#### Authentication Errors
- UNAUTHORIZED: Missing or invalid token
- FORBIDDEN: Not authorized for resource
- TOKEN_EXPIRED: JWT has expired
- INVALID_SIGNATURE: Signature verification failed

#### Resource Errors
- NOT_FOUND: Resource doesn't exist
- ALREADY_EXISTS: Resource already exists
- CONFLICT: Resource state conflict

#### Validation Errors
- INVALID_REQUEST: Malformed request
- MISSING_FIELD: Required field missing
- INVALID_FIELD: Field validation failed

#### Rate Limiting
- RATE_LIMITED: Too many requests
- QUOTA_EXCEEDED: Usage quota exceeded

#### System Errors
- INTERNAL_ERROR: Server error
- SERVICE_UNAVAILABLE: Service down
- TIMEOUT: Request timeout

## Rate Limits

### Per Endpoint
- Authentication: 5 requests/minute
- Deployment: 10 requests/minute
- Chat: 30 requests/minute
- Other: 60 requests/minute

### Response Headers
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1634567890
```

## Pagination

### Query Parameters
- limit: number (max 100)
- offset: number
- sort: string (field to sort by)
- order: string (asc|desc)

### Response Format
```
{
  "data": [...],
  "pagination": {
    "total": 500,
    "limit": 50,
    "offset": 0,
    "has_more": true
  }
}
```

## Versioning

### API Version
Current version: v1
Version in URL: `/v1/endpoint`

### Backwards Compatibility
- New fields may be added
- Existing fields won't be removed
- Deprecation notices given 30 days ahead

## Security

### HTTPS Required
All production requests must use HTTPS.

### CORS Policy
Allowed origins configured per environment.

### Request Signing
Optional HMAC signing for high-security operations.

### Audit Logging
All API calls are logged for security audit.

## SDKs

### Available SDKs
- JavaScript/TypeScript
- Python
- Go

### Example Usage
Reference implementation in each SDK repository.
