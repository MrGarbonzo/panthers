# 04. Authentication Flow Implementation

## Overview
Complete authentication and authorization system for verifying NFT ownership and managing user sessions across all services.

## Authentication Architecture

### Flow Diagram
```
User -> Frontend -> Orchestrator -> NFT Contract
         |              |              |
         |              v              |
         |         Agent Service       |
         |              ^              |
         |              |              |
         +-- JWT Token -+              |
                                       |
         Signature Verification <------+
```

## Core Components

### 1. Wallet Signature Authentication

#### Message Format
```typescript
interface AuthMessage {
  domain: "secretpanthers.com";
  purpose: "authentication";
  nft_token_id: string;
  wallet_address: string;
  timestamp: number;  // Unix timestamp
  nonce: string;      // Random string
}

// Example message to sign
const message = {
  domain: "secretpanthers.com",
  purpose: "authentication",
  nft_token_id: "123",
  wallet_address: "secret1abc...",
  timestamp: 1705762800,
  nonce: "a4f2b1c9d4e5"
};

// String representation for signing
const messageString = `
Sign this message to authenticate with Secret Panthers.

NFT Token ID: ${message.nft_token_id}
Wallet: ${message.wallet_address}
Timestamp: ${message.timestamp}
Nonce: ${message.nonce}

This signature proves you own Panther #${message.nft_token_id}.
`;
```

#### Signature Verification
```python
# Python (backend)
from secretpy.wallet import verify_arbsecp256k1_signature

def verify_signature(
    message: str,
    signature: str,
    public_key: str
) -> bool:
    """
    Verify Keplr wallet signature
    """
    try:
        # Verify the signature matches the message and public key
        is_valid = verify_arbsecp256k1_signature(
            message.encode(),
            signature,
            public_key
        )
        return is_valid
    except Exception:
        return False
```

### 2. NFT Ownership Verification

#### Contract Query
```python
async def verify_nft_ownership(
    token_id: str,
    wallet_address: str
) -> bool:
    """
    Query NFT contract to verify ownership
    """
    # Query the contract
    query_msg = {
        "owner_of": {
            "token_id": token_id
        }
    }
    
    result = await secret_client.query_contract(
        contract_address=NFT_CONTRACT_ADDRESS,
        query_msg=query_msg
    )
    
    # Check if the wallet owns the NFT
    return result.get("owner") == wallet_address
```

### 3. JWT Token Management

#### Token Structure
```typescript
interface JWTPayload {
  // Standard claims
  sub: string;        // wallet address
  exp: number;        // expiration time
  iat: number;        // issued at
  jti: string;        // JWT ID (unique)
  
  // Custom claims
  nft_token_id: string;
  permissions: string[];
  session_id: string;
  deployment_id?: string;
}
```

#### Token Generation
```python
import jwt
from datetime import datetime, timedelta, timezone

def generate_access_token(
    wallet_address: str,
    nft_token_id: str,
    permissions: List[str]
) -> str:
    """
    Generate JWT access token
    """
    payload = {
        "sub": wallet_address,
        "nft_token_id": nft_token_id,
        "permissions": permissions,
        "session_id": generate_session_id(),
        "exp": datetime.now(timezone.utc) + timedelta(hours=1),
        "iat": datetime.now(timezone.utc),
        "jti": generate_token_id()
    }
    
    token = jwt.encode(
        payload,
        JWT_SECRET_KEY,
        algorithm="HS256"
    )
    
    return token

def generate_refresh_token(
    wallet_address: str,
    nft_token_id: str
) -> str:
    """
    Generate longer-lived refresh token
    """
    payload = {
        "sub": wallet_address,
        "nft_token_id": nft_token_id,
        "type": "refresh",
        "exp": datetime.now(timezone.utc) + timedelta(days=7),
        "iat": datetime.now(timezone.utc),
        "jti": generate_token_id()
    }
    
    return jwt.encode(payload, JWT_REFRESH_SECRET, algorithm="HS256")
```

### 4. Session Management

