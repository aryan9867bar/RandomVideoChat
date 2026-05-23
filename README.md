# Real-Time Random Video Chat Platform

This project is a **real-time random video chat system** (think Omegle) built using a **Matching Server** and a **Signalling Server**, enabling users to get randomly paired and connect via **FastAPI**, **Redis**, and **WebRTC signalling** for video calls and chat.

---

## Project Structure

```
random-video-chat/
├── docker-compose.yml
├── .env.example
├── .gitignore
│
├── matching-service/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── main.py              # FastAPI app + match worker thread
│   ├── models.py            # Pydantic User model
│   └── websocket_manager.py # In-memory connection registry
│
└── signalling-service/
    ├── Dockerfile
    ├── requirements.txt
    ├── main.py              # FastAPI app + signal relay
    ├── models.py            # Pydantic SignalMessage model
    └── websocket_manager.py # In-memory connection registry
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                        Client                           │
│  (React + WebRTC + WebSocket)                           │
└────────────┬───────────────────────┬────────────────────┘
             │                       │
     WS :8000/ws/{name}      WS :8001/ws/{username}
     POST :8000/register      (after match)
             │                       │
┌────────────▼──────────┐  ┌────────▼────────────────────┐
│   Matching Service    │  │     Signalling Service      │
│   (FastAPI + Thread)  │  │     (FastAPI)               │
│                       │  │                             │
│  • Redis queue        │  │  • WebRTC offer/answer      │
│  • Random pairing     │  │  • ICE candidate relay      │
│  • Room code gen      │  │  • Room role verification   │
└────────────┬──────────┘  └────────┬────────────────────┘
             │                       │
             └──────────┬────────────┘
                        │
                ┌───────▼──────┐
                │    Redis     │
                │  • Queue     │
                │  • Rooms     │
                │  • User map  │
                └──────────────┘
```

---

## System Architecture Overview

The system is divided into **two independent backend services**:

1. Matching Server  
2. Signalling Server  

Each component has a clearly defined responsibility to keep the architecture scalable and maintainable.

---

## Matching Server

### Purpose
The Matching Server is responsible for **pairing users randomly** and assigning them a shared room.

### Responsibilities
- Accepts **register requests** from users.
- Pushes users into a **Redis-based matching queue**.
- Continuously polls the Redis queue:
  - Uses an **exponential backoff strategy** when fewer than two users are available.
- Randomly selects **two users** from the queue.
- Generates a **unique room code** for the matched pair.
- Notifies both users of the room code via **WebSocket connections**.

### Key Features
- Redis-backed queue for fast operations  
- Efficient backoff to reduce CPU usage  
- Decoupled from WebRTC logic  
- Scales independently  

---

## Signalling Server

### Purpose
The Signalling Server handles **WebRTC session setup** and **real-time messaging**.

### Responsibilities
- Acts as a **WebRTC signalling server**
- Handles:
  - Offer / Answer exchange
  - ICE candidate exchange
- Manages **WebSocket-based chat**
- Uses the **room code** received from the Matching Server to connect users

### Key Features
- WebRTC-compatible signalling flow  
- WebSocket-based messaging  
- Stateless room handling  
- Low latency communication  

---

## Prerequisites

| Tool | Version | Notes |
|------|---------|-------|
| Docker & Docker Compose | ≥ 20 / ≥ 2 | Easiest path |
| **— or —** | | |
| Python | 3.11+ | For local run |
| Redis | 7+ | Must be running on port 6379 |

---

## Quick Start — Local (No Docker)

You need Redis running locally first:

```bash
# macOS
brew install redis && brew services start redis

# Ubuntu / Debian
sudo apt install redis-server && sudo systemctl start redis
```

**Terminal 1 — Matching Service:**

```bash
cd matching-service
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --port 8000 --reload
```

**Terminal 2 — Signalling Service:**

```bash
cd signalling-service
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn main:app --port 8001 --reload
```

Both services default to `redis://127.0.0.1:6379`. Override via the `REDIS_URL` env var if needed.

---

## Quick Start — Docker (Recommended)

```bash
# 1. Clone / unzip the project
git clone <your-repo-url>
cd random-video-chat

# 2. Start all three services (Redis + Matching + Signalling)
docker compose up --build

# Services will be available at:
#   Matching Service   → http://localhost:8000
#   Signalling Service → http://localhost:8001
```

To run in the background:

```bash
docker compose up --build -d
```

To stop everything:

```bash
docker compose down
```

---

## Environment Variables

Copy `.env.example` to `.env` to customise:

| Variable | Default | Description |
|----------|---------|-------------|
| `REDIS_URL` | `redis://127.0.0.1:6379` | Full Redis connection URL |

When running via Docker Compose the `REDIS_URL` is automatically set to `redis://redis:6379` (the internal service name).

---

## API Reference

### Matching Service

#### `GET /`
Health-check endpoint.

**Response:**
```json
{ "message": "This is matching server of omegle clone." }
```

---

#### `POST /registerForMatching`

