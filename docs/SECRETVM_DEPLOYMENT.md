# Secret VM Deployment Guide (Updated)

## Overview
This guide covers deploying our Secret Panthers application to Secret VM using the Secret AI Portal or CLI with our Docker Compose configuration.

## Deployment Methods

### Method 1: Secret AI Portal (Recommended)

#### Step 1: Access the Portal
1. Visit https://secretai.scrtlabs.com
2. Sign in with:
   - Your wallet (Keplr/MetaMask) OR
   - Your Google account

#### Step 2: Create New SecretVM
1. Navigate to `SecretVMs` in the left sidebar
2. Click `Create New SecretVM` button

#### Step 3: Configure Your Machine
1. **Select VM Type**
   - Choose appropriate size based on expected load:
     - Small: Testing/Development (up to 100 users)
     - Medium: Production (100-500 users)
     - Large: High traffic (500+ users)

2. **Upload Docker Compose File**
   - Upload our `docker-compose.yaml` file (see below)

3. **Set Secret Environment Variables**
   - Add these secure environment variables:
   ```
   SECRET_AI_API_KEY=<your-key>
   JWT_SECRET=<generate-strong-secret>
   NFT_CONTRACT_ADDRESS=<contract-address>
   NFT_CONTRACT_HASH=<contract-hash>
   LCD_URL=<lcd-endpoint>
   ```

#### Step 4: Launch the VM
1. Click `Launch your SecretVM`
2. Monitor provisioning status
3. Access attestation endpoints once live

### Method 2: Secret AI CLI (Alternative)
*Documentation pending - will update when CLI instructions are available*

## Docker Compose Configuration

### Production docker-compose.yaml
```yaml
version: '3.8'

services:
  backend:
    image: ghcr.io/YOUR_ORG/panthers-backend:latest
    container_name: panthers-backend
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      # Chain Configuration (set via Secret AI Portal)
      CHAIN_ID: ${CHAIN_ID}
      LCD_URL: ${LCD_URL}
      
      # NFT Contract (set via Secret AI Portal)
      NFT_CONTRACT_ADDRESS: ${NFT_CONTRACT_ADDRESS}
      NFT_CONTRACT_HASH: ${NFT_CONTRACT_HASH}
      
      # Secret AI (set via Secret AI Portal)
      SECRET_AI_API_KEY: ${SECRET_AI_API_KEY}
      
      # Security (set via Secret AI Portal)
      JWT_SECRET: ${JWT_SECRET}
      
      # Environment
      ENVIRONMENT: ${ENVIRONMENT}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  frontend:
    image: ghcr.io/YOUR_ORG/panthers-frontend:latest
    container_name: panthers-frontend
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NEXT_PUBLIC_API_URL: http://backend:8080
      NEXT_PUBLIC_CHAIN_ID: ${CHAIN_ID}
      NEXT_PUBLIC_NFT_CONTRACT: ${NFT_CONTRACT_ADDRESS}
    depends_on:
      backend:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Nginx reverse proxy for production
  nginx:
    image: nginx:alpine
    container_name: panthers-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - frontend
      - backend
```

## Pre-Deployment Checklist

### 1. Build and Push Docker Images
```bash
# Build and push to GitHub Container Registry
docker build -t ghcr.io/YOUR_ORG/panthers-backend:latest ./backend
docker build -t ghcr.io/YOUR_ORG/panthers-frontend:latest ./frontend

docker push ghcr.io/YOUR_ORG/panthers-backend:latest
docker push ghcr.io/YOUR_ORG/panthers-frontend:latest
```

### 2. Prepare Environment Variables
Create a list of environment variables for the Secret AI Portal:

#### For Testnet
```
CHAIN_ID=pulsar-3
LCD_URL=https://lcd.testnet.secretsaturn.net
ENVIRONMENT=testnet
NFT_CONTRACT_ADDRESS=<testnet-contract>
NFT_CONTRACT_HASH=<testnet-hash>
SECRET_AI_API_KEY=bWFzdGVyQHNjcnRsYWJzLmNvbTpTZWNyZXROZXR3b3JrTWFzdGVyS2V5X18yMDI1
JWT_SECRET=<generate-random-secret>
```

#### For Mainnet
```
CHAIN_ID=secret-4
LCD_URL=https://lcd.mainnet.secretsaturn.net
ENVIRONMENT=mainnet
NFT_CONTRACT_ADDRESS=<mainnet-contract>
NFT_CONTRACT_HASH=<mainnet-hash>
SECRET_AI_API_KEY=<production-key>
JWT_SECRET=<generate-strong-secret>
```

### 3. Prepare Docker Compose File
Ensure your `docker-compose.yaml`:
- Uses correct image registry URLs
- Has proper health checks
- Includes all necessary services
- Uses environment variable placeholders

## GitHub Actions for Automated Image Building

