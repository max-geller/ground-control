# 🦞 Contributing to Ground Control

**Pull up a chair. We saved you a seat.**

I believe that running OpenClaw shouldn't require a trust fund. If you care about lightweight tooling, terminal-first design, and keeping token costs under control — welcome aboard.

---

## Before You Start

Ground Control is an umbrella project. The code lives in the individual tool repos:

- [openclaw-relay](https://github.com/max-geller/openclaw-relay) — Cloudflare Worker for API data collection
- [openclaw-telemetry](https://github.com/max-geller/openclaw-telemetry) — Cost monitoring TUI/CLI
- [openclaw-dispatch](https://github.com/max-geller/openclaw-dispatch) — Task management TUI/CLI

This repo holds the project vision, documentation, and shared assets. Contributions here are welcome for docs, architecture discussions, and new tool proposals. For code contributions, head to the relevant tool repo.

---

## Ways to Contribute

### Report Bugs

Found something broken? Open an issue in the relevant repo. Include your environment (OS, Go version, terminal emulator) and steps to reproduce. No issue is too small — if it tripped you up, it'll trip someone else up too.

### Suggest Features

Have an idea that fits the Ground Control philosophy? Open a discussion or issue. We're especially interested in ideas that close the cost feedback loop, improve agent-CLI efficiency, or extend support to new providers and environments.

### Submit Code

Whether it's a typo fix or a new feature, PRs are welcome.

1. Fork the relevant repo
2. Create a feature branch (`git checkout -b feature/your-feature`)
3. Make your changes
4. Test locally (see individual repo READMEs for setup)
5. Submit a PR with a clear description of **what** you changed and **why**

We don't have a formal code review SLA — this is a hobby project, not a startup. But we'll get to your PR, and we appreciate your patience.

### Improve Documentation

Good docs are as valuable as good code. If you can explain something more clearly, simplify a setup guide, or add examples — please do. Documentation PRs are first-class contributions here.

### Test on New Environments

Running Ground Control on something we haven't tested? Raspberry Pi, ARM, a different VPS provider, FreeBSD, an actual 1960s mainframe? Let us know how it goes. Compatibility reports (even "it just worked") are incredibly helpful.

### Propose a New Tool

Have an idea for a tool that belongs in the Ground Control ecosystem? Open a discussion in this repo. If it's lightweight, terminal-native, agent-friendly, and cost-conscious, it probably fits. We'd rather have a conversation early than review a surprise 3,000-line PR.

---

## The Ground Rules

### 1. Keep It Lightweight

Every dependency is a maintenance burden and a potential token cost. If a feature requires pulling in a massive library or standing up a new service, it probably doesn't belong here. When in doubt, ask: _"Could this be simpler?"_

### 2. Terminal First

If it can't run in a terminal, it doesn't ship. No web UIs, no Electron wrappers, no browser dependencies. The terminal is the interface — for humans via TUI, for agents via CLI.

### 3. Think About Tokens

Every interface an agent touches should be as token-efficient as possible. That's the founding principle of this project. When designing CLI output, ask: _"How many tokens will an agent need to parse this?"_ Fewer is always better.

### 4. SQLite Is the Database

We use SQLite for storage. It's zero-config, file-based, and agent-friendly. If your contribution requires PostgreSQL, Redis, or any external database service, it's probably out of scope.

### 5. Be Kind

This is a hobby project maintained by people who love this stuff. We're not a corporation. Treat every contributor — human or otherwise — with respect. Constructive feedback is welcome; rudeness is not.

---

## Code Style

- **Language:** GoLang (all CLI/TUI tools)
- **TUI Framework:** [Bubble Tea](https://github.com/charmbracelet/bubbletea) + [Lip Gloss](https://github.com/charmbracelet/lipgloss)
- **Formatting:** `gofmt` for Go
- **Commits:** Clear, descriptive commit messages. No need for conventional commits, but tell us what you did and why.
- **Tests:** If you're adding a feature, add tests where reasonable. We're not aiming for 100% coverage, but we'd like to not break things.

---

## Project Structure

```
ground-control/          ← You are here (umbrella repo)
├── README.md            ← Project vision and ecosystem overview
├── CONTRIBUTING.md      ← This file
├── assets/              ← Shared images, banners, character art
└── docs/                ← Architecture docs, guides, ADRs

openclaw-relay/          ← Cloudflare Worker (TypeScript)
openclaw-telemetry/      ← Cost monitoring TUI (Go)
openclaw-dispatch/       ← Task management TUI (Go)
```

---

## Development Setup

Each tool repo has its own setup instructions, but the general requirements are:

- **Go 1.21+** for TUI/CLI tools
- **Node.js 18+** and **Wrangler** for openclaw-relay
- **A Cloudflare account** (free tier works) for relay development
- **SQLite 3** (usually already on your system)
- **A terminal** (obviously)

---

## Recognition

All contributors are recognized. If your PR gets merged, you'll be added to the contributors list in the relevant repo. We also maintain a [Hall of Fame](#hall-of-fame) below for folks who've made significant contributions across the project.

---

## Hall of Fame

| Contributor                                  | Role                 | Focus                             |
| -------------------------------------------- | -------------------- | --------------------------------- |
| [@max-geller](https://github.com/max-geller) | Creator & Maintainer | Architecture, telemetry, dispatch |

_This table grows with the project. Your name belongs here._

---

## Questions?

Not sure where to start? Open a discussion in this repo or reach out. There are no dumb questions — only dumb token spend on bloated tooling.

---

<p align="center">
  <em>"Ground Control to all stations — we are GO for contribution."</em>
</p>
