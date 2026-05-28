# Phase 2: Graft

## Goal
Execute the plan: mass-copy files into target layout, merge pubspec deps, namespace assets, rewrite imports + asset paths + route literals, integrate router into host `main.dart`. Output: working code at 0 compile error (no host-rule adaptation yet — that's Phase 3).

## Orchestration

1. **Read** `.flutter-graft/<src-name>/plan.md`
2. **File copy** — execute the file mapping table from plan.md. Use `cp` + `mkdir -p`, not `mv`, so source repo stays intact. Drop files marked "drop" (typically `main.dart`, deleted barrel exports).

   ```bash
   SRC=<source_path>; DST=<host>/lib/features/<src-name>
   mkdir -p $DST/{theme,widgets,router}
   mkdir -p $DST/<each-sub-feature>/{pages,widgets,dialogs}
   # for each row in plan.md File Mapping: cp $SRC/<src> $DST/<dst>
   ```

3. **Pubspec deps merge** — for each row in plan.md Deps:
   - `add`: append to host pubspec dependencies block
   - `version-bump`: replace source's version constraint with host's
   - `drop`: skip
   - After all rows: `flutter pub get`; abort if it fails and emit upgrade hint

4. **Pubspec assets register** — append to host pubspec `flutter.assets:`:
   ```yaml
   - assets/images/<src-name>/
   - assets/images/<src-name>/<each-subdir>/   # MUST list each subdir explicitly; Flutter does not recurse
   - assets/icons/<src-name>_<each-iconset>/
   ```

5. **Asset file copy** — `cp -r` source's `assets/images/<subdir>/` to `host/assets/images/<src-name>/<subdir>/`, and `cp -r assets/icons/<set>/` to `host/assets/icons/<src-name>_<set>/`. Top-level loose files: copy each to `host/assets/images/<src-name>/`.

6. **Import path rewrites** (the trickiest step — see § "Import depth algorithm" below)

7. **Route literal rewrites** — `grep` and prefix every `context.go/push/pushReplacement(\s*'/` in source code (now at target paths). One `sed` per literal:

   ```bash
   sed -i '' "s|context.go('/');|context.go('/<src-name>/home');|g" lib/features/<src-name>/<file>
   sed -i '' "s|'/category|'/<src-name>/category|g" lib/features/<src-name>/<file>
   # ... one per unique literal in plan.md route rewrite table
   ```

8. **Asset path rewrites in code** — `grep` `'assets/...'` literals in `lib/features/<src-name>/` and rewrite:

   ```bash
   find lib/features/<src-name> -name "*.dart" | while read f; do
     sed -i '' \
       -e 's|assets/icons/<oldset>/|assets/icons/<src-name>_<oldset>/|g' \
       -e 's|assets/images/<subdir>/|assets/images/<src-name>/<subdir>/|g' \
       -e 's|assets/images/<toplevelfile>|assets/images/<src-name>/<toplevelfile>|g' \
       "$f"
   done
   ```

   **Verify**: after rewrite, every asset reference resolves to a real file:
   ```bash
   grep -rhoE "'assets/[^']+'" lib/features/<src-name>/ --include="*.dart" \
     | tr -d "'" | sort -u | while read p; do [ -f "$p" ] || echo "MISSING: $p"; done
   ```
   Empty output required to advance.

9. **Naming collision aliases** — for each row in plan.md alias plan, edit host `main.dart` (or wherever the collision happens):
   ```dart
   import 'features/<src-name>/<sub>/pages/<collision_class>.dart' as <src_name>_<sub>;
   // usage site:
   <src_name>_<sub>.HomePage()
   ```

10. **Router file** — create `lib/features/<src-name>/router/<src_name>_routes.dart` with all GoRoute entries (paths already prefixed) exporting `List<RouteBase> <srcName>Routes`. Cross-feature `extra` typed args (e.g. `DetailFeedArgs`) must be re-imported.

11. **Host main.dart integration** — add to `_buildRoutes()`:
    ```dart
    ...<srcName>Routes,
    ```
    plus the import line.

12. **First compile pass** — `flutter analyze lib/features/<src-name>/ lib/main.dart`. Iterate fixes until **0 error**. Common patterns:
    - Wrong relative depth → adjust (§ Import depth algorithm)
    - Dropped `widgets.dart` export → re-add or remove consumer
    - Pkg version drift → typed APIs may have changed; fix call sites

13. **Write `.flutter-graft/<src-name>/graft-log.md`** — list of (a) files copied (b) imports rewritten (c) routes prefixed (d) assets moved + verified. No user checkpoint in safe mode for Phase 2 (advance directly to Phase 3); auto mode same.

## Import depth algorithm

Source's relative imports break when target directory depth changes. Formula:

- For each file at source path `P_src`, target path `P_dst`:
- For each import in that file `'./../../X/Y.dart'`:
  - Resolve absolute path the import points to in source: `abs_src = realpath(P_src/..imports..)`
  - Map `abs_src` through the file mapping table to find target absolute path: `abs_dst`
  - Compute new relative path: `relpath(abs_dst, P_dst)`
  - Replace the import string

**In practice (this skill's batch approach)**: identify the 3-5 unique source path patterns (e.g. all source `features/X/Y_page.dart` import `'../../widgets/widgets.dart'`) and run one `sed` per pattern. Example seen in real run:

| Source pattern | Source's relative path | Target pattern | Target's relative path |
|---|---|---|---|
| `features/X/Y_page.dart` → widgets/widgets.dart | `../../widgets/widgets.dart` | `features/<src>/X/pages/Y_page.dart` → widgets/widgets.dart | `../../widgets/widgets.dart` (same — coincidence: +1 nesting balanced by introducing `pages/`) |
| `features/X/widgets/Y.dart` → theme/X.dart | `../../../theme/X.dart` | `features/<src>/X/widgets/Y.dart` → theme/X.dart | `../../theme/X.dart` (less deep — was 3 up to source's `lib/`, now 2 up to `features/<src>/`) |
| `features/X/Y_page.dart` → features/Z/Z_page.dart | `../Z/Z_page.dart` | `features/<src>/X/pages/Y_page.dart` → features/<src>/Z/pages/Z_page.dart | `../../Z/pages/Z_page.dart` (different: +1 from `pages/` AND +1 from sibling-feature jump) |

**Always verify by `flutter analyze` after each batch of sed**. The errors are clear (`Target of URI doesn't exist`); use them to drive corrections.

## Cross-feature imports

Sub-features often reference each other (e.g. `home_card_stack.dart` imports `content/content_feed.dart`). Because Phase 2 introduces `pages/` subdir, these get one extra hop:

- Source: `import '../../content/content_feed.dart';`
- Target: `import '../../content/pages/content_feed.dart';`

Sed per cross-ref pattern from plan.md.

## widgets.dart barrel handling

If source has `lib/widgets/widgets.dart` re-exporting everything, and any of the re-exported files is being dropped (e.g. host has `ToastUtil` so source's `AppToast` is dropped), remove the corresponding `export 'app_toast.dart';` line.

## Stopping conditions

- After 3 iterations of analyze-fix-analyze, still > 5 errors → abort to user; the source may have hidden dynamic-dep patterns not captured by static analysis
- `flutter pub get` fails on a dep version with no resolution path → ask user to manually resolve; do NOT auto-downgrade host pubspec
