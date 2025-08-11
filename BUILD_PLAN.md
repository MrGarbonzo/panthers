# Secret Panthers - Build Plan
*Living Document - Last Updated: January 2025*

## Project Overview
Secret Panthers: NFTs that contain and control private AI agents running in TEE infrastructure.

**Core Technologies:**
- Secret Network NFTs (encrypted metadata)
- Secret VM (GPU-enabled TEE)
- Secret AI SDK (private LLM calls)

**Vision:** Each NFT contains configuration for a unique AI agent that only the owner can deploy and use.

---

## Phase 1: Foundation (Week 1-2) - CURRENT PHASE
**Goal:** Get core pieces talking to each other

### IMPORTANT NOTE
**NFT Contract and Minting are handled by another team**
We are responsible for everything AFTER the NFT exists:
- Agent implementation
- Deployment orchestration
- Frontend dashboard
- Authentication flow

### 1.1 Development Environment Setup ✅

#### Task List:
- [x] Create Secret Network testnet wallet ✅
  - [x] Install Keplr wallet
  - [x] Get testnet SCRT from faucet
  - [x] Document wallet address
  
- [x] Configure Secret VM Access ✅
  - [x] Contact Secret team for access credentials
  - [x] Get GPU TEE allocation
  - [x] Document endpoints and keys
  - [x] Test basic deployment capability
  
- [ ] Set up IPFS Storage
  - [ ] Create Pinata account (easier than own node)
  - [ ] Get API keys
  - [ ] Test pin/unpin functionality
  - [ ] Document CID storage strategy
  
- [ ] Repository Structure
  - [ ] Create GitHub repo: `secret-panthers`
  - [ ] Set up monorepo structure:
    ```
    /contracts     - CosmWasm NFT contract
    /agent        - Python agent container
    /orchestrator - Deployment management
    /frontend     - Next.js app
    /docs         - Documentation
    ```
  - [ ] Add .gitignore for secrets
  - [ ] Create README.md
  
- [ ] Local Development Tools
  - [ ] Install Rust (for CosmWasm)
  - [ ] Install Docker Desktop
  - [ ] Install Python 3.12 + Anaconda
  - [ ] Install Node.js 20+
  - [ ] Install secretcli

#### Resources Needed:
- Secret Network testnet docs: https://docs.scrt.network/secret-network-documentation/development/getting-started
- Secret VM access form: [Need to find/request]
- Pinata signup: https://www.pinata.cloud/

#### Blockers:
- Waiting for Secret VM access credentials
- Need to determine GPU allocation limits

---

### 1.2 NFT Contract Interface ✅ (PROVIDED BY OTHERS)

#### What We Need From NFT Contract:
- Query ownership: `ownerOf(tokenId) → address`
- Get metadata: `getMetadata(tokenId) → encrypted data`
- Monitor transfer events

#### Expected Metadata Structure:
```
EncryptedMetadata {
  agent_config: {
    model: "deepseek-chat",
    system_prompt: "You are a helpful panther...",
    temperature: 0.7,
    max_tokens: 2000,
    capabilities: ["chat", "code"]
  },
  deployment: {
    image: "secretpanthers/agent:latest",
    resources: {
      cpu: "2",
      memory: "4Gi", 
      gpu: true
    }
  }
}
```

#### Integration Points:
- [ ] Document NFT contract address
- [ ] Test query functions
- [ ] Verify metadata format
- [ ] Set up event monitoring

---

### 1.3 Simple Agent Container 🔄

#### Container Requirements:
```
/agent/
├── Dockerfile
├── requirements.txt
├── src/
│   ├── main.py           - Entry point
│   ├── agent.py          - Core agent logic
│   ├── nft_client.py     - Query NFT contract
│   ├── secret_ai.py      - Secret AI SDK wrapper
│   └── api.py            - HTTP endpoints
```

#### Base Implementation Tasks:
- [ ] Create Python project structure
- [ ] Set up FastAPI server
- [ ] Implement health check endpoint
- [ ] Add Secret AI SDK
- [ ] Create basic chat endpoint
- [ ] Add environment variable configuration
- [ ] Write Dockerfile
- [ ] Build and test container locally

#### Environment Variables:
```
NFT_CONTRACT_ADDRESS=secret1...
NFT_TOKEN_ID=123
SECRET_AI_API_KEY=...
SECRET_NETWORK_RPC=https://...
AGENT_PORT=8080
```

#### API Endpoints:
- `GET /health` - Health check
- `POST /chat` - Chat with agent
- `GET /info` - Agent information
- `POST /verify` - Verify NFT ownership

---

### 1.4 Secret VM Hello World ✅

#### Deployment Steps:
- [x] Create minimal test container ✅ - Already familiar
- [x] Push to Docker Hub ✅ - Already familiar
- [x] Write Secret VM deployment config ✅ - Already familiar
- [x] Deploy via Secret VM API/CLI ✅ - Already familiar
- [x] Verify TEE attestation ✅ - Already familiar
- [x] Test GPU access ✅ - Already familiar
- [x] Document deployment process ✅ - Already familiar

