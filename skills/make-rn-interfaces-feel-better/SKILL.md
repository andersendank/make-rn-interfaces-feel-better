---
name: make-rn-interfaces-feel-better
description: Design engineering principles for making React Native interfaces feel polished. Use when building or reviewing RN UI — components, StyleSheet, Reanimated animations, Pressable feedback, shadows and elevation, border radius, optical alignment, safe-area insets, typography, micro-interactions, enter/exit and icon animations, or any visual detail work. Triggers on RN UI polish, "make it feel better", "feels off", concentric radius, hitSlop, tabular numbers, iOS shadow vs Android elevation, hairline dividers, stagger/spring animations, and the Reanimated/Skia/SVG thread model.
---

# Details that make React Native interfaces feel better

Great interfaces rarely come from a single thing. They're a collection of small details that compound into a great experience. Apply these when building or reviewing React Native UI.

React Native adds two wrinkles the web doesn't have: you are styling **two native renderers at once**, so several of these details are about making iOS and Android agree — and your animations can run on the **UI thread or the JS thread**, so a few are about keeping motion off the bridge. This skill is the [web original](https://github.com/jakubkrehel/make-interfaces-feel-better) translated honestly into RN: what maps, what changes, and what simply has no equivalent.

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

Outer radius = inner radius + padding. Compute it literally in `StyleSheet` — RN has no `rounded-xl` token to lean on. Mismatched radii on nested elements is the most common thing that makes an interface feel off.

### 2. Optical Over Geometric Alignment

When geometric centering looks off, align optically. A play triangle needs a small `marginLeft`; an icon next to text wants slightly less padding on the icon side. Fix the SVG/asset directly when you can, manual margin when you can't.

### 3. Shadows Over Borders — and Always Set `elevation`