### .github/workflows/build.yml
```yaml
name: Build and Push Docker Images

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io

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
        
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: |
            ${{ env.REGISTRY }}/${{ github.repository }}-backend
            ${{ env.REGISTRY }}/${{ github.repository }}-frontend
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest,enable={{is_default_branch}}
            
      - name: Build and push backend
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ env.REGISTRY }}/${{ github.repository_owner }}/panthers-backend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
          
      - name: Build and push frontend
        uses: docker/build-push-action@v4
        with:
          context: ./frontend
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ env.REGISTRY }}/${{ github.repository_owner }}/panthers-frontend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Nginx Configuration

### nginx/nginx.conf
```nginx
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server backend:8080;
    }
    
    upstream frontend {
        server frontend:3000;
    }
    
    server {
        listen 80;
        server_name _;
        
        # Redirect to HTTPS (if SSL configured)
        # return 301 https://$server_name$request_uri;
        
        # API routes
        location /api {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        # Frontend routes
        location / {
            proxy_pass http://frontend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
    
    # HTTPS configuration (when SSL is ready)
    # server {
    #     listen 443 ssl http2;
    #     server_name your-domain.com;
    #     
    #     ssl_certificate /etc/nginx/ssl/cert.pem;
    #     ssl_certificate_key /etc/nginx/ssl/key.pem;
    #     
    #     # ... same proxy configuration as above
    # }
}
```

## Deployment Steps

### Phase 1: Testnet Deployment

1. **Build Images**
   ```bash
   # Tag with testnet version
   docker build -t ghcr.io/YOUR_ORG/panthers-backend:testnet ./backend
   docker build -t ghcr.io/YOUR_ORG/panthers-frontend:testnet ./frontend
   
   # Push to registry
   docker push ghcr.io/YOUR_ORG/panthers-backend:testnet
   docker push ghcr.io/YOUR_ORG/panthers-frontend:testnet
   ```

2. **Create SecretVM via Portal**
   - Log into https://secretai.scrtlabs.com
   - Create new SecretVM
   - Select appropriate VM size
   - Upload docker-compose.yaml
   - Add testnet environment variables
   - Launch VM

3. **Verify Deployment**
   - Check attestation endpoints
   - Test wallet connection
   - Verify NFT queries work
   - Test chat functionality

### Phase 2: Mainnet Deployment

1. **Update Images**
   ```bash
   # Tag with version
   docker build -t ghcr.io/YOUR_ORG/panthers-backend:v1.0.0 ./backend
   docker build -t ghcr.io/YOUR_ORG/panthers-frontend:v1.0.0 ./frontend
   
   # Also tag as latest
   docker tag ghcr.io/YOUR_ORG/panthers-backend:v1.0.0 ghcr.io/YOUR_ORG/panthers-backend:latest
   docker tag ghcr.io/YOUR_ORG/panthers-frontend:v1.0.0 ghcr.io/YOUR_ORG/panthers-frontend:latest
   
   # Push all tags
   docker push ghcr.io/YOUR_ORG/panthers-backend:v1.0.0
   docker push ghcr.io/YOUR_ORG/panthers-backend:latest
   docker push ghcr.io/YOUR_ORG/panthers-frontend:v1.0.0
   docker push ghcr.io/YOUR_ORG/panthers-frontend:latest
   ```

2. **Create Production SecretVM**
   - Use larger VM size for production
   - Upload production docker-compose.yaml
   - Add mainnet environment variables
   - Configure domain/SSL if available

## Monitoring After Deployment

### Health Checks
Once your SecretVM is running, monitor health:

```bash
# Check backend health
curl http://YOUR_VM_IP:8080/health

# Check frontend health
curl http://YOUR_VM_IP:3000/api/health

# Check full flow
curl http://YOUR_VM_IP:3000
```

### Viewing Logs
Access logs through Secret AI Portal or if SSH access is provided:

```bash
# View backend logs
docker logs panthers-backend

# View frontend logs
docker logs panthers-frontend

# Follow logs
docker logs -f panthers-backend
```

### Attestation Verification
The Secret AI Portal provides attestation endpoints to verify:
- TEE is genuine
- Code hasn't been tampered with
- Running in secure enclave

## Updates and Maintenance

### Updating the Application
1. Build new Docker images with new tag
2. Push to GitHub Container Registry
3. In Secret AI Portal:
   - Navigate to your SecretVM
   - Update docker-compose.yaml with new image tags
   - Redeploy

### Rolling Back
If issues occur:
1. Keep previous image tags available
2. Update docker-compose.yaml to use previous version
3. Redeploy through portal

## Cost Considerations

### VM Sizing Guide
- **Small VM**: Development/Testing
  - Up to 100 concurrent users
  - 2 vCPU, 4GB RAM
  
- **Medium VM**: Small Production
  - 100-500 concurrent users
  - 4 vCPU, 8GB RAM
  
- **Large VM**: Full Production
  - 500+ concurrent users
  - 8 vCPU, 16GB RAM

### Optimization Tips
- Use in-memory sessions (no database costs)
- Implement proper caching
- Optimize Docker images (multi-stage builds)
- Monitor resource usage

## Security Best Practices

1. **Environment Variables**
   - Never commit secrets to Git
   - Use Secret AI Portal's secure variable storage
   - Rotate secrets regularly

2. **Docker Images**
   - Keep base images updated
   - Scan for vulnerabilities
   - Use non-root users in containers

3. **Network Security**
   - Enable HTTPS when domain is ready
   - Use proper firewall rules
   - Limit exposed ports

4. **Attestation**
   - Verify attestation after deployment
   - Share attestation proof with users
   - Monitor for any attestation failures

## Troubleshooting

### Common Issues

1. **Images won't pull**
   - Verify registry URL is correct
   - Check image tags exist
   - Ensure images are public or credentials are configured

2. **Services won't start**
   - Check environment variables are set
   - Verify health checks are passing
   - Review container logs

3. **Can't connect to services**
   - Verify ports are exposed correctly
   - Check VM firewall rules
   - Ensure services are healthy

## Support

- **Secret AI Portal Issues**: Contact through portal support
- **Application Issues**: GitHub Issues on your repository
- **Secret Network Help**: Discord/Telegram developer community

## Next Steps

1. ✅ Create account on https://secretai.scrtlabs.com
2. ✅ Build and push Docker images
3. ✅ Prepare docker-compose.yaml
4. ✅ Gather environment variables
5. ✅ Deploy to testnet first
6. ✅ Test thoroughly
7. ✅ Deploy to mainnet
8. ✅ Share attestation with community
