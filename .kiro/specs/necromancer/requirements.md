# Requirements Document

## Introduction

Necromancer is a Kiro Power for raising abandoned npm packages from the dead. It has three parts: an MCP server with three tools (`graveyard_scan`, `autopsy`, `prove`) that Kiro's agent calls; Kiro steering files and hooks that guide the rewrite; and a static web page, the Graveyard, that shows which dependencies are dead and the proof for each package that has been raised. Kiro's own agent writes the resurrected code; Necromancer supplies the facts, the scaffolding and the proof.

The first package raised is `lodash.set` 4.3.2 (CVE-2020-8203, prototype pollution).

## Glossary

- **Necromancer_Server**: the MCP server in `mcp/server.js`, started by Kiro over stdio.
- **Project_Manifest**: a `package.json` file whose `dependencies` and `devDependencies` are scanned.
- **Package_Record**: the facts gathered for one dependency (see Requirement 2).
- **Haunted**: a package whose resolved version has at least one open vulnerability in OSV.
- **Dead**: a package that is not Haunted and is either deprecated on npm or has had no release for 3 years or more.
- **Alive**: a package that is neither Haunted nor Dead.
- **Crypt_Folder**: `crypt/<folder-name>/`, where `<folder-name>` is the npm name with `/` replaced by `__`.
- **Original**: the dead package installed in the Crypt_Folder under the npm alias `original`.
- **Risen**: the rewritten package in `crypt/<folder-name>/risen/`.
- **Property**: a fast-check property that runs the Original and the Risen on the same generated arguments and asserts the results and argument changes are deeply equal.
- **Exploit**: a known attack input for a CVE of the package, with a check that reports whether the attack succeeded.
- **Mutant**: a copy of the Risen with one small deliberate change made by a mutation operator.
- **Mutation_Score**: caught Mutants divided by total Mutants.
- **Proof_Report**: the JSON result written by `prove` to `site/data/proofs/<folder-name>.json`.
- **Graveyard_Page**: the static page in `site/`.

## Requirements

### Requirement 1: MCP server

**User Story:** As a Kiro user, I want Necromancer's tools available inside Kiro, so that Kiro's agent can scan, prepare and prove packages without leaving the IDE.

#### Acceptance Criteria

1. THE Necromancer_Server SHALL expose exactly three tools named `graveyard_scan`, `autopsy` and `prove`, each with a JSON input schema and a one-sentence description.
2. WHEN Kiro starts the server using the command in `.kiro/settings/mcp.json`, THE Necromancer_Server SHALL be ready to accept tool calls over stdio within 5 seconds.
3. IF a tool call fails because of a network error, a missing file or invalid input, THEN THE Necromancer_Server SHALL return an error result with a plain-language message and SHALL keep running.
4. THE Necromancer_Server SHALL write nothing to stdout except MCP protocol messages, and SHALL write logs to stderr.

### Requirement 2: Graveyard scan

**User Story:** As a developer, I want to see which of my dependencies are dead or haunted, so that I know which ones put my project at risk.

#### Acceptance Criteria

1. WHEN `graveyard_scan` is called with the path of a Project_Manifest, THE Necromancer_Server SHALL read every entry in `dependencies` and `devDependencies`.
2. WHEN a dependency's version range is an exact version, THE Necromancer_Server SHALL use that version as the resolved version; otherwise it SHALL use the highest published version that satisfies the range, and if none does, the latest version.
3. FOR each dependency, THE Necromancer_Server SHALL record in a Package_Record: name, range, resolved version, latest version, date of the latest release, whole years since that release, weekly downloads for the last 7 days, the npm deprecation message or null, and the list of OSV vulnerabilities affecting the resolved version (id, aliases, summary).
4. THE Necromancer_Server SHALL classify each Package_Record as Haunted, Dead or Alive using the definitions in the Glossary, in that order of priority.
5. THE Necromancer_Server SHALL mark a Package_Record as risen WHEN a Proof_Report for that package exists and reports passed.
6. WHEN the scan finishes, THE Necromancer_Server SHALL write `site/data/graveyard.json` containing the scan time, the project name and all Package_Records sorted Haunted first, then Dead, then Alive, then by weekly downloads descending.
7. IF the registry, downloads or OSV call for one dependency fails, THEN THE Necromancer_Server SHALL record the error on that Package_Record and continue with the other dependencies.

### Requirement 3: Autopsy

**User Story:** As a developer, I want one command to prepare a dead package for rewriting, so that Kiro starts from the real original, a starter spec and a working test harness.

#### Acceptance Criteria

1. WHEN `autopsy` is called with a package name and an optional version, THE Necromancer_Server SHALL create the Crypt_Folder with a `package.json` that installs the Original as `"original": "npm:<name>@<version>"` (latest version if none is given) and `fast-check` as a dev dependency, and SHALL run `npm install` in it.
2. THE Necromancer_Server SHALL list the Original's exported functions by loading it in a child process: the default export if it is a function, plus every own property whose value is a function.
3. THE Necromancer_Server SHALL copy the test harness template to `test/equivalence.test.js`, and SHALL create `test/arbitraries.js` with one entry per exported function and `test/exploits.js` with one entry per CVE found in OSV, each entry marked for Kiro to complete.
4. THE Necromancer_Server SHALL create `.kiro/specs/resurrect-<folder-name>/requirements.md` containing the package name and version, its licence, its exported functions, its CVEs with summaries, and empty sections "Behaviours of the original", "Security change" and "Behaviours found by testing".
5. IF the Crypt_Folder already exists, THEN THE Necromancer_Server SHALL leave `risen/`, `test/arbitraries.js` and `test/exploits.js` unchanged and SHALL only refresh `package.json`, the installed Original and the harness template.
6. WHEN the autopsy finishes, THE Necromancer_Server SHALL return the Crypt_Folder path, the exported function names, the CVE ids and the next steps for Kiro.

