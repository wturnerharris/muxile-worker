# Muxile

Control a tmux session from your phone via a Cloudflare Worker + Durable Objects.

## Architecture

```
Phone (browser) <--> WebSocket <--> Cloudflare Worker <--> Durable Object (ChatRoom)
                                                         |
                                                    tmux client (WebSocket)
```

### Components

**Worker (`src/muxile.mjs`)** routes HTTP requests:
- `GET /` — redirects to this GitHub repo
- `GET /join/<room-id>` — serves the mobile terminal UI (`term.html`)
- `GET /api/room/<room-id>/websocket` — upgrades to WebSocket, forwarded to a Durable Object

**Durable Object (`ChatRoom`)** — each room ID gets its own persistent instance that:
- Manages WebSocket sessions between connected clients
- Broadcasts messages between all participants (phone and tmux)
- Stores the last message in durable storage so reconnecting clients see the latest terminal state

**Frontend (`src/term.html`)** — minimal mobile UI with:
- A `<pre>` block showing tmux terminal output (received as base64-encoded HTML)
- A text input + Ctrl toggle button for sending keystrokes

## Message Flow

1. **tmux client** connects to `/api/room/<id>/websocket`, identifies as `{"name": "tmux"}`
2. **Phone** opens `/join/<id>`, connects to the same room, identifies as `{"name": "phone"}`
3. **Phone to tmux**: user types a command, sends `{"message": "ls"}` — the Durable Object broadcasts it to all connected clients including the tmux client
4. **tmux to phone**: tmux client sends `{"name": "tmux", "message": "<base64-html>"}` — the phone decodes it and renders the terminal output
5. **Ctrl modifier**: the Ctrl button toggles on, and the next Enter sends `"c-<key>"` (e.g., `c-c` for Ctrl+C), which the tmux client interprets

### When tmux disconnects

- The phone receives `{"quit": "tmux"}` and dims the terminal (opacity 0.5)
- The Durable Object clears stored history

## Setup

### Prerequisites

- A Cloudflare account
- Node.js installed

### Configure

1. Set your `account_id` in `wrangler.toml` (find it in Cloudflare dashboard under Workers & Pages)
2. Log in to Cloudflare:
   ```bash
   npx wrangler login
   ```

### Develop locally

```bash
npx wrangler dev
```

Then open `http://localhost:8787/join/<any-room-id>` in your browser.

### Deploy

```bash
npx wrangler deploy
```

The worker will be available at `https://muxile.<your-subdomain>.workers.dev`.

## Testing WebSocket connections

```bash
npx wscat -c "ws://localhost:8787/api/room/<room-id>/websocket"
```

Once connected, send the initial identity message:

```json
{"name": "test"}
```

Then send commands:

```json
{"message": "hello"}
```

Use `ws://` for local dev, `wss://` for production.
