# Multi-NFT Switching Feature Specification

## Overview
Users who own multiple Secret Panthers NFTs can switch between them in the same session, with each NFT providing a different personality for the chat.

## User Experience Flow

### Initial Connection
1. User connects wallet
2. System queries ALL owned Panthers
3. If user owns 1 NFT → Direct to chat
4. If user owns multiple → Show NFT selector

### NFT Selector Interface
```
┌──────────────────────────────────────────┐
│  Your Secret Panthers (3)                │
├──────────────────────────────────────────┤
│ ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│ │  #123    │ │  #456    │ │  #789    │ │
│ │  [IMG]   │ │  [IMG]   │ │  [IMG]   │ │
│ │          │ │          │ │          │ │
│ │ DEGEN 🎮 │ │ SAGE 🧙  │ │ BUILDER 🔨│ │
│ │ Meme lord│ │ Wise one │ │ Creator  │ │
│ │ Risk: ▰▰▰│ │ Risk: ▰▱▱│ │ Risk: ▰▰▱│ │
│ │ Fun: ▰▰▰ │ │ Fun: ▰▱▱ │ │ Fun: ▰▰▱ │ │
│ └──────────┘ └──────────┘ └──────────┘ │
│      ↑ Active                            │
└──────────────────────────────────────────┘
```

### Switching Flow
1. User clicks different NFT
2. Insert system message in chat: "🔄 Switched to Panther #456 (Sage)"
3. Conversation history maintained with visual separator
4. New NFT traits loaded
5. Chat personality updates
6. NFT image changes in chat
7. Future messages use new personality

## Technical Implementation

### Frontend Components

#### NFT Selector Component
```typescript
interface NFTSelectorProps {
  ownedNFTs: NFT[];
  activeNFT: string;
  onSwitch: (tokenId: string) => void;
}

Component features:
- Grid/list view of owned NFTs (no limit on count)
- Show image, ID, and personality type
- Display key traits (personality, style, risk level)
- Visual trait indicators (bars/icons)
- Highlight active NFT
- Loading state during switch
- Hover for detailed trait preview
- Responsive grid (adapts to any number of NFTs)
```

#### State Management
```typescript
interface AppState {
  wallet: {
    address: string;
    type: 'keplr' | 'metamask';
  };
  nfts: {
    owned: NFT[];
    active: NFT | null;
    loading: boolean;
  };
  chat: {
    messages: Message[];
    sessionId: string;
  };
}
```

### Backend API Endpoints

#### GET /api/auth/owned-nfts
Get all Panthers owned by wallet.

**Request:**
```
Headers:
  Authorization: Bearer {JWT}
```

**Response:**
```json
{
  "nfts": [
    {
      "token_id": "123",
      "image_url": "ipfs://...",
      "name": "Panther #123",
      "traits": {
        "personality": "degen",
        "speaking_style": "meme",
        "expertise": ["defi", "gaming"],
        "modifiers": {
          "risk_tolerance": "high",
          "humor": "heavy",
          "energy": "chaotic"
        },
        "rarity": "uncommon"
      }
    },
    {
      "token_id": "456",
      "image_url": "ipfs://...",
      "name": "Panther #456",
      "traits": {
        "personality": "sage",
        "speaking_style": "philosophical",
        "expertise": ["wisdom", "strategy"],
        "modifiers": {
          "risk_tolerance": "low",
          "humor": "dry",
          "energy": "calm"
        },
        "rarity": "rare"
      }
    }
  ],
  "total": 2
}
```

#### POST /api/chat/switch-nft
Switch active NFT in current session.

**Request:**
```json
{
  "token_id": "456"
}
```

**Response:**
```json
{
  "success": true,
  "active_nft": "456",
  "traits": {
    "personality": "sage",
    "speaking_style": "philosophical"
  },
  "message": "Switched to Panther #456"
}
```

### Session Management Updates

#### In-Memory Session Structure
```python
sessions = {
  "session_id": {
    "wallet": "secret1...",
    "owned_nfts": ["123", "456", "789"],  # No limit
    "active_nft": "456",
    "active_traits": {...},
    "messages": [  # Preserved with switch markers
      {"role": "user", "content": "Hello"},
      {"role": "assistant", "content": "gm fren!"},
      {"role": "system", "content": "Switched to #456", "metadata": {...}},
      {"role": "user", "content": "What do you think?"},
      {"role": "assistant", "content": "*philosophical response*"}
    ],
    "created_at": timestamp,
    "last_activity": timestamp
  }
}
```

