# Phase 3: Adapt

## Goal
Apply host CLAUDE.md rules to the grafted code: flutter_screenutil suffixes everywhere, `DebounceUtil` on tappables, `ToastUtil` instead of source's toast impl, host theme integration, host route helpers (e.g. `pushSingleTop` if defined). Skip in `light` mode.

## Orchestration

1. **Read** host `CLAUDE.md` (if present) to confirm the rules in scope. If absent, advance with empty rule set (no-op for Phase 3).
2. **Read** plan.md "Host adapter rules to apply" section.
3. **Apply each rule with the techniques in this doc**. Heavy mode applies all; light mode skips screenutil + Debounce; as-is skips Phase 3 entirely.

## Rule 1: flutter_screenutil (`.w / .h / .sp / .r`) everywhere

This is the most invasive transform. **Mandatory pre-flight**:

1. **Confirm host has `flutter_screenutil` in pubspec** (direct dep or re-exported via host's `lib_core` / similar). If absent and host CLAUDE.md mentions screenutil, prompt user to set it up first.
2. **Add `import 'package:flutter_screenutil/flutter_screenutil.dart';`** to every grafted `.dart` file that didn't already have it (or import `lib_core` if that's host convention). Skip barrel/router files with no dimensional code.

### Execution order (critical — see failure-modes.md)

The naive approach "sed `fontSize: N` → `fontSize: N.sp`" fails because `.sp` is an instance getter and can't appear in `const` expressions. You must STRIP const FIRST, then add screenutil suffixes.

```bash
# Step A: Strip const from widgets that will contain instance getters
find lib/features/<src-name> -name "*.dart" | while read f; do
  perl -i -pe '
    s/\bconst\s+TextStyle\(/TextStyle(/g;
    s/\bconst\s+SizedBox\(/SizedBox(/g;
    s/\bconst\s+EdgeInsets\./EdgeInsets./g;
    s/\bconst\s+Offset\(/Offset(/g;
    s/\bconst\s+BoxConstraints\(/BoxConstraints(/g;
    s/\bconst\s+Padding\(/Padding(/g;
    s/\bconst\s+BorderRadius\./BorderRadius./g;
    s/\bconst\s+Radius\./Radius./g;
    s/\bconst\s+BoxShadow\(/BoxShadow(/g;
    s/\bconst\s+Icon\(/Icon(/g;
    s/\bconst\s+Text\(/Text(/g;
    s/\bconst\s+Positioned\(/Positioned(/g;
    s/\bconst\s+\[/\[/g;
    s/\bconst\s+<([^>]+)>\s*\[/<$1>\[/g;
  ' "$f"
done

# Step B: Add screenutil suffixes
find lib/features/<src-name> -name "*.dart" | while read f; do
  perl -i -pe '
    s/(fontSize:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/\1\2.sp/g;
    s/(\bwidth:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/\1\2.w/g;
    s/(\bheight:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/\1\2.h/g;
    s/(EdgeInsets\.all\()(\d+(?:\.\d+)?)(\))/$1$2.r$3/g;
    s/(horizontal:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.w/g;
    s/(vertical:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.h/g;
    s/(\b(?:top|bottom):\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.h/g;
    s/(\b(?:left|right):\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.w/g;
    s/(BorderRadius\.circular\()(\d+(?:\.\d+)?)(\))/$1$2.r$3/g;
    s/(Radius\.circular\()(\d+(?:\.\d+)?)(\))/$1$2.r$3/g;
    s/(blurRadius:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.r/g;
    s/(spreadRadius:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.r/g;
    s/(\bsize:\s*)(\d+(?:\.\d+)?)(?![.\d]|\s*\.[swhr]p?\b)/$1$2.r/g;
  ' "$f"
done

# Step C: Analyze; expect const_eval_extension_method errors
flutter analyze lib/features/<src-name>/ 2>&1 | grep -E "^\s*error\s"
```

### Step D: Edge cases (manual fixes per remaining error)

After Step A+B, residual errors are predictable:

| Error pattern | Fix |
|---|---|
| `Extension methods can't be used in constant expressions` on `static const TextStyle X = TextStyle(fontSize: N.sp,...)` | `static const` → `static final` |
| Same on `static const List<BoxShadow> X = [...]` | `static const` → `static final`; each inner `const Color/Offset` preserved |
| Same on top-level `const TextStyle _X = TextStyle(...)` | `const TextStyle _X` → `final TextStyle _X` |
| Same on `decoration: const BoxDecoration(...)` containing `BorderRadius.circular(N.r)` | Remove the `const`; preserve internal `const Color(...)` |
| Same on `const Column(...children: [...Text(fontSize: N.sp)])` | Remove outer `const` |
| `non_constant_default_value` on `this.x = [...]` (was `const [...]`) | Restore `const`: `this.x = const [...]` |
| `non_constant_default_value` on `this.padding = EdgeInsets.symmetric(horizontal: X)` | Restore `const`: `this.padding = const EdgeInsets.symmetric(horizontal: X)` (X must be const itself) |
| `int` parameter wrongly got `.r` (e.g. `getAssetListPaged(size: 1.r)`) | Revert: `size: 1` (sed false positive on non-pixel int args) |

### Theme tokens trade-off

