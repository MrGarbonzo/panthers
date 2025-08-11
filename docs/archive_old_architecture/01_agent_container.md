# 01. Agent Container Specification

## Overview
Python-based agent that runs in Secret VM, reads NFT configuration, and provides AI chat capabilities using Secret AI SDK.

## Directory Structure
```
/agent/
├── Dockerfile
├── requirements.txt
├── .env.example
├── src/
│   ├── main.py              # Entry point
│   ├── agent.py             # Core agent logic
│   ├── nft_client.py        # NFT contract interaction
│   ├── secret_ai_client.py  # Secret AI SDK wrapper
│   ├── auth.py              # Authentication/verification
│   ├── config.py            # Configuration management
│   └── api/
│       ├── routes.py        # API routes
│       ├── models.py        # Data models
│       └── middleware.py    # Auth middleware
```

## Technical Requirements

### Base Technologies
- Python 3.12
- FastAPI for HTTP server
- Uvicorn for ASGI server
- Secret AI SDK for LLM
- httpx for async HTTP calls
- pydantic for data validation

### Key Dependencies
- fastapi (0.109.0)
- uvicorn (0.27.0)
- secret-ai-sdk (latest)
- secretpy (for Secret Network interaction)
- redis (for session management)
- python-jose (for JWT)

## Environment Variables Required

### NFT Contract Configuration
- NFT_CONTRACT_ADDRESS - Address of the NFT contract
- SECRET_NETWORK_RPC - RPC endpoint
- SECRET_NETWORK_CHAIN_ID - Chain identifier

### Agent Identity
- NFT_TOKEN_ID - The specific NFT this agent represents
- AGENT_NAME - Display name for the agent

### Secret AI Configuration
- SECRET_AI_API_KEY - API key for Secret AI SDK
- SECRET_AI_MODEL - Model to use (e.g., deepseek-chat)

### Server Configuration
- PORT - Server port (default: 8080)
- HOST - Server host (default: 0.0.0.0)
- CORS_ORIGINS - Allowed CORS origins

### Security
- JWT_SECRET_KEY - Secret for JWT signing
- JWT_ALGORITHM - Algorithm for JWT (HS256)
- JWT_EXPIRATION_MINUTES - Token expiration time

## Core Components

### 1. Main Entry Point
Responsibilities:
- Initialize FastAPI application
- Load configuration from environment
- Set up middleware and error handlers
- Start health check endpoint
- Handle graceful shutdown

### 2. Agent Core
Main agent class that:
- Loads NFT metadata on initialization
- Connects to Secret AI SDK
- Manages conversation context
- Handles chat interactions
- Provides agent information

### 3. NFT Client
Interface to Secret Network that:
- Queries NFT ownership
- Fetches encrypted metadata
- Verifies ownership signatures
- Caches metadata with TTL

### 4. Secret AI Client
Wrapper for Secret AI SDK that:
- Initializes connection to Secret AI
- Manages system prompts
- Handles message generation
- Implements rate limiting
- Supports response streaming

## API Endpoints

### Public Endpoints

**GET /health**
- Returns service health status
- No authentication required
- Used for monitoring

**GET /info**
- Returns agent information
- Shows capabilities and status
- Public information only

### Authenticated Endpoints

**POST /auth/verify**
- Verifies NFT ownership
- Creates session token
- Input: wallet address, signature, message
- Output: JWT access token

**POST /chat**
- Main chat endpoint
- Requires valid JWT
- Input: message, optional conversation ID
- Output: AI response, conversation ID

**GET /chat/history**
- Retrieves conversation history
- Requires valid JWT
- Supports pagination

## Authentication Flow

1. User requests authentication with wallet signature
2. Agent verifies signature validity
3. Agent checks NFT ownership on-chain
4. If valid, creates JWT session token
5. Token used for all subsequent requests
6. Sessions expire after configured timeout

## Error Handling

### Standard Error Codes
- UNAUTHORIZED - Missing or invalid authentication
- FORBIDDEN - User doesn't own the NFT
- NOT_FOUND - Resource not found
- RATE_LIMITED - Too many requests
- AI_ERROR - Secret AI service error
- NFT_ERROR - NFT contract error
- INTERNAL_ERROR - Server error

### Error Response Structure
- Consistent JSON error format
- Includes error code and message
- Optional details for debugging
- Never exposes sensitive information

## Container Configuration

### Dockerfile Requirements
- Base image: Python 3.12 slim
- Non-root user for security
- Health check endpoint
- Minimal attack surface
- Environment variable support

### Resource Requirements
- Memory: 2-4GB
- CPU: 1-2 cores
- GPU: Optional (if using local models)
- Disk: 1GB for application

## Testing Requirements

### Unit Tests
- Test each component independently
- Mock external dependencies
- Cover error scenarios
- Validate data models

### Integration Tests
- End-to-end authentication flow
- Chat interaction with Secret AI
- NFT contract interaction
- Session management

### Performance Tests
- Load testing with multiple users
- Memory leak detection
- Response time benchmarks
- Rate limiting validation

## Security Considerations

1. Input validation on all endpoints
2. Rate limiting per user/IP
3. Prompt injection prevention
4. Secure secret management
5. CORS configuration
6. JWT security best practices
7. Audit logging for sensitive operations

## Deployment Considerations

### For Secret VM
- Container registry setup
- Environment variable injection
- TEE attestation verification
- Health monitoring
- Log aggregation

### Performance Optimization
- Metadata caching strategy
- Connection pooling
- Async operations throughout
- Response streaming for large outputs
- Memory management for conversations

## Monitoring Requirements

### Key Metrics
- Request rate and latency
- Secret AI API usage
- Error rates by type
- Resource utilization
- Active session count

### Logging Strategy
- Structured JSON logging
- Appropriate log levels
- No sensitive data in logs
- Request tracing with IDs