Adds a user to the matchmaking queue. Call this before opening the WebSocket.

**Request body:**
```json
{ "name": "alice" }
```

**Responses:**

| `status` | Meaning |
|----------|---------|
| `queued` | User added to the queue successfully |
| `exists` | Username is already in the queue |
| `error` | Internal server error |

```json
{ "status": "queued", "message": "User added to matching queue" }
```

---

#### `WebSocket /ws/{name}`

Persistent connection for receiving match events and sending chat messages. `{name}` must match the name used in the POST above. Keep this open while waiting for a match.

---

### Signalling Service

#### `GET /`
Health-check endpoint.

```json
{ "message": "This is signalling server of omegle clone." }
```

---

#### `WebSocket /ws/{username}`

Persistent connection used for WebRTC signalling once a room code is received. Open this **after** receiving the `matched` event from the Matching Service.

---

## WebSocket Message Reference

### Matching Service Events

#### Server → Client

**`matched`** — sent to both users when a pair is found:
```json
{
  "event": "matched",
  "room_code": "alice_bob",
  "initiator": true
}
```
The user with `"initiator": true` must send the WebRTC **offer**; the other sends the **answer**.

**`chat`** — relayed chat message from peer:
```json
{
  "event": "chat",
  "sender": "bob",
  "message": "Hello!",
  "timestamp": 1700000000.0
}
```

**`peer-disconnected`** — peer left or skipped:
```json
{
  "event": "peer-disconnected",
  "message": "bob has disconnected"
}
```

#### Client → Server

**`chat`** — send a message to the matched peer:
```json
{
  "event": "chat",
  "peer": "bob",
  "message": "Hello!"
}
```

---

### Signalling Service Events

#### Client → Server

**`join`** — verify room membership before signalling:
```json
{
  "event": "join",
  "room_code": "alice_bob",
  "target": "bob",
  "type": "offer"
}
```

**`signal`** — relay a WebRTC payload to the peer:
```json
{
  "event": "signal",
  "room_code": "alice_bob",
  "target": "bob",
  "type": "offer",
  "data": { "sdp": "...", "type": "offer" }
}
```
`type` is one of `"offer"`, `"answer"`, or `"candidate"`.

#### Server → Client

**`verified`** — room membership confirmed:
```json
{
  "event": "verified",
  "room_code": "alice_bob",
  "role": "initiator"
}
```

**`signal`** — forwarded WebRTC payload from peer:
```json
{
  "event": "signal",
  "room_code": "alice_bob",
  "from": "alice",
  "type": "offer",
  "data": { "sdp": "...", "type": "offer" }
}
```

**`error`** — invalid room, missing peer, or role mismatch:
```json
{ "event": "error", "message": "Invalid room or role" }
```

**`peer-disconnected`** — peer dropped from signalling:
```json
{ "event": "peer-disconnected", "message": "alice has disconnected" }
```

---

## End-to-End Connection Flow

```
Alice                      Matching Svc              Bob
  |                             |                     |
  |-- POST /registerForMatching →|                     |
  |-- WS /ws/alice ------------>|                     |
  |                             |<-- POST /register --|
  |                             |<-- WS /ws/bob ------|
  |                             |                     |
  |         (queue has 2 users; matcher fires)        |
  |                             |                     |
  |<-- matched {initiator:true} |  matched {false} -->|
  |                             |                     |
  |      (both open Signalling Service WS)            |
  |                                                   |
Alice                   Signalling Svc              Bob
  |-- join {room_code, type:"offer"} -->|             |
  |<-- verified {role:"initiator"} -----|             |
  |                                     |<-- join ----|
  |                                     |--> verified |
  |                                                   |
  |-- signal {type:"offer", data:SDP} ->|             |
  |                                     |-- signal -->|
  |                                     |<-- signal --|
  |<-- signal {type:"answer"} ----------|             |
  |                                                   |
  |  (ICE candidates exchanged the same way)          |
  |                                                   |
  |<========= Peer-to-peer WebRTC video call ========>|
```

---

## Communication Flow

1. User connects to the **Matching Server**
2. User is added to the Redis queue
3. Two users are matched
4. Both users receive a **room code**
5. Users connect to the **Signalling Server** using the room code
6. WebRTC peer connection is established
7. Video call and chat begin

---

## Tech Stack

### Backend
- Node.js / Python  
- Redis (Queue and State)  
- WebSockets  
- WebRTC  

### Frontend
- React  
- WebRTC APIs  
- WebSocket Client  

---

## Frontend Repository
The React + WebRTC frontend lives in a separate repository:

**https://github.com/aryan9867bar/randomvideochat-web**

Point the frontend environment variables at:
- Matching Service: `ws://localhost:8000`
- Signalling Service: `ws://localhost:8001`

---
---

## Future Improvements
- Matchmaking filters (region, language, interests)
- Moderation and reporting
- Reconnect logic
- TURN server support
- Rate limiting and abuse prevention

---

## License
MIT License

---

## Contributing
Pull requests are welcome.  
Feel free to open issues for bugs or feature requests.
