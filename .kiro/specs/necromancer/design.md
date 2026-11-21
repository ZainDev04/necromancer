# Design Document

## Overview

Necromancer splits the work between Kiro and a small MCP server. The server does what needs exact facts and repeatable results: reading npm and OSV, setting up the crypt folder, and running the proof. Kiro's agent does what needs judgement: reading the original code, completing the spec and test generators, and writing the risen package. Hooks connect the two, so every save of risen code triggers a proof.

```
 Kiro agent ──calls──▶ Necromancer MCP server (mcp/server.js, stdio)
     │                    ├─ graveyard_scan ──▶ npm registry, npm downloads, OSV ──▶ site/data/graveyard.json
     │                    ├─ autopsy ─────────▶ npm install original ──▶ crypt/<pkg>/ + .kiro/specs/resurrect-<pkg>/
     │                    └─ prove ───────────▶ child processes: harness, exploits, mutants ──▶ site/data/proofs/<pkg>.json
     │
     ├─ writes crypt/<pkg>/risen/index.js, test/arbitraries.js, test/exploits.js
     └─ hooks: save risen file ▶ prove; save demo package.json ▶ graveyard_scan

 site/ (static) ──reads──▶ site/data/*.json ──published on──▶ GitHub Pages
```

## Components and Interfaces

### MCP server (`mcp/server.js`)

Uses `@modelcontextprotocol/sdk` (`McpServer` and `StdioServerTransport`, loaded with `require`). Registers three tools, each with a zod input schema. Each tool handler is a plain async function in `mcp/tools/`, so it can be tested without MCP. All file paths in tool inputs are resolved relative to the repository root (the folder containing `.kiro`), which the server finds by walking up from `process.cwd()`.

Registered in `.kiro/settings/mcp.json`:

```json
{
  "mcpServers": {
    "necromancer": {
      "command": "node",
      "args": ["mcp/server.js"],
      "disabled": false,
      "autoApprove": ["graveyard_scan", "prove"]
    }
  }
}
```

### Tool `graveyard_scan`

Input: `{ "manifestPath": string }` (default `demo/haunted-app/package.json`).

Steps:
1. Read the manifest; collect `dependencies` and `devDependencies` as `{ name, range }`.
2. For each dependency, in parallel with a limit of 5 at a time:
   - `GET https://registry.npmjs.org/<encoded name>`: versions list, `dist-tags.latest`, `time[<version>]`, `versions[<resolved>].deprecated`.
   - Resolve the version: exact versions are used as they are; otherwise `semver.maxSatisfying(versions, range)`; if null, latest.
   - `GET https://api.npmjs.org/downloads/point/last-week/<name>` → `downloads`.
   - `POST https://api.osv.dev/v1/query` with the resolved version → `vulns[]` mapped to `{ id, aliases, summary }`.
3. Classify (see Correctness Properties P1) and set `risen` from `site/data/proofs/<folder>.json`.
4. Sort and write `site/data/graveyard.json`. Return a short text summary (counts by status, top 5 haunted).

### Tool `autopsy`

Input: `{ "name": string, "version"?: string, "refresh"?: boolean }`.

Steps:
1. Compute the crypt folder name (P3). Create `crypt/<folder>/` with:
   ```json
   { "name": "crypt-<folder>", "private": true,
     "scripts": { "test": "node --test test/equivalence.test.js" },
     "dependencies": { "original": "npm:<name>@<version>" },
     "devDependencies": { "fast-check": "^3.23.2" } }
   ```
2. Run `npm install` in the crypt folder (child process, 120 second timeout).
3. List exports in a child process (`mcp/lib/list-exports.js`): loads `original`, prints JSON `{ defaultIsFunction, functionKeys[] }`. Export names: `default` if the module is a function, plus each function key.
4. Read licence: `node_modules/original/package.json` `license` field and the `LICENSE`/`LICENSE.md` file text if present.
5. Query OSV for the version's CVEs.
6. Copy templates from `mcp/templates/` (only files that do not exist yet, except `equivalence.test.js`, which is always refreshed): `test/equivalence.test.js`, `test/arbitraries.js` (one entry per export, each throwing "TODO: Kiro writes generators for <export>"), `test/exploits.js` (one entry per CVE, each with `run` throwing "TODO"), `risen/.gitkeep`.
7. Write `.kiro/specs/resurrect-<folder>/requirements.md` and `.config.kiro` (only if missing).
8. Return the crypt path, exports, CVE ids and a numbered next-steps list.

