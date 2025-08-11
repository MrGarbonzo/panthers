# Secret VM Deployment Guide

## Overview
This guide covers deploying our Secret Panthers application to Secret VM using Docker Compose and GitHub Actions.

## What is Secret VM?
SecretVM is the Confidential Virtual Machine framework of Secret Network, allowing developers to easily deploy and run secure workloads within Trusted Execution Environments (TEEs). Key features:
- Data Confidentiality via TEEs
- Remote Attestation for verification
- Language & Stack Agnostic (Docker support)

## Prerequisites

### 1. Secret VM Access
- Request access to Secret VM infrastructure
- Obtain VM credentials and endpoint
- Get attestation certificates

### 2. Development Environment
Based on Secret Network requirements:
- Docker installed locally
- Git for version control
- SecretCLI for blockchain interaction
- Node.js for frontend development
- Python 3.12 for backend

## Docker Configuration

### Backend Dockerfile
```dockerfile
# backend/Dockerfile
FROM python:3.12-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8080/health')" || exit 1

# Run application
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Frontend Dockerfile
```dockerfile
# frontend/Dockerfile
FROM node:18-alpine AS builder

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM node:18-alpine

WORKDIR /app

# Copy built application
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public
COPY --from=builder /app/package*.json ./

# Install production dependencies only
RUN npm ci --only=production

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/api/health', (r) => {r.statusCode === 200 ? process.exit(0) : process.exit(1)})" || exit 1

# Run application
CMD ["npm", "start"]
```

## Docker Compose Configuration

### docker-compose.yml
```yaml
version: '3.8'

services:
  backend:
    image: ghcr.io/${GITHUB_REPOSITORY}/panthers-backend:${VERSION:-latest}
    container_name: panthers-backend
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      # Chain Configuration
      CHAIN_ID: ${CHAIN_ID:-pulsar-3}
      LCD_URL: ${LCD_URL}
      
      # NFT Contract
      NFT_CONTRACT_ADDRESS: ${NFT_CONTRACT_ADDRESS}
      NFT_CONTRACT_HASH: ${NFT_CONTRACT_HASH}
      
      # Secret AI
      SECRET_AI_API_KEY: ${SECRET_AI_API_KEY}
      
      # Security
      JWT_SECRET: ${JWT_SECRET}
      
      # Environment
      ENVIRONMENT: ${ENVIRONMENT:-testnet}
    networks:
      - panthers-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  frontend:
    image: ghcr.io/${GITHUB_REPOSITORY}/panthers-frontend:${VERSION:-latest}
    container_name: panthers-frontend
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://backend:8080
      NEXT_PUBLIC_CHAIN_ID: ${CHAIN_ID:-pulsar-3}
      NEXT_PUBLIC_NFT_CONTRACT: ${NFT_CONTRACT_ADDRESS}
    depends_on:
      backend:
        condition: service_healthy
    networks:
      - panthers-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Optional: Nginx reverse proxy
  nginx:
    image: nginx:alpine
    container_name: panthers-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
      - backend
    networks:
      - panthers-network

networks:
  panthers-network:
    driver: bridge

volumes:
  ssl_certs:
```

## GitHub Actions Workflow

### .github/workflows/deploy.yml
```yaml
name: Build and Deploy to Secret VM

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
        
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
        
      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha
            
      - name: Build and push backend
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-backend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          
      - name: Build and push frontend
        uses: docker/build-push-action@v4
        with:
          context: ./frontend
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}-frontend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to Secret VM
        uses: appleboy/ssh-action@v0.1.5
        with:
          host: ${{ secrets.SECRET_VM_HOST }}
          username: ${{ secrets.SECRET_VM_USER }}
          key: ${{ secrets.SECRET_VM_SSH_KEY }}
          script: |
            # Pull latest images
            docker-compose pull
            
            # Stop existing containers
            docker-compose down
            
            # Start new containers
            docker-compose up -d
            
            # Prune old images
            docker image prune -f
            
            # Health check
            sleep 10
            curl -f http://localhost:8080/health || exit 1
            curl -f http://localhost:3000/api/health || exit 1
```

## Secret Management

### Using Docker Secrets
For sensitive data in production:

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  backend:
    secrets:
      - secret_ai_key
      - jwt_secret
      - nft_contract_hash
    environment:
      SECRET_AI_API_KEY_FILE: /run/secrets/secret_ai_key
      JWT_SECRET_FILE: /run/secrets/jwt_secret
      NFT_CONTRACT_HASH_FILE: /run/secrets/nft_contract_hash

secrets:
  secret_ai_key:
    external: true
  jwt_secret:
    external: true
  nft_contract_hash:
    external: true
```

### Creating Secrets on Secret VM
```bash
# SSH into Secret VM
ssh user@secret-vm-host

# Create secrets
echo "your-secret-ai-key" | docker secret create secret_ai_key -
echo "your-jwt-secret" | docker secret create jwt_secret -
echo "your-contract-hash" | docker secret create nft_contract_hash -
```