#### Switch Logic
```python
def switch_active_nft(session_id: str, new_token_id: str):
    session = get_session(session_id)
    
    # Verify user owns this NFT
    if new_token_id not in session['owned_nfts']:
        raise ValueError("NFT not owned")
    
    # Load new traits
    new_traits = load_nft_traits(new_token_id)
    old_nft = session['active_nft']
    
    # Add switch marker to conversation
    switch_message = {
        'role': 'system',
        'content': f'Switched from Panther #{old_nft} to Panther #{new_token_id}',
        'metadata': {
            'type': 'nft_switch',
            'from': old_nft,
            'to': new_token_id,
            'personality': new_traits['personality'],
            'timestamp': time.time()
        }
    }
    
    # Update session (keep messages)
    session['active_nft'] = new_token_id
    session['active_traits'] = new_traits
    session['messages'].append(switch_message)
    
    return new_traits
```

## UI/UX Considerations

### Visual Feedback
- Smooth transition animation when switching
- Loading spinner during trait loading
- Clear indication of active NFT
- Personality preview on hover

### Chat Interface Changes
- Show active NFT badge/image
- "Panther #123 is typing..." with correct NFT
- Visual separator line with switch notification
- System message: "🔄 Switched to Panther #456 (Sage personality)"
- Conversation history preserved above separator
- Different chat bubble colors per personality
- Small NFT avatar changes to reflect active Panther

### Performance
- Cache NFT images locally
- Preload traits for owned NFTs
- Instant visual switch, async backend update
- Keep switching fast (<500ms)

## Edge Cases

### Handling Multiple Windows
- Same wallet in multiple tabs
- Sync active NFT across tabs (optional)
- Or treat each tab as separate session

### NFT Transfer During Session
- Detect if NFT sold/transferred
- Remove from available list
- Switch to another owned NFT
- Or end session if no NFTs left

### Rate Limiting
- Limit switches to prevent abuse
- Suggested: 10 switches per hour
- Clear feedback when limited

## Benefits of Multi-NFT Switching

1. **Explore Personalities**: Try different Panthers
2. **Find Favorites**: Discover preferred traits
3. **Use Case Specific**: Degen for trading, Sage for advice
4. **Collection Value**: Encourages multiple NFT ownership
5. **Entertainment**: Fun to see different responses

## Testing Requirements

### Functional Tests
- Switch between 2 NFTs
- Switch between 5+ NFTs
- Switch rapidly (rate limiting)
- Switch with no messages
- Switch mid-conversation

### UI Tests
- Selector displays correctly
- Images load properly
- Active state updates
- Responsive on mobile
- Keyboard navigation

### Integration Tests
- Traits update correctly
- Chat personality changes
- Session maintains integrity
- JWT remains valid
- Performance under load

## Implementation Priority

### Phase 1: Basic Switching
- Query owned NFTs
- Simple dropdown selector
- Switch functionality
- Update chat personality

### Phase 2: Enhanced UI
- Visual NFT grid
- Smooth animations
- Trait previews
- Loading states

### Phase 3: Optimizations
- Image caching
- Trait preloading
- Instant switching
- Cross-tab sync

## Success Metrics

- Users can see all owned NFTs
- Switching takes <1 second
- Personality changes are obvious
- No session corruption
- UI remains responsive

## Resolved Design Decisions

1. ✅ **Conversation history persists** with clear switch markers
2. ✅ **Show trait details** in selector for informed choice
3. ✅ **No limit on NFT count** - UI adapts to any number
4. ⚠️ **Rate limiting**: 10 switches per hour (prevent abuse)
5. 📱 **Mobile UI**: Scrollable horizontal cards or list view

## Implementation Notes

### Handling Large Collections
For users with many NFTs (50+):
- Paginated grid view (20 per page)
- Search/filter by trait or ID
- "Favorites" option for quick access
- Lazy load images
- Virtual scrolling for performance

### Conversation Continuity
When switching NFTs:
- Previous messages remain visible
- Clear visual separator shows switch point
- System message indicates personality change
- New responses reflect new traits
- Users understand context change
