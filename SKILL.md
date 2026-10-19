---
name: bugfix
description: "Use when fixing a bug: baseline, fix, re-analyze, compare."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Bugfix, Audit, Testing, Commit, CleanInstall]
    related_skills: [systematic-debugging, test-driven-development, hermes-agent-skill-authoring]
---

# Bugfix workflow (baseline → fix → re-analyze → compare → commit → clean install)

## When to use

When a bug is being fixed in a skill or codebase — a reported bug, an audit finding, or a bug found while working. Standing procedure for every bug fix, not only for big refactors: baseline list, fix, re-analysis, comparison, commit gate, clean install.

## Order of operations (mandatory)

1. **Baseline analysis — fresh, on the code as it is now.** Not from memory and not from the previous audit: re-run the probes. Output the list in three levels (критичные / мажорные / минорные), each item with a measurement or a case where «как есть» and «по правилу» give a DIFFERENT result. Without a measurement an item is an assumption, not a finding. Save the baseline to a file (`scratch/bugfix_baseline.md`) — the comparison must be against the recorded list, not against recollection.
2. **Fix** — one item at a time, with a behavior test that fails before the fix and passes after.
3. **Re-analysis** — run the SAME probe set over the code after the fix, plus an explicit regression hunt for the classes the fix itself introduces (new behavior is new code and can carry new bugs).
4. **Compare the two lists.** New items = regressions from the fix. Commit is allowed ONLY when the after-list has zero new items relative to the baseline. If new bugs appeared, fix them and re-compare — loop until the after-list is clean, then commit.
5. **Commit** — only after a clean comparison. The commit message states: baseline count, after count, new items = 0, and the factual test count from the actual run (never a remembered number).
6. **Clean install check** — after the commit, verify the skill installs clean on a fresh environment.

## Baseline / re-analysis probe rules

- Audit by probe, not by reading: each item = rule + minimal case where «как есть» and «с фиксом» give a different result. If they match, the bug is not proven.
- Cover the whole set of the skill's scripts, not only the layer being discussed: delivery paths (telegram, rutube, status post), calling modules, network scripts, worker/lifecycle, parsers.
- Dictionary lists (markers, lexicons, language dicts, tag word lists) are checked for Latin letters inside Cyrillic words before commit: a homoglyph gives no syntax error and no test failure — the word simply never matches and the rule silently stops working.
- Guard tests must scan BOTH skill paths (the skill dir and its sibling paths) and the whole form of the pattern, not one spelling. A narrowed pattern passes while the logic is broken.
- State in probes must be the canonical shape (some keys None), not a set of truthy fakes: if every key is faked, the case «nothing is actually there» is unobservable.
- A call-count test counts the calls of the WHOLE path, not only the intended one: if a helper measures the metric itself, the path makes two calls.
- Network scripts become testable only with an offline/dry-run mode (prints what it would send) plus a mock transport; otherwise the test checks source text and passes on broken logic.
- Spy tests work only when the caller imports the same module object: patch the attribute (`markers.scan`, `classifier.classify`) and restore it in `finally`.
- Every test class restores patched module attributes through the shared SnapshotMixin — never hand-write `finally`.
- A protocol test must read the process's startup banner before the first op response and use the protocol's actual key names: reading a response to a nonexistent op shifts every later answer (the test reads the wrong line), and a request with a wrong key name returns an empty result that reads as "the feature is broken" instead of "the test is wrong". Distinguish an empty result from a missing response, and test the unknown-op branch separately.

## Report format (to the user, in Russian)

- Three levels: критичные (меняют доставляемый результат), мажорные (меняют цифры или поведение в отдельных случаях), минорные (мёртвый код, декоративные наборы, расхождение docstring/CHANGELOG с фактическим выводом).
- Each item with the measurement or the case that proves a different result.
- Counts table «было / стало» per level, plus an explicit line: new items = 0 (or the list of new items if the fix produced them).
- Factual test count from the actual run, plus which modules are tested behaviorally and which are not tested at all.

## Commit gate

Commit only if new bugs == 0. If the fix introduced new bugs: fix them, re-run the analysis, compare again. Do not commit a fix that produced a regression — the commit is the last step of a clean round, not the first.

## Clean install check (after commit)

Run the skill's clean-install harness (or build one if absent): clone from the LOCAL repo, temporary HOME, `setup.sh`, minimal fresh Hermes (only the tables the skill needs), full run of every mode (dry-run → tick → manual → commit), verify the state dir is created in the temp HOME, verify profile/config values are read from the real config files, verify the guard tests cover both paths. Report the pass count. A compile pass (`py_compile`) is NOT a clean install: it passes scripts that fail live.

Ordering: the harness clones the COMMITTED state, so the fixes must be committed before the run — the commit gate is satisfied by the comparison, and any bug the clean-install run finds is fixed, re-verified and committed in the same round (the harness itself is part of the round and gets committed too). Network steps in the install path are verified as the printed plan, not as a real install, and the harness must say so in its report.

Two classes only a clean-install run exposes:
- **Import-time reads of installer-created artifacts.** Files that `setup.sh`/`model_setup.py` generate are not in the repo, so a module that reads them at import crashes on a fresh clone — the skill does not degrade, it breaks entirely, and the crash surfaces through the test suite (`FileNotFoundError` at import, then `TypeError` in tests that assumed a real catalog). Tests of logic must not depend on an installed model: fake the catalog and bypass the availability check; the module must answer an honest `unavailable` instead of crashing.
- **Process-global pollution.** A plugin that inserts its skill path into `sys.path` at import and never removes it makes a later import in the same process resolve to the author's home copy, not the clean clone. The test passed alone and failed in the full run — order-dependent failures are evidence of shared mutable state, not of a broken test.

## Pitfalls

- A compile pass is not verification: run one read-only live check after editing scripts.
- Never quote remembered numbers (test counts, eval metrics, memory figures) — re-measure and write the factual number.
- If a fix renames something consumers read (a response key, an env var), check the consumers before renaming; renaming a key a test asserts is a behavior change, not a cosmetic fix.
- Documentation drift is a real item: CHANGELOG/README promising behavior the code does not implement counts as a minor bug and must be fixed in the same round or listed as open.
- After editing a module, re-run the probe only after `importlib.reload` — the session kernel keeps the old import.