Prefer a subtle shadow to a border for depth on cards and buttons. But an iOS-only `shadow*` style renders **flat on Android** — Android depth comes from `elevation`. Set both, every time. Keep real dividers as borders (see #5).

### 4. Silhouette Shadows

Give a card `backgroundColor: 'transparent'` and the iOS shadow follows the content's **alpha silhouette** instead of a box outline — the trick for making an irregular shape (artwork, an icon, a cut-out) cast a true shadow. Reserve `elevation` for things that should genuinely read as lifted.

### 5. Crisp Hairline Dividers

`borderBottomWidth: StyleSheet.hairlineWidth`, never `1`. `hairlineWidth` is one **physical** pixel on the device — the density-aware way to get the thin separators the web draws with `1px`. Dividers stay borders; don't shadow them.

### 6. Image Outlines

Add a `1px` neutral outline to images for consistent depth: `borderWidth: 1` at `rgba(0,0,0,0.1)` (light) or `rgba(255,255,255,0.1)` (dark). Pure black/white only — never a tinted near-black, which reads as dirt on the edge. Budget for the layout `borderWidth` adds (RN has no `outline-offset`); use an absolute overlay if the image can't grow.

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

Any number that updates in place — timers, counters, prices, scoreboards — gets `fontVariant: ['tabular-nums']` so digits are equal width and the layout doesn't jitter. Verify your font supports it (system SF does; check custom fonts).

### 19. Load Fonts Deliberately

Custom fonts are a setup step on RN: `useFonts()` (`expo-font`) or a bare-workflow `react-native.config.js`, and gate the first render until they're loaded so there's no unstyled flash. (Font _smoothing_ — the web's `-webkit-font-smoothing` — has no RN equivalent; skip it.)

### 20. Text Wrapping Is a Known Gap

RN has **no** `text-wrap: balance` / `pretty`. There's no clean way to balance lines or kill orphans. Reach for `numberOfLines`, `adjustsFontSizeToFit`, manual line breaks, or Android-only `textBreakStrategy`. Set expectations rather than shipping a fragile JS line-balancer.

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Same `borderRadius` on parent and child | Compute `outer = inner + padding` literally |
| Icons look off-center | Nudge optically (`marginLeft`) or fix the SVG/asset |
| `shadow*` props but no Android `elevation` | Set `elevation` alongside; test on Android |
| Shadow on an opaque box, expecting a silhouette | `backgroundColor: 'transparent'` so the shadow follows the alpha |
| `borderBottomWidth: 1` divider looks chunky | `StyleSheet.hairlineWidth` |
| Tinted image outline (slate/zinc/`#111`) | Pure `rgba(0,0,0,0.1)` / `rgba(255,255,255,0.1)` |
| Tiny touch targets | `hitSlop` to 44×44 / 48dp (doesn't change layout) |
| Content under the notch / home indicator | `useSafeAreaInsets()` / `SafeAreaView` `edges` |
| Keyframe/one-shot for an interactive toggle | `withTiming` retarget + `cancelAnimation` (interruptible) |
| One container fades in as a block | Split into chunks, stagger ~60–100ms |
| Dramatic exit that steals focus | Small fixed `translateY`, shorter than the enter |
| Numbers jitter as they update | `fontVariant: ['tabular-nums']` (font must support it) |
| Expecting `text-wrap: balance` / `pretty` | No equivalent — `numberOfLines` / `adjustsFontSizeToFit` |
| Porting `-webkit-font-smoothing` | N/A on RN — drop it; load fonts via `expo-font` |
| Animating `width` / `height` / `flex` for motion | Animate `transform` / `opacity` on the UI thread |
| Driving `react-native-svg` with Reanimated props (New Arch) | rAF + `setState`, or move the node to Skia |
| Bouncy spring on chrome / toggles | `withTiming`, or `withSpring({ overshootClamping: true })` |
| Enter animation fires on every launch | Gate `entering` until after first mount |

## Review Output Format

Present changes as markdown tables with **Before** and **After** columns. Include every change you made — not just a subset. Never list findings as loose "Before:" / "After:" lines outside a table. Group changes by principle under a heading, and keep each row to a single diff so the reader can scan the whole list. Cite the specific file and the specific prop that changed when it isn't obvious from the snippet. If a principle was reviewed but nothing needed to change, omit that table — empty tables add noise.

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

- [ ] Nested rounded elements use concentric radius (`outer = inner + padding`)
- [ ] Icons are optically centered, not just geometrically
- [ ] Every `shadow*` style also sets Android `elevation`
- [ ] Silhouette shadows use a transparent background
- [ ] Dividers use `StyleSheet.hairlineWidth`, not `1`
- [ ] Image outlines are neutral `rgba(0,0,0,0.1)` / `rgba(255,255,255,0.1)`
- [ ] Interactive elements reach 44×44 / 48dp via `hitSlop`
- [ ] Screens respect safe-area insets (no hardcoded notch padding)
- [ ] Interactive animations retarget (`withTiming`), not keyframes
- [ ] Enter animations are split and staggered
- [ ] Exit animations are subtle (small fixed translate, shorter than enter)
- [ ] Icon swaps animate scale + opacity (blur omitted on RN)
- [ ] Press feedback is one consistent idiom per surface
- [ ] Springs clamp overshoot on chrome / toggles
- [ ] `entering` is gated so default-state elements don't animate on launch
- [ ] Motion animates `transform` / `opacity`, never layout props
- [ ] `react-native-svg` is driven via rAF, not Reanimated props
- [ ] Updating numbers use `fontVariant: ['tabular-nums']`

## Reference Files

- [surfaces.md](surfaces.md) — Border radius, optical alignment, shadows, hairline dividers, image outlines, hit areas
- [animations.md](animations.md) — Interruptible animations, enter/exit, icon animations, press feedback, spring vs timing
- [typography.md](typography.md) — Tabular numbers, custom font loading, the text-wrap gap
- [performance.md](performance.md) — UI-thread motion, the renderer matrix, New Architecture caveats
- [platform.md](platform.md) — iOS shadow vs Android elevation, Pressable feedback, safe-area insets
