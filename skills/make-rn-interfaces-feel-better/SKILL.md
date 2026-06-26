---
name: make-rn-interfaces-feel-better
description: Design engineering principles for making React Native interfaces feel polished. Use when building or reviewing RN UI — components, StyleSheet, Reanimated animations, Pressable feedback, shadows and elevation, border radius, optical alignment, safe-area insets, typography, micro-interactions, enter/exit and icon animations, or any visual detail work. Triggers on RN UI polish, "make it feel better", "feels off", concentric radius, hitSlop, tabular numbers, iOS shadow vs Android elevation, hairline dividers, stagger/spring animations, and the Reanimated/Skia/SVG thread model.
---

# Details that make React Native interfaces feel better

Great interfaces rarely come from a single thing. They're a collection of small details that compound into a great experience. Apply these when building or reviewing React Native UI.

React Native adds two wrinkles the web doesn't have: you are styling **two native renderers at once**, so several of these details are about making iOS and Android agree — and your animations can run on the **UI thread or the JS thread**, so a few are about keeping motion off the bridge. This skill is the [web original](https://github.com/jakubkrehel/make-interfaces-feel-better) translated honestly into RN: what maps, what changes, and what simply has no equivalent.

## Before you apply these: read the project first

These principles aren't context-free, and run blindly the checklist cries wolf. Two checks up front prevent most false positives:

