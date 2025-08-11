# Technical Specification: Secret AI SDK Integration

## Overview
This document specifies how the backend integrates with Secret AI SDK to provide LLM capabilities with trait-based personalities for each NFT.

## Technology Stack
- Python 3.12 with Anaconda
- Secret AI SDK (developer preview)
- secret-sdk>=1.8.1
- FastAPI for API server

## Environment Setup

### Installation Requirements
1. Anaconda with Python 3.12
2. Virtual environment setup:
   ```bash
   conda create -n panthers python=3.12
   conda activate panthers
   ```

3. Dependencies installation:
   ```bash
   pip install secret-ai-sdk
   pip install 'secret-sdk>=1.8.1'
   pip install fastapi uvicorn redis
   ```

### Configuration

#### Environment Variables
```bash
# Secret AI Configuration
SECRET_AI_API_KEY=bWFzdGVyQHNjcnRsYWJzLmNvbTpTZWNyZXROZXR3b3JrTWFzdGVyS2V5X18yMDI1

# Optional: Custom node URL (if default doesn't work)
SECRET_NODE_URL=<LCD_NODE_URL>  # Get from api-registry

# Chain Configuration
CHAIN_ID=pulsar-3  # testnet
# CHAIN_ID=secret-4  # mainnet (when ready)
```

## Secret AI Client Initialization

### Basic Setup
```python
from secret_ai_sdk.secret_ai import ChatSecret
from secret_ai_sdk.secret import Secret

# Initialize Secret client
# For testnet
secret_client = Secret(chain_id='pulsar-3')

# For mainnet (future)
# secret_client = Secret(chain_id='secret-4')

# If custom node URL needed
# secret_client = Secret(chain_id='pulsar-3', node_url=custom_lcd_url)

# Get available models and URLs
models = secret_client.get_models()
urls = secret_client.get_urls(model=models[0])
```

### LLM Client Configuration
```python
# Initialize the LLM client
secret_ai_llm = ChatSecret(
    base_url=urls[0],
    model=models[0],  # e.g., 'deepseek-chat'
    temperature=0.7    # Default, will be overridden by NFT traits
)
```

## Integration with NFT Traits

### Dynamic System Prompts Based on Traits

The system generates custom prompts based on NFT traits loaded from metadata:

#### Trait-Based Configuration
```python
def create_llm_client_for_nft(traits):
    """
    Create an LLM client configured with NFT-specific traits
    """
    # Get Secret AI connection
    secret_client = Secret(chain_id='pulsar-3')
    models = secret_client.get_models()
    urls = secret_client.get_urls(model=models[0])
    
    # Create LLM client with trait-specific temperature
    temperature = traits.get('modifiers', {}).get('temperature', 0.7)
    
    llm_client = ChatSecret(
        base_url=urls[0],
        model=models[0],
        temperature=temperature
    )
    
    return llm_client
```

#### System Prompt Generation
```python
def generate_system_prompt(traits):
    """
    Generate a system prompt based on NFT traits
    """
    personality = traits.get('personality', 'neutral')
    speaking_style = traits.get('speaking_style', 'casual')
    expertise = traits.get('expertise', [])
    modifiers = traits.get('modifiers', {})
    
    base_prompt = f"""You are Secret Panther, an AI with the following characteristics:
    
    Personality: {personality}
    Speaking Style: {speaking_style}
    Areas of Expertise: {', '.join(expertise) if expertise else 'General knowledge'}
    
    Behavioral Guidelines:
    - Maintain a {modifiers.get('verbosity', 'medium')} level of detail in responses
    - Use {modifiers.get('humor', 'moderate')} amounts of humor
    - Communicate with {modifiers.get('energy', 'balanced')} energy
    - Keep a {modifiers.get('formality', 'balanced')} level of formality
    """
    
    # Add personality-specific instructions
    personality_prompts = {
        "degen": "\nUse crypto slang, emojis, and express high risk tolerance. Say 'gm' and 'wagmi' frequently.",
        "sage": "\nBe thoughtful and philosophical. Ask deep questions and provide wisdom.",
        "builder": "\nFocus on solutions and technical details. Be helpful and practical.",
        "artist": "\nBe creative and use metaphors. Think abstractly and poetically.",
        "scholar": "\nBe precise and academic. Cite examples and use formal language.",
        "comedian": "\nBe witty and entertaining. Use jokes and wordplay.",
        "warrior": "\nBe bold and confident. Use strong, decisive language.",
        "mystic": "\nBe mysterious and cryptic. Speak in riddles occasionally.",
        "merchant": "\nFocus on value and strategy. Think in terms of trades and deals.",
        "explorer": "\nBe curious and adventurous. Ask questions and suggest new ideas."
    }
    
    if personality in personality_prompts:
        base_prompt += personality_prompts[personality]
    
    return base_prompt
```

## Chat Implementation

### Processing Messages with Traits

```python
def process_chat_message(llm_client, traits, user_message, conversation_history=None):
    """
    Process a user message with trait-based personality
    """
    # Generate system prompt from traits
    system_prompt = generate_system_prompt(traits)
    
    # Build message list
    messages = [
        ("system", system_prompt)
    ]
    
    # Add conversation history if exists
    if conversation_history:
        for msg in conversation_history[-10:]:  # Last 10 messages for context
            messages.append((msg['role'], msg['content']))
    
    # Add current user message
    messages.append(("human", user_message))
    
    # Get response from Secret AI
    response = llm_client.invoke(messages, stream=False)
    
    return response
```

