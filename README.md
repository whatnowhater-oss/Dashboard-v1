# start-forge-hub.ps1 — Forge Command Local Windows 11 Launcher (RTX 5070 Ti 16GB)
$ErrorActionPreference = "Stop"
Write-Host "=== Starting Forge Command Local $0/Day Stack ===" -ForegroundColor Cyan

# Cap Ollama resident models to 1 by default so 14B weights never spill into DDR5 RAM
$env:OLLAMA_MAX_LOADED_MODELS = "1"
$env:OLLAMA_KEEP_ALIVE = "5m"

if (-not (Get-Command python -ErrorAction SilentlyContinue)) {
    Write-Error "Python 3 is required on PATH."
}

Write-Host "Checking FastAPI + Uvicorn + HTTPX dependencies..." -ForegroundColor Yellow
python -m pip install --quiet fastapi uvicorn httpx pydantic python-multipart

Write-Host "Launching Forge Local Backend at http://127.0.0.1:8765 ..." -ForegroundColor Green
Start-Process "http://127.0.0.1:8765"
python .\forge_local_backend.py

# Forge Command — Windows 11 Local Backend Kit ($0/Day)
"""
Forge Command — 100% Local $0/Day Windows 11 FastAPI Backend Bridge
Hardware Target: AMD Ryzen 7 9800X3D + NVIDIA GeForce RTX 5070 Ti (16GB VRAM)

What this single file wires into ai_command_hub.html locally on Windows 11:
1. Ollama Local GPU VRAM Monitor & Spill Eviction (`GET /api/vram`, `POST /api/vram/evict` via `keep_alive: 0`)
2. 100% Offline $0 Voice Proxy (`Kokoro-82M` TTS on :8880 + `Faster-Whisper` STT on :8881)
3. Deterministic Claude Code `PreToolUse` Hook Receiver (`POST /api/hooks/pre-tool-use` -> holds risky tool calls; denies with exit-2 decision when rejected)
4. Isolated Git Worktree Spawner (`POST /api/worktrees/spawn`) so parallel coding agents never collide
5. Cross-Model Shared Memory Bus (`GET/POST /api/memory`) persisted locally to JSON (archive-only, zero delete)

Quick Start (PowerShell on Windows 11):
  pip install fastapi uvicorn httpx pydantic python-multipart
  python forge_local_backend.py
  # Then open http://127.0.0.1:8765 in your browser.
"""

from __future__ import annotations

import asyncio
import json
import os
import subprocess
import time
import uuid
from pathlib import Path
from typing import Any, Dict, List, Optional

import httpx
from fastapi import FastAPI, HTTPException, UploadFile, File
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import FileResponse, JSONResponse, Response
from pydantic import BaseModel

# ==================== LOCAL ENDPOINT CONFIGURATION ====================
HOST = os.getenv("FORGE_HOST", "127.0.0.1")
PORT = int(os.getenv("FORGE_PORT", "8765"))
GPU_TOTAL_VRAM_GB = float(os.getenv("FORGE_VRAM_GB", "16.0"))

OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://127.0.0.1:11434")
KOKORO_TTS_URL = os.getenv("KOKORO_TTS_URL", "http://127.0.0.1:8880/v1/audio/speech")
WHISPER_STT_URL = os.getenv("WHISPER_STT_URL", "http://127.0.0.1:8881/v1/audio/transcriptions")
LITELLM_PROXY_URL = os.getenv("LITELLM_PROXY_URL", "http://127.0.0.1:4000/v1")

DATA_DIR = Path(os.getenv("FORGE_DATA_DIR", "./forge_local_state"))
ARCHIVE_DIR = DATA_DIR / "99-archive"
MEMORY_FILE = DATA_DIR / "shared_memory_bus.json"
HTML_UI_PATH = Path(os.getenv("FORGE_UI_HTML", "./ai_command_hub.html"))

DATA_DIR.mkdir(parents=True, exist_ok=True)
ARCHIVE_DIR.mkdir(parents=True, exist_ok=True)

