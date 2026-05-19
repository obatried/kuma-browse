# kuma-browse

Give your coding agent (Claude Code, Codex CLI, Cursor, anything MCP-capable) the ability to drive **your real, logged-in Chrome** — not a sandboxed copy. Reads your Gmail tab, posts on your LinkedIn, scrapes a dashboard you're signed into, debugs the site you're building, all from inside the chat.

This is a setup recipe, not new code. It wires up Google's official [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp) package twice — once to attach to your running Chrome, once to spawn a fresh isolated Chrome — so your agent can pick the right one per task.

## Why two modes

`chrome-devtools-mcp` picks its mode at startup via a CLI flag. The mode is fixed once the server boots; there is no per-call switch. So if you want both available in one session, you register two MCP servers — same package, different flags.

| Server name (suggested) | Flag | What it does | When the agent picks it |
|---|---|---|---|
| `chrome-devtools` | `--autoConnect` | Attaches to your running Chrome (your cookies, sessions, open tabs) | Anything that needs *your* login: Gmail, LinkedIn, your dashboards, your accounts |
| `chrome-devtools-2` | `--isolated` | Spawns a fresh isolated Chrome with no profile | Public sites, scraping, QA where logged-in state doesn't matter |

Either works alone. Running both gives the agent the option to pick per task.

## Prerequisites

- **Google Chrome 144 or newer.** Check at `chrome://version`. Older Chrome doesn't support the auto-connect handshake.
- **Node.js 20.19+ or 22.12+** so `npx` can pull the package. (Check current `engines` in [upstream `package.json`](https://github.com/ChromeDevTools/chrome-devtools-mcp/blob/main/package.json) if you hit a version error.)
- **An MCP-capable agent** — Claude Code, Codex CLI, Cursor, Cline, Continue, etc.

## Install

There is nothing to install globally — `npx` pulls the package on first call. You only have to register the two servers with your MCP client.

### What to register

Two MCP servers, both `stdio`, both calling the same npm package with different flags:

| Field | Server 1 (attach) | Server 2 (isolated) |
|---|---|---|
| Name | `chrome-devtools` | `chrome-devtools-2` |
| Command | `npx` | `npx` |
| Args | `-y chrome-devtools-mcp@latest --autoConnect` | `chrome-devtools-mcp@latest --isolated` |

### JSON shape (common client convention — Claude Code, Cursor, Cline, etc)

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "chrome-devtools-mcp@latest", "--autoConnect"]
    },
    "chrome-devtools-2": {
      "type": "stdio",
      "command": "npx",
      "args": ["chrome-devtools-mcp@latest", "--isolated"]
    }
  }
}
```

This shape isn't part of the MCP spec itself (the spec defines the transport, not how clients store config), but most MCP clients converged on the same `command` + `args` convention. For Claude Code that's `~/.claude.json` (or use `claude mcp add` below). For Codex CLI it's `~/.codex/config.toml` in TOML form. For other clients, check their MCP docs — the `command` + `args` translates the same way.

### One-liner for Claude Code

```bash
claude mcp add chrome-devtools -- npx -y chrome-devtools-mcp@latest --autoConnect
claude mcp add chrome-devtools-2 -- npx chrome-devtools-mcp@latest --isolated
```

The `--` separator before `npx` is required — without it, `claude mcp add` tries to parse `--autoConnect` / `--isolated` as its own flags and fails.

After registering, restart your client — newly added MCPs don't show up in the session that added them.

## Enable Chrome remote debugging (one-time)

The `--autoConnect` server needs Chrome to be willing to accept a debugging session. The way to enable that on a modern Chrome:

1. Open your main Chrome window.
2. Go to `chrome://inspect/#remote-debugging`.
3. Toggle **Discover network targets** (or the equivalent remote-debugging switch) **on**.
4. No restart needed.

**Don't try** `--remote-debugging-port=9222` on the default profile. Chrome 136 and later silently refuse that flag on the default profile for security reasons. The `chrome://inspect` toggle is the supported path.

## First connect

1. After your client starts a new session, the first call into the `chrome-devtools` server will request a CDP session from Chrome.
2. Chrome shows a native permission dialog asking to allow the remote debugging session. Click **Allow**.
3. While the session is active, Chrome shows a "Chrome is being controlled by automated test software" banner at the top. That is normal.
4. Your agent now drives your real Chrome — every cookie, every signed-in account, every open tab.

