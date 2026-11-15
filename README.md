# Coding Fry Skills

Agent skills for people who build with AI, from [Coding Fry](https://codingfry.sk). Plain text files you can read before you install them. They work with Claude Code, Codex, Cursor and any agent that reads `SKILL.md` files.

## Skills

**Engineering** (plugin `engineering`)

| Skill | What it does |
| --- | --- |
| [`prelaunch-security-check`](plugins/engineering/skills/prelaunch-security-check/SKILL.md) | Checks an AI-built app against 16 security points before launch, read-only, and writes a plain-language report with fixes. |

**Marketing** (plugin `marketing`): coming next.

## Install

**Claude Code**: type these two commands into Claude Code, one after the other. They install the whole category.

```
/plugin marketplace add bladerik/skills
/plugin install engineering@codingfry
```

Or, in a terminal: `claude plugin marketplace add bladerik/skills` and `claude plugin install engineering@codingfry`.

**Codex**: run these in a terminal (needs the Codex CLI).

```bash
codex plugin marketplace add bladerik/skills
codex plugin add engineering@codingfry
```

**Any other agent** (Cursor, OpenCode, Windsurf, …): open a terminal in your app's folder and run the command below. It needs Node.js (22.20 or newer as of November 2026), asks which skills you want and for which agent, and installs into the current project by default.

```bash
npx skills@latest add bladerik/skills
```

**By hand:** copy a skill folder, e.g. `plugins/engineering/skills/prelaunch-security-check/`, into your agent's skills folder (`~/.claude/skills/`, `~/.agents/skills/`, or `.claude/skills/` in your project).

## Using `prelaunch-security-check`

**Start it yourself.** It never runs on its own: it is a check you run before launch or now and then, not something your agent should start in the middle of other work. Open your project and type:

- Claude Code: `/prelaunch-security-check`
- Codex: `$prelaunch-security-check`
- Other agents: "Use the prelaunch-security-check skill to check my app."

Write in any language; the report comes back in yours.

**What it does:** reads your code and config without changing anything, never prints your keys or passwords, and sends no requests to your live site. The only network step it may propose is a dependency audit (for example `npm audit`, which sends your dependency list to the npm registry), and it asks you first. It checks access control, leaked keys, public files, XSS and SQL injection, uploads, prompt injection, input limits, rate limiting, AI spend caps, e-mail aliases, disposable e-mails, review before deploy, dependencies, server patching, security headers and error handling.

**You get** a report in your language: a summary table, findings by severity with where, what can happen and how to fix it, and short manual tests for what code alone can't prove. Then it offers to fix findings one at a time, each only after you say yes.

It is a practical checklist from a developer, not a penetration test or a security guarantee. A Slovak walkthrough with manual tests is at [codingfry.sk/checklist/bezpecnost](https://codingfry.sk/checklist/bezpecnost).

## Did it help?

These skills are free. Bug reports and ideas are welcome as [issues](https://github.com/bladerik/skills/issues).

## License

MIT
