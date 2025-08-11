# 02. Orchestrator Service Implementation

## Overview
Service that manages the lifecycle of agent deployments in Secret VM. Handles deployment, monitoring, and cleanup when NFTs are transferred.

## Directory Structure
```
/orchestrator/
├── Dockerfile
├── requirements.txt
├── .env.example
├── src/
│   ├── main.py              # Entry point
│   ├── orchestrator.py      # Core orchestration logic
│   ├── deployments/
│   │   ├── manager.py       # Deployment manager
│   │   ├── secret_vm.py     # Secret VM interface
│   │   ├── monitor.py       # Health monitoring
│   │   └── models.py        # Deployment models
│   ├── nft/
│   │   ├── watcher.py       # NFT transfer monitoring
│   │   ├── client.py        # NFT contract interface
│   │   └── events.py        # Event handlers
│   ├── database/
│   │   ├── models.py        # SQLAlchemy models
│   │   ├── crud.py          # Database operations
│   │   └── session.py       # Database connection
│   └── api/
│       ├── routes.py        # API endpoints
│       ├── websocket.py     # WebSocket for real-time updates
│       └── auth.py          # Authentication
```

## Technical Requirements

### Dependencies
```txt
fastapi==0.109.0
uvicorn==0.27.0
sqlalchemy==2.0.25
asyncpg==0.29.0
redis==5.0.1
httpx==0.26.0
pydantic==2.5.0
python-jose[cryptography]==3.3.0
websockets==12.0
secretpy==0.1.0
docker==7.0.0
kubernetes-asyncio==29.0.0
celery==5.3.4
```

## Database Schema

### PostgreSQL Tables
```sql
-- Deployments table
CREATE TABLE deployments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nft_token_id VARCHAR(64) UNIQUE NOT NULL,
    owner_address VARCHAR(64) NOT NULL,
    deployment_id VARCHAR(256),  -- Secret VM deployment ID
    status VARCHAR(32) NOT NULL,  -- pending, deploying, running, stopped, failed
    provider VARCHAR(32) NOT NULL,  -- secret_vm, akash, local
    endpoint_url VARCHAR(512),
    resources JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_health_check TIMESTAMP,
    error_message TEXT
);

-- Deployment events
CREATE TABLE deployment_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deployment_id UUID REFERENCES deployments(id),
    event_type VARCHAR(64) NOT NULL,
    event_data JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- NFT ownership cache
CREATE TABLE nft_ownership (
    token_id VARCHAR(64) PRIMARY KEY,
    owner_address VARCHAR(64) NOT NULL,
    last_verified TIMESTAMP DEFAULT NOW(),
    block_height BIGINT
);
```

## Core Implementation

### 1. Orchestrator Core (orchestrator.py)
```python
class Orchestrator:
    """
    Main orchestration engine
    """
    
    async def deploy_agent(
        self,
        nft_token_id: str,
        owner_address: str,
        provider: str = "secret_vm"
    ) -> DeploymentResult:
        """
        Deploy new agent:
        1. Verify NFT ownership
        2. Get NFT metadata
        3. Check for existing deployment
        4. Deploy to chosen provider
        5. Store deployment info
        6. Return endpoint
        """
        
    async def stop_deployment(self, nft_token_id: str):
        """
        Stop agent deployment:
        1. Find deployment
        2. Stop container/VM
        3. Clean up resources
        4. Update database
        """
        
    async def get_deployment_status(self, nft_token_id: str):
        """
        Get current deployment status
        """
        
    async def handle_nft_transfer(self, token_id: str, new_owner: str):
        """
        Handle NFT transfer:
        1. Stop existing deployment
        2. Notify old owner
        3. Allow new owner to deploy
        """
```

