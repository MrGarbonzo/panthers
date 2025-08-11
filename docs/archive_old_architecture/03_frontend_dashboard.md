# 03. Frontend Dashboard Implementation

## Overview
Next.js-based web application for users to manage their Secret Panther agents. Provides NFT viewing, deployment management, and chat interface.

## Directory Structure
```
/frontend/
├── package.json
├── next.config.js
├── tailwind.config.js
├── .env.local.example
├── public/
│   ├── images/
│   └── fonts/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx             # Landing page
│   │   ├── dashboard/
│   │   │   ├── page.tsx         # Main dashboard
│   │   │   └── layout.tsx
│   │   ├── chat/[nftId]/
│   │   │   └── page.tsx         # Chat interface
│   │   └── api/
│   │       └── [...routes].ts   # API routes if needed
│   ├── components/
│   │   ├── wallet/
│   │   │   ├── WalletConnect.tsx
│   │   │   └── WalletProvider.tsx
│   │   ├── nft/
│   │   │   ├── NFTCard.tsx
│   │   │   ├── NFTGrid.tsx
│   │   │   └── NFTDetails.tsx
│   │   ├── deployment/
│   │   │   ├── DeployButton.tsx
│   │   │   ├── DeploymentStatus.tsx
│   │   │   └── ProviderSelector.tsx
│   │   ├── chat/
│   │   │   ├── ChatWindow.tsx
│   │   │   ├── MessageList.tsx
│   │   │   └── MessageInput.tsx
│   │   └── common/
│   │       ├── Header.tsx
│   │       ├── Footer.tsx
│   │       └── LoadingSpinner.tsx
│   ├── hooks/
│   │   ├── useWallet.ts
│   │   ├── useNFTs.ts
│   │   ├── useDeployment.ts
│   │   └── useChat.ts
│   ├── lib/
│   │   ├── secretjs/
│   │   │   ├── client.ts
│   │   │   └── contracts.ts
│   │   ├── api/
│   │   │   ├── orchestrator.ts
│   │   │   └── agent.ts
│   │   └── utils/
│   │       ├── constants.ts
│   │       └── helpers.ts
│   ├── stores/
│   │   ├── walletStore.ts
│   │   ├── nftStore.ts
│   │   └── deploymentStore.ts
│   └── types/
│       ├── nft.ts
│       ├── deployment.ts
│       └── chat.ts
```

## Technical Stack

### Core Dependencies
```json
{
  "dependencies": {
    "next": "14.1.0",
    "react": "18.2.0",
    "react-dom": "18.2.0",
    "typescript": "5.3.3",
    "@cosmjs/stargate": "0.32.2",
    "secretjs": "1.12.0",
    "keplr-types": "0.12.35",
    "zustand": "4.5.0",
    "react-query": "3.39.3",
    "@tanstack/react-query": "5.17.9",
    "axios": "1.6.5",
    "socket.io-client": "4.6.0",
    "react-hot-toast": "2.4.1",
    "react-markdown": "9.0.1",
    "framer-motion": "11.0.3",
    "@headlessui/react": "1.7.17",
    "@heroicons/react": "2.1.1"
  },
  "devDependencies": {
    "@types/react": "18.2.48",
    "@types/node": "20.11.5",
    "tailwindcss": "3.4.1",
    "autoprefixer": "10.4.17",
    "postcss": "8.4.33",
    "eslint": "8.56.0",
    "prettier": "3.2.4"
  }
}
```

## Environment Configuration

### .env.local
```bash
# Secret Network
NEXT_PUBLIC_SECRET_RPC=https://lcd.testnet.secretsaturn.net
NEXT_PUBLIC_SECRET_CHAIN_ID=pulsar-3
NEXT_PUBLIC_NFT_CONTRACT=secret1abc...

# Orchestrator Service
NEXT_PUBLIC_ORCHESTRATOR_API=https://orchestrator.secretpanthers.com
NEXT_PUBLIC_ORCHESTRATOR_WS=wss://orchestrator.secretpanthers.com

# Features
NEXT_PUBLIC_ENABLE_TESTNET=true
NEXT_PUBLIC_ENABLE_MAINNET=false

# Analytics (optional)
NEXT_PUBLIC_GA_ID=G-XXXXXXXXXX
```

