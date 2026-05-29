---
name: flutter-merge-into-host
description: "[Flutter projects only — pubspec.yaml with flutter dep present in the host cwd; source path may be given upfront or asked for after trigger.] Merges another standalone Flutter project's UI code (lib/ + assets/ + pubspec deps) into the current host Flutter project as a self-contained feature subtree. Resolves naming collisions, normalizes deps, rewrites relative imports for the new depth, prefixes all GoRoute paths + literal `context.go/push` strings, namespaces assets, and adapts to the host's CLAUDE.md rules (flutter_screenutil, DebounceUtil, ToastUtil, host theme). Triggers on EITHER (a) message contains a path to another Flutter project AND a verb like 'merge / migrate / copy / 复制 / 迁移 / 合并 / graft / 嫁接 / 接入', OR (b) message contains a standalone phrase '迁移' / 'flutter迁移' / 'Flutter 迁移' / 'flutter migration' / 'flutter project merge' / '合并 Flutter 项目' (even without a path — skill will ask user for the source path as its first step). Runs in auto mode by default (no checkpoint pauses, sensible defaults, end-to-end); opt into safe mode (checkpoint after each of 5 phases) with phrases like 'step by step' / '逐步确认' / 'let me review each phase'. Output structure: `lib/features/<src-name>/<sub-feature>/{pages,widgets,dialogs}/...` with `<src-name>` as the new feature namespace."
---

# flutter-merge-into-host

## When to use

- User provides path to another standalone Flutter project (with its own `lib/`, `pubspec.yaml`, `assets/`)
- Wants its UI code merged into the **current host** Flutter project (cwd)

### Trigger conditions (any ONE is enough)

The skill should activate on either pattern below — do not wait for both.

**Pattern A — path + verb in the same message** (most explicit):
- Message references a path with `lib/` + `pubspec.yaml` (relative or absolute)
- AND a verb keyword: `复制` / `迁移` / `合并` / `接入` / `嫁接` / `merge` / `migrate` / `graft`
- Example: `把 ../CuteAnimal-main 复制到本项目` / `merge ../foo into host`

**Pattern B — standalone intent phrase, no path yet** (skill will ask for path):
Any of these standalone phrases trigger the skill even without a path in the same message:
- `迁移` (used alone, in a Flutter context) — e.g. `帮我迁移一下` / `这个项目要迁移`
- `flutter迁移` / `Flutter 迁移` / `flutter 迁移项目` (with optional spacing)
- `flutter migration` / `flutter project migration`
- `合并 flutter 项目` / `merge flutter project` / `graft flutter project`
- `flutter 项目合并` / `把 flutter 项目接入`

When Pattern B fires WITHOUT a clear source path, the skill's first step is an `AskUserQuestion`: "你想迁移哪个 Flutter 项目？请提供项目根目录路径（含 `pubspec.yaml`）。" Do not abort or assume.

**Sanity check before activating**: host cwd must have `pubspec.yaml` with `flutter:` dep. If not, ask the user to `cd` to a Flutter project first; do not start scanning.

**Not for**: page-level Figma reproduction (use `flutter-figma-reproduce` instead), or library publishing (this skill produces in-tree feature code, not a `packages/` module — for that, manual extraction is needed post-merge). If user types `迁移` in a non-Flutter context (e.g. database migration, server migration), the skill should not activate — the host-cwd Flutter check above guards this.

## Inputs

- `source_path` (required) — absolute path to source Flutter project root
- `feature_name` (optional, default: source pubspec `name`) — top-level feature folder name under `lib/features/<feature_name>/`; auto-derived as snake_case from source project name if omitted
- `mode` (optional, default: `auto`) — `auto` for end-to-end no-pause (the default), `safe` for checkpoint-per-phase

## Mode default & safe-mode trigger detection

**Default is `auto`.** Unless the user opts into safe mode (see below), run all 5 phases end-to-end with sensible defaults and no checkpoint pauses. Announce at the start: "Running in auto mode (default) — end-to-end, no checkpoint pauses. Say 'step by step' / '逐步确认' anytime to switch to safe mode."

**Downgrade to `safe` mode** (checkpoint per phase) if the user's invoking prompt contains ANY of:

**Chinese**: `逐步` / `一步步` / `逐步确认` / `每步确认` / `分步` / `让我确认` / `让我看看每一步` / `安全模式` / `safe 模式`
**English**: `step by step` / `let me review each` / `safe mode` / `checkpoint each` / `pause at each` / `confirm each step` / `one step at a time`

When safe mode is selected, announce: "Safe mode triggered by '<phrase>' — will pause for checkpoint after Phase 0, Phase 1, and Phase 4."

Mid-run, the user can also force safe at any time with `等等` / `wait` / `暂停` / `让我看看` / `slow down` (see `references/fastlane.md` § "Mid-run mode change").

## The 5-phase pipeline (overview)