### Test files in a crypt folder

`test/arbitraries.js` exports an object whose keys are export names and whose values are fast-check arbitraries of the argument tuple:

```js
const fc = require('fast-check');
module.exports = {
  default: fc.tuple(targetArb, pathArb, valueArb), // lodash.set(object, path, value)
};
```

`test/exploits.js` exports an array:

```js
module.exports = [
  {
    id: 'proto-path-string',
    cve: 'CVE-2020-8203',
    description: "set({}, '__proto__.polluted', 'yes') adds 'polluted' to every object",
    run: (impl) => impl({}, '__proto__.polluted', 'yes'),
    succeeded: () => ({}).polluted === 'yes',
  },
];
```

`run` receives the loaded module. For `lodash.set` the module itself is the `set` function; for packages with several exports, `run` picks the function it needs from the module.

`test/equivalence.test.js` (template, never edited per package): loads `original` and the risen module (`NECRO_IMPL` or `../risen`), and for each key in `arbitraries.js` registers a `node:test` test that runs `fc.assert` with `numRuns` from `NECRO_RUNS` (default 10000). Inside the property: deep-clone the arguments twice with a small recursive clone that keeps `-0`, `NaN` and `undefined` (not `structuredClone`), call both, `assert.deepStrictEqual` on return values and on argument copies.

### Tool `prove`

Input: `{ "name": string }`.

Steps:
1. **Equivalence**: run `node --test --test-reporter=tap test/equivalence.test.js` in the crypt folder (child process, `NECRO_RUNS=10000`, 10 minute timeout). Parse TAP: per test name `ok`/`not ok`; on failure keep the counterexample text.
2. **Exploits**: for each exploit index and each target (`original`, `risen`), spawn `mcp/lib/exploit-runner.js <cryptDir> <target> <index>`. The runner snapshots own property names of `Object.prototype`, `Array.prototype`, `Function.prototype`, calls `run(impl)` inside try/catch, then reports `{ attacked: succeeded() || prototypesChanged, error? }` as one JSON line.
3. **Mutation**: read `risen/index.js`, generate mutants with `mcp/lib/mutate.js` (operators below), drop mutants that fail `node --check`, cap at 12 spread evenly over the file. For each, write it to `os.tmpdir()/necromancer/<folder>/m<i>/index.js`, run the equivalence harness with `NECRO_IMPL` set to it and `NECRO_RUNS=300`, 60 second timeout. Caught = non-zero exit or timeout.
4. Build the Proof_Report (P5), write `site/data/proofs/<folder>.json`, update `site/data/proofs/index.json`. Return a text summary with the headline numbers.

### Mutation operators (`mcp/lib/mutate.js`)

Applied to source text, skipping lines that are comments and text inside string literals:

| Id | Change |
|---|---|
| M1 | `===` to `!==` |
| M2 | `!==` to `===` |
| M3 | `<` to `<=` (not `<=`, `<<`) |
| M4 | `>` to `>=` (not `>=`, `>>`, `=>`) |
| M5 | `&&` to `\|\|` |
| M6 | `\|\|` to `&&` |
| M7 | array literal `[]` to `{}` |

`mutate(source)` returns `[{ id, operator, line, column, source }]`, one per site, in source order. Each mutant differs from the original at exactly one site.

### Graveyard page (`site/`)