app = FastAPI(title="Forge Command Local Backend", version="1.0.0")
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://127.0.0.1:8765", "http://localhost:8765", "null"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# In-memory approval queue for Claude Code PreToolUse hooks
PENDING_APPROVALS: Dict[str, Dict[str, Any]] = {}
APPROVAL_EVENTS: Dict[str, asyncio.Event] = {}


# ==================== REQUEST SCHEMAS ====================
class EvictRequest(BaseModel):
    model: str


class SpeakRequest(BaseModel):
    text: str
    voice: str = "am_michael"
    speed: float = 1.03


class WorktreeSpawnRequest(BaseModel):
    repo_path: str
    branch_name: str


class SharedMemoryEntry(BaseModel):
    scope: str = "All Models"
    text: str


class HookDecisionRequest(BaseModel):
    approval_id: str
    approve: bool
    reason: Optional[str] = None


# ==================== 1. SERVE DASHBOARD UI ====================
@app.get("/")
async def serve_dashboard() -> Any:
    if HTML_UI_PATH.exists():
        return FileResponse(HTML_UI_PATH)
    return JSONResponse(
        {
            "status": "Forge Command Backend Online ($0/day local)",
            "hint": "Place ai_command_hub.html in the same folder as forge_local_backend.py to open the UI at http://127.0.0.1:8765/",
        }
    )


# ==================== 2. OLLAMA 16GB VRAM MONITOR & SPILL EVICTION ====================
@app.get("/api/vram")
async def get_vram_status() -> Dict[str, Any]:
    """Polls Ollama GET /api/ps to detect resident weights and DDR5 system RAM spilling."""
    try:
        async with httpx.AsyncClient(timeout=3.0) as client:
            resp = await client.get(f"{OLLAMA_BASE_URL}/api/ps")
            resp.raise_for_status()
            payload = resp.json()
    except Exception as exc:
        return {
            "ollama_online": False,
            "total_gpu_gb": GPU_TOTAL_VRAM_GB,
            "used_vram_gb": 0.0,
            "ram_spill_detected": False,
            "models": [],
            "note": f"Ollama unreachable at {OLLAMA_BASE_URL}: {exc}",
        }

    models_out: List[Dict[str, Any]] = []
    used_vram_bytes = 0
    spill_detected = False

    for m in payload.get("models", []):
        size_total = int(m.get("size", 0))
        size_vram = int(m.get("size_vram", 0))
        used_vram_bytes += size_vram
        if size_total > 0 and size_vram < size_total:
            spill_detected = True
        models_out.append(
            {
                "name": m.get("name", "unknown"),
                "size_gb": round(size_total / (1024**3), 2),
                "vram_gb": round(size_vram / (1024**3), 2),
                "spilling_to_ram": size_vram < size_total,
                "expires_at": m.get("expires_at", ""),
            }
        )

    used_gb = round(used_vram_bytes / (1024**3), 2)
    return {
        "ollama_online": True,
        "total_gpu_gb": GPU_TOTAL_VRAM_GB,
        "used_vram_gb": used_gb,
        "ram_spill_detected": spill_detected or (used_gb > GPU_TOTAL_VRAM_GB),
        "models": models_out,
    }


@app.post("/api/vram/evict")
async def evict_ollama_model(req: EvictRequest) -> Dict[str, Any]:
    """Immediately frees RTX 5070 Ti VRAM by sending keep_alive: 0 to Ollama."""
    async with httpx.AsyncClient(timeout=8.0) as client:
        resp = await client.post(
            f"{OLLAMA_BASE_URL}/api/generate",
            json={"model": req.model, "keep_alive": 0},
        )
        if resp.status_code >= 400:
            raise HTTPException(status_code=resp.status_code, detail=resp.text)
    return {"evicted": req.model, "keep_alive": 0}


# ==================== 3. 100% OFFLINE $0 VOICE (KOKORO-82M + FASTER-WHISPER) ====================
@app.post("/api/voice/speak")
async def speak_kokoro_local(req: SpeakRequest) -> Response:
    """Synthesizes speech locally via self-hosted Kokoro-FastAPI (OpenAI-compatible /v1/audio/speech)."""
    async with httpx.AsyncClient(timeout=15.0) as client:
        resp = await client.post(
            KOKORO_TTS_URL,
            json={
                "model": "kokoro",
                "input": req.text,
                "voice": req.voice,
                "speed": req.speed,
                "response_format": "mp3",
            },
        )
        if resp.status_code >= 400:
            raise HTTPException(status_code=502, detail=f"Kokoro TTS error: {resp.text}")
        return Response(content=resp.content, media_type="audio/mpeg")


