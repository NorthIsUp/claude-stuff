# claude-stuff

Personal Claude Code odds and ends — statusline, helper scripts, hooks.

## bin/

| Tool | What it does |
|------|---------|
| [`cc-statusline`](plugins/northisup-utils/bin/cc-statusline) | Single-line Claude Code statusline. Branch + PR + CI + review + comments + Linear ticket on the left; context bar, quota, model, effort on the right. Shows `org/repo` (and worktree name) ahead of branch info, falls back to pwd outside a repo. Reads session JSON on stdin; caches `gh` / `linear` lookups per session. |
| [`cc-thread-prs`](plugins/northisup-utils/bin/cc-thread-prs) | Lists PRs created by a given Claude Code session by scanning the transcript for `gh pr create`, MCP github creates, and `api.github.com` POSTs. Used by the statusline to render the "other PRs this session" chip. |
| [`claude-allow`](plugins/northisup-utils/bin/claude-allow) | One-liner to add a tool permission to `~/.claude/settings.json`. `claude-allow cmd "git push"` → adds `Bash(git push:*)`. |

## Install

### As a Claude Code plugin

```
/plugin marketplace add NorthIsUp/claude-stuff
/plugin install northisup-utils@claude-stuff
```

### Or in a single project

Clone and use the bundled `.claude/settings.json`:

```bash
git clone git@github.com:NorthIsUp/claude-stuff.git
cd claude-stuff
# settings.json under .claude/ already wires the statusline to ./bin/cc-statusline
```

### Or globally

Symlink the scripts into `$PATH` and point `~/.claude/settings.json` at the statusline:

```bash
ln -s "$PWD/bin/cc-statusline" ~/.local/bin/cc-statusline
```

```jsonc
// ~/.claude/settings.json
{
  "statusLine": {
    "type": "command",
    "command": "~/.local/bin/cc-statusline",
    "padding": 0,
    "refreshInterval": 5
  }
}
```

The statusline expects:

- `gh` (for PR data)
- `jq`
- A Nerd Font in your terminal (for the glyphs)
- Optional: `LINEAR_WORKSPACE` env var to make Linear ticket labels clickable