#### Redis Session Store
```python
class SessionManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.ttl = 3600  # 1 hour
    
    async def create_session(
        self,
        wallet_address: str,
        nft_token_id: str,
        access_token: str,
        refresh_token: str
    ) -> Session:
        """
        Create new session in Redis
        """
        session_id = generate_session_id()
        
        session_data = {
            "session_id": session_id,
            "wallet_address": wallet_address,
            "nft_token_id": nft_token_id,
            "access_token": access_token,
            "refresh_token": refresh_token,
            "created_at": datetime.now().isoformat(),
            "last_activity": datetime.now().isoformat(),
            "ip_address": get_client_ip(),
            "user_agent": get_user_agent()
        }
        
        # Store in Redis with TTL
        await self.redis.setex(
            f"session:{session_id}",
            self.ttl,
            json.dumps(session_data)
        )
        
        # Track active sessions per NFT
        await self.redis.sadd(
            f"nft_sessions:{nft_token_id}",
            session_id
        )
        
        return Session(**session_data)
    
    async def validate_session(
        self,
        session_id: str
    ) -> Optional[Session]:
        """
        Validate and refresh session
        """
        session_data = await self.redis.get(f"session:{session_id}")
        
        if not session_data:
            return None
        
        session = Session(**json.loads(session_data))
        
        # Update last activity
        session.last_activity = datetime.now().isoformat()
        
        # Extend TTL
        await self.redis.expire(f"session:{session_id}", self.ttl)
        
        return session
    
    async def revoke_session(self, session_id: str):
        """
        Revoke a session
        """
        session_data = await self.redis.get(f"session:{session_id}")
        
        if session_data:
            session = json.loads(session_data)
            
            # Remove from active sessions
            await self.redis.srem(
                f"nft_sessions:{session['nft_token_id']}",
                session_id
            )
            
            # Delete session
            await self.redis.delete(f"session:{session_id}")
            
            # Add to revocation list
            await self.redis.setex(
                f"revoked:{session_id}",
                86400,  # Keep for 24 hours
                "1"
            )
    
    async def revoke_nft_sessions(self, nft_token_id: str):
        """
        Revoke all sessions for an NFT (on transfer)
        """
        session_ids = await self.redis.smembers(f"nft_sessions:{nft_token_id}")
        
        for session_id in session_ids:
            await self.revoke_session(session_id)
```

## API Endpoints

### Authentication Endpoints

#### 1. Request Challenge
```yaml
POST /auth/challenge:
  description: Get message to sign
  body: {
    wallet_address: "secret1abc...",
    nft_token_id: "123"
  }
  response: {
    message: "Sign this message...",
    nonce: "a4f2b1c9d4e5",
    expires_at: "2024-01-20T10:05:00Z"
  }
```

#### 2. Verify Signature
```yaml
POST /auth/verify:
  description: Verify signature and create session
  body: {
    wallet_address: "secret1abc...",
    nft_token_id: "123",
    signature: "0x...",
    message: "Sign this message...",
    public_key: "..."
  }
  response: {
    access_token: "eyJhbGc...",
    refresh_token: "eyJhbGc...",
    expires_in: 3600,
    session_id: "sess_abc123"
  }
```

#### 3. Refresh Token
```yaml
POST /auth/refresh:
  description: Get new access token
  body: {
    refresh_token: "eyJhbGc..."
  }
  response: {
    access_token: "eyJhbGc...",
    expires_in: 3600
  }
```

#### 4. Logout
```yaml
POST /auth/logout:
  description: Revoke session
  headers:
    Authorization: "Bearer <token>"
  response: {
    message: "Session revoked successfully"
  }
```

## Middleware Implementation

### FastAPI Middleware
```python
from fastapi import HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def verify_token(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> TokenPayload:
    """
    Verify JWT token middleware
    """
    token = credentials.credentials
    
    try:
        # Decode token
        payload = jwt.decode(
            token,
            JWT_SECRET_KEY,
            algorithms=["HS256"]
        )
        
        # Check if token is revoked
        if await is_token_revoked(payload["jti"]):
            raise HTTPException(401, "Token has been revoked")
        
        # Verify NFT ownership (cached)
        if not await verify_ownership_cached(
            payload["nft_token_id"],
            payload["sub"]
        ):
            raise HTTPException(403, "NFT ownership verification failed")
        
        return TokenPayload(**payload)
        
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token has expired")
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid token")

# Usage in routes
@app.get("/protected")
async def protected_route(
    token_payload: TokenPayload = Depends(verify_token)
):
    return {"wallet": token_payload.sub}
```

### Permission Checking
```python
def require_permissions(*required_permissions: str):
    """
    Decorator to check permissions
    """
    async def permission_checker(
        token_payload: TokenPayload = Depends(verify_token)
    ):
        user_permissions = set(token_payload.permissions)
        required = set(required_permissions)
        
        if not required.issubset(user_permissions):
            raise HTTPException(
                403,
                f"Missing required permissions: {required - user_permissions}"
            )
        
        return token_payload
    
    return permission_checker

# Usage
@app.post("/deploy")
async def deploy_agent(
    token_payload: TokenPayload = Depends(require_permissions("deploy"))
):
    # User has deploy permission
    pass
```

## Security Considerations

### 1. Message Replay Prevention
```python
class NonceManager:
    """
    Prevent replay attacks
    """
    async def is_valid_nonce(self, nonce: str) -> bool:
        # Check if nonce was used before
        if await self.redis.exists(f"nonce:{nonce}"):
            return False
        
        # Mark nonce as used (with TTL)
        await self.redis.setex(f"nonce:{nonce}", 300, "1")
        return True
```

