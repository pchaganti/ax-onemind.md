# ONEMIND.md

**Give your repo a memory.**

Most projects forget everything between sessions — decisions, rejected ideas, context. ONEMIND.md fixes that by providing a convention for agents to store decision, memories, contexts in git refs. It's a single file you drop into any git repo, and suddenly your project remembers: why you chose X over Y, what you tried and abandoned, what you learned.

No database. No service. No new dependencies. Just git.

![example](example.gif)

## Why you'd want this

- **"Why did we do it this way?"** — answered, forever, in `git log`.
- **"What did we try and reject?"** — recorded so nobody re-litigates dead ends.
- **"What did the last agent figure out?"** — the next agent picks up where it left off.
- **Decisions with reasoning**, not just code diffs.
- It travels with the repo. Clone it, you get the mind. Delete it, the mind goes with it.

## Try it now

```sh
curl -sO https://raw.githubusercontent.com/lazardanlucian/onemind.md/main/ONEMIND.md
```

That drops the spec into your repo. Tell your agent to read it.

## Pruning

Set a size limit in `AGENTS.md` (default 1GB already set in ONEMIND.md). Agents check at session start and prune stale thoughts
(`Status: dead`, `Status: deprecated`, old observations) when the mind grows past threshold.
Pruned commits become unreachable; `git gc` reclaims the space. 90-day reflog safety net.

## How it auto-bootstraps

When your agent first reads `ONEMIND.md`, it checks whether the mind exists. If not, it initializes it — one git command, no files on disk, no config changes. From that point on, every session starts by reading the mind and ends by recording what was learned.

Add the linker line to your `AGENTS.md` so the agent remembers to do this every session:

```
This project's learning memory is an in-repo git mind on `refs/mind/main`
— read `ONEMIND.md` and follow its protocol.
```

## What it feels like

Once the mind is set up, your AI agents (or you) can:

**Ask about past decisions:**
> "Why did we pick PostgreSQL over SQLite?"
> "What did we decide about authentication?"
> "What approaches did we reject for the caching layer?"

The agent searches the mind with `git log --grep` and `git grep`, finds the reasoning, and gives you the answer with full context.

**Record new learnings in real time:**
> "The rate limiter breaks under 10k concurrent connections — found during load testing."

This gets committed to the mind immediately, so the next session (or the next agent) knows about it without re-discovering it.

**Track what didn't work:**
> "Tried Redis for session storage. Too much ops overhead for our scale. Status: dead."

Dead ideas stay recorded so nobody wastes time on them again.

**Follow threads of thinking:**
> "What's the current thinking on the deployment pipeline?"

The agent traces a series of related mind commits tagged with the same thread, giving you the full evolution of an idea.

## How the mind works (briefly)

Everything lives on a hidden git ref (`refs/mind/main`) — not a branch, never checked out, invisible to `git status`. Thoughts are plain git commits. No files are written to your working directory.

Two kinds of thoughts:
- **Message-only** (default) — the commit message *is* the content. Quick notes, decisions, observations.
- **Blob thoughts** — structured documents (like ADRs) stored as git blobs. Searchable via `git grep`.

Optional trailers give thoughts machine-readable structure:
```
mind: chose Tailwind over CSS modules

Faster prototyping, consistent design tokens, no naming debates.

Tags: styling, design
Decision: accepted
Confidence: high
```

Read the full spec in [`ONEMIND.md`](ONEMIND.md) for all the details — threads, notes, ADR templates, sync, multi-person workflows.

## Syncing

```sh
git push origin refs/mind/main                # push the mind
git fetch origin refs/mind/main:refs/mind/main  # pull the mind
```

The mind lives on the same remote as your code. CI pipelines should ignore `refs/mind/*`, so pushing the mind should not trigger a build, etc.

## Safety
- ONEMIND.md instructs agents not to save PII/TOKENS, sensitive data in the mind.
- Nothing leaks into your working tree — no `mind/` folder, no `.gitignore` edits.
- Never commit secrets, tokens, or PII to the mind.
- Append only. Use `git notes` to correct past thoughts without rewriting history.
