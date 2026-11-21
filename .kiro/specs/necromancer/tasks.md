# Implementation Plan

Work in this order. Each task is small enough for one Kiro run in Default mode. Run `npm test` after every task. Tasks marked (extra) are cut first if time or credits run short.

- [ ] 1. Project setup
  - Create root `package.json` (name `necromancer`, private, `"engines": { "node": ">=22" }`, `"test": "node --test \"mcp/test/**/*.test.js\""`), add `@modelcontextprotocol/sdk`, `zod`, `fast-check` and `semver`.
  - Create `.kiro/settings/mcp.json` as in the design.
  - Create `demo/haunted-app/package.json` with `lodash.set` 4.3.2, `lodash.pick` 4.4.0, `xmldom` 0.6.0, `request` 2.88.2, `querystring` 0.2.1, and maintained packages `express`, `chalk` and `zod` (caret ranges).
  - _Requirements: 1.1, 8.1, 8.4_

- [ ] 2. Shared helpers in `mcp/lib/`
  - [ ] 2.1 `crypt-name.js`: npm name to folder name and back; property test P3.
    - _Requirements: 3.1_
  - [ ] 2.2 `classify.js`: status, years since release, sorting; property tests P1, P2, P6.
    - _Requirements: 2.3, 2.4, 2.6_
  - [ ] 2.3 `npm-client.js` and `osv-client.js`: registry metadata, weekly downloads, OSV query, with injectable `fetch`, 10 second timeout and one retry; unit tests with recorded responses.
    - _Requirements: 2.3, 2.7_

- [ ] 3. MCP server skeleton
  - `mcp/server.js` with stdio transport, three registered tools with input schemas, error results instead of crashes, logs on stderr only.
  - Check in Kiro that the server shows as connected and lists the three tools.
  - _Requirements: 1.1, 1.2, 1.3, 1.4_

- [ ] 4. `graveyard_scan`
  - Implement as in the design; run it on `demo/haunted-app/package.json`; commit the resulting `site/data/graveyard.json`.
  - _Requirements: 2.1 to 2.7_

- [ ] 5. `autopsy`
  - [ ] 5.1 Templates in `mcp/templates/`: `equivalence.test.js` (harness), `arbitraries.js`, `exploits.js`, spec skeleton.
    - _Requirements: 3.3, 3.4, 4.1 to 4.5_
  - [ ] 5.2 `mcp/lib/list-exports.js` (child process) and the `autopsy` tool, including the refresh rule.
    - _Requirements: 3.1, 3.2, 3.5, 3.6_
  - [ ] 5.3 Run `autopsy` for `lodash.set` 4.3.2 and check the crypt folder and spec skeleton.
    - _Requirements: 3.1 to 3.6_

- [ ] 6. `prove`
  - [ ] 6.1 `mcp/lib/mutate.js` with operators M1 to M7; property test P4.
    - _Requirements: 5.4_
  - [ ] 6.2 `mcp/lib/exploit-runner.js` (child process, prototype snapshot).
    - _Requirements: 5.2, 5.3, 5.8_
  - [ ] 6.3 `prove` tool: harness run, exploits, mutants, verdict (property test P5), report and index file.
    - _Requirements: 5.1 to 5.8_

- [ ] 7. Hooks (create in Kiro's hook editor; they are saved under `.kiro/hooks/`)
  - "Prove on save": when a file matching `crypt/*/risen/**/*.js` is saved, ask the agent to run `prove` for that package and report the headline numbers, explaining any counterexample.
  - "Rescan graveyard": when `demo/haunted-app/package.json` is saved, ask the agent to run `graveyard_scan` on it.
  - _Requirements: 8.2, 8.3_

- [ ] 8. Raise `lodash.set`
  - [ ] 8.1 Read the original fully; complete `.kiro/specs/resurrect-lodash.set/requirements.md` (every behaviour, the security change with CVE-2020-8203).
    - _Requirements: 6.1_
  - [ ] 8.2 Complete `test/arbitraries.js` (objects and arrays nested up to depth 4; string paths with dots, brackets, quotes, escapes and numbers; array paths of strings and numbers; values including `NaN`, `-0` and `undefined`; keys that can never form `__proto__`, `constructor` or `prototype`) and `test/exploits.js` (string path, array path, bracket path, `constructor.prototype` path).
    - _Requirements: 4.1, 6.1_
  - [ ] 8.3 Write `risen/index.js` following the necromancy rules, and `risen/LICENSE`. The hook runs `prove`; fix until `passed` is true. Add notes under "Behaviours found by testing".
    - _Requirements: 6.1, 6.2, 6.3_

- [ ] 9. Graveyard page
  - [ ] 9.1 `site/index.html`, `style.css`, `app.js` as in the design, reading the committed data files.
    - _Requirements: 7.1, 7.2, 7.3, 7.5_
  - [ ] 9.2 Accessibility pass: keyboard, contrast, 360 px width, reduced motion.
    - _Requirements: 7.4_
  - [ ] 9.3 `.github/workflows/pages.yml`; enable GitHub Pages (source: GitHub Actions); check the live page.
    - _Requirements: 7.6_

- [ ] 10. README
  - What it is, the lodash.set result with real numbers, how to install and run (MCP config, `autopsy`, `prove`), how Kiro was used, licence note that original packages are used only as test references under their own licences.

- [ ] 11. (extra) Raise `lodash.pick` 4.4.0 the same way as task 8.

- [ ] 12. (extra) Package as a Kiro Power (MCP server, steering and hooks) following Kiro's current Powers documentation.