## Core Components

### 1. Wallet Connection (components/wallet/WalletConnect.tsx)
```typescript
interface WalletConnectProps {
  onConnect?: (address: string) => void;
}

/**
 * Wallet connection component
 * - Detects Keplr installation
 * - Handles connection flow
 * - Shows connection status
 * - Manages disconnection
 */

Key features:
- Auto-connect if previously connected
- Network switching (testnet/mainnet)
- Error handling for missing Keplr
- Mobile wallet support
```

### 2. NFT Display (components/nft/NFTCard.tsx)
```typescript
interface NFTCardProps {
  nft: PantherNFT;
  deployment?: Deployment;
  onDeploy: () => void;
  onChat: () => void;
}

/**
 * NFT card component
 * - Shows Panther image
 * - Displays token ID and name
 * - Shows deployment status
 * - Quick action buttons
 */

States:
- Not deployed: Show "Deploy" button
- Deploying: Show progress spinner
- Deployed: Show "Chat" and "Manage" buttons
- Failed: Show error and "Retry" button
```

### 3. Deployment Manager (components/deployment/DeployButton.tsx)
```typescript
interface DeployButtonProps {
  nftId: string;
  provider?: DeploymentProvider;
  onSuccess: (deployment: Deployment) => void;
  onError: (error: Error) => void;
}

/**
 * Deployment button with provider selection
 * - Provider dropdown (Secret VM, Akash, Local)
 * - Cost estimation
 * - Progress tracking
 * - Error handling
 */

Flow:
1. Select provider
2. Show cost estimate
3. Confirm deployment
4. Track progress via WebSocket
5. Show success/error
```

### 4. Chat Interface (components/chat/ChatWindow.tsx)
```typescript
interface ChatWindowProps {
  nftId: string;
  agentEndpoint: string;
}

/**
 * Full chat interface
 * - Message history
 * - Real-time messaging
 * - Markdown rendering
 * - Code highlighting
 * - File uploads (future)
 */

Features:
- Auto-scroll to bottom
- Typing indicators
- Message timestamps
- Copy code blocks
- Export conversation
```

## Pages

### 1. Landing Page (app/page.tsx)
```
Sections:
- Hero: "Your AI Agents, Truly Private"
- How it works (3 steps)
- Features showcase
- Live stats (agents deployed, messages sent)
- Connect wallet CTA
```

### 2. Dashboard (app/dashboard/page.tsx)
```
Layout:
- Header with wallet info
- NFT grid (owned Panthers)
- Deployment overview
- Quick stats (active agents, total chats)
- Recent activity feed
```

### 3. Chat Page (app/chat/[nftId]/page.tsx)
```
Layout:
- Minimal header
- Full-screen chat interface
- Agent info sidebar
- Settings panel
```

## State Management (Zustand)

### Wallet Store
```typescript
interface WalletStore {
  address: string | null;
  isConnected: boolean;
  isConnecting: boolean;
  connect: () => Promise<void>;
  disconnect: () => void;
  signMessage: (message: string) => Promise<string>;
}
```

### NFT Store
```typescript
interface NFTStore {
  nfts: PantherNFT[];
  isLoading: boolean;
  error: Error | null;
  fetchNFTs: (address: string) => Promise<void>;
  getNFT: (tokenId: string) => PantherNFT | undefined;
}
```

### Deployment Store
```typescript
interface DeploymentStore {
  deployments: Map<string, Deployment>;
  activeDeployments: string[];
  deployAgent: (nftId: string, provider: string) => Promise<void>;
  stopDeployment: (nftId: string) => Promise<void>;
  subscribeToUpdates: (nftId: string) => void;
}
```

## API Integration

### Orchestrator Client
```typescript
class OrchestratorClient {
  async deployAgent(params: DeployParams): Promise<Deployment>;
  async getDeployment(nftId: string): Promise<Deployment>;
  async stopDeployment(nftId: string): Promise<void>;
  async listDeployments(wallet: string): Promise<Deployment[]>;
  
  // WebSocket for real-time updates
  subscribeToDeployment(
    nftId: string,
    onUpdate: (update: DeploymentUpdate) => void
  ): () => void;
}
```

