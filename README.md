# bugfix — Hermes Agent skill

Standing procedure for every bug fix in a skill or codebase: baseline probe list → fix →
re-analysis → comparison → commit gate → clean install. Not only for big refactors.

## Order of operations (mandatory)

1. **Baseline analysis — fresh, on the code as it is now** (not from memory, not from the
   previous audit): re-run the probes. Three levels — критичные / мажорные / минорные, each
   item with a measurement or a case where «как есть» and «по правилу» give a DIFFERENT
   result. Without a measurement an item is an assumption, not a finding. Save the baseline
   to `scratch/bugfix_baseline.md`; the comparison is against the recorded list.
2. **Fix** — one item at a time, with a behavior test that fails before the fix and passes after.
3. **Re-analysis** — the SAME probe set over the code after the fix, plus an explicit
   regression hunt for the classes the fix itself introduces.
4. **Compare the two lists.** New items = regressions from the fix.
5. **Commit gate** — commit is allowed ONLY when the after-list has zero new items relative
   to the baseline. If the fix produced new bugs, fix them and re-compare; loop until clean.
   The commit message states baseline count, after count, new items, and the factual test
   count from the actual run.
6. **Clean install check** — after the commit, verify the skill installs clean on a fresh
   environment (clone from the LOCAL repo, temporary HOME, `setup.sh`, minimal fresh Hermes,
   every mode, state dir created, guard tests cover both paths). Report the pass count.

## Probe rules (without them a test gives a fake «success»)

- Audit by probe, not by reading: rule + minimal case where «как есть» and «с фиксом» differ.
- Cover the whole set of the skill's scripts, not only the layer being discussed.
- Dictionary lists are checked for Latin homoglyphs inside Cyrillic words before commit.
- Guard tests scan BOTH skill paths and the whole form of the pattern, not one spelling.
- State in probes must be the canonical shape (some keys None), not a set of truthy fakes.
- Call-count tests count the calls of the WHOLE path, not only the intended one.
- Network scripts become testable only with an offline/dry-run mode plus a mock transport.
- Spy tests work only when the caller imports the same module object; restore in `finally`.
- Every test class restores patched module attributes through the shared SnapshotMixin.
- New modules read files with `with open(...)`; the closed-files test scans all skill paths.

## Two classes only a clean-install run exposes

- **Import-time reads of installer-created artifacts.** Files created by `setup.sh` are not in
  the repo, so a module that reads them at import crashes on a fresh clone: the skill does not
  degrade, it breaks entirely, and the crash surfaces through the test suite. Tests of logic
  must not depend on an installed model — fake the catalog and bypass the availability check;
  the module must answer an honest `unavailable` instead of crashing.
- **Process-global pollution.** A plugin that inserts its skill path into `sys.path` at import
  and never removes it makes a later import in the same process resolve to the author's home
  copy, not the clean clone. The test passed alone and failed in the full run — order-dependent
  failures are evidence of shared mutable state, not of a broken test.

## Report format (to the user, in Russian)

Three levels with measurements, a «было / стало» counts table per level, an explicit line
«new items = 0» (or the list of new items), the factual test count from the actual run, and
which modules are tested behaviorally and which are not tested at all.

## Pitfalls

- A compile pass (`py_compile`) is NOT a clean install and not verification: run one read-only
  live check after editing scripts.
- Never quote remembered numbers — re-measure and write the factual number.
- If a fix renames something consumers read (a response key, an env var), check the consumers.
- Documentation drift is a real item: CHANGELOG/README promising behavior the code does not
  implement counts as a minor bug and must be fixed in the same round or listed as open.
- After editing a module, re-run the probe only after `importlib.reload`.
