# asktube skills

Agent Skills that turn the YouTube you already follow into work you can use — grounded in real
captions, via [asktube](https://asktube.xyz).

Every claim these skills produce traces to a video id and a real timestamp, because they read
transcripts rather than recalling them.

| Skill | What it does |
|---|---|
| [`deep-research`](skills/deep-research) | Deep research on any topic: mines your asktube library, tunes YouTube discovery from what it finds, curates a corpus, reads the transcripts, and writes the deliverable in whatever shape you asked for — a report, a study plan, a one-pager, a shortlist. |

## Install

**Claude Code** — native marketplace, no CLI needed:

First, add the marketplace:

```
/plugin marketplace add asktube/skills
```

Then, add the deep research skill:

```
/plugin install asktube@asktube-skills
```

Then invoke `/asktube:deep-research <topic>`, or just ask for research and let it trigger.

**Everything else** — Cursor, Codex, Gemini CLI, opencode, Copilot, Windsurf, Amp, Goose, Kiro and
~50 more:

```
npx skills add asktube/skills
```

Add `--list` to see what's in the repo, `--skill deep-research` to take one, `-g` for a global
install.

## Prerequisites

Two MCP servers, both of which you set up yourself — the skill bundles neither. It checks both
before it starts and stops with instructions if either is missing, rather than failing halfway
through a run.

1. **asktube**, with some of your YouTube library tracked. Sign up at
   [asktube.xyz](https://asktube.xyz) and copy the MCP configuration from your dashboard
   (Settings → MCP) into your agent.
2. **A Chrome the agent can drive**, for discovery on YouTube — your own logged-in browser, not a
   fresh profile. [Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp) with
   `--autoConnect` is the default everywhere; it attaches to a Chrome that is *already open*, so
   start one first, and it needs Node. On Claude Code, **Claude in Chrome works as a fallback** if
   you'd rather not add DevTools. Caption capture happens on asktube's backend — no tab to keep open
   or in focus for that. The skill's
   [`reference/tooling.md`](skills/deep-research/reference/tooling.md) has the exact config.

Corpora are English-language for now; asktube's multi-language support is still in progress, so a
run prefers English sources and says so in its coverage section.

No YouTube Data API key. No other plugins. Nothing to configure beyond those two servers.

## Adding a skill

See [AGENTS.md](AGENTS.md) — five rules, and the one that will bite you is rule 2.

## License

MIT — see [LICENSE](LICENSE).