### Agent Client
```typescript
class AgentClient {
  constructor(private endpoint: string, private token: string);
  
  async authenticate(wallet: string, signature: string): Promise<AuthToken>;
  async sendMessage(message: string): Promise<ChatResponse>;
  async getHistory(conversationId?: string): Promise<Message[]>;
  async getInfo(): Promise<AgentInfo>;
}
```

## UI/UX Patterns

### Loading States
```typescript
// Skeleton loaders for NFT grid
<NFTGridSkeleton count={6} />

// Inline spinners for actions
<Button loading={isDeploying}>
  {isDeploying ? "Deploying..." : "Deploy Agent"}
</Button>

// Full-page loading for route transitions
<PageLoader message="Loading your agents..." />
```

### Error Handling
```typescript
// Toast notifications for actions
toast.error("Failed to deploy agent. Please try again.");
toast.success("Agent deployed successfully!");

// Inline errors for forms
<ErrorMessage error={deploymentError} />

// Full error boundaries
<ErrorBoundary fallback={<ErrorPage />}>
  <Component />
</ErrorBoundary>
```

### Responsive Design
```css
/* Mobile-first approach */
.grid {
  @apply grid grid-cols-1;
  @apply sm:grid-cols-2;
  @apply lg:grid-cols-3;
  @apply xl:grid-cols-4;
}

/* Adaptive layouts */
.sidebar {
  @apply hidden lg:block;
  @apply fixed lg:relative;
  @apply w-full lg:w-64;
}
```

## Performance Optimizations

### Code Splitting
```typescript
// Dynamic imports for heavy components
const ChatWindow = dynamic(() => import('@/components/chat/ChatWindow'), {
  loading: () => <ChatSkeleton />,
  ssr: false
});
```

### Data Fetching
```typescript
// React Query for caching and deduplication
const { data: nfts, isLoading } = useQuery({
  queryKey: ['nfts', wallet],
  queryFn: () => fetchNFTs(wallet),
  staleTime: 60000, // 1 minute
  cacheTime: 300000, // 5 minutes
});
```

### Image Optimization
```typescript
// Next.js Image component
<Image
  src={nft.imageUrl}
  alt={nft.name}
  width={400}
  height={400}
  placeholder="blur"
  blurDataURL={nft.blurHash}
/>
```

## Security Considerations

1. **Input Sanitization**: Sanitize all user inputs
2. **XSS Prevention**: Use React's built-in escaping
3. **CORS Configuration**: Whitelist allowed origins
4. **Secure Storage**: Never store private keys
5. **Content Security Policy**: Configure CSP headers
6. **API Authentication**: Validate all API calls

## Testing Strategy

### Unit Tests
```typescript
// Component testing with React Testing Library
describe('NFTCard', () => {
  it('shows deploy button when not deployed', () => {
    render(<NFTCard nft={mockNFT} />);
    expect(screen.getByText('Deploy')).toBeInTheDocument();
  });
});
```

### E2E Tests
```typescript
// Playwright tests
test('complete deployment flow', async ({ page }) => {
  await page.goto('/dashboard');
  await page.click('[data-testid="connect-wallet"]');
  await page.click('[data-testid="deploy-agent"]');
  await expect(page.locator('[data-testid="chat-window"]')).toBeVisible();
});
```

## Deployment

### Build Configuration
```javascript
// next.config.js
module.exports = {
  output: 'standalone',
  images: {
    domains: ['ipfs.io', 'gateway.pinata.cloud'],
  },
  env: {
    // Runtime environment variables
  },
};
```

### Hosting Options
1. **Vercel**: Automatic deployments, edge functions
2. **Netlify**: Static hosting with functions
3. **Self-hosted**: Docker container with Node.js
4. **IPFS**: Fully decentralized (static export)

## Analytics & Monitoring

### User Analytics
- Page views and user flows
- Wallet connection rates
- Deployment success rates
- Chat engagement metrics

### Error Tracking
- Sentry for error monitoring
- Custom error logging
- Performance monitoring
- User feedback collection
