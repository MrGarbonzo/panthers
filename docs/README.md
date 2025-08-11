# Secret Panthers Documentation Structure

## Active Documentation (Use These!)

### Implementation Plan
- `/docs/MVP_PLAN.md` - Complete MVP implementation roadmap with 10-day timeline

### Technical Specifications
- `/docs/technical/01_nft_integration_hybrid.md` - Frontend (SecretJS) + Backend (SecretPy) integration
- `/docs/technical/02_backend_nft_secretpy.md` - Backend NFT queries using permits
- `/docs/technical/03_secret_ai_integration.md` - Secret AI SDK with trait-based personalities
- `/docs/technical/04_session_management.md` - In-memory session management (no Redis!)
- `/docs/technical/05_multi_nft_switching.md` - Multi-NFT switching with conversation history

### Deployment
- `/docs/SECRETVM_DEPLOYMENT.md` - How to deploy using Secret AI Portal

## Architecture Summary

We use a **SHARED AGENT** architecture:
- One Secret AI agent shared by all users
- NFT traits modify the agent's personality per session
- In-memory session management (no database)
- Simple and cost-effective

## MVP Features
1. ✅ Keplr and MetaMask wallet connection
2. ✅ NFT ownership verification and trait reading
3. ✅ ChatGPT-style interface
4. ✅ Secret AI responses using NFT traits
5. ✅ Display NFT image as chat avatar
6. ✅ Multi-NFT switching for users with multiple Panthers
7. ✅ Docker deployment to Secret VM

## Technology Stack
- **Frontend**: Next.js 14, TypeScript, SecretJS
- **Backend**: Python 3.12, FastAPI, SecretPy
- **AI**: Secret AI SDK
- **Deployment**: Docker, Secret VM via Secret AI Portal
- **Chain**: Secret Network (testnet: pulsar-3, mainnet: secret-4)

## Getting Started for Development

1. Review `/docs/MVP_PLAN.md` for the complete roadmap
2. Check technical specs in `/docs/technical/` for detailed implementation
3. Use `/docs/SECRETVM_DEPLOYMENT.md` for deployment instructions

## Important Notes

- We do NOT use the multi-agent architecture from `/docs/specifications/` (outdated)
- We do NOT use Redis (in-memory sessions only)
- We do NOT store conversation history permanently
- We DO use a shared Secret AI agent with trait-based personalities
