# Data Models Specification

## Overview
Complete data structure definitions for the Secret Panthers system.

## NFT Metadata Structure

### Expected Format from NFT Contract
```
EncryptedMetadata {
  agent_config: {
    model: string              // "deepseek-chat"
    system_prompt: string       // Custom prompt for personality
    temperature: number         // 0.0-1.0
    max_tokens: number          // Response length limit
    capabilities: string[]      // ["chat", "code", "analysis"]
  }
  deployment: {
    image: string              // Docker image to use
    resources: {
      cpu: string              // "2"
      memory: string           // "4Gi"
      gpu: boolean             // true/false
    }
  }
}
```

## Database Schema

### Deployments Table
Tracks all agent deployments:
- id: UUID primary key
- nft_token_id: NFT identifier
- owner_address: Current owner
- deployment_id: Provider-specific ID
- status: pending|deploying|running|stopped|failed
- provider: secret_vm|akash|local
- endpoint_url: Where agent is accessible
- resources: JSON of resource allocation
- created_at: Timestamp
- updated_at: Timestamp
- last_health_check: Timestamp
- error_message: If failed

### Deployment Events Table
Audit trail of deployment actions:
- id: UUID primary key
- deployment_id: Foreign key to deployments
- event_type: created|started|stopped|failed|transferred
- event_data: JSON details
- created_at: Timestamp

### NFT Ownership Cache Table
Cached ownership for performance:
- token_id: NFT identifier (primary key)
- owner_address: Current owner
- last_verified: When last checked on-chain
- block_height: Block when verified

### Sessions Table
Active user sessions:
- session_id: UUID primary key
- wallet_address: User's wallet
- nft_token_id: Associated NFT
- access_token: JWT token
- refresh_token: For renewal
- created_at: Timestamp
- expires_at: Timestamp
- last_activity: Timestamp
- ip_address: Client IP
- user_agent: Browser info

## API Data Models

### Authentication Models

**AuthRequest**
- wallet_address: User's Secret Network address
- nft_token_id: NFT they claim to own
- signature: Signed message
- message: What was signed

**AuthResponse**
- access_token: JWT for API access
- refresh_token: For token renewal
- expires_in: Seconds until expiry
- session_id: Session identifier

### Deployment Models

**DeploymentRequest**
- nft_token_id: NFT to deploy
- provider: Where to deploy (secret_vm|akash|local)
- resources: Optional resource override

**DeploymentResponse**
- deployment_id: Unique identifier
- status: Current status
- endpoint_url: Agent URL when ready
- estimated_time: Seconds until ready
- created_at: Timestamp

**DeploymentStatus**
- deployment_id: Identifier
- status: Current state
- provider: Where deployed
- endpoint_url: Access URL
- resources: Allocated resources
- health: Health check status
- uptime: Seconds running
- error: If failed

### Chat Models

**ChatRequest**
- message: User's message
- conversation_id: Optional conversation context
- stream: Whether to stream response

**ChatResponse**
- response: AI's response
- conversation_id: Conversation identifier
- timestamp: When generated
- tokens_used: Token consumption
- model: Model used

**Conversation**
- conversation_id: Identifier
- messages: Array of messages
- created_at: Start time
- updated_at: Last activity
- token_count: Total tokens used

### Agent Models

**AgentInfo**
- name: Display name
- nft_token_id: Associated NFT
- capabilities: Available features
- model: AI model in use
- status: online|offline|error
- version: Agent version
- uptime: Seconds running

**AgentHealth**
- status: healthy|unhealthy|unknown
- last_check: Timestamp
- response_time: Milliseconds
- memory_usage: Bytes
- cpu_usage: Percentage
- error_count: Recent errors

## WebSocket Messages

### Deployment Updates
Real-time deployment status:
- type: status|progress|ready|error
- deployment_id: Which deployment
- data: Type-specific payload

### Chat Messages
Real-time chat:
- type: message|typing|error
- conversation_id: Which conversation
- data: Message content

## Redis Data Structures

### Session Storage
Key: `session:{session_id}`
Value: JSON session object
TTL: 3600 seconds

### NFT Sessions Set
Key: `nft_sessions:{nft_token_id}`
Value: Set of session IDs
Purpose: Track all sessions per NFT

### Deployment Status
Key: `deployment:{deployment_id}`
Value: JSON status object
TTL: 300 seconds

### Rate Limiting
Key: `rate:{wallet_address}:{endpoint}`
Value: Request count
TTL: 60 seconds

## Event Schemas

### NFT Transfer Event
Emitted when NFT ownership changes:
- token_id: Which NFT
- from_address: Previous owner
- to_address: New owner
- block_height: When transferred
- transaction_hash: On-chain reference

### Deployment Event
Emitted for deployment changes:
- deployment_id: Which deployment
- event_type: State change
- old_status: Previous state
- new_status: Current state
- metadata: Additional info

## Validation Rules

### NFT Token ID
- Format: Numeric string
- Range: 1-10000
- Must exist in contract

### Wallet Address
- Format: secret1 + 39 characters
- Valid bech32 encoding
- Must match signature

### Resource Limits
- CPU: 0.5-4 cores
- Memory: 1-8 GB
- GPU: 0-1 units
- Disk: 1-10 GB

### Message Limits
- Max length: 4000 characters
- Max conversation: 100 messages
- Max history: 30 days

## Data Privacy

### What's Encrypted
- NFT metadata (on-chain)
- Chat messages (in database)
- API keys and secrets
- User sessions

### What's Public
- NFT ownership
- Deployment status
- Agent endpoints
- Health metrics

### What's Logged
- API requests (without bodies)
- Errors and warnings
- Performance metrics
- Deployment events

Never logged:
- Passwords or keys
- Chat message content
- Personal information
- Wallet signatures
