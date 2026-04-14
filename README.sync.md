# gstack state sync

This directory is set up to be a private git-managed home for the durable parts
of gstack state that matter across machines.

Tracked:
- `config.yaml`
- `builder-profile.jsonl`
- `projects/**`
- the helper scripts in this directory

Ignored:
- `analytics/`
- `sessions/`
- `sidebar-sessions/`
- `slug-cache/`
- `worktrees/`
- `installation-id`
- transient marker files
- project cache like `projects/*/repo-mode.json`

Recommended usage:

1. On the first machine:
   `./sync-init <private-github-repo-url>`
   Optional first push:
   `./sync-init <private-github-repo-url> --push`

2. On a new machine:
   `git clone <private-github-repo-url> ~/.gstack`

3. Before switching machines:
   `./sync-push`

4. After landing on another machine:
   `./sync-pull`

Notes:
- Use a private repo. This directory contains personal state and project memory.
- `sync-push` stages only the allowlisted files because `.gitignore` is strict.
- `*.jsonl` uses Git's built-in `merge=union` driver to reduce conflicts on
  append-only logs.
