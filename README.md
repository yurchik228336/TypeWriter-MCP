# TypeWriter MCP

MCP server for the [Typewriter](https://typewriter.gg/) Minecraft plugin. It lets AI assistants in Cursor, Claude Desktop, and other MCP clients read and edit storylines: pages, dialogues, facts, triggers, and the rest of the entry graph.

You can work live against a running server over websocket, or edit page JSON files offline.

## What it does

Typewriter stores story content as pages and entries. The web editor is fine for manual work, but slow when you want an agent to draft whole branches, fix dialogue, or wire up triggers across dozens of entries. This server exposes that data as MCP tools so the assistant can do the editing for you.

In socket mode it uses the same protocol as the official web editor. Changes land in staging first, show up in the panel, and can be published to production without clobbering work you have open elsewhere.

## Modes

**Socket (recommended for live servers)**

Connect to the running plugin over websocket. Best for production and any setup where you want changes visible in the web panel before they go live.

**File (offline)**

Edit `plugins/Typewriter/pages/*.json` directly. Useful when the server is down or you want to batch-edit files and reload later. Run `/typewriter reload` on the server after saving.

## Install

```bash
git clone https://github.com/yurchik228336/TypeWriter-MCP.git
cd TypeWriter-MCP
npm install
npm run build
```

Requires Node.js 18+.

## Cursor / Claude Desktop config

Add this to your MCP config (for Cursor: `~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "typewriter": {
      "command": "node",
      "args": ["C:/path/to/TypeWriter-MCP/dist/index.js"],
      "env": {
        "TYPEWRITER_MODE": "socket",
        "TYPEWRITER_URL_FILE": "C:/path/to/TypeWriter-MCP/.typewriter-session-url"
      }
    }
  }
}
```

With this setup the server starts in socket mode and waits for a connect URL. When you want to work live, run `/typewriter connect` in-game and paste the full link into chat. The agent calls `connect_live` and keeps the socket open for the rest of the session. You do not need to paste the link again while that connection is still alive.

If you prefer a fixed URL in config instead:

```json
"env": {
  "TYPEWRITER_MODE": "socket",
  "TYPEWRITER_URL": "http://host:8080/#/connect?host=1.2.3.4&port=9092&token=<uuid>"
}
```

Note: with `auth: session`, that token expires when the player who generated it logs out.

**File mode:**

```json
"env": {
  "TYPEWRITER_MODE": "file",
  "TYPEWRITER_PAGES_DIR": "C:/server/plugins/Typewriter/pages"
}
```

## Typewriter plugin setup (socket)

1. Enable websocket in `plugins/Typewriter/config.yml`:

```yaml
enabled: true
websocket:
  port: 9092
  auth: session
```

Restart the server after editing the config.

2. In-game, run `/typewriter connect`. You get a URL like:

```
http://host:8080/#/connect?host=1.2.3.4&port=9092&token=<uuid>
```

3. Paste that URL to your agent or set it in `TYPEWRITER_URL`.

Plain websocket URLs work too: `ws://host:9092?token=<uuid>`

For a trusted local network you can set `auth: none` and skip tokens. Only do that if you know what you are exposing.

## Typical agent workflow

1. Paste the connect URL when starting a live session.
2. `list_pages` / `read_page` to see what exists.
3. `create_page`, `add_entry`, `connect_entries` to build or extend the story.
4. `validate_storyline` to catch broken trigger references.
5. `publish` to apply staging changes in-game (socket mode only).

If the session drops or the token expires, send a fresh connect URL.

## Tools

| Tool | Description |
|------|-------------|
| `connection_status` | Current mode, connection state, saved URL |
| `connect_live` | Connect using a `/typewriter connect` URL |
| `list_pages` | List pages (id, name, type, entry count) |
| `read_page` | Read a full page by name or id |
| `create_page` | Create a page (`sequence`, `static`, `cinematic`, `manifest`) |
| `rename_page` | Rename a page |
| `delete_page` | Delete a page and all its entries |
| `add_entry` | Add an entry (dialogue, speaker, fact, etc.) |
| `update_entry` | Update one field by dotted path (`text`, `speaker`, `criteria.0.value`) |
| `replace_entry` | Replace an entry, keeping the same id |
| `delete_entry` | Delete an entry by id |
| `connect_entries` | Append a trigger link between two entries |
| `search_entries` | Full-text search across all pages |
| `validate_storyline` | Find broken references in triggers / triggeredBy |
| `publish` | Publish staging to production (socket mode only) |

## Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `TYPEWRITER_MODE` | `socket` | `socket` or `file` |
| `TYPEWRITER_URL` | — | Optional fixed connect URL |
| `TYPEWRITER_URL_FILE` | `.typewriter-session-url` in cwd | Where the last connect URL is stored |
| `TYPEWRITER_PAGES_DIR` | `plugins/Typewriter/pages` | Pages directory for file mode |

## Not affiliated

This is an unofficial community tool. Typewriter is developed by its own team; this MCP just speaks their websocket and file formats.
