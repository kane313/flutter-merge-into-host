# Phase 0: Discover

## Goal
Build a structured inventory of the source Flutter project + a conflict matrix against the host. Output a discovery doc the user (or auto mode) will use to decide placement, adaptation depth, deps strategy in Phase 1.

## Orchestration

1. **Preflight checks**
   - Verify `source_path/pubspec.yaml` exists and contains `flutter:` dep — else abort with "source is not a Flutter project"
   - Verify host cwd has `pubspec.yaml` with `flutter:` dep
   - Verify `source_path != cwd` (refuse to merge a project into itself)

2. **Derive `<src-name>`**
   - Default = source pubspec `name` field (snake_case)
   - If contains underscores or uppercase, normalize: `cute_animal` from `name: cute_animal`; `cuteAnimal` → `cute_animal`
   - If collides with an existing host folder under `lib/features/<X>/` or `packages/<X>/`, suffix with `_v2` and warn

3. **Scan source `lib/` structure** (the 7 must-collect items)

   Run a single Bash pass and accumulate. Items required for discovery.md:

   ```bash
   # a. File count + LOC
   find SOURCE/lib -name "*.dart" | wc -l
   find SOURCE/lib -name "*.dart" -exec wc -l {} + | tail -1

   # b. Top-level lib/ layout — what feature folders, what shared dirs
   find SOURCE/lib -maxdepth 2 -type d | sort

   # c. Each feature: pages vs widgets vs no-substructure
   find SOURCE/lib/features -maxdepth 3 -type d 2>/dev/null

   # d. pubspec deps
   awk '/^dependencies:/,/^dev_dependencies:/' SOURCE/pubspec.yaml | grep -E "^  [a-z]"

   # e. Asset subdirectories
   find SOURCE/assets -maxdepth 2 -type d

   # f. main.dart + router.dart structure
   ls SOURCE/lib/main.dart SOURCE/lib/router.dart 2>/dev/null
   grep -E "GoRoute\(|path:" SOURCE/lib/router.dart 2>/dev/null

   # g. Hardcoded route literals (need /-prefix rewrite later)
   grep -rnE "context\.(go|push|pushReplacement)\s*\(\s*'/" SOURCE/lib --include="*.dart"

   # h. Self-rolled width-ratio scaling (must be removed in Phase 3 — collides with host's screenutil)
   grep -rnE "_kDesignWidth|kDesignWidth|DesignWidth" SOURCE/lib --include="*.dart"
   grep -rnE "scaleX\s*=|scaleY\s*=" SOURCE/lib --include="*.dart"
   grep -rnE "\* scaleX|\* scaleY" SOURCE/lib --include="*.dart"
   grep -rnE "MediaQuery\.(sizeOf|of\(context\)\.size).+/\s*\d+" SOURCE/lib --include="*.dart"

   # i. Category-list pattern detection (Phase 3 may upgrade to PageView linkage)
   # Find pages that have a tab/chip row + a single content list, but NOT yet PageView/TabBarView.
   # Signal A: page contains SwitchTab / TabBar / chip row widget
   grep -rlE "SwitchTab|TabBar\(|SliverPersistentHeader|filterIndex|tabIndex|_filterChip|FilterChip" \
     SOURCE/lib/features --include="*.dart" 2>/dev/null > /tmp/has-tabs
   # Signal B: page contains a single SliverList/SliverGrid/SliverMasonryGrid/GridView/ListView
   grep -rlE "SliverGrid|SliverList|SliverMasonryGrid|GridView\.|ListView\." \
     SOURCE/lib/features --include="*.dart" 2>/dev/null > /tmp/has-list
   # Signal C: page already has PageView / TabBarView (no upgrade needed)
   grep -rlE "PageView|TabBarView" SOURCE/lib/features --include="*.dart" 2>/dev/null > /tmp/has-pageview
   # Candidate = (has-tabs ∩ has-list) − has-pageview
   comm -12 <(sort /tmp/has-tabs) <(sort /tmp/has-list) | comm -23 - <(sort /tmp/has-pageview)
   ```

