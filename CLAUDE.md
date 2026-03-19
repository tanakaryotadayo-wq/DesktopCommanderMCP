# Desktop Commander MCP — Claude Code Context

## Project Overview

MCP server (`@wonderwhy-er/desktop-commander`) that gives Claude Desktop terminal access, file editing, and system control. Written in TypeScript/ESM, published to npm. Node >=18 required.

## Essential Commands

```bash
npm run build          # tsc + copy assets + build UI runtime (always run before testing)
npm test               # build + run all 40+ integration tests
npm run validate:tools # verify tool definitions are consistent
npm run watch          # TypeScript watch mode (dev)
npm start              # run built server
npm run inspector      # launch MCP inspector for interactive debugging
npm run count-tokens   # count tokens in MCP tool definitions
npm run logs:analyze   # analyze fuzzy edit match logs
```

## Architecture in One Paragraph

`index.ts` parses CLI args and boots the server. `server.ts` registers all 30+ MCP tools and wires them to handlers in `src/handlers/`. Handlers validate input with Zod schemas (`src/tools/schemas.ts`) then call implementation functions in `src/tools/`. State is managed by three singletons: `ConfigManager`, `TerminalManager`, `SearchManager`. All stdout is intercepted by `FilteredStdioServerTransport` (`src/custom-stdio.ts`) and wrapped as JSON-RPC notifications — this is required for MCP protocol compliance and means **never use bare `console.log` in code paths that run during MCP sessions**.

## Critical Gotchas

- **ESM `.js` extensions required** — all imports must end in `.js` even when importing `.ts` files: `import { foo } from './foo.js'`
- **Always build before testing** — `npm test` does this automatically; if running tests manually, run `npm run build` first
- **`dist/` is generated** — never edit files in `dist/`, only edit `src/`
- **Config lives at** `~/.claude-server-commander/config.json`, not in the repo
- **`set_config_value` only accepts known keys** — adding arbitrary keys was restricted for security (see commit `5a4806f`)
- **Command validation fails closed** — any error in `CommandManager.extractCommands()` denies execution (see commit `7090a5d`)
- **Terminal commands ignore `allowedDirectories`** — only filesystem tools respect path restrictions
- **`node:local` mode** in `improved-process-tools.ts` runs Node.js inside the MCP server process itself — handle with care

## Adding a New MCP Tool (Step-by-Step)

1. **Schema** in `src/tools/schemas.ts`:
   ```typescript
   export const MyToolArgsSchema = z.object({
     path: z.string(),
     count: z.number().optional().default(10),
   });
   ```

2. **Handler** in `src/handlers/my-handler.ts`:
   ```typescript
   import { MyToolArgsSchema } from '../tools/schemas.js';
   import type { ServerResult } from '../types.js';

   export async function handleMyTool(args: unknown): Promise<ServerResult> {
     const { path, count } = MyToolArgsSchema.parse(args);
     // implementation
     return { content: [{ type: 'text', text: result }] };
   }
   ```

3. **Register in `src/server.ts`**:
   - Add to the tool list in `ListToolsRequestSchema` handler (include `inputSchema` and `annotations`)
   - Add a `case 'my_tool':` in `CallToolRequestSchema` handler

4. **Verify**: `npm run build && npm run validate:tools`

## Key Files

| File | Purpose |
|---|---|
| `src/index.ts` | Entry point, CLI arg handling, server boot |
| `src/server.ts` | All tool definitions and MCP request handlers (~1550 lines) |
| `src/tools/schemas.ts` | Zod schemas for every tool input |
| `src/tools/filesystem.ts` | File read/write/list/move (~920 lines) |
| `src/tools/edit.ts` | Fuzzy text search-and-replace, Excel/DOCX editing |
| `src/tools/improved-process-tools.ts` | Terminal session control |
| `src/terminal-manager.ts` | Session lifecycle, REPL detection, paginated output |
| `src/command-manager.ts` | Security: command extraction and blocklist validation |
| `src/search-manager.ts` | ripgrep-based search sessions |
| `src/config-manager.ts` | Singleton config; persists to `~/.claude-server-commander/config.json` |
| `src/custom-stdio.ts` | Intercepts console output → JSON-RPC notifications |
| `src/utils/capture.ts` | Telemetry (sanitizes paths before sending) |
| `src/remote-device/device.ts` | Remote access via OAuth + WebSocket for ChatGPT/Claude Web |

## Code Conventions

- **TypeScript strict mode** — no `any`, no `@ts-ignore` without justification
- **Zod for all tool inputs** — use `.parse()` (throws on failure); only use `.safeParse()` when you need to handle errors gracefully
- **`ServerResult` return type** for all tool handlers:
  ```typescript
  { content: [{ type: 'text', text: '...' }] }           // success
  { content: [{ type: 'text', text: '...' }], isError: true }  // error
  ```
- **Error responses**: use `createErrorResponse()` from `src/error-handlers.ts`
- **Telemetry on new tools**: call `capture('tool_name', { ...sanitizedProps })` in `src/utils/capture.ts` — never include file paths or user data
- **Pagination pattern** — used consistently across file reads, process output, and search results:
  ```typescript
  { lines, totalLines, readFrom, readCount, remaining, isComplete }
  ```

## Configuration Reference

| Key | Default | Notes |
|---|---|---|
| `blockedCommands` | `["sudo","su","passwd","dd","mkfs",...]` | Never empty in production |
| `defaultShell` | OS-detected | `bash`/`zsh`/`powershell` |
| `allowedDirectories` | `[]` (unrestricted) | Filesystem tools only |
| `fileReadLineLimit` | `1000` | Per read call |
| `fileWriteLineLimit` | `50` | Per write call |
| `telemetryEnabled` | `true` | Set to `false` to opt out |

## Testing

Tests live in `test/` and run against the built server via MCP client calls — they are **integration tests**, not unit tests. Run a specific test file:

```bash
node test/specific-test.js
```

When adding a new tool, add a corresponding test file in `test/`. Follow the existing pattern: spawn the MCP server, connect as a client, call the tool, assert the response.

## File Format Support

`src/utils/files/` implements a factory pattern — `getFileHandler(filePath)` returns the right handler:
- `.xlsx/.xls/.xlsm` → ExcelJS (supports range editing: `Sheet1!A1:C10`)
- `.pdf` → unpdf/pdf-lib
- `.docx` → XML-based editing
- `.png/.jpg/.gif/.webp` → base64 image response
- Everything else → text or binary

## Security Notes (Do Not Weaken)

- `CommandManager` must fail **closed** — any parsing ambiguity should deny, not allow
- `set_config_value` must only accept keys from the known allowlist (enforced since v0.2.37)
- Telemetry must sanitize: strip all file paths, directory names, and user-identifiable data before sending
- Docker installation is the only hardened deployment — document this when users report sandbox escapes
