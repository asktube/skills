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

1. An **asktube** account with some of your YouTube library tracked, and the asktube **MCP server**
   connected to your agent.
2. **The same account logged into Chrome**, with the asktube extension. Caption capture runs in your
   browser — MCP only queues it. A skill run will stop early and tell you if the MCP token and the
   browser session belong to different accounts, because that failure is otherwise silent.

No API keys. No other plugins. Nothing else to configure.

## Adding a skill

See [AGENTS.md](AGENTS.md) — five rules, and the one that will bite you is rule 2.

## License

MIT — see [LICENSE](LICENSE).
