# Technical Specification: NFT Integration (Hybrid Approach)

## Overview
We use a hybrid approach with SecretJS on the frontend for wallet interactions and SecretPy on the backend for server-side NFT queries and Secret AI integration.

## Architecture

```
Frontend (Next.js + SecretJS)
    ├── Wallet Connection (Keplr & MetaMask)
    ├── Permit Signing
    └── Initial Authentication
              ↓
Backend (Python + SecretPy)
    ├── NFT Ownership Verification
    ├── Metadata Queries
    ├── Trait Loading
    └── Secret AI Integration
```

## Frontend: SecretJS Integration

### Technology Stack
- SecretJS library for Secret Network interaction
- Next.js 14 for React framework
- Wallet Support: **Keplr** and **MetaMask**
- Chain: **secret-4** (mainnet) or **pulsar-3** (testnet)

### Setup Requirements

#### For Mainnet:
- Chain ID: `secret-4`
- LCD URL: Get from https://github.com/scrtlabs/api-registry

#### For Testnet:
- Chain ID: `pulsar-3`
- LCD URL: Get from https://github.com/scrtlabs/api-registry

#### Wallet Support:
- **Keplr**: Desktop support, no mobile yet, supports Ledger
- **MetaMask**: Desktop and mobile support, supports Ledger

### Key Frontend Responsibilities

#### 1. Wallet Connection

##### Keplr Connection:
- Use `window.keplr.getOfflineSignerAuto()` for automatic Ledger/non-Ledger support
- Get encryption utils with `window.keplr.getEnigmaUtils(CHAIN_ID)`
- This allows transaction decryption across sessions

##### MetaMask Connection:
- Request accounts with `window.ethereum.request({ method: "eth_requestAccounts" })`
- Create wallet with `MetaMaskWallet.create(window.ethereum, ethAddress)`
- Consider passing `encryptionSeed` for consistent encryption across sessions

#### 2. Permit Generation
Generate a permit that allows the backend to query NFT data on behalf of the user.

For SNIP-721 permits:
- Method: `secretjs.utils.accessControl.permit.sign()`
- Parameters: address, chainId, permit name, contracts array, permissions array
- The permit is signed by the user's wallet (Keplr or MetaMask)

Permit allows backend to:
- Query owned tokens
- Get NFT metadata
- Verify ownership

#### 3. Authentication Flow
1. User connects wallet (Keplr or MetaMask)
2. Frontend requests user to sign a permit
3. Permit sent to backend with authentication request
4. Backend verifies and creates session

### Frontend Implementation Requirements

#### Wallet Detection and Connection

1. **Detect Available Wallets**:
   - Check for `window.keplr` (Keplr)
   - Check for `window.ethereum` (MetaMask)
   - Show appropriate connection buttons

2. **Initialize SecretJS Client**:
   ```
   For Keplr:
   - wallet: keplrOfflineSigner
   - encryptionUtils: window.keplr.getEnigmaUtils(chainId)
   
   For MetaMask:
   - wallet: MetaMaskWallet instance
   - encryptionSeed: optional for session persistence
   ```

3. **Client Configuration**:
   - url: from api-registry
   - chainId: "secret-4" or "pulsar-3"
   - wallet: wallet instance
   - walletAddress: user's address

### Frontend Data Flow

```
User Action: "Connect Wallet"
    ↓
Detect Wallet Type (Keplr/MetaMask)
    ↓
Initialize Appropriate Wallet
    ↓
Get Wallet Address
    ↓
User Action: "Access My Panther"
    ↓
Generate & Sign Permit (for NFT contract)
    ↓
Send to Backend:
{
  wallet_address: "secret1...",
  permit: {signed_permit_object},
  nft_token_id: "123",
  wallet_type: "keplr" | "metamask"
}
```

## Backend: SecretPy Integration

### Technology Stack
- SecretPy for Secret Network interaction
- Python 3.12 with FastAPI
- Redis for session management
- Secret AI SDK for LLM integration

### Setup Requirements
- SecretPy library installation
- Access to Secret Network LCD
- NFT contract address and code hash (when available)

### Key Backend Responsibilities

#### 1. Permit Verification
Receive permit from frontend and use it to query NFT data.

#### 2. NFT Ownership Verification
Using the permit, verify the user owns the claimed NFT.

