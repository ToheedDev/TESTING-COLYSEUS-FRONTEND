# Debug: Add Item (Minimal)

- Open `multiplayer-test.html` and join a match.
- In the game view, use panel "🛠️ DEBUG: Add Item".
- Pick an item and click "Add Item" — it appears immediately in "🎁 Your Item".

## Frontend
- Sends: `room.send("debug_add_item", { item_code: "fast" })`.
- Receives:
  - `debug_item_added` 
    - Example:
      ```json
      {
        "itemCode": "fast",
        "targetUserGameStateId": "<uuid>"
      }
      ```

## Supported Item Codes
`road_block`, `slow`, `fast`, `back_to_lol`, `cheating_roll`
