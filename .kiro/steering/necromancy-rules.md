---
inclusion: always
---

# Rules for raising a package

These rules apply whenever Kiro writes or changes code under `crypt/<package>/risen/`.

1. **Read the original first.** Read the original package's source in `crypt/<package>/node_modules/original/` completely before writing anything. List every exported function and every behaviour you find, including odd edge cases, in the package's spec (`.kiro/specs/resurrect-<package>/requirements.md`).
2. **Same behaviour, one exception.** For every input that is not an attack, the risen version must return the same value and make the same changes to its arguments as the original. The only allowed difference is closing the package's known security holes, and that difference must be written down in the spec with the CVE id.
3. **Rewrite, do not copy.** Write modern, readable JavaScript: `const` and `let`, small named functions, JSDoc types, clear names, comments only where the behaviour is surprising. Do not paste the original's code. The proof tests are what make a rewrite safe.
4. **No new dependencies.** A risen package has zero runtime dependencies.
5. **Keep the licence.** Copy the original's licence text and copyright notice into `risen/LICENSE`, and add a line saying the code was rewritten for Necromancer in 2026.
6. **Prove it.** A package is risen only when `prove` reports: every property passes 10,000 runs, every exploit works on the original and fails on the risen version, and at least 80% of mutants are caught. Look at every surviving mutant: if it behaves exactly like the original it is harmless, otherwise add a property that catches it. If a property fails, read the counterexample, fix the risen code (not the test), and run `prove` again.
7. **Never weaken a test to make it pass.** Do not lower the run count, narrow the generators to avoid a failing case, or delete an exploit. If a test itself is wrong, say why in the chat before changing it.
8. **Record what the tests found.** When a property fails and leads to a fix, add a short note to the package's spec under "Behaviours found by testing". These notes go into the Kiro write-up.
