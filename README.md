# Rabbit R1 LiveKit Voice Agent

**Talk to any AI model from your Rabbit R1.** Use GPT-5.2, Claude, Gemini, Qwen3.5, or any other model as a real-time voice assistant on the R1 powered by [LiveKit](https://livekit.io/).

Pair it with [local-voice-ai](https://github.com/ShayneP/local-voice-ai) to run everything locally on your own hardware w/ no cloud providers needed.

## What This Is

`SKILL.md` is a structured prompt that any AI coding assistant can follow to build a full R1 voice agent project from scratch. It works with **Claude Code, Cursor, Windsurf, Copilot, Aider, or any LLM-powered coding tool** -- just feed it the prompt and let it build.

The resulting project includes:

- A **Python voice agent** (scaffolded via `lk app create`) that runs STT -> LLM -> TTS
- A **web server** (`web-server.py`) that serves the R1 frontend and provides a LiveKit token endpoint
- An **R1 "creation"** (`creation/index.html`) -- the 240x282px voice UI that runs on the device
- An **install page** (`creation/install.html`) with a QR code for sideloading onto the R1

## How to Use

### Install the Skill

```bash
npx skills add ShayneP/rabbit-r1-livekit-skill
```

This installs `SKILL.md` so your AI coding assistant can use it automatically. Works with **Claude Code, Cursor, Windsurf, Copilot, Aider**, and other tools that support the skills format.

Once installed, just tell your assistant what you want:

> *"Build me a voice assistant for my Rabbit R1 that answers trivia questions."*

In Claude Code, you can also invoke it directly:

```
/rabbit-r1-livekit
```

### Manual Usage (Without Installing)

If you prefer not to install, copy the contents of `SKILL.md` into your assistant's context (paste it into chat, add it as a reference file, etc.) and describe what you want.

### What the Prompt Guides Your Assistant Through

1. Install the LiveKit CLI (`lk`) if needed
2. Scaffold the agent with `lk app create --template agent-starter-python agent`
3. Set up LiveKit Cloud credentials via `lk cloud auth`
4. Create the web server and R1 frontend files
5. Start all services and open an HTTPS tunnel
6. Provide an install link for the R1

## Architecture

```
R1 WebView
  --> web-server.py (port 8081, HTTPS via tunnelmole)
    --> LiveKit Cloud (WebRTC SFU + Agent Dispatch)
      --> Python Agent (STT -> LLM -> TTS)
```

**Flow:**
1. User taps the R1 screen (required for mic access)
2. Frontend fetches a LiveKit token from `/getToken`
3. Token includes a `RoomConfiguration` that tells LiveKit which agent to dispatch
4. Frontend connects to a LiveKit room via WebRTC and publishes mic audio
5. LiveKit Cloud dispatches the named agent into the room
6. Agent processes audio through STT -> LLM -> TTS and publishes audio back
7. R1 plays the response through its speaker

## Project Structure (After Running)

```
project-root/
  web-server.py              # Static file server + /getToken token API
  creation/
    index.html               # Voice agent UI for R1 (240x282px)
    install.html             # QR code page for R1 installation
  agent/                     # Scaffolded LiveKit agent
    src/agent.py             # Agent entry point
    pyproject.toml           # Dependencies (managed by uv)
    Dockerfile               # Production deployment
    tests/                   # Eval framework
```

## Prerequisites

- **Python 3.11+** with [`uv`](https://docs.astral.sh/uv/)
- **Node.js/npm** (for tunnelmole)
- A **LiveKit Cloud** account (free tier available)
- A **Rabbit R1** device (or a browser for testing)

## Running Manually

If the project is already built and you want to run it yourself:

```bash
# 1. Install web server dependency
pip install livekit-api

# 2. Install agent dependencies
cd agent && uv sync && cd ..

# 3. Start the web server (terminal 1)
python web-server.py

# 4. Start HTTPS tunnel (terminal 2)
# Install first if needed: npm install -g tunnelmole
tmole 8081

# 5. Start the agent (terminal 3)
cd agent && uv run python src/agent.py dev

# 6. Install on R1
# Open https://YOUR_TMOLE_URL/creation/install.html in a browser
# Scan the QR code with your R1
```

## Customization

| What | Where | Notes |
|------|-------|-------|
| Agent personality/instructions | `agent/src/agent.py` | The `instructions` parameter |
| Agent name | `agent/src/agent.py` + `creation/index.html` | Must match in both files |
| LLM model | `agent/src/agent.py` | e.g. `openai/gpt-4o`, `anthropic/claude-sonnet-4-20250514` |
| TTS voice | `agent/src/agent.py` | Voice ID in `inference.TTS(voice=...)` |
| UI branding/colors | `creation/index.html` | `#brand` div and `#FE5000` accent color |
| Tools (function calling) | `agent/src/agent.py` | Add tools to the Agent class |

## R1 WebView Constraints

Key limitations to be aware of if modifying the frontend:

- **HTTPS required** -- mic access (`getUserMedia`) is undefined over HTTP. Use tunnelmole or similar.
- **No WebGL** -- the Flutter-based WebView only supports Canvas 2D.
- **Screen size** -- 240x282 pixels exactly.
- **Touch quirks** -- use `document.body` touchstart with `e.preventDefault()`. Inline `onclick` in dynamic HTML won't fire.
- **Mic publishing** -- use manual `publishTrack()` instead of `setMicrophoneEnabled()` for reliability.

## R1 Native Bridges

The R1 WebView exposes these JavaScript globals:

- `PluginMessageHandler.postMessage()` -- send messages to R1 system
- `FlutterButtonHandler.postMessage()` -- button events
- `CreationStorageHandler.postMessage()` -- persistent storage
- `closeWebView` -- close the creation

Custom events on `window`: `sideClick`, `longPressStart`, `longPressEnd`, `scrollUp`, `scrollDown`