#### Secret VM Config Template:
```yaml
name: test-panther-deployment
image: secretpanthers/agent:test
resources:
  gpu: enabled
  memory: 8Gi
  cpu: 2
environment:
  - TEST_MODE=true
attestation:
  required: true
  level: strict
```

#### Verification Checklist:
- [ ] Container runs in TEE
- [ ] GPU is accessible
- [ ] Network connectivity works
- [ ] Can query Secret Network
- [ ] Attestation proof generated

---

## Week 1 Daily Goals

### Monday - Environment Setup
- [x] Complete all development environment setup ✅
- [x] Get Secret VM access sorted ✅ - Already have
- [ ] Create GitHub repository

### Tuesday - NFT Contract Start
- [ ] Set up CosmWasm project
- [ ] Implement basic NFT structure
- [ ] Write mint function

### Wednesday - NFT Contract Continued
- [ ] Add encrypted metadata support
- [ ] Implement ownership verification
- [ ] Start unit tests

### Thursday - Agent Container
- [ ] Create Python project
- [ ] Set up FastAPI
- [ ] Integrate Secret AI SDK

### Friday - Integration
- [ ] Test NFT contract on testnet
- [ ] Test agent container locally
- [ ] Attempt first Secret VM deployment

---

## Week 2 Daily Goals

### Monday - Secret VM Deep Dive
- [ ] Full Secret VM deployment working
- [ ] Document deployment process
- [ ] Test GPU capabilities

### Tuesday - NFT-Agent Connection
- [ ] Agent can query NFT contract
- [ ] Decrypt metadata successfully
- [ ] Apply configuration

### Wednesday - End-to-End Test
- [ ] Mint NFT with metadata
- [ ] Deploy agent to Secret VM
- [ ] Verify agent uses NFT config

### Thursday - Polish & Debug
- [ ] Fix any integration issues
- [ ] Improve error handling
- [ ] Add logging

### Friday - Documentation & Demo
- [ ] Document entire flow
- [ ] Create demo script
- [ ] Record demo video

---

## Success Criteria for Phase 1

✅ **Complete when:**
1. NFT contract deployed to testnet
2. Can mint NFT with encrypted metadata
3. Agent container runs in Secret VM
4. Agent can read NFT metadata
5. Basic chat functionality works
6. Ownership verification implemented

---

## Blockers & Issues Log

### Active Blockers:
- ✅ ~~Need Secret VM access credentials~~ RESOLVED - Already have access
- 🟡 Unclear on Secret VM deployment costs

### Resolved:
- ✅ [Date] - Issue description - Resolution

---

## Notes & Decisions

### Architecture Decisions:
- Using Pinata for IPFS (simpler than running own node)
- Python for agent (good AI library support)
- FastAPI for agent API (async support)
- Next.js for frontend (good Web3 integration)

### Open Questions:
1. How much GPU memory per Secret VM instance?
2. Rate limits on Secret AI SDK?
3. Cost per deployment on Secret VM?
4. Maximum metadata size for NFT?

---

## Resource Links

### Documentation:
- Secret Network Docs: https://docs.scrt.network
- Secret AI SDK: https://github.com/SecretFoundation/secret-ai-sdk
- CosmWasm: https://book.cosmwasm.com

### Repositories:
- Our Repo: [To be created]
- Secret AI Examples: https://github.com/SecretFoundation/secret-ai-getting-started

### Community:
- Secret Network Discord: [Join link]
- Office Hours: [Schedule]

---

## Phase 2 Preview: Integration (Week 3-4)
*To be detailed after Phase 1 completion*

- Enhanced NFT metadata system
- Agent-NFT deep integration
- Ownership verification service
- Complete Secret AI integration

---

## ACCELERATED TIMELINE (Secret VM Ready + Shared Agent Model)

Since Secret VM is already configured and we're using a shared agent model with unique traits:

### Revised Architecture - ONE Agent, MANY Personalities

#### What Changed:
- **OLD**: Deploy separate agent for each NFT (too expensive)
- **NEW**: One shared agent that loads unique traits per NFT
- **BENEFIT**: $1,000/month instead of $20,000/month for 100 NFTs

### Week 1 Goals - Shared Agent with Traits

#### Priority Order:
1. **Shared Agent Service** (Days 1-2)
   - Python/FastAPI with Secret AI SDK
   - Session management per NFT
   - Deploy ONE instance to Secret VM

2. **Trait System** (Days 2-3)
   - Load traits from NFT metadata
   - Generate dynamic system prompts
   - Apply personality to responses

3. **Integration** (Days 4-5)
   - NFT ownership verification
   - Trait-based chat sessions
   - End-to-end test

### What We Can Skip:
- ✅ Secret VM setup tutorials
- ✅ TEE verification learning
- ✅ Deployment debugging
- ✅ GPU access testing

### What to Focus On NOW:
1. **Get the NFT contract working** - This is the foundation
2. **Build the agent container** - Simple version first
3. **Connect them in Secret VM** - You already know how to deploy

---

## Updates Log

### January 2025
- **[Today's Date]** - Created initial build plan, starting Phase 1
- **[Today's Date]** - Discovered Secret VM already configured - accelerating timeline!
- *[Updates will be added here as we progress]*
