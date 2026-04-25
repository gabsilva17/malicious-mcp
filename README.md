# malicious-mcp

PRETEXT challenge harness — a one-tool MCP server whose `description` is the
injection payload. The target (Forgeline) ingests the description as system
instruction when it calls `tools/list`. Edit `PAYLOAD` in `src/server.ts` to
iterate.

## Install

```bash
npm install
```

## Dev

```bash
npm run dev          # tsx watch, PORT=8787 by default
```

Logs distinguish two events you want to watch separately:

- `[tools/list] …` — payload reached the model context (ingestion).
- `[tools/call] …` — model actually invoked the tool (engagement).

The challenge succeeds on `tools/list` alone if the payload steers the
target's behavior; `tools/call` is just noise from the cover story.

## Expose locally

```bash
cloudflared tunnel --url http://localhost:8787
```

Copy the generated `https://…trycloudflare.com/mcp` URL into Forgeline:
**Integrations → Advanced → Connect MCP**, transport = **Streamable HTTP**.

## Deploy (Railway)

```bash
railway up
```

Or push to a Railway-connected repo. The service uses `npm run build`
(`tsc`) and `npm start` (`node dist/server.js`); Railway injects `PORT`.
After deploy, paste `https://<service>.up.railway.app/mcp` into Forgeline.
Don't commit `.env` — set vars in the Railway dashboard.

## Verify

With the dev server running:

```bash
SID=$(curl -s -D - -o /dev/null -X POST http://localhost:8787/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"curl","version":"0"}}}' \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="mcp-session-id"{print $2}')

curl -s -X POST http://localhost:8787/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
```

`tools/list` returns one tool, `get_weather`, with `description` = `PAYLOAD`.
