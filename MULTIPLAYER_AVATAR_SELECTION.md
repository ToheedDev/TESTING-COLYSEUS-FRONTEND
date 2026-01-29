# Multiplayer Avatar Selection - Frontend Integration Guide

## Overview
During the 60-second waiting phase, players can select avatars and see each other's selections in real-time via Colyseus.

**NEW: Cross-Game Avatar Support**
In multiplayer mode, users can select ANY avatar from ANY game they own. The avatar is no longer restricted to the current game being played. This allows players to use their favorite avatars across all multiplayer maps.

### Colyseus Server
- **Message Handler**: `change_avatar` - Processes avatar changes during waiting phase
- **Broadcast Event**: `avatar_changed` - Notifies all players of avatar changes

## Frontend Integration

### 1. Get User's Available Avatars

**Option A: All Avatars (Recommended for Multiplayer)**
```javascript
// Fetch ALL available avatars from all games
// In multiplayer, users can select avatars from any game
async function fetchUserAvatars(backendUrl, userToken) {
  const response = await fetch(`${backendUrl}/user/api/v1/user-nft/`, {
    headers: {
      'Authorization': `Bearer ${userToken}`
    }
  });
  const data = await response.json();
  return data.data || data.result || [];
}
```

**Option B: Game-Specific Avatars (Optional)**
```javascript
// Fetch available avatars for a specific game (if you want to filter UI)
async function fetchUserAvatarsForGame(backendUrl, gameId, userToken) {
  const response = await fetch(`${backendUrl}/user/api/v1/user-nft/?id=${gameId}`, {
    headers: {
      'Authorization': `Bearer ${userToken}`
    }
  });
  const data = await response.json();
  return data.data || data.result || [];
}
```

### 2. Send Avatar Change to Colyseus
```javascript
// When user selects an avatar from the UI
function changeAvatar(avatarCode, itemId, nftCategory) {
  // Only allow during waiting phase
  if (room.state.status !== 'waiting') {
    console.warn('Avatar can only be changed during waiting phase');
    return;
  }
  
  room.send('change_avatar', {
    avatar_code: avatarCode,
    item_id: itemId,
    nft_category: nftCategory  // 'minted', 'game_pass', or 'free'
  });
}
```

### 3. Listen for Avatar Change Events
```javascript
// Listen for your own avatar change confirmation
room.onMessage('avatar_changed', (message) => {
  console.log('Avatar changed:', message);
  // message = {
  //   sessionId: 'xyz123',
  //   username: 'PlayerName',
  //   seat: 1,
  //   avatarAssetKey: 'G001ACSB',
  //   avatarRarity: 'common',
  //   avatarMovement: 'walk'
  // }
  
  // Update UI to show the new avatar for this player
  updatePlayerAvatar(message.seat, {
    assetKey: message.avatarAssetKey,
    rarity: message.avatarRarity,
    movement: message.avatarMovement
  });
});

// Listen for errors
room.onMessage('error', (message) => {
  if (message.message.includes('Avatar')) {
    alert(message.message);
  }
});
```

## Colyseus Events

### Send to Server
- **Message**: `change_avatar`
- **Payload**: `{ avatar_code, item_id?, nft_category? }`
- **Restrictions**: Only during waiting phase

### Receive from Server
- **Message**: `avatar_changed`
- **Payload**: `{ sessionId, username, seat, avatarAssetKey, avatarRarity, avatarMovement }`
- **Broadcast**: Sent to all players in the room

## Flow Diagram

```
User clicks avatar
    ↓
Frontend sends 'change_avatar' message to Colyseus
    ↓
Colyseus validates (waiting phase only)
    ↓
Colyseus calls Django internal API
    ↓
Django updates UserActiveGameAvatar + user_game_config
    ↓
Django returns avatar metadata
    ↓
Colyseus updates player state
    ↓
Colyseus broadcasts 'avatar_changed' to all players
    ↓
All clients update UI to show new avatar
```
