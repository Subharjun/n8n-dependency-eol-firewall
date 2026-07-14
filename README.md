# 🛡️ Dependency Deprecation & EOL Firewall — an n8n multi-agent workflow

An **end-to-end n8n workflow** that watches a GitHub repo's dependencies and, when one becomes
**deprecated, end-of-life, or a major version behind**, it reads *your actual code* to gauge the
blast radius, drafts a migration plan, **rewrites the affected source files**, and **opens a
merge-ready pull request** — automatically.

Unlike Dependabot (which bumps versions blindly), this reasons about **your specific usage** before
telling you to act, using **four cooperating LLM agents** — and the fourth one actually applies the
code change, so the PR isn't just a version bump with a to-do list.

> Works with any LLM provider node in n8n. This repo ships configured for **Groq** (`openai/gpt-oss-120b`),
> but you can swap in OpenAI, Anthropic, Ollama, etc. by replacing the model node.

---

## Why this exists

Auto-upgrade bots open noisy version-bump PRs with no judgment. Security scanners flag CVEs but not
*"this library is abandoned"* or *"you're one breaking major behind."* Worse, a blind bump can break
your app — e.g. bumping `chalk` to 5.x (ESM-only) while your code still `require()`s it throws
`ERR_REQUIRE_ESM` the moment it's merged.

This workflow closes that gap: a conservative triage agent decides what's genuinely worth acting on,
a second agent reads your repo to size the work, a third writes a ready-to-review PR body, and a
fourth **rewrites the affected source files** (e.g. `require()` → `import`, adds `"type":"module"`
when needed) so the PR is actually mergeable. On a healthy repo it does **nothing** — no spam.

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
Find Usage In Repo               ← fetch the git tree + grep source files for the dependency
      │
🤖 Agent 2 — Blast-Radius        ← which symbols do YOU actually use? how much breaks?
      │
🤖 Agent 3 — Migration Drafter   ← writes the PR body: summary, risk, upgrade cmd, migration plan
      │
Prep PR Vars → Get Default Branch → Get Base SHA → Create Branch →
Bump package.json (deterministic) → Commit
      │
🤖 Agent 4 — Migration Coder     ← rewrites each affected source file for the new version
      │
Split / commit each rewritten file (+ "type":"module" when an ESM migration is detected)
      │
Open Pull Request                ← one merge-ready PR per outdated dependency
```

**Design principle — LLMs judge, code edits.** The agents *reason* (risk verdict, blast radius,
migration notes) and *propose* the new file contents. But the actual `package.json` version bump is
done by a deterministic regex, so the version diff is always correct and range-prefix-safe
(`^`, `~` preserved). Agent 4 returns the **full rewritten file** as plain text (never JSON-wrapped),
and deterministic code commits it — so a mangled tool-call can't corrupt your source.

---

## The four agents

| Agent | Job | Output |
|-------|-----|--------|
| **1 — Watcher/Triage** | Is this a genuine deprecation / EOL / major-gap risk? | `{at_risk, risk_type, severity, summary}` |
| **2 — Blast-Radius** | Which symbols/files does the repo actually use? | `{used_symbols, files_touched, breaking_calls, estimated_effort}` |
| **3 — Migration Drafter** | Write the human-readable PR description | GitHub-flavored Markdown |
| **4 — Migration Coder** | Rewrite the affected source files for the new version | Full file contents (committed deterministically) |

---

## Requirements

- An **n8n** instance (Cloud or self-hosted; Community edition is fine — no Enterprise features used).
- A **Groq API key** — https://console.groq.com (free tier works). *Or swap the model node for another provider.*
- A **GitHub Personal Access Token** with `repo` scope — https://github.com/settings/tokens

---

## Setup (5 minutes)

1. **Import** the workflow: n8n → *Workflows* → *Import from File* → `dependency-eol-firewall-autopr.json`.
2. **Create two credentials** and select them on the highlighted nodes (they import with placeholder IDs):
   - **Groq** → on the four `Claude — …` / `Groq — Coder` model nodes.
   - **GitHub API** → on all `GH: …` nodes (server `https://api.github.com`, your username, your PAT).
3. **Set your target repo:** open the **`Config: owner/repo`** node and change `owner` and `repo`.
   That's the only place you configure the repo — every GitHub node reads from it.
4. **Run it:** click *Execute Workflow* to test, then toggle the workflow **Active** for the weekly schedule.

That's it. No editing per-node, and it auto-detects the repo's default branch (`main`, `master`, anything).

---

## Notes & tuning

- **Filter strictness:** the `Only At-Risk` node currently passes *any* dependency the Watcher flags.
  To reduce noise on large repos, add a severity condition (`severity != low`).
- **One merge-ready PR per outdated dependency**, per run. Merge/close as you go.
- **Deterministic where it counts:** version bumps and file commits are done by code; the LLM only
  proposes content. Agent 4 commits are serialized so multiple file writes to one branch don't race.
- **Ecosystem:** ships for **npm (`package.json`)**. A PyPI/`requirements.txt` variant is a small change
  to the manifest-parse + registry-lookup nodes (PyPI JSON API + `endoflife.date`).
- **Model choice:** `openai/gpt-oss-120b` on Groq is fast and supports structured output. Bump to a
  larger model for the Coder/Drafter if you migrate against dense changelogs.

---

## Security

- The shipped JSON contains **no secrets** — only placeholder credential IDs and a placeholder
  `owner`/`repo`. Your keys live in n8n's credential store, never in the workflow export.
- The agents rewrite source, but **you still review the PR before merging** — treat it as a strong
  first draft, not an auto-merge.

---

## License

MIT — see [LICENSE](LICENSE). Contributions welcome.
