# YGG Games Multiplayer System - Complete Frontend Guide

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Setup & Getting Started](#setup--getting-started)
- [Integration Flow](#integration-flow)
- [API Reference](#api-reference)
- [State Synchronization](#state-synchronization)
- [Avatar System](#avatar-system)
- [Game State Management](#game-state-management)
- [Event System](#event-system)
---

## Overview

The YGG Games multiplayer system enables real-time competitive gameplay where multiple players participate in the same game session simultaneously. The system uses a three-tier architecture:

1. **Django Backend** - Handles authentication, game initialization, roll processing, and match finalization
2. **Colyseus Server** - Manages real-time WebSocket connections and state synchronization
3. **Frontend Client** - Connects to both systems for seamless multiplayer experience

### Key Features
- ✅ Real-time state synchronization across all players
- ✅ Simultaneous play (non-turn-based)
- ✅ Prize pool system with winner-takes-all rewards
- ✅ Free rolls per player (doesn't consume wallet rolls)
- ✅ Match time limits (default: 10 minutes)
- ✅ Points-based winner determination
- ✅ Avatar synchronization (asset, rarity, movement)
- ✅ Complete game state storage per player
- ✅ Board configuration sharing

---

## Architecture

```
┌─────────────┐         ┌──────────────────┐         ┌─────────────────┐
│   Frontend  │◄───────►│ Django Backend   │         │ Colyseus Server │
│   Client    │  HTTPS  │ (REST API)       │  HTTPS  │ (WebSocket)     │
└─────────────┘         └──────────────────┘◄───────►└─────────────────┘
      │                          │                            │
      │         1. Initialize    │                            │
      ├─────────────────────────►│                            │
      │         2. Get Token     │                            │
      │◄─────────────────────────┤                            │
      │         3. Connect (WebSocket)                        │
      ├──────────────────────────────────────────────────────►│
      │         4. Validate Token│                            │
      │                          │◄───────────────────────────┤
      │         5. Real-time State Updates                    │
      │◄──────────────────────────────────────────────────────┤
      │         6. Roll Actions  │                            │
      │──────────────────────────┼───────────────────────────►│
      │                          │◄──── Process Roll ─────────┤
      │         7. Match End     │                            │
      │◄─────────────────────────┼────────────────────────────┤
```

**Data Flow:**
1. Frontend initializes match with Django → Receives token & connection details
2. Frontend connects to Colyseus with token
3. Colyseus validates token with Django → Authenticates player
4. Player state synced to all clients in real-time
5. Players roll simultaneously → Colyseus processes via Django → Updates broadcast
6. Match ends → Winner determined → Prize awarded

---

## Integration Flow

### Phase 1: Initialize Match

**Endpoint:** `POST /game/api/v1/multiplayer/{game_id}/initialize/`

```javascript
const response = await fetch(`${DJANGO_URL}/game/api/v1/multiplayer/${gameId}/initialize/`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${authToken}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    "multiplier": 1,
    "max_players": 2, // 3, 4
    "entry_points_cost": 10,
  })
});

const data = await response.json();
```

**Response Structure:**
```json
{
  "success": true,
  "result": {
    "id": "user-game-state-uuid",
    "game": {
      "id": "game-uuid",
      "game": "ygg_city",
      "label": "YGG City"
    },
    "mode": "premium",
    "multiplier": 1,
    "avatar": {
      "asset_key": "G001ACSB",
      "rarity": "Common",
      "movement": "walk"
    },
    "user_game_state": { /* complete game state */ },
    "user_game_config": { /* board configuration */ },
    "token": "colyseus-auth-token",
    "colyseus_url": "wss://col-dev.lolreel.com",
    "colyseus_room_id": "ygg_city_abc123",
    "match_id": "match-uuid",
    "game_id": "game-uuid",
    "game_name": "ygg_city",
    "entry_rolls_cost": 20,
    "entry_points_cost": 10,
    "prize_pool_points": 10,
    "free_rolls": 20,
    "current_players": 1,
    "max_players": 2
  }
}
```

**What Happens:**
1. Django validates user has enough rolls/points
2. Entry costs deducted from wallet
3. Finds existing waiting match OR creates new one
4. Entry points added to prize pool
5. User game state created and linked to match
6. Avatar resolved from user's active avatar
7. Complete game state and board config serialized
8. Auth token generated (24-hour validity)

### Phase 2: Connect to Colyseus

```javascript
const client = new Colyseus.Client(matchData.colyseus_url);

const room = await client.joinOrCreate('generic_game_room', {
  token: matchData.token,
  matchRoomId: matchData.colyseus_room_id,
  matchUUID: matchData.match_id,
  gameId: matchData.game_id,
  gameName: matchData.game_name,
  multiplier: matchData.multiplier,
  mode: matchData.mode,
  maxPlayers: matchData.max_players,
  entryRollsCost: matchData.entry_rolls_cost,
  entryPointsCost: matchData.entry_points_cost,
  prizePoolPoints: matchData.prize_pool_points,
  freeRollsPerPlayer: matchData.free_rolls
});

console.log("Connected! Session ID:", room.sessionId);
```

**What Happens:**
1. Colyseus validates token with Django
2. Player authenticated and added to room
3. Player state synchronized to all clients
4. `room.sessionId` assigned (your unique identifier)

### Phase 3: Setup Event Listeners

```javascript
// State changes
room.onStateChange((state) => {
  console.log("Status:", state.status); // waiting, in_progress, finished
  updateGameUI(state);
});

// Player joined
room.onMessage("player_joined", (data) => {
  console.log(`${data.username} joined (Seat ${data.seat})`);
  addPlayerToUI(data);
});

// Player ready
room.onMessage("player_ready", (data) => {
  console.log(`${data.username} is ready`);
  updatePlayerReady(data.username);
});

// Game started
room.onMessage("game_started", (data) => {
  console.log("Game started! Prize pool:", data.prizePoolPoints);
  showGameInterface();
});

// Roll result
room.onMessage("roll_result", (data) => {
  displayRollActivity(data);
  updatePlayerProgress(data);
});

// Match ended
room.onMessage("game_ended", (data) => {
  console.log("Winner:", data.winner.username);
  console.log("Prize:", data.winner.prizePoolAwarded);
  showResults(data);
});

// Errors
room.onMessage("error", (data) => {
  console.error("Error:", data.message);
  showError(data.message);
});

// Disconnection
room.onLeave((code) => {
  console.log("Left room, code:", code);
  handleDisconnect();
});
```

### Phase 4: Lobby (Waiting for Players)

```javascript
// Mark as ready
function markReady() {
  room.send("player_ready");
}

// Display lobby state
function updateLobby() {
  const state = room.state;
  console.log(`Players: ${state.players.size}/${state.maxPlayers}`);
  console.log(`Prize Pool: ${state.prizePoolPoints} points`);
  
  state.players.forEach((player, sessionId) => {
    console.log(`- ${player.username} ${player.ready ? '✓' : '⏳'}`);
  });
}
```

**Auto-start:** Game starts when ALL players mark ready

### Phase 5: Gameplay (In Progress)

```javascript
// Perform a roll
function performRoll() {
  const myPlayer = room.state.players.get(room.sessionId);
  
  if (myPlayer.freeRollsRemaining <= 0) {
    console.log("No free rolls remaining!");
    return;
  }
  
  room.send("roll", {});
}

// Update UI on roll results
room.onMessage("roll_result", (data) => {
  if (data.sessionId === room.sessionId) {
    // Your roll
    renderBoard(data.userGameState);
    showWinnings(data.userGameState.winning_amount);
  }
  
  // Update leaderboard for all
  updateLeaderboard();
});
```

**Key Points:**
- Simultaneous play (not turn-based)
- Each player has fixed free rolls
- `pointsEarnedThisMatch` determines winner
- Match ends when all rolls used OR time limit reached

### Phase 6: Match End & Results

```javascript
room.onMessage("game_ended", (data) => {
  console.log("Reason:", data.reason);
  console.log("Winner:", data.winner.username);
  console.log("Prize:", data.winner.prizePoolAwarded);
  
  data.results.forEach((player, index) => {
    console.log(`${index + 1}. ${player.username}`);
    console.log(`   Points: ${player.points_earned}`);
    console.log(`   Rolls Used: ${player.free_rolls_used}`);
  });
  
  showResultsScreen(data);
});
```

**What Happens:**
1. Colyseus determines winner (highest `pointsEarnedThisMatch`)
2. Django backend awards prize pool to winner
3. All player states deactivated
4. Match marked as `finished`
5. Room disconnects all clients after 10 seconds

---

## API Reference

### Django Backend Endpoints

#### Initialize Multiplayer Match
```
POST /game/api/v1/multiplayer/{game_id}/initialize/
Authorization: Bearer {token}
Content-Type: application/json

{
  "multiplier": 1,
  "max_players": 2,
  "entry_points_cost": 10,
}
```


### Colyseus Room Messages

#### FROM Client TO Colyseus Server

**Player Ready:**
```javascript
room.send("player_ready");
```

**Roll Action:**
```javascript
room.send("roll", {});
```

#### FROM Colyseus Server TO Client

**player_joined:**
```javascript
{
  sessionId: "abc123",
  username: "PlayerName",
  seat: 1,
  freeRolls: 20
}
```

**game_started:**
```javascript
{
  startedAt: 1703500000000,
  prizePoolPoints: 20,
  freeRollsPerPlayer: 20,
  maxDuration: 600000
}
```

**roll_result:**
```javascript
{
  sessionId: "abc123",
  username: "PlayerName",
  rollValue: 5,
  jackpot: false,
  jackpotAmount: 0,
  gameState: "base_game",
  userGameState: { /* complete game state */ },
  userGameConfig: { /* board configuration */ },
  balance: 1500,
  freeRollsRemaining: 15,
  pointsEarnedThisMatch: 250,
  avatarAssetKey: "G001ACSB",
  avatarRarity: "Common",
  avatarMovement: "walk"
}
```

**game_ended:**
```javascript
{
  reason: "all_rolls_used",
  results: [ /* all players */ ],
  winner: {
    userId: "uuid",
    username: "PlayerName",
    pointsEarned: 500,
    prizePoolAwarded: 20
  },
  prizePoolPoints: 20,
  duration: 180000
}
```

---

## State Synchronization

### Room State Schema

```javascript
room.state = {
  matchUUID: "uuid",
  matchRoomId: "ygg_city_abc123",
  gameId: "uuid",
  gameName: "ygg_city",
  status: "waiting", // or "in_progress" or "finished"
  maxPlayers: 2,
  entryRollsCost: 20,
  entryPointsCost: 10,
  prizePoolPoints: 20,
  freeRollsPerPlayer: 20,
  winnerId: "",
  winnerUsername: "",
  winnerPointsEarned: 0,
  startedAt: 0,
  finishedAt: 0,
  players: MapSchema<PlayerState>
}
```

### Player State Schema

```javascript
playerState = {
  userId: "uuid",
  username: "PlayerName",
  userGameStateId: "uuid",
  seat: 1,
  connected: true,
  ready: false,
  freeRollsUsed: 5,
  freeRollsRemaining: 15,
  pointsEarnedThisMatch: 250,
  balance: 1500,
  
  // Avatar
  avatarAssetKey: "G001ACSB",
  avatarRarity: "Common",
  avatarMovement: "walk",
  
  // Complete game state (JSON strings - parse with JSON.parse())
  userGameState: "{...}",
  userGameConfig: "{...}"
}
```

### Accessing State

```javascript
// Your player
const myPlayer = room.state.players.get(room.sessionId);
console.log("Free rolls:", myPlayer.freeRollsRemaining);
console.log("Points:", myPlayer.pointsEarnedThisMatch);

// Avatar
console.log("Avatar:", myPlayer.avatarAssetKey);
console.log("Rarity:", myPlayer.avatarRarity);

// Game state (parse JSON)
const gameState = JSON.parse(myPlayer.userGameState);
const boardConfig = JSON.parse(myPlayer.userGameConfig);
console.log("Position:", gameState.current_position);
console.log("Board size:", Object.keys(boardConfig.board).length);

// All players
room.state.players.forEach((player, sessionId) => {
  console.log(player.username, player.pointsEarnedThisMatch);
});
```


## Game State Management

### Storage Structure

Each player has two JSON-stringified fields:
- `userGameState` - Current game state
- `userGameConfig` - Board configuration

**Updated:**
- On join (initial state from Django)
- After each roll (updated from roll processing)

### Safe JSON Parsing

```javascript
function safeParseJSON(jsonString, defaultValue = {}) {
  if (!jsonString || jsonString === '{}') {
    return defaultValue;
  }
  try {
    return JSON.parse(jsonString);
  } catch (e) {
    console.error('JSON parse error:', e);
    return defaultValue;
  }
}

// Usage
const gameState = safeParseJSON(player.userGameState);
const boardConfig = safeParseJSON(player.userGameConfig);
```

### Game State Fields

```javascript
{
  game_state: "base_game",           // Game phase
  current_position: 15,              // Board position
  previous_position: 10,             // Previous position
  current_roll_value: 5,             // Dice roll
  total_winning_amount: 1540,        // Total balance
  winning_amount: 40,                // Last win
  available_rolls: 17,               // Rolls left
  jackpot: false,                    // Jackpot hit?
  jackpot_amount: 0,                 // Jackpot value
  free_spin_left: 0,                 // Free spins
  board_refresh: false,              // Board refreshed?
  win_index: [10, 11, 12, 13]       // Winning tiles
}
```

### Board Configuration

```javascript
{
  board: {
    "0": { name: "24H", value: -1, stop_type: "c" },
    "1": { name: "Spade", value: 4, stop_type: "r" },
    // ... 28 tiles total
  },
  game_type: "base_game",
  cor_values: [2, 5, 10, 20, 30, 50, 70, 100, 150, 200, 350, 500],
  avatar_rarity: "Common"
}
```

### Rendering Board

```javascript
function renderBoard(sessionId) {
  const player = room.state.players.get(sessionId);
  const boardConfig = safeParseJSON(player.userGameConfig);
  const gameState = safeParseJSON(player.userGameState);
  
  const boardDiv = document.getElementById('gameBoard');
  boardDiv.innerHTML = '';
  
  Object.entries(boardConfig.board).forEach(([index, tile]) => {
    const tileDiv = document.createElement('div');
    tileDiv.className = `tile ${tile.stop_type}`;
    
    // Highlight current position
    if (parseInt(index) === gameState.current_position) {
      tileDiv.classList.add('current');
    }
    
    tileDiv.innerHTML = `
      <div class="tile-name">${tile.name}</div>
      <div class="tile-value">${tile.value > 0 ? tile.value : ''}</div>
    `;
    
    boardDiv.appendChild(tileDiv);
  });
}

// Update on roll
room.onMessage("roll_result", (data) => {
  if (data.sessionId === room.sessionId) {
    renderBoard(room.sessionId);
  }
});
```

### Shared Board State

In multiplayer:
- First player to refresh sets `shared_board_state`
- Subsequent players use same board
- Ensures fair gameplay with identical tile placement

---

## Event System

### Listening to State Changes

```javascript
// Entire state changes
room.onStateChange((state) => {
  console.log("State updated:", state.status);
});

// Player map changes
room.state.players.onAdd = (player, sessionId) => {
  console.log(`${player.username} added`);
};

room.state.players.onRemove = (player, sessionId) => {
  console.log(`${player.username} removed`);
};

room.state.players.onChange = (player, sessionId) => {
  console.log(`${player.username} updated`);
};
```

---