@app.post("/api/voice/transcribe")
async def transcribe_whisper_local(file: UploadFile = File(...)) -> Dict[str, Any]:
    """Transcribes mic audio locally via Faster-Whisper server (OpenAI-compatible /v1/audio/transcriptions)."""
    audio_bytes = await file.read()
    async with httpx.AsyncClient(timeout=15.0) as client:
        files = {"file": (file.filename or "speech.webm", audio_bytes, file.content_type or "audio/webm")}
        data = {"model": "Systran/faster-whisper-large-v3-turbo"}
        resp = await client.post(WHISPER_STT_URL, data=data, files=files)
        if resp.status_code >= 400:
            raise HTTPException(status_code=502, detail=f"Faster-Whisper error: {resp.text}")
        return resp.json()


# ==================== 4. CLAUDE CODE DETERMINISTIC PreToolUse HOOK GATE ====================
# Wire in ~/.claude/settings.json:
# {
#   "hooks": {
#     "PreToolUse": [{
#       "matcher": "Bash|mcp__*",
#       "hooks": [{ "type": "http", "url": "http://127.0.0.1:8765/api/hooks/pre-tool-use" }]
#     }]
#   }
# }
RISKY_PATTERNS = ("git push", "gh pr merge", "rm ", "del ", "Remove-Item", "send_sms", "stripe")


@app.post("/api/hooks/pre-tool-use")
async def claude_pre_tool_use_hook(event: Dict[str, Any]) -> Dict[str, Any]:
    """Holds risky Claude Code tool calls until approved in Forge Command UI."""
    tool_name = str(event.get("tool_name", ""))
    tool_input = event.get("tool_input", {})
    raw_cmd = json.dumps(tool_input)

    # Hard block any destructive delete (enforcing strict 99-archive non-negotiable rule)
    if any(tok in raw_cmd for tok in ("rm -rf", "Remove-Item -Recurse", "DROP TABLE")):
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Blocked by Forge 99-Archive Zero-Delete Hook.",
            }
        }

    if not any(pat in raw_cmd or pat in tool_name for pat in RISKY_PATTERNS):
        return {}

    ap_id = f"ap-{uuid.uuid4().hex[:8]}"
    ev = asyncio.Event()
    PENDING_APPROVALS[ap_id] = {
        "id": ap_id,
        "tool_name": tool_name,
        "command": raw_cmd[:240],
        "created_at": time.strftime("%H:%M:%S"),
        "status": "Pending",
        "approved": False,
    }
    APPROVAL_EVENTS[ap_id] = ev

    try:
        await asyncio.wait_for(ev.wait(), timeout=120.0)
    except asyncio.TimeoutError:
        PENDING_APPROVALS[ap_id]["status"] = "Timed Out (Denied)"
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "deny",
                "permissionDecisionReason": "Timed out awaiting human approval in Forge Command.",
            }
        }

    decision = PENDING_APPROVALS[ap_id]
    if decision.get("approved"):
        return {
            "hookSpecificOutput": {
                "hookEventName": "PreToolUse",
                "permissionDecision": "allow",
            }
        }
    return {
        "hookSpecificOutput": {
            "hookEventName": "PreToolUse",
            "permissionDecision": "deny",
            "permissionDecisionReason": decision.get("reason") or "Denied in Forge Command UI (Exit 2).",
        }
    }


@app.get("/api/hooks/pending")
async def list_pending_hooks() -> List[Dict[str, Any]]:
    return [v for v in PENDING_APPROVALS.values() if v["status"] == "Pending"]


@app.post("/api/hooks/resolve")
async def resolve_pending_hook(req: HookDecisionRequest) -> Dict[str, Any]:
    item = PENDING_APPROVALS.get(req.approval_id)
    if not item:
        raise HTTPException(status_code=404, detail="Approval ID not found")
    item["approved"] = req.approve
    item["status"] = "Approved" if req.approve else "Denied"
    item["reason"] = req.reason
    if req.approval_id in APPROVAL_EVENTS:
        APPROVAL_EVENTS[req.approval_id].set()
    return item


