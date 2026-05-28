# Fastlane (Safe vs Auto mode)

Quick reference for the exact behavior differences. Detailed phase steps are in `0N-*.md`; this doc only captures what changes with `mode=auto`.

## Mode summary

| Mode | Phase 0 | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|---|---|---|---|---|---|
| `safe` (default) | CHECKPOINT | CHECKPOINT (3 AskUserQuestion) | advance | advance | CHECKPOINT |
| `auto` | log + advance | log + apply defaults | advance | advance | log + advance |

In `auto` mode, every step in `0N-*.md` that says "wait for user reply / CHECKPOINT pause / approval" is replaced by:

1. Write phase output files normally
2. Emit a one-line `[AUTO] Phase N done: <key decision/fact>` to the chat
3. For Phase 1 specifically: apply defaults (see below)
4. Advance immediately

After Phase 4, emit ONE final summary block. Informational — no wait.

## Auto-mode Phase 1 defaults

| Decision | Default |
|---|---|
| Placement | Single root: `lib/features/<src-name>/<sub>/{pages,widgets,dialogs}/` |
| Adaptation depth | `heavy` if host `CLAUDE.md` exists and mentions `flutter_screenutil` or `DebounceUtil` or `ToastUtil`; else `light` |
| Deps | Add all source deps; bump versions to host's range when host is more restrictive |
| Asset namespace | `assets/images/<src-name>/<subdir>/` ; `assets/icons/<src-name>_<set>/` |
| Route prefix | `/<src-name>/` ; source `/` route → `/<src-name>/home` |
| Naming collision | `import as <src-name>_<sub>` at consumer site |

## Mandatory checks that auto mode does NOT skip

(Repeat from SKILL.md for emphasis — silent failure modes that compile-time checks miss.)

1. **Naming collision grep** (Phase 0/2)
2. **Hardcoded route literal grep + prefix** (Phase 2)
3. **Asset disk-existence grep** (Phase 2/4)
4. **Pubspec asset subdir explicit registration** (Phase 2)
5. **flutter pub get success** (Phase 2)
6. **flutter analyze 0 error** (Phase 2/3/4)

Auto mode runs all six. Skipping any produces silent-at-compile-time, broken-at-runtime defects. The user pause is removed; the mechanical work is not.

## Mid-run mode change

If the user interrupts an auto-mode run with phrases suggesting they want to review (`等等` / `wait` / `pause` / `让我看看` / `暂停` / `slow down`), downgrade to `safe` mode for remaining phases. Emit the next CHECKPOINT as if it had been safe all along (include all the state context the user would have seen at that phase boundary).

## Auto-mode trigger phrases

Refer to SKILL.md § "Auto-mode trigger detection". Phrases include:
- Chinese: `不再询问` / `不用问` / `连续执行` / `自动执行` / `一口气` / `跑完为止` / `直接执行`
- English: `continue without asking` / `no checkpoint` / `autopilot` / `auto mode` / `yolo` / `just do it`

Substring match, case-insensitive. Any match → auto mode for the whole run.

## What auto mode is NOT

- ❌ A "quality" mode. Scorecard thresholds in Phase 4 are identical to safe.
- ❌ A "shortcut" mode. Pre-flight checks remain mandatory.
- ❌ A "non-destructive" mode. Same file writes happen as in safe.

The ONLY difference is the absence of user-facing pause. Use it when you're confident the source-host pairing is straightforward (small source, no exotic deps, host with clear CLAUDE.md). For unknown pairings, default to safe so you can inspect Phase 0/1 outputs before Phase 2 starts moving files.
