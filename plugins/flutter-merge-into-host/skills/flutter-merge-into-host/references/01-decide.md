# Phase 1: Decide

## Goal
Convert Phase 0 inventory into concrete plan: placement, adaptation depth, deps strategy. Three decisions, then write `plan.md`.

## Orchestration

1. **Read** `.flutter-graft/<src-name>/discovery.md` (from Phase 0)
2. **Safe mode**: surface the 3 decisions via `AskUserQuestion` (one tool call, 3 questions to avoid round-trip). **Auto mode**: skip the questions, use defaults below; emit `[AUTO] Phase 1 done: defaults applied.`

## The 3 decisions

### Decision 1: Placement

| Option | Effect | When |
|---|---|---|
| **Single root `lib/features/<src-name>/<sub>/{pages,widgets,dialogs}/`** ✓ default | All source code under one feature namespace. Clean boundary. | Most cases; recommended. |
| Each sub-feature as top-level `lib/features/<sub>/` | Source's home/settings/etc each become top-level features. Mixes with host's own. | Only if source is intended to fully replace host UI. |
| `packages/lib_<src-name>/` | Becomes a path-dependency package. Highest cost (Pubspec, DI, route exports). | Only if downstream multi-app reuse is planned. |

### Decision 2: Adaptation depth

| Option | Effect | When |
|---|---|---|
| **Heavy: full host CLAUDE.md compliance** ✓ default | Stage 2 transforms: screenutil all dims; DebounceUtil on buttons; ToastUtil replace; go_router API match; theme tokens host-style. | When host has a CLAUDE.md with strict rules. |
| Light: only fix breaks | Only resolve compile errors and dep conflicts. Keep source's own theme, screenutil-free, original toast. | When source is short-lived prototype or host doesn't enforce rules. |
| Copy-as-is, optimize later | Phase 2 only does file move + import fix + 0 error. Stage 2 deferred to manual follow-up tasks. | Quick smoke-merge; not recommended for shipping. |

### Decision 3: Deps strategy

| Option | Effect | When |
|---|---|---|
| **Add all source deps; adjust versions to host** ✓ default | Each source dep is appended to host pubspec; version conflicts resolved by bumping source's request to host's range. | Most cases. |
| Add only used deps | For each source dep, `grep` source code to check usage; drop unused deps from host pubspec. | When source's pubspec is bloated and you want lean host pubspec. Cost: extra scan. |
| User decides per dep | Skill asks for each conflict. | Only when there are 3+ conflicts and they're load-bearing (e.g. firebase auth differs by major version). |

## AskUserQuestion call (safe mode)

Emit a single `AskUserQuestion` with 3 questions matching the tables above. Each question has 3 options with the recommended one marked. Capture answers; write to `plan.md`. If user picks a non-default for Decision 2 (light/as-is), set a flag for Phase 3 to skip the heavy transforms.

## Auto-mode defaults

- Decision 1: `single root`
- Decision 2: `heavy` IF host has `CLAUDE.md` with explicit `flutter_screenutil` / `DebounceUtil` / `ToastUtil` mention; else `light`
- Decision 3: `add all + adjust versions to host`

## Sub-decisions encoded in plan.md (no user pause)

### Naming collisions

For each class collision (from Phase 0): record `<src-class-name>` → `import 'features/<src-name>/<sub>/<src-class>.dart' as <src-name>_<sub>;`. Use lowercase snake_case alias. Plan.md records the alias-import lines that Phase 2 will inject.

### Route prefix

All source GoRoute paths AND all hardcoded `context.go/push` literals get `/<src-name>/` prefix. Record the rewrite plan (source path → target path):

| Source | Target |
|---|---|
| `/splash` | `/<src-name>/splash` |
| `/` (root) | `/<src-name>/home` (avoid host's `/` if any) |
| `/<X>` | `/<src-name>/<X>` |

### Asset namespace

All `assets/images/<subdir>/...` → `assets/images/<src-name>/<subdir>/...`. Top-level files (e.g. `assets/images/logo.png`) → `assets/images/<src-name>/logo.png`. Icons: `assets/icons/<subdir>/` → `assets/icons/<src-name>_<subdir>/` (icon dirs are usually flat-named like `lucide/`, not nestable inside a project namespace).

### Sub-feature → host directory layout

From Phase 0 source structure, derive the explicit file mapping. Default rule:

| Source path | Host target |
|---|---|
| `lib/main.dart` | drop (host has own) |
| `lib/router.dart` | rewrite as `lib/features/<src-name>/router/<src-name>_routes.dart` exporting `List<RouteBase> <srcName>Routes` |
| `lib/theme/X.dart` | `lib/features/<src-name>/theme/X.dart` |
| `lib/widgets/X.dart` | `lib/features/<src-name>/widgets/X.dart` |
| `lib/widgets/widgets.dart` (barrel) | move; strip any export of widgets being dropped (e.g. AppToast if host has ToastUtil) |
| `lib/features/<sub>/<sub>_page.dart` | `lib/features/<src-name>/<sub>/pages/<sub>_page.dart` (introduce `pages/` subdir per host convention) |
| `lib/features/<sub>/widgets/X.dart` | `lib/features/<src-name>/<sub>/widgets/X.dart` |
| `lib/features/<sub>/<sub>_popup.dart` (dialog-ish) | `lib/features/<src-name>/<sub>/dialogs/<sub>_popup.dart` |

Record every mapping in `plan.md`'s "File mapping" section so Phase 2 just executes.

## Write `.flutter-graft/<src-name>/plan.md`

Use `templates/plan.md.tmpl`. Sections:
- 3 decisions (option chosen + rationale)
- Naming-collision alias plan
- Route prefix rewrite table
- Asset namespace rewrite table
- File mapping (source → target)
- Deps merge plan (with version actions)
- Host adapter rules to apply in Phase 3 (one row per CLAUDE.md rule)

## Emit CHECKPOINT (safe mode)

```
CHECKPOINT Phase 1 — Decide done.
Placement: <Decision 1 choice>
Adaptation: <Decision 2 choice>
Deps: <Decision 3 choice>
Plan written to `.flutter-graft/<src-name>/plan.md`.
Reply `continue` to start Phase 2 (Graft).
```

Auto mode: `[AUTO] Phase 1 done: placement=single-root, adaptation=heavy|light, deps=add-all.`

## Stopping conditions

- User picks "User decides per dep" but the next round-trip yields ambiguous answers — fall back to `add all + adjust to host`
- Collision count > 20 AND user picks "light" adaptation — warn that compile errors are likely; offer to fall back to "heavy"