# ==================== 5. GIT WORKTREE ISOLATION FOR PARALLEL AGENTS ====================
@app.post("/api/worktrees/spawn")
async def spawn_git_worktree(req: WorktreeSpawnRequest) -> Dict[str, Any]:
    repo = Path(req.repo_path).expanduser().resolve()
    if not (repo / ".git").exists():
        raise HTTPException(status_code=400, detail=f"Not a git repository: {repo}")

    safe_branch = "".join(ch for ch in req.branch_name if ch.isalnum() or ch in ("-", "_", "/"))
    wt_dir = repo.parent / f"{repo.name}__{safe_branch.replace('/', '_')}"
    proc = subprocess.run(
        ["git", "-C", str(repo), "worktree", "add", "-b", safe_branch, str(wt_dir)],
        capture_output=True,
        text=True,
        timeout=15,
    )
    if proc.returncode != 0:
        raise HTTPException(status_code=400, detail=proc.stderr.strip() or proc.stdout.strip())
    return {"repo": str(repo), "branch": safe_branch, "worktree_path": str(wt_dir)}


# ==================== 6. CROSS-MODEL SHARED MEMORY BUS (ARCHIVE-ONLY, ZERO DELETE) ====================
def _read_memory() -> List[Dict[str, Any]]:
    if not MEMORY_FILE.exists():
        return []
    return json.loads(MEMORY_FILE.read_text(encoding="utf-8"))


def _write_memory(items: List[Dict[str, Any]]) -> None:
    MEMORY_FILE.write_text(json.dumps(items, indent=2), encoding="utf-8")


@app.get("/api/memory")
async def get_shared_memory() -> List[Dict[str, Any]]:
    return _read_memory()


@app.post("/api/memory")
async def pin_shared_memory(entry: SharedMemoryEntry) -> Dict[str, Any]:
    items = _read_memory()
    record = {
        "id": f"sm-{uuid.uuid4().hex[:8]}",
        "scope": entry.scope,
        "text": entry.text,
        "pinned_at": time.strftime("%Y-%m-%d %H:%M:%S"),
    }
    items.insert(0, record)
    _write_memory(items)
    return record


@app.post("/api/memory/{item_id}/archive")
async def archive_shared_memory(item_id: str) -> Dict[str, Any]:
    """Moves memory entry to 99-archive instead of deleting it."""
    items = _read_memory()
    kept = []
    archived_item = None
    for it in items:
        if it.get("id") == item_id:
            archived_item = it
        else:
            kept.append(it)
    if not archived_item:
        raise HTTPException(status_code=404, detail="Memory item not found")
    _write_memory(kept)
    archive_log = ARCHIVE_DIR / f"archived_memory_{int(time.time())}_{item_id}.json"
    archive_log.write_text(json.dumps(archived_item, indent=2), encoding="utf-8")
    return {"archived_to": str(archive_log)}


if __name__ == "__main__":
    import uvicorn

    print(f"Forge Command Local Backend starting on http://{HOST}:{PORT} ($0/day local mode)")
    uvicorn.run(app, host=HOST, port=PORT)
## Included Files
1. `forge_local_backend.py` — FastAPI bridge (`127.0.0.1:8765`) connecting `ai_command_hub.html` to:
   - Ollama GPU VRAM monitor (`/api/ps`) & instant `keep_alive: 0` eviction
   - Local offline `Kokoro-82M` TTS (`:8880`) & `Faster-Whisper` STT (`:8881`)
   - Deterministic Claude Code `PreToolUse` approval queue + `99-archive` zero-delete guard
   - Isolated `git worktree` spawner for parallel sub-agents
   - Local JSON Shared Memory Bus (`forge_local_state/shared_memory_bus.json`)
2. `start-forge-hub.ps1` — 1-click PowerShell launcher for Windows 11.
3. `claude_hooks_settings.json` — Merge into `~/.claude/settings.json` to route risky tool calls to your UI Approval Gat