The `chrome-devtools-2` server doesn't need any of this; it spawns its own Chrome on first use.

## When to use which (routing the agent picks for you)

Most agents will figure this out from the server names + descriptions, but here is the rule in plain text if you want to put it in a system prompt or `CLAUDE.md`:

> Need my logged-in Chrome session (cookies, accounts, open tabs)? → `chrome-devtools` (attach).
> Just need any browser instance? → `chrome-devtools-2` (fresh isolated).
> When uncertain, default to isolated.

Examples that need attach:
- Reply to a Gmail thread on your real account.
- Post to LinkedIn / TikTok Studio / Meta Business / Beehiiv.
- Pull from a dashboard you're signed into.
- Drive a tab you already have open.

Examples that should use isolated:
- QA a public or production site.
- Scrape a competitor or unauth site.
- Test a flow you can re-auth into in a few seconds via SSO.

## Security

Read this part. While remote debugging is on, **any local process on your machine** can drive that Chrome instance — read every page, every cookie, every signed-in account. The debugging port is localhost-only, so nothing remote can reach it, and the toggle is something you flip in Chrome's UI yourself. But anything running on your laptop (a stray dev tool, a sketchy npm postinstall, a malicious VS Code extension) has the same access your agent does, for as long as the toggle is on.

Reasonable hygiene:

- Only enable remote debugging on a profile you're comfortable having a program see.
- If you want hard isolation, run a dedicated Chrome profile (`chrome://settings/manageProfile`) and only flip the toggle on for that one.
- Turn it off when you're not actively using it. One click at `chrome://inspect/#remote-debugging`.

## Known gotchas

These are real failure modes I've hit, not theoretical concerns.

**"The selected page has been closed."** The attach server tracks one "selected page" as implicit context. If that page closes, navigates to a redirecting URL, or crashes, subsequent calls can return this error — including `list_pages`, which should let you recover and sometimes doesn't. Once it sticks, the server usually can't self-heal and you need to restart it (restart your client, usually). Workaround: prefer opening new tabs over `navigate_page` on URLs that might redirect.

**Duplicate config entries — last wins.** If you accidentally have two `chrome-devtools` entries (one `--autoConnect`, one `--isolated`), only the last one is registered. Symptom: you thought you had attach mode but you're in isolated. Check your config for duplicates.

**`fill` can silently no-op on React-controlled textareas.** Big controlled `<textarea>` elements (think paste-a-transcript boxes) sometimes visually show the filled value but never fire React's `onChange`, so Submit stays disabled. Workaround: use `evaluate_script` with the native setter pattern:

```js
const ta = document.querySelector('textarea[placeholder*="..."]');
const setter = Object.getOwnPropertyDescriptor(window.HTMLTextAreaElement.prototype, 'value').set;
setter.call(ta, '<your text>');
ta.dispatchEvent(new Event('input', { bubbles: true }));
```

Single-line `<input>` usually works fine; the trick is for controlled multi-line textareas and contenteditable editors (LinkedIn DMs, etc).

**Old `--remote-debugging-port=9222` flag.** Doesn't work on the default profile in Chrome 136+. Use the `chrome://inspect` toggle instead. The flag still works on a non-default `--user-data-dir`, but then you lose your logins, which defeats the point.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `chrome-devtools` calls fail with no clear error | Chrome remote debugging is off | Re-toggle at `chrome://inspect/#remote-debugging` |
| MCP "connected" but every tool call fails | Selected-page bug after a redirect | Restart your client |
| Permission dialog never appears in Chrome | Chrome version below 144 | Update Chrome |
| Both servers show up but agent picks the wrong one | Server names not descriptive enough for your agent | Add the routing rule to your system prompt / `CLAUDE.md` |
| Fresh isolated Chrome doesn't have an extension you need | That's by design | Use the attach server (`chrome-devtools`) for that flow |

## What this is built on

- [`chrome-devtools-mcp`](https://github.com/ChromeDevTools/chrome-devtools-mcp) — Google's official MCP server for Chrome DevTools Protocol. All credit there.
- [Model Context Protocol](https://modelcontextprotocol.io) — the spec your agent uses to talk to MCP servers.

## License

MIT. See [LICENSE](./LICENSE).
