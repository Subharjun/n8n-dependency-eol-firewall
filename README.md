# 🛡️ Dependency Deprecation & EOL Firewall — an n8n multi-agent workflow

An **end-to-end n8n workflow** that watches a GitHub repo's dependencies and, when one becomes
**deprecated, end-of-life, or a major version behind**, it reads *your actual code* to gauge the
blast radius, drafts a migration plan, and **opens a pull request (or files an issue)** — automatically.

Unlike Dependabot (which bumps versions blindly), this reasons about **your specific usage** before
telling you to act, using three cooperating LLM agents.

> Works with any LLM provider node in n8n. This repo ships configured for **Groq** (`openai/gpt-oss-120b`),
> but you can swap in OpenAI, Anthropic, Ollama, etc. by replacing the model node.

---

## Why this exists

Auto-upgrade bots open noisy version-bump PRs with no judgment. Security scanners flag CVEs but not
*"this library is abandoned"* or *"you're one breaking major behind."* This workflow closes that gap:
a conservative triage agent decides what's genuinely worth acting on, a second agent greps your repo to
size the work, and a third writes a ready-to-review PR. On a healthy repo it does **nothing** — no spam.

---

## How it works

```
Schedule (weekly)
      │
Config: owner/repo          ← the ONE place you set the target repo
      │
Read package.json  →  Parse Manifest (1 → N deps)  →  npm Registry Lookup  →  Assess Registry Facts
      │
🤖 Agent 1 — Watcher/Triage      ← is this a real deprecation / EOL / major-gap risk?  (conservative)
      │
Only At-Risk  (filter)
      │
Search Repo For Usage  →  Summarize Usage
      │
🤖 Agent 2 — Blast-Radius        ← which symbols do YOU actually use? how much breaks?
      │
🤖 Agent 3 — Migration Drafter   ← writes the PR/issue: summary, risk, upgrade cmd, migration plan, diff
      │
Prep PR Vars → Get Default Branch → Get Base SHA → Create Branch → Bump package.json (deterministic) →
Commit → Open Pull Request
```

**Design principle — LLMs judge, code edits.** The agents only *reason* (risk verdict, blast radius,
migration notes). The actual `package.json` version bump is done by a deterministic regex, so the diff
is always correct and range-prefix-safe (`^`, `~` preserved).

---

## Two variants

| File | Delivery | Use when |
|------|----------|----------|
| [`dependency-eol-firewall-autopr.json`](dependency-eol-firewall-autopr.json) | Opens a **Pull Request** (branch + commit + PR) | You want hands-off, mergeable upgrades |
| [`dependency-eol-firewall.json`](dependency-eol-firewall.json) | Files a **GitHub Issue** | You want a heads-up without touching code |

Both are self-contained and importable.

---

## Requirements

- An **n8n** instance (Cloud or self-hosted; Community edition is fine — no Enterprise features used).
- A **Groq API key** — https://console.groq.com (free tier works). *Or swap the model node for another provider.*
- A **GitHub Personal Access Token** with `repo` scope — https://github.com/settings/tokens

---

## Setup (5 minutes)

1. **Import** the workflow: n8n → *Workflows* → *Import from File* → pick one of the JSON files.
2. **Create two credentials** and select them on the highlighted nodes (they import with placeholder IDs):
   - **Groq** → on the three `Claude — …` model nodes.
   - **GitHub API** → on `Read package.json`, `Search Repo For Usage`, and all `GH: …` nodes
     (server `https://api.github.com`, your username, your PAT).
3. **Set your target repo:** open the **`Config: owner/repo`** node and change `owner` and `repo`.
   That's the only place you configure the repo — every GitHub node reads from it.
4. **Run it:** click *Execute Workflow* to test, then toggle the workflow **Active** for the weekly schedule.

That's it. No editing per-node, and it auto-detects the repo's default branch (`main`, `master`, anything).

---

## Notes & tuning

- **Filter strictness:** the `Only At-Risk` node currently passes *any* dependency the Watcher flags.
  To reduce noise on large repos, add a severity condition (`severity != low`).
- **One PR per outdated dependency**, per run. Merge/close as you go.
- **Ecosystem:** ships for **npm (`package.json`)**. A PyPI/`requirements.txt` variant is a small change
  to the manifest-parse + registry-lookup nodes (PyPI JSON API + `endoflife.date`).
- **Model choice:** `openai/gpt-oss-120b` on Groq is fast and supports structured output. Bump to a
  larger model for the Drafter if you parse dense legal/compiler changelogs.

---

## Security

- The shipped JSON files contain **no secrets** — only placeholder credential IDs. Your keys live in
  n8n's credential store, never in the workflow export.
- Keep the auto-PRs as *drafts you review* on important repos; don't auto-merge.

---

## License

MIT — see [LICENSE](LICENSE). Contributions welcome.
