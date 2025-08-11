# Secret Panthers MVP Implementation Plan

## MVP Scope Definition

### Core Requirements
1. ✅ Keplr and MetaMask wallet connection
2. ✅ NFT ownership verification and trait reading
3. ✅ ChatGPT-style interface for conversations
4. ✅ Secret AI responses using NFT traits
5. ✅ Display NFT image as chat avatar
6. ✅ Deploy to Secret VM via Docker Compose and GitHub Actions
7. ✅ Multi-NFT switching (if user owns multiple Panthers)

### What's NOT in MVP
- ❌ Conversation history persistence
- ❌ User preferences/settings
- ❌ Analytics/metrics
- ❌ Mobile app
- ❌ Advanced features

## System Architecture for MVP

```
Frontend (Next.js)
    ├── Wallet Connection (Keplr/MetaMask)
    ├── NFT Query & Display
    └── Chat Interface
            ↓
    Backend (FastAPI)
    ├── NFT Verification (SecretPy)
    ├── Session Management (In-Memory)
    └── Secret AI Integration
            ↓
    Deployment (Secret VM)
    └── Docker Compose Stack
```

## Component Breakdown

### 1. Frontend Components

#### Wallet Connection Component
```
/components/WalletConnect.tsx
- Detect Keplr/MetaMask
- Connect button
- Show connected address
- Handle disconnection
```

#### NFT Display Component
```
/components/NFTDisplay.tsx
- Show current NFT image
- Display token ID
- Show basic traits
- Loading states
```

#### NFT Selector Component
```
/components/NFTSelector.tsx
- Query all owned NFTs
- Display NFT grid/list
- Show current selection
- Handle NFT switching
- Update chat context on switch
```

#### Chat Interface Component
```
/components/ChatInterface.tsx
- Message input
- Message display
- NFT image as avatar
- Typing indicators
- Auto-scroll
```

### 2. Backend Services

#### API Structure
```
/api
├── /auth
│   └── POST /verify-nft    # Verify ownership & create session
├── /chat
│   └── POST /message        # Send message to AI
└── /health
    └── GET /                # Health check
```

#### Core Services
```
/services
├── nft_service.py          # Query NFT & traits
├── session_service.py       # In-memory sessions
├── ai_service.py           # Secret AI integration
└── auth_service.py         # JWT generation
```

### 3. Docker Configuration

#### docker-compose.yml
```yaml
version: '3.8'
services:
  backend:
    image: ghcr.io/[org]/panthers-backend:latest
    ports:
      - "8080:8080"
    environment:
      - CHAIN_ID=pulsar-3
      - SECRET_AI_API_KEY=${SECRET_AI_API_KEY}
      - NFT_CONTRACT_ADDRESS=${NFT_CONTRACT_ADDRESS}
    restart: unless-stopped

  frontend:
    image: ghcr.io/[org]/panthers-frontend:latest
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8080
      - NEXT_PUBLIC_CHAIN_ID=pulsar-3
    depends_on:
      - backend
    restart: unless-stopped
```

## Implementation Phases

### Phase 1: Backend Core (Days 1-3)
**Goal: Working API with NFT verification**

Day 1:
- Set up FastAPI project structure
- Implement NFT verification with SecretPy
- Create session management

Day 2:
- Integrate Secret AI SDK
- Implement trait-based prompts
- Create chat endpoint

Day 3:
- Add JWT authentication
- Complete API endpoints
- Local testing with mock NFT

### Phase 2: Frontend Core (Days 4-6)
**Goal: Working UI with wallet connection**

Day 4:
- Set up Next.js project
- Implement wallet connection (Keplr/MetaMask)
- Create basic layout

Day 5:
- Build chat interface
- Add NFT image display
- Connect to backend API

Day 6:
- Polish UI/UX
- Add loading states
- Error handling

### Phase 3: Integration (Days 7-8)
**Goal: Complete flow working end-to-end**

Day 7:
- Full integration testing
- Fix connection issues
- Optimize performance

Day 8:
- User flow testing
- Bug fixes
- Final adjustments

### Phase 4: Deployment (Days 9-10)
**Goal: Running on Secret VM**

Day 9:
- Create Dockerfiles
- Set up GitHub Actions
- Build and push images

Day 10:
- Deploy to Secret VM
- Configure domain/SSL
- Production testing

## File Structure

