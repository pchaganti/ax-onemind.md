# ONEMIND.md — turn any git repo into a persistent "mind", with zero files on disk

**A one-file convention that gives your project a memory, using only git.**

No database. No service. No API key. No extra software to install or keep alive. And **no
working-directory files**: every thought lives only as a git object on a hidden ref
(`refs/mind/main`), written through plumbing. Nothing is written to your disk except the repo's own
`.git`. `git status` of your code repo never shows the mind.

## Why this exists
Most "memory" tools ask you to rent or run another system: a vector DB, a note app, a SaaS. That's a
dependency, a bill, and a thing that can disappear on you. This uses the tool that's already in the
repo and already versioned: **git itself**. The memory is *bound to the project it's about* — clone
the repo, you get its mind; delete the repo, its mind goes with it, correctly. It never gets
separated from the code it explains.

## What you get
- **Decisions with reasons**, not just code. "Why did we strip imported photos?" → one `git log`
  away, forever.
- **A "why NOT" log.** Record rejected ideas so you (and your future collaborators) don't waste
  time re-litigating dead ends.
- **Full history + diff + blame** on your thinking, for free, by a tool you already trust.
- **Portable & lock-in-free.** It's plain git; anyone can read it, any agent can use it, nothing
  phones home.
- **Zero disk footprint.** The mind is pure git metadata — no `mind/` folder, no `.gitignore`
  edits, no helper script.

## How it differs from the other variants
- `MIND.md` isolates the mind in a *separate* `.mind/` git repo plus a `mind.ps1`/`mind` helper
  script.
- A folder-based `ONEMIND.md` kept mind files in a git-ignored `mind/` directory on disk.
- **This version uses neither.** Thoughts are git objects on `refs/mind/main`, created entirely via
  plumbing (`hash-object`, `update-index`, `commit-tree`, `update-ref`). No `.mind/`, no `mind/`
  directory, no script. Your working tree stays exactly as it was.

---

# The spec (pure-ref mode)

> Drop a copy of this file into a git repo (as `ONEMIND.md`). The repo becomes a mind on the hidden
> ref `refs/mind/main`. Nothing else changes in how you use git for code. This file is the *spec*;
> it lives at repo root like any doc and is **not** mind content.

## What a "thought" is
Two equivalent, disk-free forms:
- **Message-only thought (recommended default):** a commit on `refs/mind/main` whose *message is the
  content* — prose plus trailers. The tree is unchanged. Best for notes, decisions, realizations.
- **Blob thought (optional, for file-like retrieval):** the content is stored as a git *blob* in the
  ref's tree under a path like `mind/decisions/<slug>.md`, but the blob is created from stdin via
  `git hash-object -w --stdin` — **never written to disk**. Later you can
  `git show refs/mind/main:mind/decisions/<slug>.md` or `git grep` over the tree, with no local file.

Both forms leave your working directory and your code branch (`main`/`master`) completely untouched.

