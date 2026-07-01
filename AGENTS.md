# AGENTS.md

This is a personal Obsidian vault of **study notes** analyzing the OpenClaw
(and occasionally OpenCode) source code. There is no build/test/lint/CI —
it's markdown only. Notes are written in **Chinese (simplified)**; match
that when adding or editing content.

## Repo layout — two distinct note types, don't mix them up

- Root `*.md` — deep technical/architecture notes (研究笔记): source-cited,
  precise, written for the author's own understanding. Cite exact file
  paths and line refs from the source repo.
- `blog/*.md` — reader-facing "101" explainer articles (科普向), derived
  from the same research but rewritten for an outside audience (problem →
  intuition → source-backed example). Don't add deep-dive content here;
  put it in a root note and link to it instead (see cross-links at the
  bottom of `blog/skills-101.md` / `blog/rag-101.md` for the pattern).
- Each of `README.md` (root) and `blog/README.md` is a **manual index**
  listing every note with a 2-3 line summary. When adding a new note file,
  add an entry to the matching README — this is easy to forget (two
  existing notes, `agent-loop-notes.md` and `opencode-plan-mode.md`, are
  currently missing from the root index).
- `.obsidian/` is Obsidian's vault config (workspace layout, plugins) —
  don't hand-edit `workspace.json`; it's UI state, not content.

## Source of truth lives outside this repo

This folder is a nested git repo (own history/remotes) living inside a
checkout of the OpenClaw source at the parent directory
(`/Users/clz/Code/openclaw`, one level up from here). That parent repo has
its own `AGENTS.md`/`docs/` — notes here are written by reading *that*
source, not by re-deriving architecture from prose.

Some notes (e.g. `opencode-plan-mode.md`) instead study a **sibling repo**,
referenced in the note as `../opencode` (i.e. `/Users/clz/Code/opencode`
relative to this vault's parent). If a note cites a sibling repo, verify
claims against that repo's actual source before editing/extending the note
— don't trust the note's paraphrase alone.

When writing or updating a note:
- State which source repo/version the note is based on (existing notes
  open with a blockquote like `> 基于 2026.6.2 版本源码，关键文件：...`
  or `> 基于 sibling 仓库 ../opencode 源码实读`) — follow that convention.
- Prefer citing concrete `file:line` references over paraphrased claims.

## Git remotes

Two remotes are configured and both are pushed to: `origin` (GitHub,
`FadingLight9291117/openclaw-notes`) and `gitea` (private instance at
`git.fadinglight.cn`). Don't assume only one remote — push to both if
asked to publish, or ask which one if unclear.