```
secret-panthers/
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py
│   ├── api/
│   │   ├── auth.py
│   │   ├── chat.py
│   │   └── health.py
│   ├── services/
│   │   ├── nft_service.py
│   │   ├── session_service.py
│   │   ├── ai_service.py
│   │   └── auth_service.py
│   └── models/
│       ├── requests.py
│       └── responses.py
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── next.config.js
│   ├── pages/
│   │   ├── index.tsx
│   │   └── api/
│   ├── components/
│   │   ├── WalletConnect.tsx
│   │   ├── NFTDisplay.tsx
│   │   ├── ChatInterface.tsx
│   │   └── Layout.tsx
│   ├── hooks/
│   │   ├── useWallet.ts
│   │   └── useChat.ts
│   └── utils/
│       ├── secretjs.ts
│       └── api.ts
│
├── docker-compose.yml
├── .github/
│   └── workflows/
│       └── deploy.yml
└── README.md
```

## GitHub Actions Workflow

### .github/workflows/deploy.yml
```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push backend
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/panthers-backend:latest
      
      - name: Build and push frontend
        uses: docker/build-push-action@v4
        with:
          context: ./frontend
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/panthers-frontend:latest
```

## Environment Variables

### Backend (.env)
```bash
# Chain Configuration
CHAIN_ID=pulsar-3
LCD_URL=https://lcd.testnet.secretsaturn.net

# NFT Contract
NFT_CONTRACT_ADDRESS=secret1...
NFT_CONTRACT_HASH=...

# Secret AI
SECRET_AI_API_KEY=bWFzdGVyQHNjcnRsYWJzLmNvbTpTZWNyZXROZXR3b3JrTWFzdGVyS2V5X18yMDI1

# Security
JWT_SECRET=generate-strong-secret
```

### Frontend (.env.local)
```bash
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_CHAIN_ID=pulsar-3

# Contract
NEXT_PUBLIC_NFT_CONTRACT=secret1...
```

## Testing Plan

### Unit Tests
- NFT verification logic
- Trait extraction
- Session management
- API endpoints

### Integration Tests
- Wallet connection flow
- NFT query and display
- Chat conversation flow
- End-to-end user journey

### User Acceptance Tests
1. Connect Keplr wallet ✓
2. Verify NFT ownership ✓
3. See NFT image displayed ✓
4. Send chat message ✓
5. Receive trait-based response ✓
6. Disconnect and reconnect ✓

## Deployment Checklist

### Pre-Deployment
- [ ] All tests passing
- [ ] Environment variables configured
- [ ] Docker images building
- [ ] GitHub Actions workflow ready

### Deployment Steps
1. [ ] Push code to GitHub
2. [ ] GitHub Actions builds images
3. [ ] SSH to Secret VM
4. [ ] Pull docker-compose.yml
5. [ ] Set environment variables
6. [ ] Run `docker-compose up -d`
7. [ ] Verify health endpoints
8. [ ] Test with real NFT

### Post-Deployment
- [ ] Monitor logs
- [ ] Test all features
- [ ] Check performance
- [ ] Verify Secret VM attestation

## Success Criteria

### Technical Success
- Wallet connection works for both Keplr and MetaMask
- NFT ownership correctly verified
- Traits successfully extracted and applied
- Chat responses reflect NFT personality
- System handles 50+ concurrent users
- Deploys successfully to Secret VM

### User Experience Success
- Connection process < 5 seconds
- Chat response time < 2 seconds
- NFT image loads properly
- Interface is intuitive
- No critical errors

## Risk Mitigation

### Technical Risks
| Risk | Mitigation |
|------|------------|
| NFT query fails | Implement retry logic |
| Secret AI unavailable | Error message with retry |
| Wallet connection issues | Support multiple wallets |
| Secret VM deployment fails | Have backup deployment ready |

### Timeline Risks
| Risk | Mitigation |
|------|------------|
| Integration delays | Start integration early |
| Bug fixes take longer | Time buffer in schedule |
| Testing reveals issues | Continuous testing |

## Next Steps

### Immediate Actions
1. Set up development environment
2. Get NFT contract address for testing
3. Create GitHub repository
4. Start backend development

### Week 1 Goals
- Backend API complete
- Frontend wallet connection working
- Basic chat interface ready

### Week 2 Goals
- Full integration complete
- Docker deployment working
- Ready for Secret VM deployment

## Questions to Resolve

1. Exact NFT contract address for testing?
2. Which NFT collection for initial testing?
3. Domain name for production?
4. SSL certificate approach?
5. Monitoring/logging solution?

## Definition of Done

The MVP is complete when:
1. ✅ User can connect Keplr or MetaMask
2. ✅ System verifies NFT ownership
3. ✅ NFT traits are loaded
4. ✅ Chat interface works
5. ✅ Responses use NFT personality
6. ✅ NFT image displays as avatar
7. ✅ Deployed to Secret VM
8. ✅ Accessible via public URL
