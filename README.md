# Necromancer

Raise abandoned npm packages from the dead with Kiro, and prove they still behave the same.

Built for Kiroween 2026 (Resurrection category).

## Status

The specs and steering files in `.kiro/` are written. The code is being built from `.kiro/specs/necromancer/tasks.md`, so the commands below describe the design until those tasks are ticked. Results will be added here after the first full proof run, and every number will come from that run.

## The problem

Many projects depend on npm packages that nobody maintains any more. Some have known security holes that will never get a fix. Rewriting one by hand is risky, because other code depends on its exact behaviour, including the odd edge cases.

Our first target is `lodash.set` 4.3.2, last released in August 2016, with an open prototype pollution hole (CVE-2020-8203). One call changes every object in the program:

```js
const set = require('lodash.set'); // 4.3.2
set({}, '__proto__.isAdmin', true);
console.log({}.isAdmin); // true
```

## How it works

Necromancer is an MCP server with three tools that Kiro's agent calls:

- `graveyard_scan` reads a project's dependencies and looks each one up in the npm registry, the npm download counts and the OSV vulnerability database. A package with a known hole is haunted. A package that is deprecated, or has had no release for three years, is dead.
- `autopsy` installs the original package into a crypt folder, lists what it exports, and writes a starter spec and a test harness.
- `prove` checks the rebuilt package against the original and writes a proof report.

Kiro's agent reads the original code, completes the spec and writes the new version. A hook runs `prove` each time that code is saved.

## What the proof checks

A package counts as risen only if it passes all three checks:

1. Equivalence: for every exported function, 10,000 random inputs give the same result and the same changes to the arguments as the original. The inputs come from fast-check.
2. Exploits: each known attack works on the original and is blocked on the new version. Attacks run in separate child processes.
3. Mutation: we break the new code on purpose, and the tests must catch at least 80% of the broken copies.

## Requirements

- Node.js 22 or newer
- Kiro
- No API keys: the npm and OSV endpoints are public

## Run it

```bash
git clone https://github.com/ZainDev04/necromancer.git
cd necromancer
npm install
npm test
```

Open the folder in Kiro. It reads `.kiro/settings/mcp.json` and starts the server. Then ask Kiro's agent:

1. "Run graveyard_scan on demo/haunted-app/package.json."
2. "Run autopsy on lodash.set 4.3.2."
3. "Raise lodash.set, following its spec and the necromancy rules."
4. "Run prove on lodash.set."

Results are written to `site/data/`, and the Graveyard page in `site/` shows them.

## Repository layout

- `.kiro/`: steering files, specs, hooks and MCP settings
- `mcp/`: the MCP server, its tools and its tests
- `crypt/`: one folder per package, with the original as the test reference, the risen version and its tests
- `demo/haunted-app/`: a sample `package.json` with dead and living dependencies
- `site/`: the Graveyard page

## How we used Kiro

- Specs: we wrote requirements, design and tasks before any code, and Kiro builds one task at a time. `autopsy` creates a spec for each package it opens.
- Steering: `necromancy-rules.md` is the contract for every resurrection. The other three files describe the product, the tech and the structure.
- Hooks: one runs `prove` when risen code is saved, and one runs `graveyard_scan` when the demo project's dependencies change.
- MCP: the server does the work that needs exact, repeatable answers, and Kiro's agent does the work that needs judgement.

## Team

- Shaikh Muhammad Zain ([ZainDev04](https://github.com/ZainDev04))
- Syed Mohammad Huzaifa ([huzaifaguru](https://github.com/huzaifaguru))
- Maaz Rehman ([maazrehman99](https://github.com/maazrehman99))

## Licence

MIT, see [LICENSE](LICENSE). Original packages are used only as test references, under their own licences. Each risen package keeps the original licence and copyright notice.
