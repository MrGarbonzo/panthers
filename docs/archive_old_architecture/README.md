# Secret Panthers - System Specifications

This directory contains detailed specifications for building the Secret Panthers system. These documents are designed for use with Claude Code or developers to implement the system.

## Project Assumptions
- NFT contract is already deployed and minting works
- NFTs contain encrypted metadata with agent configuration  
- We have access to Secret VM for deployments
- Someone else handles NFT contract and minting

## Specification Documents

### Core System Specs
1. **01_agent_container_spec.md** - Python agent running in Secret VM
2. **02_orchestration_spec.md** - Deployment management service
3. **03_frontend_spec.md** - User dashboard and chat interface
4. **04_authentication_spec.md** - NFT ownership verification system
5. **05_deployment_pipeline_spec.md** - Automated deployment workflow

### Data & Integration Specs
- **data_models.md** - All data structures and schemas
- **api_specification.md** - Complete API documentation
- **integration_points.md** - How services connect

### Infrastructure Specs
- **secret_vm_requirements.md** - Secret VM deployment details
- **monitoring_requirements.md** - Logging and metrics
- **security_requirements.md** - Security considerations

## Implementation Phases

### Phase 1: Core Agent (Week 1)
Build the agent container that:
- Connects to Secret AI SDK
- Reads NFT metadata
- Provides chat interface
- Runs in Secret VM

### Phase 2: Orchestration (Week 2)  
Build the orchestrator that:
- Manages deployments
- Monitors NFT ownership
- Handles lifecycle

### Phase 3: Frontend (Week 3)
Build the dashboard that:
- Shows owned NFTs
- Deploys agents
- Provides chat UI

### Phase 4: Integration (Week 4)
- Connect all services
- Test end-to-end
- Deploy to production

## Key Integration Points

### With NFT Contract (provided by others)
```
We need from NFT contract:
- Query ownership: ownerOf(tokenId) → address
- Get metadata: getMetadata(tokenId) → encrypted data
- Transfer events: for monitoring ownership changes
```

### With Secret VM (we handle)
```
We provide to Secret VM:
- Container image with agent
- Deployment configuration
- Environment variables
- Resource requirements
```

### With Secret AI SDK
```
We use Secret AI for:
- LLM inference
- Private chat responses
- System prompt application
```

## Success Criteria

System is complete when:
1. User connects wallet and sees owned NFTs
2. One-click deployment to Secret VM works
3. Agent uses NFT configuration
4. Chat interface is functional
5. Ownership changes revoke access
