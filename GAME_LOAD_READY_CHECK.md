# Game Load Ready Check - Frontend Integration Guide

## Overview (Simple)
Game starts only after every player confirms load.

## Quick Flow
- Server prepares board, then sends `game_ready`
- Client loads assets → sends `game_load_finished`
- Server broadcasts `player_loaded` (progress)
- When all ready → server sends `game_started`
