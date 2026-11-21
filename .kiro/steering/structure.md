---
inclusion: always
---

# Repository structure

```
necromancer/
  .kiro/
    steering/            product, tech, structure, necromancy rules
    specs/               necromancer (this project) and resurrect-<package> specs
    settings/mcp.json    registers the Necromancer MCP server with Kiro
    hooks/               agent hooks (created in Kiro's hook editor)
  mcp/
    server.js            MCP server entry point (stdio)
    tools/
      graveyard-scan.js  graveyard_scan tool
      autopsy.js         autopsy tool
      prove.js           prove tool
    lib/                 shared helpers: npm and OSV clients, file writing, mutation operators
    templates/           files that autopsy copies into a new crypt folder
    test/                tests for the MCP tools
  crypt/
    <package>/
      package.json       depends on the original via an npm alias: "original": "npm:<name>@<version>"
      risen/index.js     the resurrected package, written by Kiro
      risen/LICENSE      original licence and copyright, kept
      test/arbitraries.js   fast-check generators for each exported function's arguments
      test/exploits.js      known attack inputs for the package's CVEs
      test/equivalence.test.js   generic harness: original vs risen
  demo/
    haunted-app/package.json   sample project with dead and living dependencies
  site/
    index.html, style.css, app.js   the Graveyard page
    data/graveyard.json             written by graveyard_scan
    data/proofs/<package>.json      written by prove
  README.md
  LICENSE
```

- A "crypt folder" is `crypt/<package>/`. `<package>` is the npm name with `/` replaced by `__` (for scoped packages).
- Generated data files under `site/data/` are committed, so the deployed page works without running anything.