## The mind ref
`refs/mind/main` — a hidden ref: not a branch, never checked out, invisible to `git branch`. Your
code commits are never touched. (Multi-person works fine — it's just a shared ref; CI won't fire on
`refs/mind/*` pushes since those aren't `refs/heads/*` or tags.)

## Structure lives in git, not on disk
**Commit trailers (the metadata layer).** End a commit message with key-value trailers (same
mechanism as `Signed-off-by:`). Gives thoughts machine-readable structure with zero file schema:
```
mind: pass on generic med-reminder app

Saturated market, weak monetization; only viable with a sharp wedge
(caregiver, B2B, or localized/Romania).

Tags: medication, saas, vetoed
Idea: medication-management-site
Status: dead
Confidence: high
```
Suggested vocabulary (all optional): `Tags:`, `Idea:`, `Decision:`, `Status:`
(proposed|accepted|deprecated|dead), `Thread:`, `Confidence:`, `Related:` (a SHA).
Recall by trailer via grep (see below). Note: `git log --format='%(trailers:key=Tags)'` only renders
whitelisted trailer keys by default, so use `git log --grep` for custom keys like `Tags:`/`Status:`.

**git notes (annotate the past without falsifying it).** Correct a past decision *on* its original
commit, without rewriting it:
```
git notes add -m "Superseded by a1b2c3d — shipped photos-stripped, risk lower than feared." <sha>
git log --show-notes refs/mind/main
```

## Recall (all plain git, all on the mind ref)
1. **Thought log / text in messages:** `git log --grep="<term>" -i refs/mind/main`
2. **Text inside blob thoughts:** `git grep -n -i "<term>" refs/mind/main -- mind/`
3. **Change-aware (pickaxe):** `git log -S"<string>" refs/mind/main` (when a term appeared/left a
   blob); `git log -G"<regex>" refs/mind/main` (diffs touching a pattern). For message-only thoughts
   use `--grep`.
4. **By milestone:** `git tag -n` (annotated tags only), `git describe`.
5. **Follow the graph:** `Related:` SHA trailers; `git log --grep="Thread: <name>" refs/mind/main`
   reconstructs a thread (prefer a `Thread:` trailer over a branch).

## Threads (use sparingly — the mind ref is the default)
A "thread" is a sustained line of thinking. Tag a series of commits with a shared `Thread: <name>`
trailer; they stay on `refs/mind/main`, visible to `git grep`/`git log`, no checkout dance. Reserve
an actual git branch only for exploring something you might throw away. Do **not** open a branch per
thought.

## Usage
1. **Session start:** read this file, then `git log -5 refs/mind/main`, `git tag -n`, and optionally
   `git ls-tree -r refs/mind/main --name-only` (lists any blob thoughts). Open with a short recap.
2. **Remember by committing:** write a message-only or blob thought and commit it to the mind ref
   with a clear message + trailers (commands below).
3. **Recall before deciding:** run the recall commands so you build on prior learning.
4. **Correct with notes:** `git notes add` on the original commit rather than rewriting.
5. **Tag milestones** (annotated) when reached.
6. **Keep current:** append, don't overwrite; preserve history.

### Decision record (ADR) template
```
# Decision: <Title>
## Context  /  ## Options (A: pro/con · B: pro/con)  /  ## Decision  /  ## Why  /  ## Status
```
Commit with matching trailers (`Decision:`, `Status:`, `Tags:`) so it's queryable.

### Always-on & transparent
The mind is **always on** — consult it before deciding/recalling, and **write the moment something
is learned**. Tell the user when you read from / write to the mind ("From mind: …" / "Recording to
mind: …"). The user should never be surprised the mind was used.

### Session-start recap
```
=== What's on my mind ===
Recent thoughts:  git log -5 refs/mind/main
Milestones:       git tag -n
```
That's just `git log -5 refs/mind/main` + `git tag -n` — no tooling required.

## Setup in a repo (one time, plain git)

**Bash / Git Bash / macOS / Linux:**
```
git update-ref refs/mind/main "$(git commit-tree "$(git mktree </dev/null)" -m 'mind: init

Tags: meta, bootstrap
Status: accepted')"
```

**PowerShell (Windows) — `git mktree` doesn't accept piped empty input on PS:**
```
# create an empty tree via a temp index (no disk files left behind)
$idx = "$env:TEMP\mind_init_$([guid]::NewGuid().ToString('N').Substring(0,8))"
$env:GIT_INDEX_FILE = $idx
git read-tree --empty
$tree = git write-tree
Remove-Item Env:\GIT_INDEX_FILE; Remove-Item $idx -EA SilentlyContinue

$msg = "mind: init`n`nTags: meta, bootstrap`nStatus: accepted"
$f = "$env:TEMP\mind_init_msg.txt"
[System.IO.File]::WriteAllText($f, $msg, [System.Text.UTF8Encoding]::new($false))
$sha = git commit-tree $tree -F $f
Remove-Item $f
git update-ref refs/mind/main $sha
```

That's the whole "install." Nothing is written to your working directory. (This file, `ONEMIND.md`,
is the spec and lives at repo root like any doc; it is not mind content.)

## Writing to the mind (pure plumbing — no checkout, no branch switch, no disk file)
Use a **temporary index** so your real index is never touched.

**A) Message-only thought (recommended default):**
```
NEW=$(git commit-tree "$(git rev-parse refs/mind/main^{tree})" \
        -p "$(git rev-parse refs/mind/main)" -m "mind: <summary>

<prose: context, options, decision, why>

Tags: ...
Status: ...
Idea: ...")
git update-ref refs/mind/main "$NEW"
```

**B) Blob thought (file-like, but never on disk):**
```
# content goes straight into a git blob from stdin — no file is created on disk
BLOB=$(git hash-object -w --stdin <<'EOF'
# Decision: <Title>
## Context / ## Options / ## Decision / ## Why / ## Status
EOF
)
# build the tree in a temp index (no working directory needed)
TMP=$(mktemp -d)
GIT_INDEX_FILE="$TMP/idx" git read-tree refs/mind/main
GIT_INDEX_FILE="$TMP/idx" git update-index --add --cacheinfo 100644,"$BLOB",mind/decisions/<slug>.md
TREE=$(GIT_INDEX_FILE="$TMP/idx" git write-tree)
NEW=$(git commit-tree "$TREE" -p "$(git rev-parse refs/mind/main)" -m "mind: <summary>

Tags: ...
Status: ...")
git update-ref refs/mind/main "$NEW"; rm -rf "$TMP"
```
- `git show refs/mind/main:mind/decisions/<slug>.md` reads it back.
- `git grep -n -i "<term>" refs/mind/main -- mind/` searches it.

Both methods leave your working directory and `main`/`master` completely untouched.

> **Windows / PowerShell note:** don't build the commit message with `Set-Content -Encoding UTF8`
> — PowerShell 5.1 prepends a UTF-8 BOM that leaks into the commit subject. Use
> `[System.IO.File]::WriteAllText($path, $text)` (UTF-8, no BOM) and `git commit-tree -F <file>`, or
> pass `-m` directly. Avoid multi-line `-m` via here-strings (they can mis-parse); prefer `-F`.
>
> **PowerShell equivalent for message-only thought (A):**
> ```
> $tree = git rev-parse "refs/mind/main^{tree}"
> $parent = git rev-parse refs/mind/main
> $body = "mind: <summary>`n`n<prose>`n`nTags: ...`nStatus: ..."
> $f = "$env:TEMP\mind_msg.txt"
> [System.IO.File]::WriteAllText($f, $body, [System.Text.UTF8Encoding]::new($false))
> $NEW = git commit-tree $tree -p $parent -F $f
> Remove-Item $f
> git update-ref refs/mind/main $NEW
> ```

> Optional human convenience only (not part of the spec): a shell alias for recall,
> `mind-recall(){ git grep -ni "$1" refs/mind/main -- mind/ 2>/dev/null; git log --oneline refs/mind/main --grep="$1" -i; }`.
> The mind works with raw git; the alias is sugar for your terminal.

## Multi-person
It's a shared ref. Everyone fetches, commits (plumbing), and pushes it. If two people push
`refs/mind/main` and diverge, the second push is rejected (non-fast-forward, like any branch) —
`git fetch origin refs/mind/main` then re-commit onto the fetched ref (`read-tree` the fetched tree,
`write-tree`, `commit-tree -p <fetched>`, `update-ref`). No checkout, no conflict with code.

## Sync (one upstream — the whole point)
Single remote, single `git push`:
```
git push origin refs/mind/main                 # the mind, on the same repo's upstream
git push origin 'refs/notes/*'                 # if you use notes for corrections
```
Pull: `git fetch origin refs/mind/main:refs/mind/main`. CI triggered on `refs/heads/*` / tags
ignores `refs/mind/*`, so pushing the mind never kicks off a build.

## Safety bounds
- The mind is invisible on disk; there is no `mind/` folder to leak into code commits. Keep all mind
  commands scoped to `refs/mind/main`; never run them against `HEAD`/`main`/`master`.
- **Never commit PII, tokens, keys, credentials, or sensitive data** to the mind ref.
- Do not force-push or rewrite shared mind history unless explicitly instructed. `git notes` is the
  tool for revising understanding — annotate, don't rewrite.
- Prefer appending over overwriting; preserve history.

## Linker for agents (optional)
Add one line to the repo's `AGENTS.md` so any agent working there uses the mind:
> This project's learning memory is an in-repo git mind on `refs/mind/main` — read `ONEMIND.md` and
> follow its protocol. The mind uses **no working-directory files**; all thoughts are git objects
> written via plumbing (read at session start; commit learnings via the plumbing commands).
