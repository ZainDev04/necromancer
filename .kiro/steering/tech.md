---
inclusion: always
---

# Tech stack and conventions

## Runtime

- Node.js 22 or newer (development machine: Node 24 on Windows 10; test file patterns need 22+). CommonJS (`require`, `module.exports`). No TypeScript build step; use JSDoc comments for types.
- Paths must work on Windows and Linux: use `path.join`, never hard-coded `/` or `\` separators.

## Dependencies (keep this list short)

- `@modelcontextprotocol/sdk` for the MCP server (stdio transport), loaded with `require('@modelcontextprotocol/sdk/server/mcp.js')` and `require('@modelcontextprotocol/sdk/server/stdio.js')`.
- `zod` for the tools' input schemas (the MCP SDK expects zod schemas).
- `fast-check` for property-based tests.
- `semver` for resolving version ranges.
- Node built-ins for everything else: `node:test`, `node:assert/strict`, `node:child_process`, `node:fs`, global `fetch`.
- Do not add any other dependency without saying why in the chat first.
- The Graveyard page uses plain HTML, CSS and JavaScript with no framework and no build step.

## External APIs (public, no keys)

- npm registry metadata: `https://registry.npmjs.org/<name>` (versions, `time`, `deprecated`).
- npm downloads: `https://api.npmjs.org/downloads/point/last-week/<name>`.
- OSV vulnerabilities: `POST https://api.osv.dev/v1/query` with `{"package":{"name":"<name>","ecosystem":"npm"},"version":"<version>"}`.
- Scoped package names (`@scope/name`) must be URL-encoded as `@scope%2Fname` for the registry.
- Every network call has a timeout (10 seconds) and a clear error message; a failed call never crashes the MCP server.

## Testing

- `npm test` runs `node --test "mcp/test/**/*.test.js"`. Always pass file patterns to `node --test`, never a bare folder name.
- Property tests use fast-check with a fixed seed when reproducing a failure, and print the seed on failure.
- Code that can pollute prototypes (the original packages) is only ever run with exploit inputs inside a separate child process, never in the main process.

## Git

- Commits are authored by "Zain" only. Never add `Co-authored-by` lines or any mention of AI tools in commit messages.
- Never commit `node_modules`, `.env` files or downloaded tarballs. Never put `.kiro` in `.gitignore`.
