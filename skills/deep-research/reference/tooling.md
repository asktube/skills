# tooling — required servers, how to check them, and their tool mappings

Two MCP servers are required and **neither is bundled with this skill** — you configure both, in
whatever client you run. This file is what the preflight in `SKILL.md` and the phase files read:
which call proves a server is live, what to say when one isn't, and which tool name stands for which
capability in your client.

## The two hard requirements

**1. The asktube MCP server**, authenticated as you. Everything in `reference/asktube.md`
(`me_get`, `videos_search`, `videos_get`, `captions_get`, `channels_*`, `app_playlists_*`,
`captions_prioritize`, `captions_status`) is an MCP tool on that server. Tool *names* are the same
everywhere; only the way your client loads them differs. There is no fallback and no library-only
mode — without it the run stops.

**2. Browser automation, for discovery on youtube.com.** Discovery is the *only* thing the browser
does here. Caption capture runs on asktube's backend — `captions_prioritize` queues it and
`captions_status` reports it — so nothing has to be open, focused or logged in for a capture to
complete.

- **Chrome DevTools MCP is the default**, in every client including Claude Code.
- **Claude in Chrome is the fallback, and only on Claude Code.** If DevTools isn't connected there,
  use it. Everywhere else, no DevTools means no run.
- The automation must be able to run JavaScript in the page: the search-results extractor in
  `phases/explore.md` is the only thing that returns video **ids**, and it reads the rendered DOM.
- It must drive **the user's own Chrome**. YouTube meets clean profiles with consent and bot
  interstitials, and the related/recommended sidebar is personalized — a fresh profile gets a
  different, usually worse, page. A clean-profile Playwright is not a supported replacement.

## Preflight probes

Run these before the first phase. `SKILL.md` carries the halt copy; this is the mechanism.

| Check | Claude Code | Codex, Cursor, other IDEs |
|---|---|---|
| Are the tools there? | `ToolSearch` with a required-term query — `+asktube`, `+chrome-devtools`, `+claude-in-chrome`. A missing server returns `No matching deferred tools found`, which is an unambiguous negative. | The client either exposes the tool or it doesn't. |
| asktube is live | `me_get({ include: ["access", "usage"] })` | same |
| DevTools is live | `list_pages` | same |
| Claude in Chrome is live | `tabs_context_mcp` | n/a — not available outside Claude Code |

`me_get` is the right asktube probe because it separates *not configured* from *stale config* from
*account without MCP access*, and hands back the library counters for free. `list_pages` is the
right browser probe because `--autoConnect` needs a Chrome that is **already running**, so it
catches an attached-to-nothing server that a presence check would wave through.

**A bad or missing asktube config reads as "failed to connect", not as a sign-in prompt** — the
server answers `401` with nothing for a client to negotiate against. So when asktube shows unhealthy,
suspect the configuration before the network, and re-copy it from the dashboard.

## Capability map

Browser rows depend on what the preflight resolved into `BROWSER`.

| Capability | `BROWSER = devtools` (default) | `BROWSER = cic` (Claude Code fallback) |
|---|---|---|
| Browser context | `list_pages` | `tabs_context_mcp` |
| New tab | `new_page` | `tabs_create_mcp` |
| Navigate | `navigate_page` | `navigate` |
| Run the extractor | `evaluate_script` — takes a **function**; use the function-form block in `explore.md` | `javascript_tool` — **REPL semantics**, the last expression is the result; use the REPL-form block in `explore.md` |
| Sequence | No batch tool: one operation at a time | `browser_batch` is allowed; every item carries an explicit `tabId` and the batch stops at its first error |

Everything below is harness, not browser, and varies by client rather than by `BROWSER`:

| Capability | Claude Code | Codex, Cursor, and other IDEs |
|---|---|---|
| Load MCP tools | One `ToolSearch` call per group, e.g. `select:mcp__asktube__me_get,mcp__asktube__videos_search,…`. | Use the configured tools directly; bare asktube names (`me_get`, `videos_search`, …) are the contract. |
| Ask the user | `AskUserQuestion` — once, per Invariant 10. | The client's user-question capability — Codex calls it `request_user_input` — once, per Invariant 10. |
| Parallel readers | Sub-agents, one per sub-topic, **never onto the browser**. | Subagents where available, one per sub-topic, **never onto the browser**; otherwise read groups sequentially. The one-question-one-owner rule still applies, as does "don't paste session history into the prompt". |
| Read a file | The `Read` tool. | Any file-read capability. |
| Invocation arguments | `$ARGUMENTS`. | The user's request, verbatim. |
| Waiting | A background `sleep` between polls. | Any non-tight-loop wait; poll every 5–10s. |

## Setup

Both servers are user-side in every client. Nothing here is installed by the skill.

### asktube — get it from your dashboard

1. Sign up at [asktube.xyz](https://asktube.xyz) and track some of your YouTube library.
2. Copy the MCP configuration from your dashboard (Settings → MCP).
3. Add it to your client and restart.

The dashboard is the single source of truth for the endpoint, transport and credential, so this file
deliberately doesn't restate them — a copy here would be one more thing to go stale, and a stale
connection detail fails silently at the first tool call.

### chrome-devtools-mcp

Prefer `--autoConnect`: it attaches to your already-running Chrome's default profile — the logged-in
browser discovery wants — with no `--remote-debugging-port` juggling.

```bash
# Claude Code
claude mcp add chrome-devtools -- npx chrome-devtools-mcp@latest --autoConnect
```

```toml
# Codex — ~/.codex/config.toml
[mcp_servers.chrome-devtools]
command = "npx"
args = ["chrome-devtools-mcp@latest", "--autoConnect"]
```

```json
// Cursor — .cursor/mcp.json, or Cursor Settings → MCP
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["chrome-devtools-mcp@latest", "--autoConnect"]
    }
  }
}
```

`--autoConnect` requires a Chrome to already be running. To point at an explicit instance instead,
start Chrome with `--remote-debugging-port=9222` and swap the arg for
`--browser-url=http://127.0.0.1:9222`.

### Claude in Chrome

Nothing to configure — it is the Chrome extension plus Claude Code's built-in browser tools. It
cannot be installed by a plugin or a CLI, which is why it is the fallback rather than the default.

## Oversized tool output

Claude Code persists a tool response above ~40KB to a file and returns the path plus a ~2KB preview.
The call *succeeded* — the payload is in the file, not the response. On a long `captions_get({ method: "cat" })`
that means: read the persisted JSON back, or grep it for the passages you need, rather than
concluding the transcript is short.

This is **harness behaviour, not asktube's**. asktube returns the whole transcript (recovering
oversized ones from its R2 archive). On another client the response may simply arrive whole; if
yours truncates differently, page with `head` and `nextFrom` — see
[asktube.md](asktube.md) → *Reading captions at length*.

## What this skill does not depend on

No YouTube Data API key. No other skill, plugin or marketplace. No network access beyond asktube and
youtube.com in the browser. The skill installs nothing and bundles nothing: it checks its two
prerequisites and stops if either is missing. If you find yourself reaching for anything else, the
phase file is wrong and that belongs in `retro.md`.