If source has design tokens like `AppSpacing.lg = 16` and host CLAUDE.md mandates screenutil, you have two paths:

| Path | Pros | Cons |
|---|---|---|
| **Keep `const double` tokens** (do not change `AppSpacing` itself) | All `const SizedBox(height: AppSpacing.xxl)` keep working | ~7% visual drift between source designSize (e.g. 375) and host designSize (e.g. 402) for spacing-only values |
| Change tokens to `static double get xxx => N.r` getters | True screenutil-adaptive | Breaks 30-60 `const` callsites; cascading const-chain removal |

Default for auto mode: **keep const tokens**. This is the trade-off seen in real run (cute_animal merge); the 7% drift is invisible on spacing. Heavy-mode users wanting strict compliance can switch later by manually de-const-ing affected sites.

### Step E: Remove self-rolled width-ratio scaling

Phase 0 collected (item `h`) all sites with `_kDesignWidth = N` / `scaleX = size.width / N` / `X * scaleX` patterns. These are the source's own attempt at responsive sizing and **directly collide with host's screenutil**:

- `97.w * scaleX` is **double-scaled**: `.w` already divides by host's `designSize.width`; multiplying again by `scaleX = device.width / 402` divides a second time. On the baseline device (402), `scaleX = 1` so the bug is invisible during dev; on any other device, content shrinks/grows wrong.

**Mechanical replacement** (when source's designWidth equals host's screenutil baseline, the common case):

| Source pattern | Replacement |
|---|---|
| `final scaleX = size.width / 402;` (or `_kDesignWidth`) | delete the line |
| `final scaleY = size.height / 874;` | delete the line |
| `X * scaleX` for a horizontal dimension | `X.w` |
| `X * scaleX` for an icon/square dimension | `X.r` (preferred — handles narrow/tall device variance better than `.w`) |
| `Y * scaleY` for a vertical dimension | `Y.h` |
| `AppSpacing.xl * scaleX` | `AppSpacing.xl.w` |
| const `_kDesignWidth = 402` / `_kDesignHeight = 874` declarations | delete |
| `Builder(builder: (context) { final scaleX = MediaQuery.sizeOf(context).width / 402; return ...; })` | strip the Builder wrapper; use `.w/.h/.r` directly inside the inner widget |

After all replacements, `grep -rn "scaleX\\s*=\\|scaleY\\s*=\\|\\* scaleX\\|\\* scaleY\\|_kDesignWidth\\|kDesignWidth"` in the grafted code must output empty.

**Preserve these MediaQuery usages** (not scaling, real-pixel needs):
- `CustomPainter` paints the actual device canvas — needs `MediaQuery.sizeOf(context).width` directly, not `.w`
- Animations like `Offset(-screenW * 1.2 * (1 - t), 0)` (card slides in from off-screen) — needs real device width
- Layouts like `width: screenW - AppSpacing.xl * 2` (fill viewport minus padding) — needs real device width

The distinguishing question: "Is this number measuring something physical about the device (canvas size, slide-from-edge distance, viewport-minus-padding), or trying to be design-relative (an icon designed at 34px, a card designed at 280px)?" Physical → keep MediaQuery; design-relative → use `.w/.h/.sp/.r`.

### Step F: Watch for `height: N.h` in TextStyle (semantic bug)

`TextStyle.height` is a **line-height multiplier** (unitless, e.g. `height: 1.2` means line is 120% of fontSize). The mass-sed `height: N` → `height: N.h` rule wrongly scales this. Catch by:

```bash
grep -rnE "TextStyle\([^)]*height:\s*\d+(\.\d+)?\.h" lib/features/<src-name>/ --include="*.dart"
```

Any hit → revert that specific `.h` (e.g. `height: 1.h` → `height: 1`). The sed in Step B should ideally pattern-match `height:` only inside `SizedBox` / `Container` / `Positioned`, not inside `TextStyle`, but Bash regex can't do that AST-level disambiguation cheaply. Manual revert pass after Step B.

## Rule 2: DebounceUtil on tappables

Locate the source's base button widgets (typically `app_button.dart`, `app_circle_icon_button.dart`, or whatever wraps the main `onPressed`/`onTap` API). Wrap in **state-level cache** to avoid rebuilding the debounced closure each `build`:

```dart
class _AppButtonState extends State<AppButton> {
  VoidCallback? _debouncedOnPressed;

  @override
  void initState() {
    super.initState();
    _debouncedOnPressed = DebounceUtil.debounce(widget.onPressed);
  }

  @override
  void didUpdateWidget(AppButton oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (!identical(oldWidget.onPressed, widget.onPressed)) {
      _debouncedOnPressed = DebounceUtil.debounce(widget.onPressed);
    }
  }

  // in build:
  // onTap: _debouncedOnPressed
}
```

Why state-cache and not `onTap: DebounceUtil.debounce(widget.onPressed)`: latter creates a new wrapper every build, breaking debounce state (each tap → fresh lastClickTime).

Add `import 'package:lib_core/lib_core.dart';` (or wherever host re-exports DebounceUtil).

**Do not wrap individual `GestureDetector.onTap` sites in the grafted code** — too invasive. Base-widget wrap covers ~80% of taps; the remaining are rare enough to defer.

## Rule 3: Toast replacement

If source has a custom toast impl (e.g. `AppToast.show(context, msg)`):

