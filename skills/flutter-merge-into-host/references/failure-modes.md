# Failure Modes (Flutter Project Merge)

AI scans this at the start of every phase to avoid repeating known mistakes.

## Phase 0/1 (Discover / Decide)

- ✗ Treating the source's `lib/main.dart` as part of the merge. **The source's `main.dart` is always dropped** — host has its own initialization chain (DI, theme, splash, deep-link). Source's `MaterialApp.router(...)` would create a nested MaterialApp inside host's, which Flutter forbids.

- ✗ Letting the user pick "User decides per dep" for dep conflicts without a bail-out. Once chosen, every conflict becomes a round-trip. **Set a 3-conflict ceiling**; beyond that, fall back to "add all + adjust to host" with an inline note. Document the diverging deps in plan.md so user can review post-merge.

- ✗ Treating sub-feature folders as if they all have widgets/. Some sources have `features/X/X_page.dart` flat and others have `features/X/widgets/Y.dart`. Phase 0 must enumerate per-feature structure, not assume uniformity, so Phase 2 file mapping is accurate.

- ✗ Auto-default `feature_name` collides silently with existing host folder. Example: host has `lib/features/home/`, source's pubspec `name: home_app` → derived `home_app` is unique, OK. But source `name: home` → collides. Phase 0 must detect and suffix `_v2` with a warn.

## Phase 2 (Graft) — Import depth

- ✗ Assuming relative imports stay correct because "the depth went up by one for everyone". Actually:
  - `features/X/Y_page.dart` → `features/<src>/X/pages/Y_page.dart` is +2 nestings BUT introduces `pages/` so `../../widgets/widgets.dart` (-2) still points to the right place
  - `features/X/widgets/Y.dart` → `features/<src>/X/widgets/Y.dart` is +1 nesting; original `../../../theme/X.dart` (-3) now points 3 levels up which is `features/` not `lib/` → BROKEN; correct is `../../theme/X.dart` (-2)

  **Always verify with `flutter analyze` after import-rewrite sed; the "URI does not exist" errors drive the corrections.** Do not trust manual depth math alone.

- ✗ Cross-feature imports forgetting the new `pages/` subdir. Source `import '../detail/detail_page.dart'` becomes `'../detail/pages/detail_page.dart'` (+1 hop) AND if the importing file is itself moved into a `pages/` subdir, it's `'../../detail/pages/detail_page.dart'` (+2 hops).

## Phase 2 — Route prefixing

- ✗ Only prefixing the GoRoute `path:` entries and forgetting `context.go/push/pushReplacement` literals scattered in code. Symptoms: app navigates fine until a specific button is tapped → runtime "no route matches '/category'". **Mandatory full grep**:
  ```bash
  grep -rnE "context\.(go|push|pushReplacement)\s*\(\s*'/" lib/features/<src-name>/
  ```
  Every match must include `/<src-name>/`. Phase 4 verification re-runs this grep.

- ✗ Prefixing `context.go('/')` with `/<src-name>/` and getting `/<src-name>/`. The source's `/` root route should map to `/<src-name>/home` (or whatever the source's home was) — not a trailing-slash variant. Always check the rewrite table in plan.md for the `/` → `<sensible>` mapping.

## Phase 2 — Asset namespace

- ✗ Pubspec `flutter.assets:` lists `assets/images/<src-name>/` and assumes subdirs are included. **Flutter does not recurse**. Each subdir needs its own line.

- ✗ Sed rewrite of asset paths in code uses a regex that matches host's own subdirs too. Always scope the find: `find lib/features/<src-name> -name "*.dart"`, NOT `find lib/`. Otherwise host's `assets/images/banner/` (if it exists) gets wrongly prefixed.

- ✗ Forgetting top-level loose files (`assets/images/logo.png`, `assets/images/empty_box.png`) — these don't match the subdir pattern. Add individual sed rules for each loose-file path that appears in source code.

- ✗ Asset rewrite "compiled fine" but image shows blank at runtime. The compile-time check (`flutter analyze`) does not validate asset paths; only runtime does. **Phase 4 must run disk-existence grep** for every asset literal in grafted code.

## Phase 2 — Naming collisions