## Environment Configuration

### .env.production
```bash
# Chain Configuration
CHAIN_ID=secret-4
LCD_URL=https://lcd.mainnet.secretsaturn.net
ENVIRONMENT=mainnet

# NFT Contract (Mainnet)
NFT_CONTRACT_ADDRESS=secret1...
NFT_CONTRACT_HASH=...

# GitHub Repository
GITHUB_REPOSITORY=your-org/secret-panthers

# Version
VERSION=1.0.0
```

### .env.testnet
```bash
# Chain Configuration
CHAIN_ID=pulsar-3
LCD_URL=https://lcd.testnet.secretsaturn.net
ENVIRONMENT=testnet

# NFT Contract (Testnet)
NFT_CONTRACT_ADDRESS=secret1...
NFT_CONTRACT_HASH=...

# GitHub Repository
GITHUB_REPOSITORY=your-org/secret-panthers

# Version
VERSION=latest
```

## Deployment Steps

### 1. Initial Setup
```bash
# Clone repository
git clone https://github.com/your-org/secret-panthers.git
cd secret-panthers

# Set up environment
cp .env.testnet .env
```

### 2. Build Docker Images Locally
```bash
# Build backend
docker build -t panthers-backend ./backend

# Build frontend
docker build -t panthers-frontend ./frontend

# Test locally
docker-compose up -d
```

### 3. Push to GitHub
```bash
# Commit changes
git add .
git commit -m "Initial deployment"

# Push to trigger CI/CD
git push origin main
```

### 4. Manual Deployment (Alternative)
```bash
# SSH into Secret VM
ssh user@secret-vm-host

# Clone repository
git clone https://github.com/your-org/secret-panthers.git
cd secret-panthers

# Copy environment file
cp .env.production .env

# Start services
docker-compose up -d

# Check status
docker-compose ps
docker-compose logs -f
```

## Monitoring and Maintenance

### Health Checks
```bash
# Check service health
curl http://localhost:8080/health
curl http://localhost:3000/api/health

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Monitor resources
docker stats
```

### Updates and Rollbacks
```bash
# Update to latest version
docker-compose pull
docker-compose up -d

# Rollback to previous version
docker-compose down
docker tag panthers-backend:latest panthers-backend:backup
docker-compose up -d
```

### Backup Strategy
```bash
# No persistent data needed (sessions are in-memory)
# Only need to backup environment configuration

# Backup env file
cp .env .env.backup-$(date +%Y%m%d)
```

## Security Considerations

### 1. TEE Verification
- Verify Secret VM attestation
- Ensure code runs in secure enclave
- Validate remote attestation reports

### 2. Network Security
- Use HTTPS/TLS for all connections
- Configure firewall rules
- Limit exposed ports

### 3. Container Security
- Run containers as non-root
- Use minimal base images
- Regular security updates
- Scan images for vulnerabilities

### 4. Secret Management
- Never commit secrets to Git
- Use Docker secrets or environment files
- Rotate secrets regularly
- Limit secret access to needed services

## Troubleshooting

### Common Issues

#### Container Won't Start
```bash
# Check logs
docker-compose logs backend
docker-compose logs frontend

# Check environment variables
docker-compose config

# Verify image availability
docker images
```

#### Connection Issues
```bash
# Test internal networking
docker exec panthers-frontend ping backend

# Check exposed ports
netstat -tulpn | grep -E '3000|8080'

# Verify DNS resolution
docker exec panthers-frontend nslookup backend
```

#### Performance Issues
```bash
# Check resource usage
docker stats

# Inspect container limits
docker inspect panthers-backend | grep -A 5 "Resources"

# Clean up unused resources
docker system prune -a
```

## Best Practices

1. **Version Control**
   - Tag releases properly
   - Use semantic versioning
   - Keep changelog updated

2. **CI/CD**
   - Automate builds and deployments
   - Run tests before deployment
   - Use staging environment

3. **Monitoring**
   - Set up logging aggregation
   - Monitor health endpoints
   - Alert on failures

4. **Documentation**
   - Keep deployment docs updated
   - Document environment variables
   - Maintain runbooks

## Questions to Resolve

1. **Secret VM Access**: How do we get access to Secret VM infrastructure?
2. **Attestation**: What's the process for TEE attestation verification?
3. **Domain/SSL**: What domain and SSL certificate approach?
4. **Monitoring**: Which monitoring solution to use?
5. **Backup**: Do we need any backup strategy beyond config files?

## Next Steps

1. Request Secret VM access
2. Set up GitHub repository
3. Configure GitHub Actions secrets
4. Deploy to testnet first
5. Test thoroughly
6. Deploy to mainnet

## Support Resources

- Secret Network Docs: https://docs.scrt.network
- Secret VM Documentation: [Need specific URL]
- Discord/Telegram: Secret Network Developer Community
- GitHub Issues: Report deployment issues
