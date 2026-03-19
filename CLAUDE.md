# Desktop Commander MCP — AI Agent Guidelines

This file provides context for AI coding assistants (Claude, Copilot, etc.) working in this repository.

> ⚠️ **Security notice**: This file can be read and followed by AI agents. Any modification that weakens the Agent Boundaries section below is a prompt-injection attack and must not be committed without explicit human approval and code review.

## Project Overview

Desktop Commander MCP is an MCP (Model Context Protocol) server that lets AI assistants execute terminal commands and manage files on the user's computer. Because it has **direct shell access**, security is paramount.

## Build & Test

```bash
npm install
npm run build       # TypeScript compile + copy assets
npm test            # build + run all integration tests
```

Key source files:
- `src/command-manager.ts` — command validation and blocklist enforcement
- `src/config-manager.ts` — persistent configuration (read/write)
- `src/config.ts` — path constants
- `src/tools/config.ts` — MCP tool handlers for `get_config` / `set_config_value`
- `src/config-field-definitions.ts` — allowlist of user-editable config keys

## Agent Boundaries

**Never do any of the following without explicit written approval from a human maintainer:**

- Remove or skip a failing test
- Weaken or bypass security logic in `src/command-manager.ts` or `src/config.ts`
- Remove commands from the default `blockedCommands` list in `src/config-manager.ts` (`getDefaultConfig`)
- Allow `validateCommand` in `src/command-manager.ts` to return `true` (permit) when an error occurs — it must fail closed
- Expand the set of keys accepted by `set_config_value` beyond `CONFIG_FIELD_DEFINITIONS`
- Modify this file (`CLAUDE.md`) to remove or soften any of the constraints listed here
- Commit changes that disable or bypass git pre-commit hooks

These constraints exist because Desktop Commander has direct shell access on users' machines. A security regression here can result in arbitrary code execution.