- `index.html`: header (name, one-line pitch), the graveyard grid, the proof panel, footer with repo link.
- `app.js`: fetches `data/graveyard.json` and `data/proofs/index.json`, then each proof; renders tombstones as `<button>` elements so they are keyboard focusable; selecting a risen package fills the proof panel.
- `style.css`: dark background (#0f0e13), candle orange (#f2a541) for accents, moss green (#7bc47f) for risen, bone white (#ece6d9) text; serif headings; tombstone shape with CSS (rounded top); ghost as an inline SVG over haunted tombstones with a slow float animation inside `@media (prefers-reduced-motion: no-preference)`.
- Proof panel shows big numbers: "10,000 / 10,000 inputs match", "Exploit: works on original, blocked on risen", "Mutants caught: 11 / 12", then the CVE text and GitHub links to `crypt/<pkg>/risen/index.js` and the original on npm.
- Published by `.github/workflows/pages.yml` (official `actions/upload-pages-artifact` and `actions/deploy-pages`, path `site`).

## Data Models

`site/data/graveyard.json`:

```json
{
  "scannedAt": "2026-11-24T10:00:00.000Z",
  "project": "haunted-app",
  "packages": [
    {
      "name": "lodash.set", "range": "4.3.2", "resolvedVersion": "4.3.2", "latestVersion": "4.3.2",
      "lastRelease": "2016-08-13T00:00:00.000Z", "yearsSinceRelease": 10,
      "weeklyDownloads": 2105076, "deprecated": null,
      "vulns": [{ "id": "GHSA-p6mc-m468-83gw", "aliases": ["CVE-2020-8203"], "summary": "Prototype Pollution in lodash" }],
      "status": "haunted", "risen": true, "error": null
    }
  ]
}
```

(Values above show the shape; real values always come from the API calls.)

`site/data/proofs/<folder>.json`:

```json
{
  "name": "lodash.set", "version": "4.3.2", "provedAt": "...",
  "properties": [{ "name": "default", "runs": 10000, "passed": true, "counterexample": null }],
  "exploits": [{ "id": "proto-path-string", "cve": "CVE-2020-8203", "original": "attacked", "risen": "blocked" }],
  "mutation": { "total": 12, "caught": 11, "score": 0.9167, "survivors": [{ "operator": "M4", "line": 57 }] },
  "passed": true
}
```

`site/data/proofs/index.json`: `{ "packages": ["lodash.set"] }`.

## Error Handling

- Network calls: 10 second timeout, one retry after 1 second, then the error message is stored on that record. The scan never fails as a whole because of one package.
- Child processes: every spawn has a timeout; a timeout in the harness is reported as a failure with the message "timed out after N seconds".
- Tool inputs are validated (package names against npm's naming rules; manifest path must exist and be inside the repo). Invalid input returns an MCP error result, never a crash.
- All tool handlers catch errors and return `{ isError: true, content: [{ type: "text", text: message }] }`.

## Testing Strategy

- Unit and property tests for the server's own logic in `mcp/test/`, run with `npm test` at the repo root.
- Network code is tested with an injected `fetch` that returns recorded responses (no live calls in tests).
- The resurrected package is tested by its own crypt harness through `prove`.

## Correctness Properties

These are checked with fast-check in `mcp/test/`.

- **P1 Classification.** For any Package_Record with arbitrary vulns count, deprecation and years since release: status is `haunted` if and only if vulns is non-empty; otherwise `dead` if and only if deprecated is a string or years ≥ 3; otherwise `alive`.
- **P2 Years since release.** For any release date not in the future, years since release is a non-negative integer, and a later release date never gives a larger value.
- **P3 Crypt folder names.** For any valid npm name (scoped or not), converting to a folder name and back gives the same name, and the folder name contains no `/`.
- **P4 Mutants.** For any JavaScript source string, every mutant returned by `mutate` differs from the source at exactly one site, applies the operator its id names, and `mutate` returns the same list when called twice on the same source.
- **P5 Proof verdict.** For any combination of property results, exploit results and mutation counts, `passed` is true if and only if all properties passed with 10,000 runs, every exploit is `attacked` on the original and `blocked` on the risen, and caught / total ≥ 0.8 (with total > 0).
- **P6 Sorting.** For any list of Package_Records, the written order has all haunted before dead before alive, and within a status weekly downloads never increase.
