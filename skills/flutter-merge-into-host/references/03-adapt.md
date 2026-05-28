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

## Optional follow-up: PageView linkage with tab/chip rows

After merge is functionally complete, the user often asks for a small UX upgrade that the source didn't ship: **"swipe left/right between content lists, and have the selected tab/chip auto-scroll into view"**. Common shape:

- Source category page has a top tab row (already shrink/pin via SliverPersistentHeader) and a single content grid below
- User wants the content grid to become a horizontal PageView so swiping switches between filtered lists
- Wants the corresponding tab/chip on top to: (a) reflect the active page (b) auto-scroll into view if outside the visible window

**Identify the correct linkage target first**. A category page often has **two tab rows**: a top-level SwitchTab (`category` axis) and a secondary filter chip row (`filter` axis). Ask the user which one drives the PageView before coding — they're not interchangeable. The closer-to-the-list row is the conventional choice (the filter chip), but confirm.

### Implementation pattern

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

### Bidirectional sync without re-entrant `setState`

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

### Auto-scroll-into-view for variable-width chips

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

### Trade-offs vs alternatives

- `TabController` + `TabBarView`: simpler but doesn't compose well with custom pinned/shrinking SliverPersistentHeader animations. PageView gives more control.
- `ExtendedNestedScrollView` (pub package): adds proper outer-header / inner-PageView linkage for shrink animations. Adds a dep. Default to stock NestedScrollView until proven necessary.

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