### 2. Secret VM Deployment (deployments/secret_vm.py)
```python
class SecretVMDeployer:
    """
    Interface to Secret VM
    """
    
    async def deploy(
        self,
        deployment_name: str,
        image: str,
        environment: Dict[str, str],
        resources: ResourceRequirements
    ) -> DeploymentInfo:
        """
        Deploy to Secret VM:
        1. Prepare deployment manifest
        2. Submit to Secret VM API
        3. Wait for ready state
        4. Get endpoint URL
        5. Verify attestation
        """
        
    async def stop(self, deployment_id: str):
        """
        Stop deployment in Secret VM
        """
        
    async def get_status(self, deployment_id: str):
        """
        Get deployment status
        """
        
    async def get_logs(self, deployment_id: str, lines: int = 100):
        """
        Retrieve deployment logs
        """
```

### 3. NFT Watcher (nft/watcher.py)
```python
class NFTWatcher:
    """
    Monitor NFT transfers and ownership changes
    """
    
    async def start_watching(self):
        """
        Start monitoring:
        1. Subscribe to contract events
        2. Poll for ownership changes
        3. Trigger handlers on changes
        """
        
    async def check_ownership(self, token_id: str) -> str:
        """
        Query current owner from contract
        """
        
    async def handle_transfer_event(self, event: TransferEvent):
        """
        Handle transfer:
        1. Verify event
        2. Update ownership cache
        3. Trigger deployment cleanup
        4. Notify services
        """
```

### 4. Deployment Monitor (deployments/monitor.py)
```python
class DeploymentMonitor:
    """
    Monitor health of all deployments
    """
    
    async def health_check_loop(self):
        """
        Continuous monitoring:
        1. Check each deployment health
        2. Restart if unhealthy
        3. Clean up failed deployments
        4. Update metrics
        """
        
    async def check_deployment_health(self, deployment: Deployment):
        """
        Health check:
        1. HTTP health endpoint
        2. Resource usage
        3. Response time
        4. Error rates
        """
```

## API Endpoints

### Deployment Management
```yaml
POST /deploy:
  description: Deploy agent for NFT
  body: {
    nft_token_id: "123",
    wallet_address: "secret1...",
    signature: "0x...",
    provider: "secret_vm"  # optional, default: secret_vm
  }
  response: {
    deployment_id: "uuid",
    status: "deploying",
    endpoint_url: null,  # Will be populated when ready
    estimated_time: 30
  }

GET /deployment/{nft_token_id}:
  description: Get deployment status
  response: {
    deployment_id: "uuid",
    status: "running",
    endpoint_url: "https://agent-123.secret-vm.io",
    created_at: "2024-01-20T10:00:00Z",
    resources: {
      cpu: 2,
      memory: "4Gi",
      gpu: true
    },
    health: {
      status: "healthy",
      last_check: "2024-01-20T10:30:00Z",
      uptime: 1800
    }
  }

DELETE /deployment/{nft_token_id}:
  description: Stop deployment
  headers:
    Authorization: "Bearer <token>"
  response: {
    status: "stopped",
    message: "Deployment stopped successfully"
  }

GET /deployments:
  description: List all deployments for wallet
  headers:
    Authorization: "Bearer <token>"
  response: {
    deployments: [...],
    total: 5
  }
```

### Monitoring Endpoints
```yaml
GET /health:
  description: Service health
  response: {
    status: "healthy",
    deployments_active: 42,
    version: "1.0.0"
  }

GET /metrics:
  description: Prometheus metrics
  response: |
    deployments_total{status="running"} 42
    deployments_total{status="failed"} 3
    deployment_duration_seconds{quantile="0.99"} 28.5

WS /ws/deployment/{nft_token_id}:
  description: WebSocket for real-time deployment updates
  messages:
    - { type: "status", status: "deploying" }
    - { type: "progress", percent: 50 }
    - { type: "ready", endpoint_url: "https://..." }
    - { type: "error", message: "..." }
```

## Deployment Strategies