4. **Collision detection against host** (5 categories)

   ```bash
   # i. Top-level class names that may shadow host's
   grep -rhE "^class [A-Z]" SOURCE/lib --include="*.dart" | awk '{print $2}' | sort -u > /tmp/src-classes
   grep -rhE "^class [A-Z]" HOST/lib --include="*.dart" | awk '{print $2}' | sort -u > /tmp/host-classes
   comm -12 /tmp/src-classes /tmp/host-classes  # intersection → collisions

   # ii. Route path collisions (GoRoute paths)
   # Compare source `path: '/X'` with host `path: '/X'`

   # iii. Asset folder collisions (banner/, settings/, etc.)
   ls SOURCE/assets/images/ HOST/assets/images/ 2>/dev/null

   # iv. Dep version conflicts
   for dep in $(awk '/^dependencies:/,/^dev_dep/' SOURCE/pubspec.yaml | grep -E "^  [a-z]" | awk '{print $1}' | tr -d :); do
     hostver=$(grep "^  $dep:" HOST/pubspec.yaml)
     srcver=$(grep "^  $dep:" SOURCE/pubspec.yaml)
     [ -n "$hostver" ] && [ "$hostver" != "$srcver" ] && echo "CONFLICT: $dep | src=$srcver | host=$hostver"
   done

   # v. Host CLAUDE.md rules to enforce (read host/CLAUDE.md if exists)
   ```

5. **Detect host conventions**
   - `lib/features/<X>/pages/` vs flat `lib/features/<X>/X_page.dart`?
   - `lib/shared/widgets/` exists?
   - `lib/theme/` exists, or per-feature inline?
   - `flutter_screenutil` in host pubspec? what `designSize`?
   - `ToastUtil` / `DebounceUtil` re-exported from a known package?
   - Read host `CLAUDE.md` if present — record mandatory rules

6. **Write `.flutter-graft/<src-name>/discovery.md`** using `templates/discovery.md.tmpl`. Sections:
   - Run config (paths, sizes, file count)
   - Source structure (feature → file mapping)
   - Source deps + version conflicts
   - Source assets (subdir list + top-level files)
   - Source routes (path + builder summary)
   - Naming collisions (class, route, asset)
   - Host conventions inventory

7. **Emit CHECKPOINT** (safe mode):
   ```
   CHECKPOINT Phase 0 — Discover done.
   <N> files / <M> LOC; <K> features; <D> deps (<C> conflicts); <R> routes; <P> collisions.
   Review `.flutter-graft/<src-name>/discovery.md`. Reply `continue` to advance.
   ```
   Auto mode: log `[AUTO] Phase 0 done: <X> files / <Y> deps / <Z> collisions detected.`

## Stopping conditions

- Source project has no `lib/` or no Dart files → abort, source unsupported
- Host has no `pubspec.yaml` → abort, wrong cwd
- File count > 200 OR LOC > 30000 → warn user; merging projects this large is rare and likely a sign the source should be a `packages/` package instead. Continue only after explicit confirmation in safe mode.
- Collision matrix > 20 entries → warn; merge is feasible but Phase 2 will require many `import as` aliases.

## Output: discovery.md

See `templates/discovery.md.tmpl`. Key sections the user reviews:

- **File mapping table** — auto-generated `source path → target path` for each `.dart` file
- **Class collision list** — for each colliding class, decision deferred to Phase 2 (always `import as`, never rename source)
- **Route prefix rewrite preview** — show the 5-10 `context.go('/X')` calls that will be prefixed
- **Asset namespace rewrite preview** — show which `assets/images/<dir>/` will move to `assets/images/<src-name>/<dir>/`
- **Deps merge plan** — table: source dep | source version | host version | action (add / version-bump / drop)
- **Self-rolled scaling inventory** — list of sites using `scaleX = size.width / N` patterns; Phase 3 must replace with `.w/.h/.sp/.r`. If `N` matches host's screenutil baseline, the rewrite is mechanical (`X * scaleX` → `X.w`). If `N` differs (e.g. source designed for 375 but host baseline is 402), record the visual-fidelity trade-off.
- **Category-list pattern candidates** — list of files matching item `i` (has tab/chip row + has single content list + NOT already PageView/TabBarView). For each candidate, Phase 3 Rule 6 will upgrade it to PageView-linkage. Include in discovery.md a per-page mini-report:

  ```
  - lib/features/<sub>/<file>.dart
    - tab-row signal: <which grep hit>
    - list signal: <SliverMasonryGrid / GridView / ...>
    - tab rows detected: <1 or 2>  (if 2, Phase 1 must ask which one drives PageView)
    - candidate linkage target: <closest-to-list row's name>
  ```

  Use this report in Phase 1 to surface a 4th `AskUserQuestion` decision ("which tab row drives PageView") when 2 rows are detected; if only 1, decision is mechanical and auto mode applies it.
