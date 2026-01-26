# Multiplayer Avatar Selection - Frontend Integration Guide

## Overview
During the 60-second waiting phase, players can select avatars and see each other's selections in real-time via Colyseus.

## Backend Implementation (Completed)

### Colyseus Server
- **Message Handler**: `change_avatar` - Processes avatar changes during waiting phase
- **Broadcast Event**: `avatar_changed` - Notifies all players of avatar changes
- **File**: `/colyseus-server/src/rooms/GenericGameRoom.ts`

### Django Backend
- **Internal Endpoint**: `/game/api/v1/multiplayer/internal/update-avatar/`
- **Validates**: Only allows changes during waiting phase
- **Updates**: UserActiveGameAvatar, user_game_config, and broadcasts to room

## Frontend Integration

### 1. Get User's Available Avatars (for a specific game)
```javascript
// Fetch available avatars for the current game
// NOTE: correct path is /user/api/v1/user-nft/?id={gameId}
async function fetchUserAvatars(backendUrl, gameId, userToken) {
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

### 4. Display Player Avatars in Lobby
```javascript
// When room state updates
room.state.players.onAdd = (player, sessionId) => {
  // Display player with their current avatar
  displayPlayer(player.seat, {
    username: player.username,
    avatarAssetKey: player.avatarAssetKey,
    avatarRarity: player.avatarRarity,
    avatarMovement: player.avatarMovement
  });
};

// Listen for player avatar changes
room.state.players.onChange = (player, sessionId) => {
  // Update the player's displayed avatar
  updatePlayerAvatar(player.seat, {
    assetKey: player.avatarAssetKey,
    rarity: player.avatarRarity,
    movement: player.avatarMovement
  });
};
```

## Complete Example

```html
<!DOCTYPE html>
<html>
<head>
  <title>Multiplayer Avatar Selection</title>
