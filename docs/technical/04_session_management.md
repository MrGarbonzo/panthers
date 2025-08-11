# Technical Specification: In-Memory Session Management

## Overview
This document specifies a simple in-memory session management system that runs entirely within the VM's RAM, requiring no external dependencies like Redis or databases. Perfect for up to 500-1000 users.

## Architecture Decision

### Why In-Memory Storage?
- **User Count**: 500 users = ~1.5MB RAM (negligible)
- **Session Duration**: 1 hour max (ephemeral by design)
- **Simplicity**: No external dependencies
- **Performance**: Nanosecond access times
- **Cost**: $0 additional infrastructure

### What We're NOT Using
- ❌ Redis (unnecessary complexity)
- ❌ PostgreSQL (no persistent storage needed)
- ❌ File storage (no backup requirements)
- ❌ Distributed cache (single VM is sufficient)

## Memory Requirements

### Per Session Storage
```
Each session contains:
- session_id: ~36 bytes
- wallet_address: ~45 bytes  
- nft_token_id: ~10 bytes
- traits: ~500 bytes (personality data)
- last_10_messages: ~2KB (context)
- timestamps: ~50 bytes

Total per session: ~3KB
```

### Capacity Planning
```
Concurrent Users | Memory Usage | VM Requirement
-----------------|--------------|----------------
50               | 150KB        | Any VM
100              | 300KB        | Any VM
500              | 1.5MB        | Any VM
1000             | 3MB          | Any VM

Even 10,000 concurrent users = 30MB (still trivial)
```

## Implementation Design

### Core Session Store Structure

The entire session management system is a single Python class:

```python
class InMemorySessionStore:
    """
    Thread-safe in-memory session storage.
    All sessions expire after 1 hour of inactivity.
    """
```

### Key Features

#### 1. Session Creation
- Generate unique session ID
- Store wallet address and NFT ID
- Cache NFT traits
- Initialize empty message history
- Set expiration timer

#### 2. Session Retrieval
- Validate session exists
- Check expiration (1 hour inactivity)
- Auto-cleanup if expired
- Update last activity timestamp
- Return session data

#### 3. Message History
- Store last 10 messages only
- Automatic rotation (FIFO)
- Provides context for conversations
- Cleared on session expiry

#### 4. Automatic Cleanup
- Background task every 5 minutes
- Remove expired sessions
- Free memory immediately
- No garbage accumulation

### Data Structure

```
sessions = {
    "session_id_123": {
        "wallet": "secret1abc...",
        "nft_id": "456",
        "traits": {
            "personality": "degen",
            "speaking_style": "meme",
            "expertise": ["defi", "gaming"],
            "modifiers": {...}
        },
        "messages": [
            {"role": "user", "content": "..."},
            {"role": "assistant", "content": "..."}
        ],
        "created_at": 1234567890.0,
        "last_activity": 1234567890.0,
        "expires_at": 1234571490.0
    }
}

# Lookup index for wallet -> session
user_sessions = {
    "secret1abc...": "session_id_123"
}
```

## Thread Safety

### Concurrency Handling
Since FastAPI uses async operations, we need thread-safe operations:

```python
import threading

class InMemorySessionStore:
    def __init__(self):
        self.sessions = {}
        self.user_sessions = {}
        self.lock = threading.RLock()  # Reentrant lock
```

All operations use the lock to prevent race conditions:
- Create session: Acquire lock
- Get session: Acquire lock
- Update session: Acquire lock
- Cleanup: Acquire lock

## Session Lifecycle

### 1. Session Creation Flow
```
User connects with wallet
    ↓
Verify NFT ownership
    ↓
Create session in memory
    ↓
Return session ID in JWT
    ↓
Session active for 1 hour
```

### 2. Session Usage Flow
```
User sends chat message
    ↓
Extract session ID from JWT
    ↓
Retrieve session from memory
    ↓
Check if expired
    ↓
Update last activity
    ↓
Process message with traits
    ↓
Store in message history
    ↓
Return response
```

### 3. Session Expiration Flow
```
No activity for 1 hour
    ↓
Cleanup task runs (every 5 min)
    ↓
Identify expired sessions
    ↓
Remove from memory
    ↓
Memory freed immediately
```

## API Integration

### Authentication Endpoint
```
POST /auth/verify
- Verify NFT ownership
- Create in-memory session
- Return JWT with session ID
```

### Chat Endpoint
```
POST /chat
- Get session from memory
- Verify not expired
- Process with Secret AI
- Update message history
- Return response
```

### Health Check
```
GET /health
- Report active sessions count
- Report memory usage
- Report uptime
```

## Cleanup Strategy

### Automatic Cleanup Task
Runs every 5 minutes as background task:

