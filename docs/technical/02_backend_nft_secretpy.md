# Technical Specification: Backend NFT Integration with SecretPy

## Overview
This document specifies how the Python backend uses SecretPy to interact with SNIP-721 NFTs using permits received from the frontend.

## Technology Stack
- Python 3.12+
- SecretPy SDK (secret-sdk)
- FastAPI for REST API
- Redis for session management
- Secret AI SDK for LLM integration

## Environment Configuration

### Development (Testnet)
```
CHAIN_ID=pulsar-3
LCD_URL=<from api-registry>
ENVIRONMENT=testnet
```

### Production (Mainnet)
```
CHAIN_ID=secret-4
LCD_URL=<from api-registry>
ENVIRONMENT=mainnet
```

### Easy Environment Switching
The system uses environment variables to switch between testnet and mainnet:
- Single configuration file with environment-specific settings
- No code changes required to switch networks
- All chain-specific parameters in environment variables

## SecretPy Setup

### Installation
```bash
pip install -U secret-sdk
```

### LCD Client Initialization
The backend creates an LCD client to interact with Secret Network:

```
For Testnet:
- chain_id: "pulsar-3"
- url: testnet LCD endpoint

For Mainnet:
- chain_id: "secret-4" 
- url: mainnet LCD endpoint
```

### Async vs Sync Client
- Use AsyncLCDClient for non-blocking operations
- Particularly useful for batch NFT queries
- Must properly close sessions after use

## NFT Query Operations with SecretPy

### Client Initialization

#### For Development (Testnet)
```python
from secret_sdk.client.lcd import LCDClient
from secret_sdk.client.localsecret import LocalSecret, test_net_chain_id

# Option 1: Using LCDClient for remote connection
secret = LCDClient(
    chain_id="pulsar-3",
    url=testnet_lcd_url  # from api-registry
)

# Option 2: Using LocalSecret for local node
secret = LocalSecret(chain_id=test_net_chain_id)
```

#### For Production (Mainnet)
```python
from secret_sdk.client.lcd import LCDClient
from secret_sdk.client.localsecret import LocalSecret, main_net_chain_id

# Option 1: Using LCDClient for remote connection
secret = LCDClient(
    chain_id="secret-4",
    url=mainnet_lcd_url  # from api-registry
)

# Option 2: Using LocalSecret for local node
secret = LocalSecret(chain_id=main_net_chain_id)
```

### SNIP-721 Query with Permit

When the frontend sends a permit, we include it in the query message:

#### 1. Query Owned Tokens
```python
def query_owned_tokens(secret, contract_address, contract_hash, wallet_address, permit):
    """
    Query tokens owned by a wallet using a permit
    """
    query_msg = {
        "with_permit": {
            "query": {
                "tokens": {
                    "owner": wallet_address,
                    "viewer": wallet_address,
                    "start_after": None,
                    "limit": 30
                }
            },
            "permit": permit  # Permit object from frontend
        }
    }
    
    result = secret.wasm.contract_query(
        contract=contract_address,
        query=query_msg,
        contract_hash=contract_hash,  # Optional but faster
        timeout=30
    )
    
    return result.get('token_list', {}).get('tokens', [])
```

#### 2. Query NFT Metadata
```python
def query_nft_metadata(secret, contract_address, contract_hash, token_id, permit):
    """
    Get NFT metadata including private metadata using permit
    """
    query_msg = {
        "with_permit": {
            "query": {
                "nft_info": {
                    "token_id": token_id
                }
            },
            "permit": permit
        }
    }
    
    result = secret.wasm.contract_query(
        contract=contract_address,
        query=query_msg,
        contract_hash=contract_hash,
        timeout=30
    )
    
    return result.get('nft_info', {})
```

#### 3. Query All NFT Details
```python
def query_all_nft_info(secret, contract_address, contract_hash, token_id, permit):
    """
    Get complete NFT information including metadata and ownership
    """
    query_msg = {
        "with_permit": {
            "query": {
                "all_nft_info": {
                    "token_id": token_id
                }
            },
            "permit": permit
        }
    }
    
    result = secret.wasm.contract_query(
        contract=contract_address,
        query=query_msg,
        contract_hash=contract_hash,
        timeout=30
    )
    
    return result
```

### Async Operations for Better Performance

For handling multiple NFT queries efficiently:

```python
import asyncio
from secret_sdk.client.lcd import AsyncLCDClient

async def batch_query_nfts(contract_address, contract_hash, token_ids, permit):
    """
    Query multiple NFTs in parallel for better performance
    """
    async with AsyncLCDClient(chain_id="pulsar-3", url=testnet_lcd_url) as secret:
        tasks = []
        for token_id in token_ids:
            query_msg = {
                "with_permit": {
                    "query": {
                        "nft_info": {
                            "token_id": token_id
                        }
                    },
                    "permit": permit
                }
            }
            
            task = secret.wasm.contract_query(
                contract=contract_address,
                query=query_msg,
                contract_hash=contract_hash,
                timeout=30
            )
            tasks.append(task)
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        await secret.session.close()  # Important: close the session
        
        return results
```

### Error Handling

```python
from secret_sdk.exceptions import LCDResponseError

def safe_query_nft(secret, contract_address, contract_hash, token_id, permit):
    """
    Query NFT with proper error handling
    """
    try:
        query_msg = {
            "with_permit": {
                "query": {
                    "nft_info": {
                        "token_id": token_id
                    }
                },
                "permit": permit
            }
        }
        
        result = secret.wasm.contract_query(
            contract=contract_address,
            query=query_msg,
            contract_hash=contract_hash,
            timeout=30
        )
        
        return {"success": True, "data": result}
        
    except LCDResponseError as e:
        return {"success": False, "error": str(e)}
    except Exception as e:
        return {"success": False, "error": f"Unexpected error: {str(e)}"}
```

### Verifying NFT Ownership

```python
def verify_nft_ownership(secret, contract_address, contract_hash, wallet_address, token_id, permit):
    """
    Verify that a wallet owns a specific NFT
    """
    # Query owned tokens
    owned_tokens = query_owned_tokens(
        secret, 
        contract_address, 
        contract_hash, 
        wallet_address, 
        permit
    )
    
    # Check if token_id is in owned tokens
    return token_id in owned_tokens
```

### Extracting Traits from Metadata

```python
def extract_traits_from_metadata(nft_info):
    """
    Extract personality traits from NFT metadata
    """
    # Default traits
    default_traits = {
        "personality": "neutral",
        "speaking_style": "casual",
        "expertise": [],
        "modifiers": {
            "temperature": 0.7,
            "verbosity": "medium",
            "humor": "moderate"
        }
    }
    
    try:
        # Navigate to private metadata
        private_metadata = nft_info.get('nft_info', {}).get('private_metadata', {})
        extension = private_metadata.get('extension', {})
        traits = extension.get('traits', {})
        
        # Merge with defaults
        return {**default_traits, **traits}
        
    except Exception as e:
        # Return defaults if extraction fails
        return default_traits
```

## Questions Remaining

1. Exact NFT contract address for testnet?
2. How to use permits with SecretPy contract queries?
3. Do you have SNIP-721 query message formats for SecretPy?