### Requirement 4: Equivalence harness

**User Story:** As a maintainer, I want the risen package compared with the original on thousands of random inputs, so that I can trust it as a drop-in replacement.

#### Acceptance Criteria

1. FOR each entry in `test/arbitraries.js`, THE harness SHALL run a Property that deep-clones the generated arguments twice, calls the Original's function with one copy and the Risen's function with the other, and asserts with `assert.deepStrictEqual` that the return values and the argument copies are equal.
2. THE harness SHALL use 10,000 runs per Property by default, and SHALL read a different run count from the environment variable `NECRO_RUNS`.
3. THE harness SHALL load the Risen from the path in the environment variable `NECRO_IMPL` when it is set, and from `../risen` otherwise.
4. IF a Property fails, THEN THE harness SHALL report the shrunk counterexample and the seed.
5. THE harness SHALL never call the Original with an Exploit input.

### Requirement 5: Proof

**User Story:** As a maintainer, I want a single proof report, so that I can see at a glance whether a package is safe to switch to.

#### Acceptance Criteria

1. WHEN `prove` is called with a package name, THE Necromancer_Server SHALL run the equivalence harness in a child process with 10,000 runs per Property and record, per Property, the runs completed and whether it passed.
2. FOR each Exploit, THE Necromancer_Server SHALL run it in a separate child process against the Original and again against the Risen, and record whether the attack succeeded in each.
3. THE Necromancer_Server SHALL check for prototype pollution after each Exploit by comparing the own property names of `Object.prototype`, `Array.prototype` and `Function.prototype` before and after the call, in addition to the Exploit's own check.
4. THE Necromancer_Server SHALL create up to 12 Mutants of the Risen using the mutation operators in the design, run the harness against each with 300 runs per Property, and count a Mutant as caught WHEN any Property fails or the run times out after 60 seconds.
5. THE Necromancer_Server SHALL write a Proof_Report containing: package name and version, timestamp, per-Property results, per-Exploit results for Original and Risen, Mutation_Score with the list of surviving Mutants (operator and line), and `passed`.
6. THE Proof_Report SHALL set `passed` to true only WHEN every Property passed with 10,000 runs, every Exploit succeeded on the Original and failed on the Risen, and the Mutation_Score is at least 0.8.
7. WHEN the Proof_Report is written, THE Necromancer_Server SHALL add the package to `site/data/proofs/index.json` if it is not already listed.
8. IF an Exploit does not succeed on the Original, THEN THE Proof_Report SHALL mark that Exploit as "not reproduced" and `passed` SHALL be false.

### Requirement 6: Resurrection of lodash.set

**User Story:** As a judge, I want to see one real package raised end to end, so that I can see Necromancer works.

#### Acceptance Criteria

1. THE repository SHALL contain `crypt/lodash.set/` with the Risen written by Kiro, `risen/LICENSE` with lodash's MIT licence and copyright, completed `test/arbitraries.js` and `test/exploits.js`, and a completed `.kiro/specs/resurrect-lodash.set/requirements.md`.
2. THE Proof_Report for `lodash.set` SHALL have `passed` set to true.
3. THE Risen for `lodash.set` SHALL block paths containing `__proto__`, `constructor` or `prototype` in the same way as lodash 4.17.21 (stop at the first such key and return the object).

### Requirement 7: Graveyard page

**User Story:** As a visitor, I want to see the graveyard and the proofs on one page, so that I understand the risk and the fix in seconds.

#### Acceptance Criteria

1. THE Graveyard_Page SHALL load `data/graveyard.json` and every Proof_Report listed in `data/proofs/index.json` and render without a build step or server code.
2. THE Graveyard_Page SHALL show each Package_Record as a tombstone with the package name, resolved version, years since release and weekly downloads; Haunted packages SHALL show a ghost and their CVE ids; risen packages SHALL be visibly marked as risen.
3. WHEN a visitor selects a risen package, THE Graveyard_Page SHALL show its Proof_Report: runs matched out of runs, Exploit results for Original and Risen, the Mutation_Score, and links to the original and risen source on GitHub.
4. THE Graveyard_Page SHALL be usable with a keyboard, have text contrast of at least 4.5:1, work at 360 pixels wide, and turn off animations WHEN the visitor's system asks for reduced motion.
5. IF a data file fails to load, THEN THE Graveyard_Page SHALL show a plain message saying which file failed.
6. THE Graveyard_Page SHALL be published on GitHub Pages from the `site/` folder.

### Requirement 8: Kiro integration

**User Story:** As a Kiro user, I want the workflow automated, so that the proof runs without me remembering to run it.

#### Acceptance Criteria

1. THE repository SHALL register the Necromancer_Server in `.kiro/settings/mcp.json`.
2. WHEN a file under `crypt/*/risen/` is saved, an agent hook SHALL run `prove` for that package and report the result in the chat.
3. WHEN `demo/haunted-app/package.json` is saved, an agent hook SHALL run `graveyard_scan` on it.
4. THE repository SHALL contain `demo/haunted-app/package.json` listing at least `lodash.set@4.3.2`, `lodash.pick@4.4.0`, `xmldom@0.6.0`, `request@2.88.2` and at least three actively maintained packages.
