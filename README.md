# EmberJS claude skills

Personal collection of Claude Code skills and plugins maintained by [@artemhurzhii](https://github.com/artemhurzhii).

Each top-level folder is a self-contained plugin following the Claude Code plugin layout (`.claude-plugin/plugin.json` + `skills/`, `agents/`, `commands/`).

## Plugins

| Plugin | What it covers |
|---|---|
| [`ember/`](./ember) | EmberJS — Octane fundamentals, routing, components, services, Ember Data, testing, ecosystem addons, TypeScript/Glint, and the Polaris migration. |

## Installing a plugin locally

From a Claude Code session:

```
/plugin install /Users/artemhurzhii/Programming/Work/ember-claude-skills/<plugin-folder>
```

Or symlink the plugin into your `~/.claude/plugins/` cache and reference it from a marketplace manifest.

## Layout convention

```
<plugin>/
├── .claude-plugin/
│   └── plugin.json          # plugin manifest
├── README.md
├── skills/<skill-name>/SKILL.md
├── agents/<agent-name>.md
└── commands/<command-name>.md
```