1. Grep `AppToast.show(` in grafted code
2. Replace each call with host's `ToastUtil.show(msg)` (drop `context` param; host's util uses overlay or scaffold)
3. If `AppToast` is exported by a barrel: edit the barrel to remove that export
4. Delete the `app_toast.dart` source file (already not copied per plan.md drop list)

If source has no toast impl, skip.

## Rule 4: Host theme integration

If host has `lib/theme/` with `ThemeData`, the grafted code's hardcoded colors should ideally migrate. **Conservative default**: leave source colors alone (per-feature constants). Migration to host theme is a follow-up refactor, not part of merge.

Only do the migration if source colors map 1:1 to host theme tokens. Otherwise visual fidelity suffers.

## Rule 5: go_router version match

If source used go_router APIs that changed in the host's version (e.g. v17 → v14):

- `state.uri.queryParameters` — same in v6+
- `state.extra` — same
- `GoRoute(path, builder)` — same
- `ShellRoute`, redirects: API may have changed; check release notes

Fix any breakage per analyze errors. If APIs diverge irreconcilably, ask user whether to bump host go_router or fork the source code.

## Rule 6: Category-list pattern → PageView linkage (auto-detect + transform)

**When this rule fires**: Phase 0 item `i` produced a non-empty "Category-list pattern candidates" list — pages that have a tab/chip row + a single content list but NOT yet a PageView/TabBarView. These pages ship a stale UX (tap a tab to filter, no swipe between lists) that the host's CLAUDE.md (if it exists) typically expects to be upgraded. **Auto mode runs this transform mandatorily** for every Phase 0 candidate; safe mode surfaces a checkpoint per candidate.

**⚠️ Detection is by widget tree, not filename.** Do NOT use the filename (e.g. "it's called `category_page.dart` so apply", or "it's not called `category_page.dart` so skip") as either the trigger or the skip reason. The signals are: (a) the page mounts a tab/chip widget row, AND (b) the page's main body is a scrollable list widget, AND (c) the page does NOT already use PageView/TabBarView. A page called `feed.dart` / `discover.dart` / `index.dart` with this combination is just as much a candidate; a page called `category_page.dart` whose body is actually a chooser dialog (not a content list) is NOT.

**⚠️ "Preserving source's original structure" is NOT a valid skip reason.** Auto mode's contract is precisely to make these UX-consistency transforms without asking. The three legitimate skip conditions are listed in § "Skip conditions" below; anything else means the transform must run. If you're tempted to skip with reasoning like "降低改造面" / "minimize churn" / "source UX was deliberate" — stop, run the transform, and document any concrete UX trade-off in the verify report instead.

**⚠️ Source code with placeholder/static data is NOT a skip reason.** Sources often ship category pages whose chip rows don't yet filter (TODO for API integration). The PageView transform still runs — each page renders the same placeholder list, marked with a clear `// TODO: per-pageIndex data once API ready` at the SliverChildBuilderDelegate. When the API lands later, only the data source switches; the layout / linkage is already in place. Skipping because "it would look like N duplicate pages" reverses the intent of the rule (preparing the UX shape ahead of the data).

**Why this is not optional**: a "category + single list" page produces the wrong runtime expectation when shown next to other pages in the host that already use PageView linkage. Users habitually swipe; the unmigrated page feels broken. The grafted code being "functionally complete" but UX-inconsistent is a real silent failure — analyzer can't catch it, only manual smoke-test will, and only if the reviewer happens to swipe that exact page.

### Step 1: Identify the linkage target row

A category page typically has one OR two tab rows:

| Source layout | Linkage target |
|---|---|
| Single top-level SwitchTab/TabBar row + content list | that row drives PageView |
| Two rows: top SwitchTab (broad category) + secondary chip row (filter) + content list | the **row closest to the list** drives PageView (filter chip in this example) |
| Two rows where both rows together filter the list | drive PageView from chip row; top row stays as broad navigation (Phase 1 should have asked user, but if auto mode picked default, this is it) |

Detect by reading the source file's widget tree: the `tabIndex` / `filterIndex` (or equivalents) that's used to compute the list's data subset is the linkage target. If two indices both filter the list, the one whose row sits visually closer to the list (smaller y-offset gap) wins by convention.

Confirm via Phase 1 plan.md if it recorded a user answer; otherwise apply default and emit a note in the verify report.

### Step 2: Implementation pattern

