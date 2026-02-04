
# FAST & SLOW (Minimal Frontend Guide)

## Overview
- Effects are server-authoritative.
- Backend sets `active_effect.applied_at` and `active_effect.expires_at` when applied.
- Expiry is auto-checked by Django on each roll (no client messages for expiry).
- There is no `effect_expired` or `effect_cleared` message. Expiry is reflected in roll payloads.

## Data Fields (active_effect)
- `type`: "fast" | "slow"
- `speed_multiplier`: number (optional)
- `duration_seconds`: number (optional)
- `applied_at`: ISO string
- `expires_at`: ISO string
- `applied_by`: string (present for SLOW; user id of applier)

## Client Behavior
- Do not send `effect_expired`.
- Parse `userGameState` from roll payloads to get `active_item` and `active_effect`.
- Use `duration_seconds`, `applied_at`, and `expires_at` to handle timing locally on the client (purely visual countdowns/indicators). No events need to be sent.
- Remove effect indicators when `active_effect` is absent in the latest roll payload:
  - For self: check `mine_roll_result.userGameState.active_effect`.
  - For others: check `roll_result.userGameState.active_effect` for that player.

## Colyseus Messages
- `mine_roll_result` (to roller only): contains `userGameState` with `active_item` and `active_effect`.
- `roll_result` (to others): same shape.
- `item_used`: clear the current item UI after successful use.

## Removed Events
- `effect_expired` and `effect_cleared`


