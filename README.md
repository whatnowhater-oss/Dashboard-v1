# Forge Command — Windows 11 Local Backend Kit ($0/Day)

## Included Files
1. `forge_local_backend.py` — FastAPI bridge (`127.0.0.1:8765`) connecting `ai_command_hub.html` to:
   - Ollama GPU VRAM monitor (`/api/ps`) & instant `keep_alive: 0` eviction
   - Local offline `Kokoro-82M` TTS (`:8880`) & `Faster-Whisper` STT (`:8881`)
   - Deterministic Claude Code `PreToolUse` approval queue + `99-archive` zero-delete guard
   - Isolated `git worktree` spawner for parallel sub-agents
   - Local JSON Shared Memory Bus (`forge_local_state/shared_memory_bus.json`)
2. `start-forge-hub.ps1` — 1-click PowerShell launcher for Windows 11.
3. `claude_hooks_settings.json` — Merge into `~/.claude/settings.json` to route risky tool calls to your UI Approval Gat