#### 3. Metadata Extraction
Query NFT metadata to get trait information:
```
Expected Metadata Structure:
{
  "public_metadata": {
    "extension": {
      "image": "panther_image_url",
      "name": "Panther #123"
    }
  },
  "private_metadata": {
    "extension": {
      "traits": {
        "personality": "degen",
        "speaking_style": "meme",
        "expertise": ["defi", "gaming"],
        "modifiers": {
          "temperature": 0.8,
          "humor": "heavy"
        }
      }
    }
  }
}
```

#### 4. Trait Application
Load traits from metadata and apply to Secret AI prompts.

### Backend Data Flow

```
Receive from Frontend:
{permit, wallet_address, nft_token_id, wallet_type}
    ↓
Use Permit to Query NFT Contract
    ↓
Verify Ownership
    ↓
Extract Traits from Metadata
    ↓
Create Session with Traits
    ↓
Return JWT to Frontend
```

## SNIP-721 Specific Queries

### Queries We Need (Using Permit)

#### 1. Get Owned Tokens
Verify user owns the specific NFT they claim.

#### 2. Get NFT Info
Retrieve both public and private metadata containing traits.

#### 3. Token Details
Get specific information about the NFT including metadata extensions.

## Session Management

### Session Creation
1. Frontend sends permit + NFT ID + wallet type
2. Backend verifies ownership via permit
3. Backend loads traits from NFT metadata
4. Session created with traits cached
5. JWT returned to frontend

### Session Structure
```
Session {
  session_id: unique_identifier,
  wallet_address: user_wallet,
  wallet_type: "keplr" | "metamask",
  nft_token_id: panther_id,
  traits: loaded_from_nft,
  permit: stored_for_session_duration,
  expires_at: timestamp
}
```

## API Endpoints

### Authentication Endpoint
```
POST /api/auth/verify-nft

Request:
{
  wallet_address: string,
  wallet_type: "keplr" | "metamask",
  nft_token_id: string,
  permit: object  // Signed permit from SecretJS
}

Response:
{
  access_token: JWT,
  session_id: string,
  traits_summary: object,  // Basic trait info for UI
  expires_in: seconds
}
```

### Chat Endpoint
```
POST /api/chat

Headers:
  Authorization: Bearer {JWT}

Request:
{
  message: string
}

Response:
{
  response: string,  // AI response with traits applied
  session_id: string
}
```

## Security Considerations

### Permit Handling
- Permits are temporary and session-specific
- Never store permits permanently
- Validate permit signature on backend
- Refresh permits as needed

### Session Security
- Short-lived JWT tokens (1 hour)
- Sessions tied to specific NFT + wallet
- Automatic cleanup on NFT transfer
- Rate limiting per wallet

### Wallet-Specific Security
- **Keplr**: Uses encryptionUtils for consistent encryption
- **MetaMask**: Consider implementing encryptionSeed storage
- Both support Ledger for hardware wallet security

## Error Handling

### Common Scenarios
1. Invalid permit → Request new permit from frontend
2. NFT not owned → Authentication fails
3. Metadata missing traits → Use default personality
4. Permit expired → Frontend regenerates
5. Wallet not connected → Prompt user to connect
6. Unsupported wallet → Show supported wallet list

## Benefits of Hybrid Approach

1. **Optimal Library Usage**: Each library used where it works best
2. **Security**: Permits signed in browser, verified on backend
3. **User Experience**: Support for multiple popular wallets
4. **Developer Experience**: Python backend for AI work
5. **Flexibility**: Can update backend without touching wallet code
6. **Wide Compatibility**: Desktop, mobile (MetaMask), Ledger support

## Next Steps

1. Confirm NFT contract address when available
2. Verify metadata structure with NFT contract team
3. Set up development environment with both libraries
4. Test permit generation and verification flow
5. Implement trait extraction from metadata
6. Test both Keplr and MetaMask flows

## Questions to Resolve

1. Exact structure of trait metadata in NFT?
2. Will traits be in public or private metadata?
3. Permit expiration time preferences?
4. Should we cache permits or regenerate each session?
5. Are we using mainnet (secret-4) or testnet (pulsar-3) for development?
6. Should we implement encryptionSeed storage for MetaMask users?
7. Priority: Keplr or MetaMask for initial implementation?