The outer scroll has to stay a `NestedScrollView` (so the existing pinned SliverPersistentHeader's shrink animation keeps working with inner-page scroll); body becomes a `PageView`; each page is its own `CustomScrollView`.

```dart
NestedScrollView(
  headerSliverBuilder: (context, _) => [
    SliverOverlapAbsorber(
      handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
      sliver: SliverPersistentHeader(pinned: true, delegate: existingDelegate),
    ),
  ],
  body: PageView.builder(
    controller: _pageController,
    onPageChanged: _onPageChanged,  // sets selectedIndex + scrolls chip into view
    itemCount: filterCount,
    itemBuilder: (context, pageIndex) => Builder(
      builder: (context) => CustomScrollView(
        key: PageStorageKey<int>(pageIndex),  // preserves per-page scroll position
        slivers: [
          SliverOverlapInjector(
            handle: NestedScrollView.sliverOverlapAbsorberHandleFor(context),
          ),
          SliverPadding(
            padding: ...,
            sliver: SliverMasonryGrid(...), // or any inner sliver
          ),
        ],
      ),
    ),
  ),
)
```

### Step 3: Bidirectional sync without re-entrant `setState`

```dart
bool _isAnimatingPage = false;  // guard

void _onFilterSelected(int i) {
  if (_filterIndex == i) return;
  _isAnimatingPage = true;
  setState(() => _filterIndex = i);
  _scrollFilterIntoView(i);
  _pageController.animateToPage(i,
    duration: Duration(milliseconds: 300), curve: Curves.easeInOut,
  ).whenComplete(() => _isAnimatingPage = false);
}

void _onPageChanged(int i) {
  if (_isAnimatingPage) return;  // skip when triggered by tap, animateToPage handled it
  setState(() => _filterIndex = i);
  _scrollFilterIntoView(i);
}
```

### Step 4: Auto-scroll-into-view for variable-width chips

Tab rows with **fixed-width** slots can use offset math (`i * slotWidth + gap` → `animateTo`). Chip rows with **text-length-dependent widths** ("全部" vs "孝顺的宠物") cannot — `Scrollable.ensureVisible` reads the chip's actual `RenderBox` and computes a correct centering offset:

```dart
late final List<GlobalKey> _chipKeys =
    List.generate(filters.length, (_) => GlobalKey());

void _scrollFilterIntoView(int i) {
  WidgetsBinding.instance.addPostFrameCallback((_) {
    final ctx = _chipKeys[i].currentContext;
    if (ctx == null) return;
    Scrollable.ensureVisible(ctx,
      alignment: 0.5,  // 0=left edge, 0.5=center, 1=right edge
      duration: Duration(milliseconds: 300), curve: Curves.easeInOut,
    );
  });
}
```

`alignment: 0.5` centers the chip; `0.0` snaps to leading edge (good for "always show next chip preview"); `1.0` snaps to trailing edge.

### Step 5: Verification (added to Phase 4 scorecard)

Each transformed page must satisfy:

```bash
# Linkage target row was wired
grep -E "_pageController\.animateToPage" lib/features/<src-name>/<sub>/pages/<file>.dart
# PageView present
grep -E "PageView\.builder|PageView\(" lib/features/<src-name>/<sub>/pages/<file>.dart
# Per-page state preserved
grep -E "PageStorageKey" lib/features/<src-name>/<sub>/pages/<file>.dart
# Variable-width chip uses ensureVisible (not offset math)
grep -E "Scrollable\.ensureVisible" lib/features/<src-name>/<sub>/pages/<file>.dart
# Re-entrancy guard
grep -E "_isAnimatingPage|isAnimating" lib/features/<src-name>/<sub>/pages/<file>.dart
```

All five must hit at least once. Auto mode aborts that page's transform and adds it to verify-report "manual follow-up" if any check fails.

### Trade-offs vs alternatives

- `TabController` + `TabBarView`: simpler but doesn't compose well with custom pinned/shrinking SliverPersistentHeader animations. PageView gives more control.
- `ExtendedNestedScrollView` (pub package): adds proper outer-header / inner-PageView linkage for shrink animations. Adds a dep. Default to stock NestedScrollView until proven necessary.

### Skip conditions (do NOT apply Rule 6 even when Phase 0 detected the pattern)

- The list is short enough to fit one screen (no scroll). PageView between short lists is a UX downgrade — less discovery, no swipe affordance to user.
- The "filter" is single-toggle on/off (e.g. "show only favorites"). Use a switch widget, not multi-page PageView.
- The page is a settings / form / detail with one section per tab. Tabs are organizational, not data-axis; users don't expect swipe-to-switch-section.

If skipped, record reason in verify-report under "Rule 6 skipped pages".

## Rule 7: Splash (开屏) — replace source splash with host's central module

**When this rule fires**: Phase 0 items `j` (source splash) AND `k` (host central splash module) both produced a hit. The host has its own splash flow (typically agreement gate → device register → token bootstrap → ad → home) that is **load-bearing initialization** — the source's splash widget would bypass it, breaking host's startup contract.

**Why this is not optional**: source splash usually only shows a logo and a timer-then-navigate. Letting it stay as the entry point means: (a) host's agreement gate never appears (legal/compliance risk) (b) device-register and user-token bootstrap never run → API calls 401 (c) splash ad never serves → revenue loss. These are silent failures that don't show as compile errors; only manual end-to-end smoke would catch them.

**Skip conditions**:

- Host has no central splash module (Phase 0 item `k` empty) → leave source splash at its prefixed route, no transform.
- Source splash is the ONLY route shown to first-time users for a non-launch reason (e.g. it's actually an onboarding page mislabeled "splash") → keep as feature page, not a true launch splash.

### Step 1: Locate host's splash integration point AND widget injection capability

Phase 0 item `k` recorded the host's splash entry. **Critical sub-question**: does the host's splash widget expose a *widget injection slot* for fully custom UI (animation / video / 帧序列), or does it only accept a single logo asset?

Grep host's splash class:

```bash
# Find splash class file from item `k`
SPLASH_FILE=<from-item-k>

# Look for widget injection parameters
grep -nE "Widget\? (backgroundChild|splashBackground|customSplash|splashContent|splashBuilder)" "$SPLASH_FILE"
grep -nE "WidgetBuilder\? splashBuilder" "$SPLASH_FILE"

# Verify the factory exposes it (sometimes the SplashPage has the param but the factory doesn't pass-through)
grep -nE "splashBackground|backgroundChild|customSplash|splashBuilder" <host-splash-route-factory-file>
```

Three host capability levels:

| Level | What's exposed | Strategy enabler |
|---|---|---|
| **L0** — logo only | `splashLogoAsset: String?` (or `setLogoIcon(...)`); no widget param | Path A (logo replacement) only |
| **L1** — widget injection exists in SplashPage but not exposed in route factory | `SplashPage` has `backgroundChild: Widget?` etc., but `buildBasePageRoutes()` doesn't pass it through | Phase 3 Step 2.5: add the pass-through (one-line factory change); now treat as L2 |
| **L2** — widget injection fully exposed | Factory has `splashBackground: Widget?` or `splashBuilder: WidgetBuilder` | Path B (inject-as-background) available |

If host is L0 and source has more than a single-image visual (animation/video), **Phase 3 must extend the host first** (see Step 6 below). Document this in plan.md; it's a host-side change that the merge skill is allowed to make because it's strictly additive (existing callers using `splashLogoAsset` keep working).

Also read host's integration guide (Phase 0 grepped its path) to confirm the documented API; the guide is the source of truth.

### Step 2: Classify source splash by visual + side-effects

```bash
grep -nE "await |\.then\(|Future\.|Timer|Provider\.of|GetIt|init\(|register|fetch|load" \
  <source-splash-file>
```

| Classification | Signal | Phase 3 path |
|---|---|---|
| **Logo-only** | `Stack/Image.asset` + `Timer(Duration, () => context.go('/'))` only | **Path A**: replace with `splashLogoAsset` |
| **Animation / video / 帧序列 (visual-only)** | Multiple frames + Timer.periodic frame stepping, AnimationController, VideoPlayerController, Lottie, etc. WITHOUT business calls | **Path B**: extract to content widget, inject via `splashBackground` (preferred — preserves designer's intended visual) |
| **Bespoke business init** | Repository / fetch / GetIt / `await registerDevice()` / state mutation | **Path C**: extend-host, distribute side-effects to host's init chain; for the visual part, also Path B if visual is non-trivial |
| **Onboarding pretending to be splash** | First-time-user-only content (welcome / permission ask) | NOT a real splash → keep at prefixed route as feature page |

**Default precedence**: when source has any animation/video, prefer Path B over Path A — collapsing a frame animation to a single logo is a visual regression the designer didn't intend. Path A only when source visual is truly a static logo + timer.

### Step 3: Execute Path A (logo-only)

1. **Extract source logo asset** — from Phase 0's recorded asset list, pick the primary logo image.
2. **Drop source splash route + widget files** — remove GoRoute, route constant, splash widget file(s).
3. **Update host's splash factory call** — patch `lib/main.dart`:
   ```dart
   ...buildBasePageRoutes(splashLogoAsset: 'assets/images/<src-name>/splash/logo.png'),
   ```
4. **Keep the logo asset file** at its namespaced path (already there from Phase 2).
5. **Search for stale references** — `grep -rnE "context\.(go|push)\s*\(\s*'/<src-name>/splash" lib/` must be empty.

### Step 4: Execute Path B (inject-as-splashBackground — preferred for animation/video)

Goal: keep source's bespoke visual experience (frame animation, video, Lottie) but defer ALL navigation / lifecycle / business logic to host's SplashViewModel.

1. **Extract content widget** from `<sub>/splash/pages/splash_page.dart`:
   - Rename: `lib/features/<src-name>/splash/widgets/<src_name>_splash_content.dart`
   - Class rename: `SplashPage` → `<SrcName>SplashContent` (e.g. `CuteAnimalSplashContent`)
   - **Strip the following** (host owns these now):
     - `Scaffold` wrapper → return directly the visual `Stack` / `Center` / `Image.asset`
     - `context.go(...)` / `context.push(...)` navigation calls → delete; host's `SplashViewModel` handles routing
     - `WidgetsBindingObserver` + lifecycle handlers → delete; host handles foreground/background
     - `SystemChrome.set*` calls → delete; host sets overlay style
     - Any business init (`await xxxRepository.X`, `GetIt.I<X>`, `register*`, `fetch*`) → goes to Path C destination, NOT this widget
   - **Keep**:
     - Animation controllers, frame timer, `precacheImage` (pure visual)
     - The visual `Image.asset` / `VideoPlayer` / `Lottie` widget
     - Animation completion: when reached, just `timer.cancel()` and hold final frame (do NOT call `context.go`)
2. **Delete source splash route + page file** — keep route off, the content widget will be injected at host's existing splash route.
3. **Inject in `lib/main.dart`**:
   ```dart
   ...buildBasePageRoutes(
     splashBackground: const <SrcName>SplashContent(),
   ),
   ```
4. **Verify host capability** — if Step 1 marked host as L0, you must first do Step 6 (extend host).

### Step 5: Execute Path C (extend-host for bespoke business init)

Use when source splash has business side-effects beyond visual. Often combined with Path B if the visual part is non-trivial.

1. **Audit each side-effect line-by-line**, classify destination:
   | Side-effect kind | Destination |
   |---|---|
   | One-shot config (`setApiHost`, `loadFeatureFlag`) | `ServiceCenterConfig.builder()` chain or before `initServiceCenter` |
   | Per-session bootstrap (`registerDevice`, `loadUserProfile`) | Verify already in host's SplashViewModel; if missing, file host-side TODO |
   | Visual preload (`precacheImage`, `precacheNetwork`) | Keep in the content widget (Path B), it's still visual |
   | First-launch-only logic (`if !hasShownGuide`) | Likely belongs in host's guidePage flow, not splash |
2. **Drop source splash widget + route** (or extract content for Path B if visual is non-trivial).
3. **Document each migration** in verify-report.

### Step 6: Extend the host (when L0 + source has animation/video)

**This is a host-side change the merge skill is allowed to make**, because:
(a) it's strictly additive (existing `splashLogoAsset` callers continue working)
(b) the integration guide adoption is part of the merge contract
(c) without it, Path B is impossible and Path A would silently regress visual fidelity

Concrete changes (use real shapes seen in the wild as templates):

**Change 1 — pass-through in the route factory**:

```dart
// Before:
List<RouteBase> buildBasePageRoutes({String? splashLogoAsset}) => [
  GoRoute(path: ..., builder: (_, __) => SplashPage(splashLogoAsset: splashLogoAsset ?? '')),
];

// After:
List<RouteBase> buildBasePageRoutes({
  String? splashLogoAsset,
  Widget? splashBackground,
}) => [
  GoRoute(path: ..., builder: (_, __) => SplashPage(
    splashLogoAsset: splashLogoAsset ?? '',
    backgroundChild: splashBackground,
  )),
];
```

If `SplashPage` already has `backgroundChild` parameter (often pre-wired), only the factory needs editing. If not, the SplashPage's `build` also needs a Stack child that uses `backgroundChild` when provided, falls back to `Image.asset(splashLogoAsset)`.

**Change 2 — integration guide section**: add to the host's `INTEGRATION_GUIDE.md` (or equivalent) a "自定义开屏页 UI" / "Custom Splash UI" section. Required contents:

- Two-layer architecture diagram: 视觉层 (custom widget) + 逻辑层 (host's ViewModel)
- Two API usages: `splashLogoAsset` (single image) vs `splashBackground` (widget)
- "Writing your splash content" constraints list — 6 don'ts (the same as Step 4 strip list)
- A 30-line example of a frame-animation content widget

**Change 3 — table in the guide's symbol index**: update the row that documents `buildBasePageRoutes(...)` to include the new param.

After Step 6, treat host as L2 and proceed with Step 4 (Path B).

### Step 7: Verification (added to Phase 4 scorecard)

```bash
# Source splash route deleted from routes file
! grep -E "path:\s*'/<src-name>/splash" lib/features/<src-name>/router/<src-name>_routes.dart
# No orphan source splash route references
! grep -rnE "context\.(go|push)\s*\(\s*'/<src-name>/splash" lib/
# Path A: host main.dart references the new logo via splash factory
# OR
# Path B: host main.dart references the new content widget via splashBackground
grep -E "splashLogoAsset:|setLogoIcon\(|splashBackground:" lib/main.dart | grep "<src-name>"
# Path B: content widget exists and is properly stripped
if [ "<PATH>" = "B" ]; then
  test -f "lib/features/<src-name>/splash/widgets/<src_name>_splash_content.dart"
  # Forbidden patterns inside content widget
  ! grep -E "Scaffold\(|context\.(go|push)|WidgetsBindingObserver|SystemChrome\." \
    lib/features/<src-name>/splash/widgets/<src_name>_splash_content.dart
fi
# Step 6 (if executed): host factory has the new param
if [ "<HOST_EXTENDED>" = "yes" ]; then
  grep -E "Widget\? splashBackground" <host-splash-route-factory-file>
fi
```

All applicable checks must pass.

## Rule 9: Base button label overflow guard (preemptive fix)

**When this rule fires**: source has its own base button widget (typically `AppButton` / `AppCircleIconButton` / `CommonButton` — anything wrapping `onPressed`/`onTap` plus a label) AND the page composes buttons via a `Row` + Expanded/flex pattern (e.g. `AppButtonRow(secondary, primary)` with a 1:2.6 ratio). Default to running this rule when Rule 2 (DebounceUtil) targets the same widget — they patch the same files.

**Why this is not optional**: source's label is typically `Row(mainAxisSize: min, [Text, gap, Icon])`. Works in source's standalone runtime because there source has its own fonts (PingFang SC on iOS, designer's spacing). After merge into a host that ships under different fonts (Android with Noto Sans CJK fallback), Chinese glyphs are 3–5% wider; in a narrow flex slot (≤ ~50w container, common when secondary button gets ~30% of a 350w row), the content widens past the container by 5–15 px. **Runtime crash-free but visible "overflowed by N pixels" yellow-black stripe**; analyzer can't catch this — only device run does.

`MainAxisSize.min` doesn't help: each child's intrinsic width is fixed (Text width = fontSize × glyph count + kerning; Icon width = size). Row can't shrink below sum of children.

### The fix

Wrap the label `Row` in `FittedBox(fit: BoxFit.scaleDown)` and harden the Text:

```dart
// Before
final Widget label = Row(
  mainAxisSize: MainAxisSize.min,
  children: [
    Text(widget.label, style: ...),
    if (widget.icon != null) ...[
      SizedBox(width: AppSpacing.xs),
      AppIcon(icon: widget.icon!, size: _iconSize, color: ...),
    ],
  ],
);

// After
final Widget label = FittedBox(
  fit: BoxFit.scaleDown,
  child: Row(
    mainAxisSize: MainAxisSize.min,
    children: [
      Text(
        widget.label,
        maxLines: 1,
        softWrap: false,
        style: ...,
      ),
      if (widget.icon != null) ...[
        SizedBox(width: AppSpacing.xs),
        AppIcon(icon: widget.icon!, size: _iconSize, color: ...),
      ],
    ],
  ),
);
```

**Why `scaleDown` not `scale`**: `scaleDown` only shrinks when content exceeds the container; on normal-width buttons it's a no-op. `BoxFit.scale` would also enlarge content on wide buttons, ruining typography.

**Why also `maxLines: 1, softWrap: false`**: belt-and-suspenders. If a future change uses `FittedBox(fit: contain)` instead of `scaleDown`, the Text-only safety net still prevents soft-wrap inside a 1-line button.

### Step 1: locate source's base button widget(s)

Same as Rule 2 — the file already has DebounceUtil wired in by Rule 2:

```bash
grep -rlE "class App(Button|CircleIconButton|TextButton)" lib/features/<src-name>/widgets/
```

Apply the FittedBox+Text guard to the `label` (or equivalent body) widget of each.

### Step 2: do NOT wrap inline Text in arbitrary GestureDetectors

This rule only applies to **base button widgets** the source maintains. Don't blanket-wrap every Text in the grafted code — invasive and unnecessary.

### Step 3: Verification (added to Phase 4 scorecard)

```bash
# Each grafted base button widget contains FittedBox.scaleDown around its label
for f in $(grep -rlE "class App(Button|CircleIconButton)" lib/features/<src-name>/widgets/); do
  grep -E "FittedBox\(\s*fit:\s*BoxFit\.scaleDown" "$f" > /dev/null || echo "MISSING_GUARD: $f"
done
```

Empty output required.

### Skip conditions

- Source's button is icon-only with no Text label → Row content is just an Icon → fixed width with no font fallback issue → no guard needed.
- Source's button uses `Expanded(child: Text)` inside its own internal Row → already handles narrow case correctly → no need to add FittedBox (would scale-down even when ellipsis is more appropriate). Leave alone.
- Source's button is composed into a parent that ALWAYS gives it `width: double.infinity` (full-width buttons only, never narrow) → font drift produces equal-distribution Text width, no overflow → guard is harmless but unnecessary; record skip reason.

## Rule 8: Widget file split when files exceed readability thresholds

**When this rule fires**: scan every grafted `.dart` file under `lib/features/<src-name>/`. Any file that trips ANY of these thresholds must be split:

- file LOC > 300
- file contains > 5 `class _<Foo> extends (Stateless|Stateful)Widget` private widget classes
- file declares > 10 file-level / static `const` design tokens (colors, asset paths, spec rows)

Sources written as a single mega-file (1000+ LOC pages with 15+ private widgets) are common; grafting drops them into the host unchanged. Reviewing or surgically editing those files later — by a human or by Claude — is hostile when each file has a dozen widgets fighting for attention.

### Scan command (Phase 3 runs this once after Phase 2 succeeds)

```bash
printf "%-90s %6s %6s\n" "file" "LOC" "_priv"
find lib/features/<src-name> -name "*.dart" | sort | while read f; do
  loc=$(wc -l < "$f")
  priv=$(grep -cE "^class _[A-Z]" "$f")
  if [ "$loc" -gt 300 ] || [ "$priv" -gt 5 ]; then
    printf "%-90s %6d %6d\n" "$f" "$loc" "$priv"
  fi
done
```

Record matches in plan.md's "Widget split targets" section. Auto mode splits every match; safe mode surfaces a single CHECKPOINT confirming the split list before executing.

### Target layout per split file

```
lib/features/<src-name>/<sub>/
├── pages/
│   └── <sub>_page.dart           # page entry; thin (Scaffold + Stack + composed widgets)
├── widgets/                      # one file per logical region
│   ├── <sub>_<region_a>.dart
│   ├── <sub>_<region_b>.dart
│   └── ...
├── dialogs/                      # popup-style flows
└── models/                       # data specs / enums shared across widgets
    └── <name>.dart
```

### Step 1: Group widgets by region, not one-per-file

A page with 12 private widgets does NOT split into 12 files. Group widgets that compose together (region-of-the-screen, or "card + its parts"). Each group becomes one file; widgets used only inside the group stay `_` private inside that file. Only the entry widget the page directly mounts goes public.

Example from cute_animal merge (settings_popup.dart had 10 private widgets):

| File | Public entry | Private internals |
|---|---|---|
| `widgets/settings_header_section.dart` | `SettingsHeader`, `SettingsVipBanner` | `_GuestHeaderActions`, `_PhonePill`, `_VipBannerButton` |
| `widgets/settings_card.dart` | `SettingsCard`, `SettingsRow`, `SettingsDivider` | — |
| `widgets/settings_footer.dart` | `SettingsFooter` | — |
| `models/user_membership_state.dart` | `enum UserMembershipState` | — |

Three widget files + one model file replaced the 722-line monolith. Page entry shrank to 321 lines.

### Step 2: Rename private widgets used cross-file → public with feature prefix

`_FooBar` used outside its origin file → `<SrcName><FooBar>` (e.g. `_SettingsHeader` → `SettingsHeader`, `_GridCard` → `CategoryGridCard`). Same-file-only private widgets keep their `_` prefix.

**Why feature prefix**: source merger may have multiple sub-features all calling something `_HeaderCard`; once exported as `HeaderCard`, host import resolution becomes ambiguous. `<SrcName>HeaderCard` keeps them distinct.

### Step 3: Top-level `const` accessed across files needs file-level scope

Source's mega-file often has `class _FooDelegate { static const double _kHorizPad = 16; }` referenced by the enclosing State class via `_FooDelegate._kHorizPad`. Once `_FooDelegate` moves to a new file, the State class can't reach `._kHorizPad` (private static).

Two options:

| Path | Edit | Trade-off |
|---|---|---|
| Promote class to public AND const to public | `_FooDelegate._kHorizPad` → `FooDelegate.kHorizPad` | More cross-file access, but verbose call sites |
| Lift const to file-level top of new file | `const double kFooDelegateHorizPad = 16;` outside the class | Cleaner call sites; needs the same const accessible from the new file's exports |

Default to **file-level top consts** (cleaner) when ≤ 3 consts cross the boundary. Promote class statics when there are many.

### Step 4: Sed-rewrite the page entry's references

Use `perl -i -pe` (BSD `sed \b` is unsupported) to rename in one pass:

```bash
perl -i -pe '
  s/\b_FooBar\b/<SrcName>FooBar/g;
  s/\b_FooDelegate\._kHorizPad\b/k<SrcName>FooHorizPad/g;
  s/\b_FooDelegate\b/<SrcName>FooDelegate/g;
' lib/features/<src-name>/<sub>/pages/<sub>_page.dart
```

Order matters when one rename is a substring of another (e.g. `_Foo` and `_FooBar`) — list the longer match first.

### Step 5: Truncate the page file + add imports

```bash
# Drop the extracted lines (preserving everything before the first private widget)
awk 'NR<=<last-keep-line>' lib/features/<src-name>/<sub>/pages/<sub>_page.dart > /tmp/p.dart
mv /tmp/p.dart lib/features/<src-name>/<sub>/pages/<sub>_page.dart

# Add imports for the new widget/model files
# (Edit tool to insert next to existing relative imports)
```

After truncation, `flutter analyze` will surface stale imports (now `unused_import` warnings). Delete them.

### Step 6: Verification (added to Phase 4 scorecard)

```bash
# Each split-target file under threshold
find lib/features/<src-name> -name "*.dart" | while read f; do
  loc=$(wc -l < "$f")
  priv=$(grep -cE "^class _[A-Z]" "$f")
  [ "$loc" -gt 300 ] && echo "STILL_OVER: $f ($loc lines)"
  [ "$priv" -gt 5 ] && echo "STILL_PRIV: $f ($priv private widgets)"
done
# Output should be empty OR only files documented in plan.md as "intentionally over" with a reason
```

### Skip conditions (file stays unsplit even when over threshold)

- **State class itself is the bulk** (e.g. `_StudioPageState` 800 lines with init / dispose / lifecycle / 20+ build helpers). Widget split won't help — needs a State-level refactor (extract camera controller, gallery controller, animation controller etc. into mixins or controller-style classes). Out of merge scope; flag in verify-report as "follow-up: State class refactor for `<file>` (LOC=N)".
- **Generated code** (build_runner output `.g.dart` / `.freezed.dart`) — never split.
- **Single huge enum / data table** that's read-only data, not widget code — keep as-is.

### Anti-patterns

- ✗ Splitting into too many files (< 50 lines each) — composition tax beats readability gain. Aim 80-250 lines per file.
- ✗ Promoting EVERY private widget to public — bloats the feature's public surface area. Only the entry widgets the page mounts should be public.
- ✗ Re-exporting via a `widgets.dart` barrel after splitting — kills the locality benefit of splitting. Each page imports only the widget files it needs.
- ✗ Leaving stale relative imports after extraction — analyzer flags as `unused_import`; cleanup as part of Step 5.

## Final pass: flutter analyze 0 error

After all rule applications:
```bash
flutter analyze lib/features/<src-name>/ lib/main.dart
```
must show `0 error`. Info-level lints (`always_use_package_imports`, `prefer_const_constructors`) are acceptable; they're style-only and the const-strip step inevitably introduces some.

Auto mode: log `[AUTO] Phase 3 done: <N> screenutil sites, <M> Debounce wraps, <K> manual const fixes.`

## Stopping conditions

- After 3 sed-iterate-fix cycles in screenutil rule, > 10 errors remain → abort to user; the source may have unusual const patterns (e.g. `const factory` constructors)
- DebounceUtil not exported by host's lib_core / similar → abort, ask user to point to the right import