- ✗ Renaming the source's `HomePage` class to `CuteAnimalHomePage` to avoid host's `HomePage`. Now the source diverges from upstream; future re-syncs require redoing the rename. **Always use `import as <alias>` at the consumer site** (host's `main.dart`), leave source intact.

- ✗ Aliasing too aggressively — every cross-file import inside source gets an alias. Only the consumer site (host main.dart, or wherever the colliding class is used) needs the alias. Source's internal references to its own classes never collide with host.

## Phase 3 (Adapt) — screenutil

- ✗ **Leaving source's self-rolled width-ratio scaling in place after introducing screenutil**. Source likely has `_kDesignWidth = 402; final scaleX = size.width / _kDesignWidth;` and uses `X * scaleX` everywhere. If you only add `.w/.h` to other dimensions without removing this, you get **double-scaling**: `97.w * scaleX` divides by host's designSize once (via `.w`) AND again (via `scaleX`). Symptom is invisible on the baseline device (`scaleX = 1`) but visible everywhere else — content shrinks on smaller screens, grows on larger. **Phase 3 Step E removes ALL `_kDesignWidth` const + `scaleX/scaleY` variables + `* scaleX/* scaleY` arithmetic**; pure `.w/.h/.r` instead. Grep `scaleX\s*=` to verify zero residual.

- ✗ Mass-sed `height: N` → `height: N.h` blindly. **`TextStyle.height` is a line-height multiplier (unitless, e.g. 1.2 = 120% of fontSize), not a pixel value.** Scaling it by `.h` makes line-height device-dependent in a way no designer ever intended. Phase 3 Step F greps `TextStyle(... height: N.h ...)` and reverts those specific sites.

- ✗ Stripping `Builder(builder: (context) { final scaleX = ...; return Row(...); })` wrappers without rebalancing closing parens. The Builder's `return X;` body becomes the direct child — leaving the `);` `},` `)` trailing parens that paired with Builder/lambda creates "Expected to find ';'" / "Unexpected token" cascades. Always count parens after removing a Builder wrapper; or use Read+Edit on the exact region instead of regex sed.

- ✗ Running `sed 'fontSize: N' → 'fontSize: N.sp'` before stripping `const` from outer widgets. `.sp` is an instance getter; `const TextStyle(fontSize: 14.sp)` produces `const_eval_extension_method` error. **Order: strip const first, then add suffixes.**

- ✗ Stripping `const [` from constructor default params: `this.items = [...]` was `this.items = const [...]`. After strip, compiles only if all `[...]` contents are themselves const. **Restore `const`**: `this.items = const ['图片']`.

- ✗ Sed adds `.r` to `size:` parameters that take `int` (not `double`). Example: `photo_manager`'s `getAssetListPaged(size: int)`. Symptom: `argument_type_not_assignable double → int`. Manual revert: `size: 1.r` → `size: 1`. Watch for analyzer hint "expected int" lines after each pass.

- ✗ Promoting `static const TextStyle` to `static final` but forgetting to update inner `const Color(...)` / `const Offset(...)`. These remain valid const sub-expressions even when the outer is `final`; you don't need to strip them. Leaving them const preserves a tiny amount of optimization. Don't over-strip.

- ✗ Strip-and-suffix produces `const Icon(Icons.X, size: 14.r)`. The `Icons.X` is const, the `14.r` is not, the wrapping `const Icon` fails. Strip the `const Icon` (sed `const Icon(` → `Icon(`). Same for `Text`, `Positioned`, `AppIcon` (any source-defined widget that wraps screenutil-scaled props).

- ✗ Top-level `const TextStyle _titleStyle = TextStyle(fontSize: N.sp, ...)`: file-level `const` field can't reference instance getter. Change to `final TextStyle _titleStyle = TextStyle(...)`. Same for `static const List<BoxShadow> _shadows = [BoxShadow(...blurRadius: N.r)]` → `static final List<BoxShadow>`.

- ✗ Converting source's design-token class (`AppSpacing.lg = 16` → `static double get lg => 16.r`) to enable screenutil scaling. This breaks every `const SizedBox(height: AppSpacing.xxl)` site in the source code (50-100+ callsites). **Trade-off**: keep tokens as `const double` unless host CLAUDE.md is strict; visual drift at ~7% for spacing is invisible. Document the trade-off in verify-report.md.

## Phase 3 — DebounceUtil

- ✗ Wrapping `onTap` inline at every site: `onTap: DebounceUtil.debounce(widget.onPressed)`. Creates a new wrapper every `build`; the wrapper's `lastClickTime` resets to 0 each rebuild → debounce never fires. **Cache in State**:
  ```dart
  @override void initState() { super.initState(); _debounced = DebounceUtil.debounce(widget.onPressed); }
  @override void didUpdateWidget(Old o) { if (!identical(o.onPressed, widget.onPressed)) _debounced = DebounceUtil.debounce(widget.onPressed); }
  // build: onTap: _debounced
  ```

- ✗ Wrapping every `GestureDetector.onTap` in the grafted code. ~100+ sites; mostly low-value (list-item taps that users expect to feel instant). **Wrap only the base button widgets** (e.g. source's `AppButton`, `AppCircleIconButton`); covers ~80% of user-facing buttons.

## Phase 4 — Verify

- ✗ Reporting success after `flutter analyze` 0 error without re-grepping asset paths on disk. Asset 404 has no compile signal. Always Phase 4 step 2.

- ✗ Telling user "hot reload" when they need "hot restart". The asset manifest reloads only on **R** (capital), not r. Include the gotcha in every verify report's smoke-test checklist line 1.

- ✗ Missing structural conflict warnings. Source's `HomePage` self-Scaffolds with its own `BottomNavigationBar`; if host wraps it in another `Scaffold + bottomNavigationBar`, two nav bars stack visually. Compile is clean. Phase 4 must grep this pattern and warn in the report.

## Phase 3 (Adapt) — PageView linkage for tabbed content

- ✗ Implementing "tab + content list" linkage by computing the tap's slot offset from the chip count: `targetOffset = i * (slotWidth + tabGap)`. Works only when chips are **fixed-width** (top-level SwitchTab pattern). Filter chip rows with **text-length-dependent widths** (`'全部'` is 4px wide, `'孝顺的宠物'` is 5×) produce wrong centering. **正解**: tag every chip with a `GlobalKey` and call `Scrollable.ensureVisible(key.currentContext!, alignment: 0.5, duration: ..., curve: ...)`. Reads the real RenderBox after layout and centers exactly.

- ✗ Calling `Scrollable.ensureVisible` synchronously inside `_onFilterSelected` or `_onPageChanged`. The `GlobalKey.currentContext` may be `null` at that point because the chip's RenderBox isn't laid out yet (especially right after a `setState` that changes the active index). **Always wrap in `WidgetsBinding.instance.addPostFrameCallback`** so the call fires after the frame's layout phase finishes.

- ✗ Wiring PageView's `onPageChanged` to `setState(_filterIndex = i)` without a re-entrancy guard. Tap on a chip triggers: `setState` → `animateToPage` → page settles → fires `onPageChanged` → another `setState`. The double setState rebuilds the chip row twice and can cause a visible flicker on slower devices. **正解**: a `bool _isAnimatingPage` flag set true before `animateToPage`, cleared in `.whenComplete`; `onPageChanged` early-returns when the flag is true.

- ✗ Forgetting `PageStorageKey<int>(pageIndex)` on each inner `CustomScrollView` in the PageView body. Swiping right then back left will scroll the previous page back to top (the inner scroll state isn't preserved). PageStorageKey gives each page its own slot in the storage tree.

- ✗ Picking the wrong tab row when there are two. Pages with a top SwitchTab AND a secondary filter chip row will look ambiguous — "下面的分类" could mean either. The chip row closest to the content list is the conventional linkage target, but ask the user before wiring. Re-wiring later is cheap; first-time mis-wire wastes a round-trip.

## Phase 3 (Adapt) — post-cleanup

- ✗ Adding `import 'package:flutter_screenutil/...'` to every grafted .dart file as a Phase 3 prep step, then forgetting to remove it from files that turned out not to need it (after self-rolled scaling code was stripped). Symptom: `unused_import` warnings, host's `very_good_analysis` (or similar lint) treats as actionable. **Final pass**: `grep -L "\.\\(sp\\|w\\|h\\|r\\)\\b" lib/features/<src-name>/**/*.dart | xargs -I {} sed -i '' "/^import 'package:flutter_screenutil\\/.*';$/d" {}` — removes the import from files that don't reference any scaling getter.

## Shell scripting

- ✗ Using `perl -pe 's|foo|bar|g'` where `bar` contains alternation `|`. Perl reads `|` as the regex delimiter end, then parses subsequent chars as modifiers → "Unknown regexp modifier" cascade. **Always switch delimiter** for alternation: `s,foo|baz,bar,g` or `s{foo|baz}{bar}g`.

- ✗ Forgetting `sed -i ''` (BSD syntax on macOS) vs `sed -i` (GNU on Linux). Skill should detect platform once and use the right one, or always quote `''` and accept GNU-side benign warning.
