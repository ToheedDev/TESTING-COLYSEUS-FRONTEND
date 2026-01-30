# Multiplayer Timing (Minimal)

Configure match timing via MultiplayerGameConfiguration.

## Config Keys (seconds)
- `max_match_duration_seconds` — Total match duration.
- `waiting_duration_seconds` — Lobby wait time before start.
- `bot_join_lead_seconds` — How many seconds before waiting ends to auto-add the bot (default 1).


## How it’s used
- Django reads these and returns them in initialize response.
- Frontend passes to Colyseus on room join as:
  - `maxDurationSeconds`
  - `waitingDurationSeconds`
  - `botJoinLeadSeconds`

## API Path
- Initialize Multiplayer (returns timing fields):
  - `GET /game/api/v1/multiplayer/{game_id}/initialize?multiplier=<int>`
  - Response includes: `max_duration_seconds`, `waiting_duration_seconds`, `bot_join_lead_seconds`