### Streaming Responses

For real-time chat experience:

```python
def stream_chat_response(llm_client, traits, user_message, conversation_history=None):
    """
    Stream response for real-time chat
    """
    system_prompt = generate_system_prompt(traits)
    
    messages = [
        ("system", system_prompt),
        ("human", user_message)
    ]
    
    # Stream response
    response_stream = llm_client.invoke(messages, stream=True)
    
    for chunk in response_stream:
        yield chunk  # Send chunk to frontend via WebSocket/SSE
```

## Session-Based Chat Management

### Managing LLM Clients per Session

```python
class ChatSessionManager:
    """
    Manages LLM clients and conversations per NFT session
    """
    def __init__(self):
        self.sessions = {}  # session_id -> llm_client mapping
        self.secret_client = Secret(chain_id='pulsar-3')
        self.models = self.secret_client.get_models()
        self.urls = self.secret_client.get_urls(model=self.models[0])
    
    def create_session(self, session_id, traits):
        """
        Create a new chat session with trait-specific configuration
        """
        temperature = traits.get('modifiers', {}).get('temperature', 0.7)
        
        llm_client = ChatSecret(
            base_url=self.urls[0],
            model=self.models[0],
            temperature=temperature
        )
        
        self.sessions[session_id] = {
            'client': llm_client,
            'traits': traits,
            'history': []
        }
        
        return session_id
    
    def chat(self, session_id, user_message):
        """
        Process a chat message for a session
        """
        if session_id not in self.sessions:
            raise ValueError("Session not found")
        
        session = self.sessions[session_id]
        
        # Process message
        response = process_chat_message(
            session['client'],
            session['traits'],
            user_message,
            session['history']
        )
        
        # Update history
        session['history'].append({'role': 'human', 'content': user_message})
        session['history'].append({'role': 'assistant', 'content': response})
        
        # Keep history limited
        if len(session['history']) > 20:
            session['history'] = session['history'][-20:]
        
        return response
    
    def end_session(self, session_id):
        """
        Clean up session
        """
        if session_id in self.sessions:
            del self.sessions[session_id]
```

## API Integration

### FastAPI Endpoints

```python
from fastapi import FastAPI, Depends, HTTPException
from pydantic import BaseModel

app = FastAPI()
session_manager = ChatSessionManager()

class ChatRequest(BaseModel):
    message: str

class ChatResponse(BaseModel):
    response: str
    session_id: str

@app.post("/api/chat")
async def chat_endpoint(
    request: ChatRequest,
    session_id: str = Depends(get_session_from_jwt)
):
    """
    Chat endpoint with trait-based personality
    """
    try:
        response = session_manager.chat(session_id, request.message)
        
        return ChatResponse(
            response=response,
            session_id=session_id
        )
        
    except ValueError as e:
        raise HTTPException(status_code=404, detail="Session not found")
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

## Error Handling

### Common Issues and Solutions

1. **Model Not Available**
   ```python
   models = secret_client.get_models()
   if not models:
       # Fallback or error
       raise Exception("No models available")
   ```

2. **Connection Issues**
   ```python
   try:
       urls = secret_client.get_urls(model=models[0])
   except Exception as e:
       # Try custom node URL
       secret_client = Secret(chain_id='pulsar-3', node_url=backup_url)
   ```

3. **Rate Limiting**
   ```python
   # Implement rate limiting per session
   from functools import lru_cache
   import time
   
   @lru_cache(maxsize=1000)
   def check_rate_limit(session_id):
       # Implementation
       pass
   ```

## Performance Optimization

### Caching LLM Clients
- Reuse LLM clients across sessions when possible
- Cache model and URL information
- Implement connection pooling

### Response Caching
- Cache common responses for similar inputs
- Use Redis for distributed caching
- Implement smart cache invalidation

## Monitoring and Logging

### Key Metrics
- Response time per chat
- Token usage per session
- Model availability
- Error rates
- Session duration

### Logging Strategy
- Log all chat requests (without content)
- Track token usage
- Monitor model performance
- Alert on errors

## Testing

### Unit Tests
- Test trait-based prompt generation
- Mock Secret AI responses
- Test session management
- Validate error handling

### Integration Tests
- Test with real Secret AI connection
- Verify trait application
- Test conversation continuity
- Load test with multiple sessions

## Security Considerations

1. **API Key Protection**: Never expose SECRET_AI_API_KEY
2. **Input Sanitization**: Clean user messages before sending
3. **Rate Limiting**: Implement per-user limits
4. **Session Security**: Validate JWT for every request
5. **Content Filtering**: Optional profanity/inappropriate content filter

## Migration Path

### From Testnet to Mainnet
1. Update CHAIN_ID from 'pulsar-3' to 'secret-4'
2. Update node URL to mainnet endpoint
3. No other code changes required
4. Re-test all functionality

## Important Notes

- Secret AI SDK is in developer preview (not for production)
- Monitor for SDK updates and breaking changes
- Join Secret Network Developer Telegram for support
- Be prepared for potential model changes

## Next Steps

1. Set up Anaconda environment
2. Install Secret AI SDK
3. Test connection to Secret AI
4. Implement trait-based prompts
5. Integrate with session management
6. Test with different NFT traits
7. Monitor performance and costs