### 2. Rate Limiting
```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(
    key_func=get_remote_address,
    default_limits=["100 per minute"]
)

@app.post("/auth/verify")
@limiter.limit("5 per minute")  # Strict limit for auth
async def verify_signature(request: Request):
    pass
```

### 3. Token Rotation
```python
async def rotate_tokens_if_needed(
    session: Session
) -> Optional[Tuple[str, str]]:
    """
    Rotate tokens if close to expiry
    """
    token_age = datetime.now() - session.created_at
    
    if token_age > timedelta(minutes=45):  # 75% of lifetime
        # Generate new tokens
        new_access = generate_access_token(...)
        new_refresh = generate_refresh_token(...)
        
        # Update session
        await update_session_tokens(session.id, new_access, new_refresh)
        
        return new_access, new_refresh
    
    return None
```

### 4. Multi-Factor Authentication (Optional)
```python
async def verify_2fa(
    wallet_address: str,
    code: str
) -> bool:
    """
    Verify TOTP code for additional security
    """
    secret = await get_2fa_secret(wallet_address)
    
    if not secret:
        return True  # 2FA not enabled
    
    totp = pyotp.TOTP(secret)
    return totp.verify(code, valid_window=1)
```

## Cross-Service Authentication

### Service-to-Service Auth
```python
class ServiceAuth:
    """
    Internal service authentication
    """
    def generate_service_token(
        self,
        service_name: str
    ) -> str:
        """
        Generate token for service-to-service calls
        """
        payload = {
            "sub": service_name,
            "type": "service",
            "permissions": self.get_service_permissions(service_name),
            "exp": datetime.now(timezone.utc) + timedelta(minutes=5)
        }
        
        return jwt.encode(payload, SERVICE_SECRET, algorithm="HS256")
    
    def verify_service_token(self, token: str) -> bool:
        """
        Verify service token
        """
        try:
            payload = jwt.decode(token, SERVICE_SECRET, algorithms=["HS256"])
            return payload.get("type") == "service"
        except:
            return False
```

### Forwarding Authentication
```python
async def forward_auth_to_agent(
    agent_endpoint: str,
    user_token: str,
    request_data: dict
) -> dict:
    """
    Forward authenticated request to agent
    """
    headers = {
        "Authorization": f"Bearer {user_token}",
        "X-Forwarded-For": get_client_ip(),
        "X-Request-ID": generate_request_id()
    }
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{agent_endpoint}/chat",
            json=request_data,
            headers=headers
        )
        
    return response.json()
```

## Testing Authentication

### Unit Tests
```python
def test_signature_verification():
    """Test wallet signature verification"""
    message = create_auth_message("secret1abc", "123")
    signature = sign_message(message, private_key)
    
    assert verify_signature(message, signature, public_key)

def test_jwt_generation():
    """Test JWT token generation"""
    token = generate_access_token("secret1abc", "123", ["chat"])
    payload = jwt.decode(token, JWT_SECRET_KEY, algorithms=["HS256"])
    
    assert payload["sub"] == "secret1abc"
    assert payload["nft_token_id"] == "123"

def test_session_management():
    """Test session creation and validation"""
    session = await session_manager.create_session(...)
    retrieved = await session_manager.validate_session(session.id)
    
    assert retrieved.id == session.id
```

### Integration Tests
```python
async def test_full_auth_flow():
    """Test complete authentication flow"""
    # 1. Request challenge
    challenge = await client.post("/auth/challenge", json={...})
    
    # 2. Sign message
    signature = sign_message(challenge["message"])
    
    # 3. Verify signature
    auth_response = await client.post("/auth/verify", json={
        "signature": signature,
        ...
    })
    
    # 4. Use token
    protected = await client.get(
        "/protected",
        headers={"Authorization": f"Bearer {auth_response['access_token']}"}
    )
    
    assert protected.status_code == 200
```

## Monitoring & Logging

### Authentication Metrics
```python
# Track authentication metrics
auth_attempts = Counter("auth_attempts_total")
auth_successes = Counter("auth_successes_total")
auth_failures = Counter("auth_failures_total")
active_sessions = Gauge("active_sessions")
token_refreshes = Counter("token_refreshes_total")
```

### Audit Logging
```python
async def log_auth_event(
    event_type: str,
    wallet_address: str,
    nft_token_id: str,
    success: bool,
    details: dict = None
):
    """
    Log authentication events for audit
    """
    event = {
        "timestamp": datetime.now().isoformat(),
        "type": event_type,
        "wallet": wallet_address,
        "nft_id": nft_token_id,
        "success": success,
        "ip_address": get_client_ip(),
        "user_agent": get_user_agent(),
        "details": details
    }
    
    await audit_logger.log(event)
```
