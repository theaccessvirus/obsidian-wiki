---
name: wiki-issues-dashboard
description: >
  Generate or refresh a live GitHub Issues dashboard note for a git repo using the
  GitHub Embeds Obsidian plugin. Use when the user says "dashboard my issues",
  "add an issues board for this repo", "make a github issues dashboard",
  "/wiki-issues-dashboard", or wants a repo's open issues embedded as a live view in
  the vault. Also invoked by wiki-update when a synced project has a GitHub remote
  with open issues.
---

# Wiki Issues Dashboard — Live GitHub Issues in the Vault

Writes a one-note-per-repo dashboard that embeds a repo's GitHub issues as a live
view via the **GitHub Embeds** plugin (`id: github-embeds`). The note holds a single
`github-issues` fenced block; the plugin fetches and renders the issues at read time
using the token stored in its own settings. No script executes — this skill only
writes the block.

## Before You Start

1. **Resolve config** — Config Resolution Protocol in `llm-wiki/SKILL.md` (inline
   `@name` override → walk up CWD for `.env` → `~/.obsidian-wiki/config` → prompt
   setup). This gives `OBSIDIAN_VAULT_PATH`.
2. Confirm the GitHub Embeds plugin is installed:
   `$OBSIDIAN_VAULT_PATH/.obsidian/plugins/git-snip/` (plugin id `github-embeds`).
   If absent, tell the user to install it and stop — without it the note renders as a
   plain code block, not a live issue list.

## Step 1: Identify the Repo

Run from the target repo (default: current working directory):

```bash
git -C "$REPO" remote get-url origin
```

Parse `owner/name` from either remote form:

```bash
slug=$(git -C "$REPO" remote get-url origin \
  | sed -E 's#(git@github.com:|https://github.com/)##; s#\.git$##')
repo_name=${slug##*/}
```

- ssh form: `git@github.com:owner/name.git`
- https form: `https://github.com/owner/name.git`

If origin is not a GitHub remote, tell the user and stop.

## Step 2: Gate on Issue Usage

Only build a dashboard for repos that actually use issues:

```bash
gh issue list -R "$slug" --state open -L 1 --json number -q 'length'
```

- `> 0` → proceed.
- `0` → no open issues. When invoked **directly**, ask whether to create it anyway
  (with `state: all`) or skip. When invoked **from wiki-update**, skip silently.

Requires `gh` authenticated to an account with read access to the repo. If `gh` is
unavailable, report it and stop (the snippet still works if written manually — the
gate is only to avoid creating empty boards).

## Step 3: Write the Dashboard Note

Target path: `$OBSIDIAN_VAULT_PATH/dashboards/github/<repo_name>.md`
(create `dashboards/github/` if it does not exist). The filename is the repo name
verbatim — do not slugify (`onlyStems` stays `onlyStems`).

Write exactly this block — **overwrite** if the file exists, never append (compile,
don't retrieve). The snippet is shown indented here; write it as a real fenced block:

    ```github-issues
    repo: <slug>
    state: open
    ```

- `state:` accepts `open` | `closed` | `all`. Default `open`; honor a user request
  for a different state.
- Write **no frontmatter** — the plugin reads only the `repo:`/`state:` lines, and
  the note is a pure live view, not a compiled wiki page.
- Close the fence with a proper ```` ``` ````. Do not emit a `...` line or any other
  content.

## Step 4: Update Tracking

Append to `$OBSIDIAN_VAULT_PATH/log.md`:

```
- [TIMESTAMP] WIKI_ISSUES_DASHBOARD repo=<slug> state=<state> file=dashboards/github/<repo_name>.md
```

No `.manifest.json` or `index.md` update — dashboards are live queries, not compiled
pages (same policy as `wiki-dashboard`).

## Backfill / Batch

To cover several repos at once, loop Steps 1–4 per repo path or per `owner/name`
slug. To find candidates across a GitHub account:

```bash
gh repo list <owner> --limit 200 --json nameWithOwner -q '.[].nameWithOwner'
```

then keep only slugs where Step 2 returns `> 0`.

## Notes

- **No secret in the note.** The GitHub token lives in the plugin's settings
  (`.obsidian/plugins/git-snip/data.json`), not in the snippet — generated notes are
  safe to commit. Keep `.obsidian/` out of the vault's git history regardless.
- **Snippet syntax is fixed by the plugin.** Confirmed from its parser: the block
  language is `github-issues`; `repo:` matches `owner/name`; `state:` matches
  `open|closed|all`. Other keys are ignored.
- **Related:** `wiki-dashboard` (Bases/Dataview views over vault notes);
  `wiki-update` (invokes this skill when syncing a repo that has open issues).