### Secret VM Deployment
```python
deployment_config = {
    "name": f"panther-agent-{nft_token_id}",
    "image": "secretpanthers/agent:latest",
    "replicas": 1,
    "env": {
        "NFT_TOKEN_ID": nft_token_id,
        "NFT_CONTRACT_ADDRESS": contract_address,
        "SECRET_AI_API_KEY": secret_ai_key
    },
    "resources": {
        "requests": {
            "cpu": "1",
            "memory": "2Gi",
            "gpu": "1"
        },
        "limits": {
            "cpu": "2",
            "memory": "4Gi",
            "gpu": "1"
        }
    },
    "tee": {
        "enabled": true,
        "attestation": "strict"
    },
    "networking": {
        "expose": true,
        "port": 8080,
        "domain": f"agent-{nft_token_id}.secret-vm.io"
    }
}
```

### Deployment State Machine
```
States:
- PENDING: Request received, validating
- DEPLOYING: Sending to provider
- STARTING: Container starting
- RUNNING: Healthy and serving
- UNHEALTHY: Failed health checks
- STOPPING: Shutdown initiated
- STOPPED: Fully stopped
- FAILED: Deployment failed

Transitions:
PENDING -> DEPLOYING -> STARTING -> RUNNING
RUNNING -> UNHEALTHY -> RUNNING (auto-recovery)
RUNNING -> STOPPING -> STOPPED
ANY -> FAILED (on error)
```

## Background Tasks

### Using Celery
```python
@celery_app.task
async def cleanup_orphaned_deployments():
    """
    Run every hour:
    1. Find deployments with transferred NFTs
    2. Stop orphaned deployments
    3. Clean up resources
    """

@celery_app.task
async def refresh_ownership_cache():
    """
    Run every 5 minutes:
    1. Check all active deployments
    2. Verify NFT ownership
    3. Update cache
    """

@celery_app.task
async def collect_metrics():
    """
    Run every minute:
    1. Collect deployment metrics
    2. Check resource usage
    3. Store in time-series DB
    """
```

## Error Handling

### Deployment Failures
1. **Insufficient Resources**: Queue deployment or suggest alternatives
2. **Image Pull Error**: Retry with exponential backoff
3. **Secret VM Unavailable**: Fallback to alternative provider
4. **NFT Transfer During Deploy**: Cancel deployment, notify user

### Recovery Strategies
- Automatic retry with backoff
- Circuit breaker for provider failures
- Graceful degradation to alternative providers
- Manual intervention webhooks

## Security Considerations

1. **Deployment Isolation**: Each deployment in separate namespace/context
2. **Resource Limits**: Enforce CPU/memory/GPU limits
3. **Network Policies**: Restrict inter-deployment communication
4. **Secret Management**: Rotate Secret AI keys regularly
5. **Audit Logging**: Log all deployment actions

## Monitoring & Alerting

### Key Metrics
- Deployment success rate
- Average deployment time
- Resource utilization
- Provider availability
- Cost per deployment

### Alerts
- Deployment failures > 5%
- Provider unavailable
- Resource exhaustion
- Unusual NFT transfer patterns
- Cost overruns

## Cost Management

### Tracking
```python
class CostTracker:
    async def calculate_deployment_cost(
        self,
        deployment: Deployment
    ) -> Cost:
        """
        Calculate costs:
        - Compute time
        - GPU usage
        - Network egress
        - Storage
        """
        
    async def get_user_spending(
        self,
        wallet_address: str,
        period: TimePeriod
    ) -> SpendingSummary:
        """
        User spending report
        """
```

### Optimization
- Auto-scale down idle deployments
- Use spot instances where possible
- Implement spending limits
- Provide cost estimates before deployment

## Testing Strategy

### Unit Tests
- Test deployment logic
- Mock Secret VM API
- Test state transitions
- Test error handling

### Integration Tests
- Deploy to test Secret VM
- Test NFT transfer handling
- Test monitoring loops
- Test cost calculations

### Load Tests
- Simulate 100+ simultaneous deployments
- Test queuing system
- Monitor resource usage
- Test recovery under load
