# Ever — control your browser from any AI agent

[Ever](https://foreverbrowsing.com) lets your AI coding agent drive a **real Chrome browser** — navigate, read the page, click, type, extract data, screenshot, and run autonomous web tasks. It works with Claude Code, Codex, Cursor, Gemini CLI, and ~30 other agents.

This repo is the install hub: a **plugin** (skill + MCP server) for Claude Code & Codex, a **skill** for every other agent, and the `@everbrowser/cli` package that powers them all.

---

## Prerequisites (required for everyone)

Ever automates **your** Chrome, so two one-time steps are needed no matter how you install:

1. **Install the Chrome extension** → **[Ever on the Chrome Web Store](https://chromewebstore.google.com/detail/ever/codfpjgkcdackkjhjhfefmfbckijjjnf)** → *Add to Chrome*
2. **Sign in** → click the Ever icon in the toolbar → the side panel opens → **sign in with Google**

> If a command later says the browser is "not connected," it almost always means the extension isn't installed or you're not signed in — do the two steps above and retry.

---

## Install

### ⭐ Claude Code & Codex — install the plugin (recommended)

One step wires up both the skill **and** the native `browser_*` tools (via MCP):

**Claude Code**
```
/plugin marketplace add namuh-eng/everbrowser
/plugin install ever-browser@everbrowser
```

**Codex**
```bash
codex plugin marketplace add namuh-eng/everbrowser
```
Then enable **ever-browser** in the Codex plugin directory and restart.

### Any other agent (Cursor, Gemini CLI, Copilot, Amp, …) — install the skill

```bash
npx skills add namuh-eng/everbrowser
```
Installs to the universal `.agents/skills/` directory (covers Cursor, Codex, Gemini CLI, GitHub Copilot, Amp, Cline, OpenCode, Warp, and [30+ more](https://github.com/nicepkg/nice-skills)), with Claude Code, Augment, and Continue available during setup.

### Just want the CLI or a raw MCP server?

```bash
npm install -g @everbrowser/cli   # gives you `ever` and `ever mcp`
ever --version
```
Point any MCP client at `ever mcp` (stdio). Full command reference: **[@everbrowser/cli on npm](https://www.npmjs.com/package/@everbrowser/cli)**.

---

## Verify it works

```bash
ever start --url https://example.com
ever snapshot
ever stop
```
If all three succeed, you're set. (If `start` reports "not connected," revisit the two prerequisite steps.)

---

## How it works

```
your AI agent ──▶ Ever (CLI commands  or  MCP browser_* tools)
                       └──▶ everd (local daemon) ──▶ Chrome extension (CDP) ──▶ your browser
```

- **Skill** = teaches the agent to run `ever` **CLI commands** in the terminal.
- **MCP** = gives the agent **native `browser_*` tools** it calls directly (`ever mcp`).
- **Plugin** = ships **both** in one install — best for Claude Code & Codex.

All paths drive the same daemon → extension → browser, and all need the prerequisites above.

---

## Core workflow

1. `ever start --url <url>` — open a session
2. `ever snapshot` — capture the DOM with `[id]` annotations on interactive elements
3. `ever click <id>` / `ever input <id> "text"` — act on elements by id
4. Re-snapshot after the page changes — old ids become stale
5. `ever stop` — end the session

For scripted automation and scraping, use `ever exec` (injects `page` + `browser` JS globals):
```bash
ever exec "await page.goto('https://example.com'); await browser.snapshot()"
ever exec --file ./scrape.js
```

Run `ever --help` for the full command set, or see the [npm README](https://www.npmjs.com/package/@everbrowser/cli).

---

## Available skills

| Skill | Description |
|-------|-------------|
| [ever-browser](./skills/ever-browser/) | Browser control commands, the `exec` scripting API, video recording, and scraping recipes |

---

## Contributing a skill or recipe

A skill is just a folder with a `SKILL.md` and optional recipe scripts.

1. **Get it working** interactively first with `ever` commands and `ever exec`.
2. **Save reusable scripts** under `skills/<name>/recipes/` (see the [x-feed-scraper recipe](./skills/ever-browser/recipes/x-feed-scraper.js)).
3. **Write `SKILL.md`** with frontmatter (`name`, `description`) and clear sections: Quick Start, Core Workflow, Recipes, Error Recovery.
4. **Open a PR** to [namuh-eng/everbrowser](https://github.com/namuh-eng/everbrowser), or share from your own repo via `npx skills add <you>/<repo>`.

---

If Ever is useful, a ⭐ on [the repo](https://github.com/namuh-eng/everbrowser) helps others find it.
