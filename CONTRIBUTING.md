# Contributing to egnyte-for-ai

## Adding a reference doc

1. Create `skills/egnyte/references/<domain>.md`
2. Structure: **Overview** → **MCP patterns** → **CLI equivalents** → **Examples** → **Common errors**
3. Add a row to the routing table in `skills/egnyte/SKILL.md`

## Fixing CLI commands

Check `egnyte schema --list` and `egnyte schema <operation>` for authoritative parameter documentation. The [`@egnyte/agentic-cli`](https://www.npmjs.com/package/@egnyte/agentic-cli) npm package page has the full changelog.

## Updating guardrails

Edit `rules/egnyte.mdc`. Each rule needs:
- The trigger condition
- The required action (confirm / warn / dry-run)
- One sentence explaining why

## Adding IDE plugin support

1. Copy `.claude-plugin/` to `.<ide>-plugin/`
2. Update `plugin.json` with IDE-specific required fields
3. `skills/egnyte/SKILL.md` is IDE-agnostic — no changes needed there

## Commit format

```
<scope>: <description>
```

Scopes: `skill`, `rules`, `plugin`, `references`, `docs`

Examples:
- `skill: add agents routing pattern`
- `rules: add guardrail for bulk delete`
- `references: expand search with advanced filters`