1. Iterate through all sessions
2. Check last_activity timestamp
3. Remove if older than 1 hour
4. Free memory immediately
5. Log cleanup statistics

### Manual Cleanup Triggers
- User explicitly logs out
- NFT transfer detected
- Admin command
- Memory pressure (optional)

## Monitoring and Metrics

### Key Metrics to Track
```
Active Sessions: Current count
Peak Sessions: Maximum concurrent
Memory Usage: Current RAM usage
Cleanup Rate: Sessions/hour expired
Average Session Duration: Minutes
Message Count: Messages per session
```

### Health Monitoring
```python
def get_system_stats():
    return {
        "active_sessions": len(sessions),
        "memory_usage_kb": calculate_memory(),
        "uptime_seconds": time.time() - start_time,
        "total_sessions_created": session_counter,
        "cleanup_last_run": last_cleanup_time
    }
```

## Failure Scenarios

### VM Restart
- **Impact**: All sessions lost
- **Recovery**: Users must reconnect
- **Mitigation**: Not needed (sessions are ephemeral)

### Memory Pressure
- **Unlikely**: 500 users = 1.5MB
- **Detection**: Monitor memory usage
- **Action**: Reduce message history size

### Session Corruption
- **Prevention**: Thread-safe operations
- **Detection**: Validation on retrieval
- **Recovery**: Delete corrupt session

## Scaling Path

### Current: 500 Users
```
Single VM with in-memory storage
- Simple Python dictionary
- 3KB per session
- 1.5MB total RAM
```

### Growth: 1000 Users
```
Same architecture
- Still only 3MB RAM
- No changes needed
```

### Growth: 5000 Users
```
Options:
1. Still use in-memory (15MB RAM) 
2. Add Redis if needed
3. Multiple VMs with sticky sessions
```

## Implementation Guidelines

### Best Practices
1. **Keep sessions small**: Only essential data
2. **Limit message history**: 10 messages max
3. **Aggressive cleanup**: 1 hour timeout
4. **Monitor memory**: Track usage patterns
5. **Log sparingly**: Don't log message content

### Security Considerations
1. **No persistent storage**: Nothing to breach
2. **Auto-expiration**: Limits exposure window
3. **Memory only**: No disk forensics
4. **Clean shutdown**: Clear memory on stop

### Performance Tips
1. **Use dictionaries**: O(1) lookups
2. **Batch cleanup**: Every 5 minutes
3. **Async operations**: Non-blocking
4. **Minimal locking**: Quick operations only

## Testing Strategy

### Unit Tests
- Session creation
- Session expiration
- Message rotation
- Thread safety
- Cleanup logic

### Load Tests
- 500 concurrent sessions
- 1000 requests/second
- Memory usage monitoring
- Cleanup under load

### Failure Tests
- Sudden VM restart
- Memory exhaustion
- Corrupt session data
- Race conditions

## Deployment Configuration

### VM Requirements
```
Minimum:
- 1 vCPU
- 2GB RAM (1.5MB for sessions, rest for OS/App)
- 10GB disk (code only, no data)

Recommended:
- 2 vCPUs  
- 4GB RAM
- 20GB disk
```

### Environment Variables
```bash
# Session Management
SESSION_TIMEOUT_SECONDS=3600  # 1 hour
SESSION_CLEANUP_INTERVAL=300  # 5 minutes
MAX_MESSAGES_PER_SESSION=10
MAX_SESSIONS_PER_USER=1

# Monitoring
ENABLE_METRICS=true
LOG_LEVEL=INFO
```

## Migration Path

### If We Need Persistence Later
1. Add Redis adapter interface
2. Implement Redis store with same API
3. Switch via configuration
4. No code changes needed

### If We Need Distribution Later
1. Add session affinity to load balancer
2. Or implement Redis for shared state
3. Or use JWT for stateless operation

## Advantages of This Approach

1. **Zero Dependencies**: No Redis, no DB
2. **Zero Configuration**: Just run the app
3. **Zero Cost**: No additional infrastructure
4. **Maximum Performance**: Memory-only access
5. **Perfect Privacy**: Nothing persisted
6. **Simple Deployment**: Single Python process
7. **Easy Debugging**: All state in one place
8. **Natural Cleanup**: Expires automatically

## Limitations (Acceptable)

1. **No Persistence**: Sessions lost on restart
2. **Single VM**: Not distributed
3. **Memory Bound**: Limited by RAM (not an issue)
4. **No History**: Previous sessions not retrievable

## Summary

This in-memory session management provides:
- Perfect solution for 500 users
- Trivial resource usage (1.5MB RAM)
- Zero infrastructure dependencies
- Nanosecond access times
- Automatic cleanup
- Natural expiration
- Complete privacy

The entire "database layer" is one Python class with dictionaries. Simple, fast, and exactly what we need for this scale.
