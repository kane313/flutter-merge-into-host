# Phase 4: Verify

## Goal
Run mechanical checks that catch the silent failures (asset 404, route 404 at runtime) which compile-time `flutter analyze` misses. Write a scorecard. Prompt user to hot-restart and exercise the grafted routes.

## Orchestration

1. **flutter analyze recheck** — full host:
   ```bash
   flutter analyze lib/ 2>&1 | grep -cE "^\s*error\s|^\s*warning\s"
   ```
   Must equal 0.

2. **Asset existence verification** — grep + disk check:
   ```bash
   grep -rhoE "'assets/[^']+'" lib/features/<src-name>/ --include="*.dart" \
     | tr -d "'" | sort -u | while read p; do
       [ -f "$p" ] || echo "MISSING: $p"
     done
   ```
   Must output empty. Any `MISSING:` line means `Image.asset` will fail at runtime with `Unable to load asset` — silent at compile time.

3. **Asset pubspec registration verification** — for each unique parent directory under `assets/images/<src-name>/` and `assets/icons/<src-name>_*/`:
   ```bash
   find assets/images/<src-name> assets/icons/<src-name>_* -mindepth 1 -type d 2>/dev/null \
     | while read d; do
       grep -q "    - $d/" pubspec.yaml || echo "UNREGISTERED: $d/"
     done
   ```
   Must output empty. Flutter does not recurse into subdirs — each must be listed explicitly under `flutter.assets:`.

4. **Route literal verification** — confirm no source route literal slipped past Phase 2:
   ```bash
   grep -rnE "context\.(go|push|pushReplacement)\s*\(\s*'/" lib/features/<src-name>/ \
     --include="*.dart" | grep -v "/<src-name>/"
   ```
   Must output empty. Any line shown means a route call still points to a path that doesn't exist after prefix.

5. **GoRoute registration verification** — every path used in `context.go/push` literals must have a matching `GoRoute(path:)` in the grafted router file:
   ```bash
   # Extract called paths
   grep -rhoE "context\.(go|push)\s*\(\s*'/<src-name>/[^']*'" lib/features/<src-name>/ \
     | sed -E "s/.*'(\/<src-name>\/[^?']+).*/\1/" | sort -u > /tmp/called

   # Extract registered paths from routes file
   grep -E "path:\s*'/<src-name>/" lib/features/<src-name>/router/*.dart \
     | sed -E "s/.*path:\s*'([^']+)'.*/\1/" | sort -u > /tmp/registered

   comm -23 /tmp/called /tmp/registered  # called but not registered → broken
   ```

6. **Naming collision residual check** — ensure no host file accidentally uses the unaliased source class:
   ```bash
   for cls in $(cat .flutter-graft/<src-name>/plan.md | sed -nE 's/.*<collision> ([A-Z][A-Za-z]+) .*/\1/p'); do
     # Find references to bare class name outside the src-name namespace
     grep -rnE "\b$cls\b" lib/ --include="*.dart" | grep -v "lib/features/<src-name>/" | head -3
   done
   ```
   Each hit should be inside an `as <alias>` reference or a comment. Otherwise the collision wasn't aliased properly.

7. **Run-time conflict warnings** — emit a `[WARN]` block in the verify report for any structural conflicts that compile fine but visually break:
   - Source `HomePage` contains `Scaffold + BottomNavigationBar`; host wraps it in another `Scaffold + BottomNavigationBar` → two stacked nav bars
   - Source `<X>Page` contains `extendBodyBehindAppBar: true` but host wraps it without a transparent AppBar → status bar covered
   - Source declares its own font in pubspec.yaml that host doesn't have → fallback rendering

   These are best-effort detected by grepping the source pages for `Scaffold` + `bottomNavigationBar:` combos; the report flags them and recommends the user manually decide whether to strip the source's BottomNav.