</head>
<body>
  <div id="lobby">
    <h2>Waiting for Players... <span id="timer"></span></h2>
    
    <!-- Players display -->
    <div id="players">
      <!-- Will be populated dynamically -->
    </div>
    
    <!-- Avatar selection panel (only shown during waiting) -->
    <div id="avatar-selection" style="display: none;">
      <h3>Select Your Avatar</h3>
      <div id="avatar-list">
        <!-- Will be populated with user's avatars -->
      </div>
    </div>
  </div>

  <script src="https://unpkg.com/colyseus.js@^0.15.0/dist/colyseus.js"></script>
  <script>
    let room = null;
    let currentGameId = null;

    // Initialize multiplayer
    async function initializeMultiplayer() {
      const response = await fetch('/game/api/v1/multiplayer/initialize/', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${userToken}`
        },
        body: JSON.stringify({
          multiplier: 1
        })
      });
      
      const data = await response.json();
      const authData = data.data;
      currentGameId = authData.game_id;
      
      // Connect to Colyseus
      const client = new Colyseus.Client(authData.colyseus_url);
      room = await client.joinById(authData.colyseus_room_id, {
        token: authData.token
      });
      
      setupRoomListeners();
      
      // Show avatar selection during waiting phase
      if (room.state.status === 'waiting') {
        loadAvatarSelection();
      }
    }

    function setupRoomListeners() {
      // Listen for waiting timer
      room.onMessage('waiting_timer_started', (message) => {
        startCountdown(message.duration);
      });
      
      // Listen for players
      room.state.players.onAdd = (player, sessionId) => {
        displayPlayer(player);
      };
      
      // Listen for avatar changes
      room.onMessage('avatar_changed', (message) => {
        updatePlayerAvatar(message.seat, message);
      });
      
      // Listen for game start
      room.onMessage('game_started', () => {
        hideAvatarSelection();
      });
    }

    async function loadAvatarSelection() {
      const avatars = await fetchUserAvatars(currentGameId);
      
      const avatarList = document.getElementById('avatar-list');
      avatarList.innerHTML = '';
      
      avatars.forEach(avatar => {
        const avatarCard = document.createElement('div');
        avatarCard.className = 'avatar-card';
        avatarCard.innerHTML = `
          <img src="${avatar.properties.meta_data.image}" alt="${avatar.properties.meta_data.name}">
          <p>${avatar.properties.meta_data.name}</p>
          <p>Rarity: ${avatar.properties.rarity}</p>
        `;
        
        avatarCard.onclick = () => {
          room.send('change_avatar', {
            avatar_code: avatar.properties.avatar_code,
            item_id: avatar.item_id,
            nft_category: avatar.nft_category
          });
        };
        
        avatarList.appendChild(avatarCard);
      });
      
      document.getElementById('avatar-selection').style.display = 'block';
    }

    async function fetchUserAvatars(gameId) {
      const response = await fetch(`/user/api/v1/user-nft/?id=${gameId}`, {
        headers: {
          'Authorization': `Bearer ${userToken}`
        }
      });
      const data = await response.json();
      return data.data || data.result || [];
    }

    function displayPlayer(player) {
      const playersDiv = document.getElementById('players');
      const playerCard = document.createElement('div');
      playerCard.id = `player-${player.seat}`;
      playerCard.innerHTML = `
        <h4>Seat ${player.seat}: ${player.username}</h4>
        <img id="avatar-${player.seat}" src="/static/avatars/${player.avatarAssetKey}.png" alt="Avatar">
        <p>Rarity: <span id="rarity-${player.seat}">${player.avatarRarity}</span></p>
      `;
      playersDiv.appendChild(playerCard);
    }

    function updatePlayerAvatar(seat, avatarData) {
      const avatarImg = document.getElementById(`avatar-${seat}`);
      const raritySpan = document.getElementById(`rarity-${seat}`);
      
      if (avatarImg) {
        avatarImg.src = `/static/avatars/${avatarData.avatarAssetKey}.png`;
      }
      if (raritySpan) {
        raritySpan.textContent = avatarData.avatarRarity;
      }
    }

    function hideAvatarSelection() {
      document.getElementById('avatar-selection').style.display = 'none';
    }

    function startCountdown(seconds) {
      const timerEl = document.getElementById('timer');
      let remaining = seconds;
      
      const interval = setInterval(() => {
        timerEl.textContent = `${remaining}s`;
        remaining--;
        
        if (remaining < 0) {
          clearInterval(interval);
        }
      }, 1000);
    }
  </script>
</body>
</html>
```

## API Endpoints

### Get User Avatars
- **URL**: `/user/api/v1/user-nft/?id={gameId}`
- **Method**: GET
- **Auth**: Required
- **Response**:
```json
{
  "data": [
    {
      "item_id": "123",                 
      "avatar_id": "G001ACSB",
      "nft_category": "minted",
      "properties": {
        "avatar_code": "cubie_common_walk",
        "rarity": "Common",
        "meta_data": {
          "name": "Cubie",
          "image": "https://..."
        }
      }
    }
  ]
}
```

### Internal change-avatar endpoint (called by Colyseus)
- URL: `/game/api/v1/multiplayer/internal/update-avatar/`
- Method: POST
- Body: `{ user_game_state_id, avatar_code, item_id?, nft_category? }`
- Only allowed during waiting phase; updates `UserActiveGameAvatar` and `user_game_config`, then the room broadcasts `avatar_changed`.

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

## Testing

1. Start two browser sessions with different users
2. Both users click "Initialize Multiplayer"
3. Both users should see avatar selection UI
4. User 1 selects an avatar → Both users see the change
5. User 2 selects an avatar → Both users see the change
6. Timer reaches 0 → Game starts, avatar selection hidden
7. Try changing avatar after game starts → Should show error

## Notes

- Avatar changes are only allowed during the **waiting phase** (before game starts)
- All players see real-time updates when anyone changes their avatar
- The system validates avatar ownership on the backend
- Bot players use default Cubie avatar and cannot change it
- Avatar metadata (rarity, movement) is automatically propagated to game state
