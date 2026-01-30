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