8. **Write `.flutter-graft/<src-name>/verify-report.md`** using `templates/verify-report.md.tmpl`. Sections:
   - Compile status (error / warning count)
   - Asset count + missing list (should be empty)
   - Asset pubspec registration check
   - Route called-vs-registered diff
   - Naming collision residuals
   - Structural conflict warnings
   - **Manual smoke-test checklist** — list every grafted route path with a checkbox; user runs the app and tickes each as they verify

9. **Emit CHECKPOINT (safe mode)**:
   ```
   CHECKPOINT Phase 4 — Verify done.
   Compile: 0 error.
   Assets: <N> referenced, <0> missing.
   Routes: <K> registered, <0> orphaned calls.
   Collisions: <M> resolved via alias.
   Warnings: <P> structural conflicts (see report).

   Hot-restart the app (capital R, NOT hot reload — asset manifest reloads on restart only).
   Reply `accept` to close, or describe issues for follow-up.
   ```

   Auto mode: emit the same scorecard as a `[REPORT]` block, no wait.

## Stopping conditions

- Any of checks 2/3/4/5/6 emits non-empty output → DO NOT report success; fix the issue first
- Structural conflict count > 3 → recommend a Stage 2 follow-up task (manual UI integration); do not block but call out clearly

## Runtime overflow detection (manual smoke step)

Static analysis cannot detect `RenderFlex overflowed by N pixels` — it's a layout-time exception emitted only when the offending widget paints. The verify report MUST include a manual smoke-test instruction calling this out explicitly, otherwise users mark "0 error, merge complete" and ship with visible yellow/black overflow stripes.

### Smoke command

```bash
# After Phase 3 finishes
flutter run -d <device>            # device list: flutter devices
# Or for hot-test on simulator:
flutter run -d "iPhone 15"
```

Once running, navigate to each grafted route (see "Manual smoke-test checklist" below). Watch the `flutter run` console — any overflow surfaces as:

```
══╡ EXCEPTION CAUGHT BY RENDERING LIBRARY ╞════════════════════════════════════
The following assertion was thrown during layout:
A RenderFlex overflowed by N pixels on the (top|bottom|left|right).

The relevant error-causing widget was:
  (Row|Column) ... :file:///.../<file>:<line>:<col>
The overflowing RenderFlex has an orientation of Axis.(horizontal|vertical).
```

### Common root causes (mapped to Phase 3 rules)

| Overflow symptom | Likely root cause | Fix |
|---|---|---|
| Row "overflowed by 5-15 px on right" inside narrow flex button (`AppButtonRow` secondary) | Base button's label Row has no FittedBox guard; cross-font drift | Rule 9 (apply FittedBox.scaleDown to base button label) |
| Column "overflowed by <1 px on bottom" inside fixed-height SizedBox in grid item | screenutil multi-axis (.r / .h / .sp) drift + font strut on baseline non-matching devices | Use `MainAxisSize.min` on inner Column + remove fixed height SizedBox; let parent Row align by max child height |
| Row "overflowed by 50+ px" in card with long title | Text not wrapped; no `Flexible` / `Expanded` / `overflow: ellipsis` | Wrap Text in `Flexible/Expanded` with `overflow: TextOverflow.ellipsis, maxLines: 1` |
| "BoxConstraints forces an infinite width/height" | unbounded ListView/Column inside another unbounded ListView/Column | Wrap with `Expanded` if inside Row/Column, or use `shrinkWrap: true` (only if list count is small) |

### Per-route smoke checklist for verify report

The report's manual checklist enumerates every grafted route with a checkbox. User checks each off as they verify "no overflow stripe + content visible". If any route reports overflow, the offending file:line is captured in the next-action list and Phase 3 re-runs only that file's rule(s).

## Hot-restart gotcha (always include in verify report)

Flutter's asset manifest is re-read only on **hot restart** (capital `R` in `flutter run`), not hot reload (lowercase `r`). After Phase 2/3, the user MUST hot-restart for new asset paths to resolve. Verify report's manual checklist starts with: "1. Press R (capital) in the terminal running flutter — not r."