1. **Platform targets — is Android actually shipped?** Look for a real `android/` directory and an Android build profile (e.g. in `eas.json`) — **not** the `android` block in `app.json` / `app.config`, which Expo writes whether or not you ship Android. If the app is **iOS-only**, skip the Android half of these rules: [platform.md](platform.md)'s `elevation` / ripple guidance is moot, and you should *not* flag an iOS-only shadow for "missing `elevation`."
2. **Design system — read the tokens.** Find the radius / color / spacing / shadow / font tokens before flagging anything. A radius **scale** changes how concentric radius applies (#1); a **monospace** number font makes tabular figures a no-op (#18); a warm or branded neutral changes the default outline color (#6); skeuomorphic surfaces use 1px bevel edges that look like dividers but aren't (#5).

When a principle below assumes cross-platform or token-less defaults, these two facts override it.

## Quick Reference

| Category | When to Use |
| --- | --- |
| [Surfaces](surfaces.md) | Border radius, optical alignment, shadows, hairline dividers, image outlines, hit areas |
| [Animations](animations.md) | Interruptible motion, enter/exit, icon animations, press feedback, spring vs timing |
| [Typography](typography.md) | Tabular numbers, custom font loading, the text-wrap gap |
| [Performance](performance.md) | UI-thread motion, the renderer matrix, New Architecture |
| [Platform](platform.md) | iOS shadow vs Android elevation, Pressable/ripple, safe-area insets |

## Core Principles

### 1. Concentric Border Radius

Outer radius = inner radius + padding, computed in `StyleSheet`. If the project has a **radius token scale** (e.g. `{ card: 6, control: 8 }`), reconcile the two: concentric math wins for **tightly-nested** elements (a control inside a track, a cell inside a ladder); the **token scale governs independent surfaces**. Mismatched radii on nested elements is the most common thing that makes an interface feel off.

### 2. Optical Over Geometric Alignment

When geometric centering looks off, align optically. A play triangle needs a small `marginLeft`; an icon next to text wants slightly less padding on the icon side. Fix the SVG/asset directly when you can, manual margin when you can't.

### 3. Shadows Over Borders — Set `elevation` if Android Ships

Prefer a subtle shadow to a border for depth on cards and buttons. **If Android is a shipped target** (see the preflight), an iOS-only `shadow*` style renders flat there — Android depth comes from `elevation`, so set both. Two cases where `elevation` is *wrong*, not just missing: a **silhouette shadow** (transparent background, #4) — `elevation` traces a box or renders nothing; and an **offset/directional shadow** (e.g. an upward shadow) — `elevation` is symmetric and can't reproduce it. Keep real dividers as borders (see #5).

### 4. Silhouette Shadows

Give a card `backgroundColor: 'transparent'` and the iOS shadow follows the content's **alpha silhouette** instead of a box outline — the trick for making an irregular shape (artwork, an icon, a cut-out) cast a true shadow. Reserve `elevation` for things that should genuinely read as lifted.

### 5. Crisp Hairline Dividers — but Know What's a Divider

For real **separators**, use `borderBottomWidth: StyleSheet.hairlineWidth`, never `1` — `hairlineWidth` is one **physical** pixel, the density-aware way to draw the thin lines the web does with `1px`. But a 1px **bevel / inset / depth edge** (a skeuomorphic light-or-shadow border that fakes a raised or recessed surface) is **not** a divider — hairlining it flattens the effect. Separators → `hairlineWidth`; depth edges → leave them at `1`. Dividers stay borders; don't shadow them.

### 6. Image Outlines

Add a `1px` neutral outline to **rectangular** images for consistent depth. The non-negotiable part is the **hue**: never a **saturated or palette-derived** tint (slate, zinc, your accent), which reads as dirt on the edge. `rgba(0,0,0,0.1)` / `rgba(255,255,255,0.1)` is the safe default — but on a warm/branded surface a **desaturated neutral matched to the surface temperature**, at a fitting alpha, is more cohesive than a literal cold black. Don't outline **irregular / cut-out alpha art** — a border boxes it; let its silhouette shadow (#4) carry the depth. Budget for the layout `borderWidth` adds (RN has no `outline-offset`); use an absolute overlay if the image can't grow.

### 7. Minimum Hit Area via `hitSlop`

Interactive elements need ~44×44 (iOS) / 48dp (Android). Use the `hitSlop` prop to extend the touch target **without affecting layout** — this is cleaner than the web's pseudo-element trick. Never let two controls' hit areas overlap.

### 8. Respect Safe-Area Insets

Never hardcode notch / home-indicator / status-bar padding. Read `useSafeAreaInsets()` from `react-native-safe-area-context` and apply the specific edges you need, or use `SafeAreaView` with explicit `edges`.

### 9. Interruptible Animations

Users change intent mid-gesture. Reanimated's `withTiming` / `withSpring` **retarget toward the latest value** when you reassign a shared value — that's your interruptible "transition." Use `cancelAnimation` to stop one. Reserve one-shot `withSequence` / keyframe-like chains for staged sequences that run once.

### 10. Split & Stagger Enter Animations

Don't animate one big container. Break content into semantic chunks (title, body, actions) and stagger them ~60–100ms apart, each combining `opacity` and a small `translateY`. Either `withDelay(i * stagger, withTiming(...))` or layout `entering={FadeInDown.delay(i * 80)}`.

### 11. Subtle Exit Animations

Exits should be softer than enters. Use a small fixed `translateY` (e.g. `-12`), not the full height, and a shorter duration. Reanimated `exiting={FadeOutUp}` is the `AnimatePresence` exit equivalent — keep some directional motion so the element reads as _going_ somewhere.

### 12. Contextual Icon Animations

When an icon swaps on state change (play ↔ pause, like ↔ liked), animate it instead of hard-cutting: scale `0.25 → 1` and opacity `0 → 1`. **Blur is dropped on RN** — there's no cheap per-node blur (Skia can, at a cost; treat it as optional/advanced). Keep both icons mounted and cross-fade, or use a layout `entering`/`exiting` pair.

### 13. Consistent Press Feedback

RN has no `:active`. Pick one feedback and apply it consistently: scale `0.96` (never below `0.95`), an opacity dim (~`0.6–0.7`), or a 1px `translateY` "key press." For interruptible feedback use a Reanimated handler (`withTiming(0.96)`); for a snap, `Pressable`'s `style={({ pressed }) => ...}`. Add a `static` prop to opt a button out.

### 14. Choose Spring or Timing Deliberately

Two valid motion languages. **Timing** (`withTiming` + `Easing.*`) is mechanical and deterministic — right for chrome, toggles, tabs, and choreographed staggers. **Spring** (`withSpring`) is physical and velocity-aware — right for drag-release and gesture-following content. Default chrome to **no visible overshoot** (`withTiming`, or `withSpring({ overshootClamping: true })`); be consistent within a surface. See [animations.md](animations.md) — neither house is universal.

### 15. Skip the Animation on First Mount

Elements already in their default state shouldn't animate in on launch — only on later state changes. Reanimated layout `entering` runs on mount by default, so gate it behind a "mounted" ref / flag for toggles, tabs, and segmented controls. Don't gate a deliberate hero entrance.

### 16. Animate Explicit, Compositor-Friendly Props

There's no `transition: all` in RN, but the spirit holds: animate `transform` and `opacity` (the UI thread can drive them off-bridge) and enumerate exactly the shared values you mean. **Never animate `width` / `height` / `flex` / `margin` for motion** — layout animation runs on the JS thread and drops frames.

### 17. Drive Motion on the Right Renderer

Where a frame is computed decides whether it's smooth. A `View` transform via Reanimated `useAnimatedStyle` runs on the UI thread. Skia reads shared values on the UI thread via `useDerivedValue`. **`react-native-svg` cannot** be driven by Reanimated props under the New Architecture — animate it with `requestAnimationFrame` + `setState`, or move the node to Skia. See [performance.md](performance.md).

### 18. Tabular Numbers

Any number that updates in place — timers, counters, prices, scoreboards — gets `fontVariant: ['tabular-nums']` so digits are equal width and the layout doesn't jitter. **If the number is set in a monospace font, every glyph is already equal-width — this is a no-op; skip it (and don't flag its absence).** For proportional fonts, verify the font supports tabular figures (system SF does; check custom fonts).

### 19. Load Fonts Deliberately

Custom fonts are a setup step on RN: `useFonts()` (`expo-font`) or a bare-workflow `react-native.config.js`, and gate the first render until they're loaded so there's no unstyled flash. (Font _smoothing_ — the web's `-webkit-font-smoothing` — has no RN equivalent; skip it.)

### 20. Text Wrapping Is a Known Gap

RN has **no** `text-wrap: balance` / `pretty`. There's no clean way to balance lines or kill orphans. Reach for `numberOfLines`, `adjustsFontSizeToFit`, manual line breaks, or Android-only `textBreakStrategy`. Set expectations rather than shipping a fragile JS line-balancer.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Same `borderRadius` on parent and child | Compute `outer = inner + padding` literally |
| Icons look off-center | Nudge optically (`marginLeft`) or fix the SVG/asset |
| `shadow*` with no Android `elevation` (and Android ships) | Set `elevation` too — but never on a silhouette/directional shadow; iOS-only app? ignore |
| Shadow on an opaque box, expecting a silhouette | `backgroundColor: 'transparent'` so the shadow follows the alpha |
| `borderBottomWidth: 1` on a real divider looks chunky | `StyleSheet.hairlineWidth` (but leave bevel/inset depth edges at `1`) |
| **Saturated** image outline (slate/zinc/accent) | Neutral hue; `rgba(0,0,0,0.1)` / `rgba(255,255,255,0.1)` default, surface-fit alpha on branded UIs |
| Tiny touch targets | `hitSlop` to 44×44 / 48dp (doesn't change layout) |
| Content under the notch / home indicator | `useSafeAreaInsets()` / `SafeAreaView` `edges` |
| Keyframe/one-shot for an interactive toggle | `withTiming` retarget + `cancelAnimation` (interruptible) |
| One container fades in as a block | Split into chunks, stagger ~60–100ms |
| Dramatic exit that steals focus | Small fixed `translateY`, shorter than the enter |
| Numbers jitter as they update | `fontVariant: ['tabular-nums']` (no-op on a mono font; verify other fonts) |
| Expecting `text-wrap: balance` / `pretty` | No equivalent — `numberOfLines` / `adjustsFontSizeToFit` |
| Porting `-webkit-font-smoothing` | N/A on RN — drop it; load fonts via `expo-font` |
| Animating `width` / `height` / `flex` for motion | Animate `transform` / `opacity` on the UI thread |
| Driving `react-native-svg` with Reanimated props (New Arch) | rAF + `setState`, or move the node to Skia |
| Bouncy spring on chrome / toggles | `withTiming`, or `withSpring({ overshootClamping: true })` |
| Enter animation fires on every launch | Gate `entering` until after first mount |

## Review Output Format

Present changes as markdown tables with **Before** and **After** columns. Include every change you made — not just a subset. Never list findings as loose "Before:" / "After:" lines outside a table. Group changes by principle under a heading, and keep each row to a single diff so the reader can scan the whole list. Cite the specific file and the specific prop that changed when it isn't obvious from the snippet. If a principle was reviewed but nothing needed to change, don't make an empty before/after table — instead list it under a short **"Reviewed, already correct"** line at the end, so the report shows coverage (what you checked), not just diffs.

### Example

#### Concentric border radius
| Before | After |
| --- | --- |
| `track: { borderRadius: 14, padding: 2 }` + `knob: { borderRadius: 14 }` | `knob: { borderRadius: 12 }` (14 − 2) |
| `card` and inner `tile` both `borderRadius: 16`, `padding: 8` | Outer `24`, inner `16` |

#### Shadows & elevation
| Before | After |
| --- | --- |
| iOS-only `shadowColor/Offset/Opacity/Radius` on a card | Added `elevation: 6` for Android parity |
| Opaque card expecting the art to cast a shadow | `backgroundColor: 'transparent'` → shadow follows the silhouette |

#### Hit area
| Before | After |
| --- | --- |
| Bare 24×24 icon `Pressable` | `hitSlop={{ top: 12, right: 12, bottom: 12, left: 12 }}` → 48×48 |

#### Press feedback
| Before | After |
| --- | --- |
| `scale(0.9)` on press | Raised to `0.96` — anything below `0.95` feels exaggerated |
| Three buttons, three different press effects | Standardized on opacity `0.65` across the surface |

## Review Checklist

> Run the **preflight first** (confirm platform targets, read the design tokens) — several items below are conditional on those.

- [ ] Nested rounded elements use concentric radius (`outer = inner + padding`), reconciled with any radius-token scale
- [ ] Icons are optically centered, not just geometrically
- [ ] If Android ships, every `shadow*` style also sets `elevation` — but not on silhouette/directional shadows
- [ ] Silhouette shadows use a transparent background
- [ ] Real dividers use `StyleSheet.hairlineWidth`, not `1` — bevel/inset depth edges are left alone
- [ ] Image outlines are a neutral hue (not a saturated/palette tint); alpha fits the surface
- [ ] Interactive elements reach 44×44 / 48dp via `hitSlop`
- [ ] Screens respect safe-area insets (no hardcoded notch padding)
- [ ] Interactive animations are interruptible — Reanimated retarget (`withTiming`) or `Animated` + `useNativeDriver` scroll/gesture interpolation; not one-shot keyframes
- [ ] Enter animations are split and staggered
- [ ] Exit animations are subtle (small fixed translate, shorter than enter)
- [ ] Icon swaps animate scale + opacity (blur omitted on RN)
- [ ] Press feedback is one consistent idiom per surface
- [ ] Springs clamp overshoot on chrome / toggles
- [ ] `entering` is gated so default-state elements don't animate on launch
- [ ] Motion animates `transform` / `opacity`, never layout props
- [ ] `react-native-svg` is driven via rAF, not Reanimated props
- [ ] Updating numbers use `fontVariant: ['tabular-nums']` (skip on mono fonts)

> **Verifiability.** The surface items (radius, shadows, hairlines, outlines, hit area, safe-area) are screenshot-checkable — though hairline and ~2px radius deltas are sub-pixel at typical screenshot-harness resolution, so confirm those at full device resolution. The motion items (interruptible, enter/exit, icon swaps, press feedback, springs, `entering`) are **device/interaction-verified**, not screenshot-verifiable — confirm by interacting or reading the code.

## Reference Files

- [surfaces.md](surfaces.md) — Border radius, optical alignment, shadows, hairline dividers, image outlines, hit areas
- [animations.md](animations.md) — Interruptible animations, enter/exit, icon animations, press feedback, spring vs timing
- [typography.md](typography.md) — Tabular numbers, custom font loading, the text-wrap gap
- [performance.md](performance.md) — UI-thread motion, the renderer matrix, New Architecture caveats
- [platform.md](platform.md) — iOS shadow vs Android elevation, Pressable feedback, safe-area insets