| Phase | Name | Mandatory checkpoint | Safe | Auto |
|---|---|---|---|---|
| 0 | Discover | yes | wait | log + advance |
| 1 | Decide | yes (3 AskUserQuestion items) | wait | use defaults |
| 2 | Graft | no | advance | advance |
| 3 | Adapt | no | advance | advance |
| 4 | Verify | yes | wait for accept | report + advance |

Detailed phase orchestration: read `references/0N-<phase>.md` (one file per phase) when entering that phase. DO NOT read all phases upfront.

## Checkpoint mechanism

**Auto mode** (default): every "wait for user" instruction in `references/0N-*.md` is replaced by:

1. Write phase output files normally
2. Log a one-line `[AUTO] Phase N done: <key decisions taken>` to chat (replaces full CHECKPOINT block)
3. Apply Phase 1 defaults below
4. Advance immediately

After full run, emit ONE final summary listing: phases completed, defaults taken, blockers, files produced. This is informational, not a prompt.

**Safe mode** (only when the user opts in — see § "Mode default & safe-mode trigger detection"):

1. At end of each checkpoint phase, write phase output files to `.flutter-graft/<src-name>/`
2. Emit a `CHECKPOINT Phase N` block listing approval items
3. Wait for reply: `continue` / `ok` / `过` / `next` (approve) OR any other content (treat as modification → adjust and re-emit CHECKPOINT)
4. Only after approval, advance

### Phase 1 auto defaults (no user pause)

- **Placement**: source code lands at `lib/features/<src-name>/<sub-feature>/{pages,widgets,dialogs}/`
- **Adaptation depth**: heavy (full CLAUDE.md compliance: screenutil + DebounceUtil + ToastUtil + go_router version-match)
- **Deps**: add all source deps; adjust versions to satisfy host constraints (e.g. `permission_handler ^11.3.1` → host's `^12.0.1` if host enforces newer)
- **Asset namespace**: `assets/images/<src-name>/<sub-dir>/` for images, `assets/icons/<src-name>_<sub-dir>/` for icons
- **Route prefix**: `/<src-name>/` for all GoRoute paths AND every `context.go/push` literal

### Auto-mode mandatory pre-flight checks (skipping = silent runtime failure)

Removing user pauses does NOT remove these mechanical verifications. Each MUST execute even in auto mode:

1. **Naming collisions** — `grep "^class HomePage\|^class AppButton\|..." host/lib/ vs source/lib/`. If hit, use `import as <alias>` not silent rename. (See `references/02-graft.md` § "Naming collisions".)
2. **Hardcoded route literals** — `grep -rE "context\\.(go|push)\\s*\\(\\s*'/" source/lib/` and prefix every match with `/<src-name>`. Missing one causes runtime 404 only when that branch is exercised.
3. **Asset path namespace rewrite** — after asset folder move, `grep -rhoE "'assets/[^']+'" lib/features/<src-name>/ | tr -d \"'\" | while read p; do [ -f \"$p\" ] || echo MISSING: \"$p\"; done` must output empty. Silent failure: image not found at runtime, no compile error.
4. **pubspec deps version-resolve** — run `flutter pub get` and abort if it fails; emit the upgrade hint inline.
5. **flutter analyze 0 error** — after each codegen pass.

These checks are encoded in `references/04-verify.md`; auto mode runs them all, only the user-facing pause is removed.

## Session-scoped state

All intermediate artifacts live under `.flutter-graft/<src-name>/` in the **host** project:

- `discovery.md` — Phase 0 scan output
- `plan.md` — Phase 1 decisions
- `graft-log.md` — Phase 2 actions (files moved, imports rewritten, routes prefixed)
- `verify-report.md` — Phase 4 final scorecard

Templates for skill-owned outputs: `templates/discovery.md.tmpl`, `templates/plan.md.tmpl`, `templates/verify-report.md.tmpl`.

## Cross-cutting references

- `references/failure-modes.md` — known footguns from real-world runs; consult at every phase start
- `references/fastlane.md` — exact behavior differences between safe and auto modes

## Constraints

- **Never skip Phase 0 checkpoint in safe mode** — the discovery file is the contract; downstream phases require it. (In auto mode, the default, there is no Phase 0 checkpoint pause — but the `discovery.md` file is STILL written; auto skips the user pause, never the artifact.)
- **Never modify host files outside `lib/main.dart` (route integration), `pubspec.yaml` (deps + assets), and the `lib/features/<src-name>/` + `assets/.../<src-name>/...` namespace** without explicit user acknowledgement
- **Never rename source classes/widgets to avoid collisions** — always `import as <alias>` instead; renaming makes the source diverge from upstream and breaks future re-syncs
- **Never blindly downgrade host deps to match source** — if source uses `permission_handler ^11` and host requires `^12`, source code must adapt (or the merge aborts), not host
- **Auto mode is not a "quality mode"** — every pre-flight check above remains mandatory; only user-facing pauses are removed
