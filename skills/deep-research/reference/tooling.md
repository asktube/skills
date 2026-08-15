# tooling — required clients and their tool mappings

The phase files preserve Claude Code's exact names where that is the active runtime. This file is the
translation layer for every other client: Codex, Cursor, and other IDEs use the same asktube MCP and
Chrome DevTools MCP contract. Read the selected client row before entering a phase.

## The two hard requirements

**1. The asktube MCP server**, authenticated as the user. Everything in `reference/asktube.md`
(`me_get`, `videos_search`, `videos_get`, `captions_get`, `channels_*`, `app_playlists_*`,
`captions_prioritize`, `captions_status`) is an MCP tool on that server. Tool *names* are the same
everywhere; only the way your client loads them differs.

**2. Browser automation, for discovery on youtube.com.** Discovery is the *only* thing the browser
does here. Claude Code requires Claude in Chrome. Codex, Cursor, and other IDEs require
[Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp). Caption capture runs
on asktube's backend — `captions_prioritize` queues it and `captions_status` reports it — so nothing
has to be open, focused or logged in for a capture to complete.

- The automation must be able to run JavaScript in the page: the search-results extractor in
  `phases/explore.md` is the only thing that returns video **ids**, and it reads the rendered DOM.
- Driving the user's own Chrome is required outside Claude Code. YouTube meets clean profiles with
  consent and bot interstitials, and the related/recommended sidebar is personalized — a fresh
  profile gets a different, usually worse, page. A clean-profile Playwright is not a supported
  replacement.
- **Outside Claude Code, Chrome DevTools MCP is required.** Run it with `--autoConnect` and it
  attaches to your *already-running Chrome's default profile* — the logged-in browser this step
  wants, with no `--remote-debugging-port` juggling (that flag is the fallback; see *Chrome DevTools
  MCP setup* below). `evaluate_script` runs the extractor. It takes a **function**, so wrap the
  `explore.md` extractor as `async () => { …; return { … }; }` — it already `await`s and returns an
  object; only the outer form differs from Claude's raw-body `javascript_tool`.

If either required MCP server is unavailable, stop before starting the run and name the missing
requirement. This skill does not offer a library-only mode.

## Capability map

| Capability | Claude Code | Codex, Cursor, and other IDEs |
|---|---|---|
| Load MCP tools | One `ToolSearch` call per group, e.g. `select:mcp__asktube__me_get,mcp__asktube__videos_search,…`. | Use the configured asktube and Chrome DevTools MCP tools directly; bare asktube names (`me_get`, `videos_search`, …) are the contract. |
| Browser automation | The `claude-in-chrome` MCP: `tabs_context_mcp`, `tabs_create_mcp`, `navigate`, `javascript_tool`. | `chrome-devtools-mcp`: `list_pages` (context), `new_page` (new tab), `navigate_page`, `evaluate_script` (the extractor, wrapped as a function — see above). |
| Browser sequence | `browser_batch` is allowed; every item carries an explicit `tabId` and the batch stops at its first error. | No batch tool: issue `navigate_page` then `evaluate_script` sequentially in a fresh tab. |
| Ask the user | `AskUserQuestion` — once, per Invariant 10. | Use the client's user-question capability — Codex calls it `request_user_input` — once, per Invariant 10. |
| Parallel readers | Sub-agents, one per sub-topic, **never onto the browser**. | Subagents where available, one per sub-topic, **never onto the browser**; otherwise read groups sequentially. The one-question-one-owner rule still applies, as does "don't paste session history into the prompt". |
| Read a file | The `Read` tool. | Any file-read capability. |
| Invocation arguments | `$ARGUMENTS`. | The user's request, verbatim. |
| Waiting | A background `sleep` between polls. | Any non-tight-loop wait; poll every 5–10s. |

## Chrome DevTools MCP setup (Codex, Cursor, and other IDEs)

Two MCP servers, both user-side — nothing is bundled and there is no API key (see *What this skill
does not depend on*). Configure both in your client's MCP settings before running the skill.

1. **asktube**, authenticated as you — the same server Claude Code uses; add it as an MCP server in
   Codex, Cursor, or your chosen IDE.
2. **`chrome-devtools-mcp`**, configured in the same client. Prefer `--autoConnect` so it drives your
   logged-in Chrome. In Cursor, add it in `.cursor/mcp.json` (or Cursor Settings → MCP):

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["chrome-devtools-mcp@latest", "--autoConnect"]
    }
  }
}
```

`--autoConnect` requires a Chrome to already be running and attaches to its default profile — the
logged-in browser discovery wants. If you would rather point at an explicit instance, start Chrome
with `--remote-debugging-port=9222` and swap the arg for `--browser-url=http://127.0.0.1:9222`.
There is **no batch tool**, so where a phase refers to a Claude `browser_batch`, run `navigate_page`
then `evaluate_script` as two sequential calls in a fresh tab (`new_page`) — never the user's active
tab.

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
youtube.com in the browser. If you find yourself reaching for one of those, the phase file is wrong
and that belongs in `retro.md`.
