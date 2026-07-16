# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A GitHub profile README stats generator (adapted from Andrew6rant's profile). A single Python script, [today.py](today.py), queries the GitHub GraphQL API for user stats (age/uptime, commits, stars, repos, followers, lines of code) and rewrites the values in-place inside `dark_mode.svg` (the only theme — light mode was removed). The profile README just embeds that SVG.

There are no tests and no linter — the GitHub Actions run is the de facto verification.

## Running

```bash
pip install -r cache/requirements.txt
ACCESS_TOKEN=<fine-grained PAT> USER_NAME=<github username> python today.py
```

Both env vars are required (`today.py` reads them at import time and crashes without them). The required PAT scopes are documented in the comment at the top of [today.py](today.py#L9-L12).

CI: [.github/workflows/build.yaml](.github/workflows/build.yaml) runs the script on every push to `main`, daily at 04:00 UTC, and on manual dispatch, then commits the regenerated SVGs and cache back to `main` ("Updated README" commits are from the bot). Secrets `ACCESS_TOKEN` and `USER_NAME` supply the env vars.

## Architecture

**Data flow:** GraphQL queries → per-repo LOC cache → SVG text replacement.

- **LOC cache** (`cache/<sha256-of-username>.txt`): counting lines of code requires paginating every commit of every repo, so results are cached. The file has a 7-line comment header (the `comment_size=7` argument threaded through `loc_query`/`cache_builder`/`commit_counter`), then one line per repo: `sha256(nameWithOwner) total_commits my_commits loc_added loc_deleted`. A repo's LOC is only re-fetched when its commit count changes; if the repo *count* changes, the whole cache is flushed and rebuilt. On API failure mid-run, `force_close_file` saves partial cache state before raising.

- **SVG templating:** the SVGs are hand-authored, terminal-neofetch-style images. `svg_overwrite` locates `<tspan>` elements by `id` (e.g. `commit_data`, `loc_data`, `loc_add`) and replaces their text. Each value has a sibling `<id>_dots` tspan; `justify_format` adjusts the number of dots so columns stay aligned as value widths change. The right column is aligned to exactly 67 characters wide — if you edit SVG text content or the width arguments passed to `justify_format` in `svg_overwrite`, keep that alignment intact.

## Repo-specific gotchas

- The birthday used for the "Uptime" line is hardcoded at the call site near the bottom of [today.py](today.py#L450) — not an env var.
- The `add_archive()` path (deleted-repo stats from `cache/repository_archive.txt`) is intentionally disabled via `if False:` — it only applied to the original author. The file doesn't exist here; don't enable it.
- `recursive_loc` deliberately does not use `simple_request` so it can save the cache before raising. GitHub's undocumented anti-abuse limit (403) is a known failure mode on large accounts.
