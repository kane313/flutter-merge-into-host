# flutter-merge-into-host

A Claude Code skill that merges another standalone Flutter project's UI code into the current host Flutter project as a self-contained feature subtree.

## What it does

Given a path to another Flutter project (with its own `lib/`, `assets/`, `pubspec.yaml`) and the current host Flutter project as cwd, this skill:

1. **Discovers** source structure, dependencies, naming collisions, host conventions
2. **Decides** placement, adaptation depth, dep strategy
3. **Grafts** code into `lib/features/<src-name>/<sub>/{pages,widgets,dialogs}/`, namespaces assets, prefixes routes, aliases colliding class names, integrates routes into host `main.dart`
4. **Adapts** to host CLAUDE.md rules: `flutter_screenutil` suffixes, `DebounceUtil` on base buttons, `ToastUtil` replacement, host theme integration, go_router API match
5. **Verifies** via `flutter analyze` 0 error + asset disk-existence grep + route registration check + structural conflict warnings

## When to use

- User provides path to another Flutter project and wants its UI merged into current cwd
- Trigger phrases (Chinese): `复制`, `迁移`, `合并`, `接入`, `嫁接`
- Trigger phrases (English): `merge`, `migrate`, `graft`

## When NOT to use

- Single-page Figma reproduction → use [`flutter-figma-reproduce`](https://github.com/kane313/flutter-figma-skills) instead
- Building reusable packages → manually extract to `packages/` after this skill produces in-tree feature code
- The source is itself a library/package (no `lib/main.dart`, no `MaterialApp`) → simpler `cp` will do

## Modes

| Mode | Behavior |
|---|---|
| `safe` (default) | Checkpoint pause after Phase 0, Phase 1 (3 AskUserQuestion), Phase 4 |
| `auto` | End-to-end with sensible defaults; mandatory pre-flight checks still run |

Auto trigger phrases: `连续执行`, `自动执行`, `不用问`, `autopilot`, `yolo`, `just do it`.

## Installation

This skill targets Claude Code's skill system. Place under your skills directory:

```
~/.claude/plugins/<plugin-name>/skills/flutter-merge-into-host/
```

Or use this repo as a Claude Code plugin source.

## Layout

```
skills/flutter-merge-into-host/
├── SKILL.md                          # entry point — read first
├── references/
│   ├── 00-discover.md
│   ├── 01-decide.md
│   ├── 02-graft.md
│   ├── 03-adapt.md
│   ├── 04-verify.md
│   ├── failure-modes.md              # known footguns
│   └── fastlane.md                   # safe vs auto mode rules
└── templates/
    ├── discovery.md.tmpl
    ├── plan.md.tmpl
    └── verify-report.md.tmpl
```

## Origin

Distilled from a real-world run merging the `CuteAnimal` project (7k LOC / 31 files / 9 deps / 8 sub-features) into an existing Flutter host project (`flutter_center`, `flutter_screenutil` baseline 402×874, with CLAUDE.md mandating screenutil + DebounceUtil + ToastUtil). The failure-modes doc enumerates the specific gotchas encountered.

## License

Apache 2.0
