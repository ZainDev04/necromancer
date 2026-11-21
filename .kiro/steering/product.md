---
inclusion: always
---

# Product: Necromancer

## What it is

Necromancer raises abandoned npm packages from the dead. It finds dependencies that nobody maintains any more, downloads the dead package, writes a spec of exactly how it behaves, lets Kiro rebuild it as clean modern code, and proves with property-based tests that the new version behaves the same as the original on thousands of random inputs, except for its known security hole, which is closed.

It is built for Kiroween 2026, category **Resurrection** ("bring your favorite dead technology back to life").

## Who it is for

Developers and maintainers whose projects depend on packages that are old, deprecated or have unpatched CVEs, and who cannot simply switch to an alternative because they do not know what behaviour would change.

## The demo story

1. `lodash.set` 4.3.2: about 2.1 million downloads a week, last released 13 Aug 2016, open hole CVE-2020-8203 (prototype pollution). One line, `set({}, '__proto__.polluted', 'yes')`, adds a property to every object in the program.
2. The Graveyard shows a sample project's dependencies as tombstones; dead packages with known holes have a ghost over them.
3. Inside Kiro: `autopsy` downloads the package and writes the starter spec and test harness; Kiro raises the package from the spec; a hook runs `prove` on every save.
4. `prove` shows: 10,000 of 10,000 random inputs give the same result as the original; the exploit works on the original and fails on the risen version; deliberately broken copies of the risen code are all caught by the tests.
5. Next target: `xmldom` 0.6.0 (about 1.6 million downloads a week, 16 open CVEs).

## Product rules

- Every number shown to users (downloads, dates, CVE counts, test counts) comes from a real API call or a real test run. Never type a number in by hand.
- Proof over promises: a package only counts as "risen" when `prove` passes.
- The original package's licence and copyright notice are kept in every risen package.
- Public text (README, page copy, video) says "we" and uses plain language: short sentences, no hype words, no emoji.
