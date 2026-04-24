---
description: Enable NorthIsUp's cc-statusline for the current user
---

Wire up the `cc-statusline` shipped with this plugin into the user's Claude Code settings.

1. Read `~/.claude/settings.json` (create empty `{}` if missing).
2. Set `statusLine` to:
   ```json
   {
     "type": "command",
     "command": "${CLAUDE_PLUGIN_ROOT}/bin/cc-statusline",
     "padding": 0,
     "refreshInterval": 5
   }
   ```
   …where `${CLAUDE_PLUGIN_ROOT}` resolves to this plugin's install path.
3. Write the file back, preserving every other key.
4. Tell the user the statusline is enabled and that they may need to restart Claude Code for it to take effect.

Do not touch any other field in the settings file.
