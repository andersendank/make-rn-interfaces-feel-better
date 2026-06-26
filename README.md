# Make React Native Interfaces Feel Better

A React Native port of the [Agent Skill](https://docs.anthropic.com/en/docs/claude-code/skills) [`make-interfaces-feel-better`](https://github.com/jakubkrehel/make-interfaces-feel-better) by [Jakub Krehel](https://jakub.kr), based on his article [_Details that make interfaces feel better_](https://jakub.kr/writing/details-that-make-interfaces-feel-better).

The original is written for the web (CSS, Tailwind, Framer Motion). This port teaches the same compounding design-engineering details in **React Native** terms — `StyleSheet`, Reanimated, `Pressable`, shadows + elevation, `hitSlop`, safe-area insets, and the UI-thread/renderer model — and it is honest about which web details have **no** RN equivalent.

## What it covers

- Concentric border radius for nested elements
- Optical vs geometric alignment
- Shadows over borders — and always setting Android `elevation`
- Silhouette shadows (transparent background → the shadow follows the alpha shape)
- Crisp hairline dividers (`StyleSheet.hairlineWidth`)
- Image outlines for depth
- Minimum 44×44 / 48dp hit areas via `hitSlop`
- iOS shadow vs Android elevation parity
- `Pressable` feedback idioms (pressed state, ripple, hitSlop)
- Safe-area insets
- Interruptible animations (`withTiming` retarget, not keyframes)
- Split & stagger enter animations
- Subtle exit animations
- Contextual icon animations (scale + opacity)
- Press feedback (scale `0.96` / opacity / translate)
- Spring vs timing — choosing a motion language
- Driving motion on the right thread/renderer (Reanimated / Skia / `react-native-svg`)
- Tabular numbers for dynamic values
- Custom font loading

## What's different from the web original

| Web original | React Native port |
| --- | --- |
| `box-shadow` (layered, inset) | iOS `shadow*` props **plus** Android `elevation`; no layered/inset shadows → nest `View`s or use Skia |
| `-webkit-font-smoothing` | **Dropped** — the OS owns text rendering on RN |
| `text-wrap: balance` / `pretty` | **No equivalent** — `numberOfLines` / `adjustsFontSizeToFit` workarounds |
| `outline` (no layout cost) | `borderWidth` (costs layout) or an absolutely-positioned overlay |
| `will-change` / GPU layers | The **thread & renderer model**: Reanimated runs on the UI thread, Skia is drivable, `react-native-svg` is not (under the New Architecture) |
| `:active` / `:hover` | `Pressable` `pressed` state and `android_ripple` (there is no hover) |
| `transition: all` footgun | Animate explicit shared values; never animate layout props for motion |

## Who it's for

Any React Native developer — Expo or bare workflow. The surface, typography, and platform principles are dependency-free (`StyleSheet` + `Pressable`). The animation chapters assume [`react-native-reanimated`](https://docs.swmansion.com/react-native-reanimated/) (recommended, not required — a built-in `Animated` fallback is noted). The Skia/SVG threading notes only matter if you use [`@shopify/react-native-skia`](https://shopify.github.io/react-native-skia/) or [`react-native-svg`](https://github.com/software-mansion/react-native-svg).

**Context first.** These principles aren't context-free — `SKILL.md` opens with a short preflight: confirm your platform targets and read the design-system tokens. Much of `platform.md` is moot on an **iOS-only** app, and a **radius-token scale** or a **monospace** number font changes how several principles apply. The skill is written to respect the system that already exists rather than impose cross-platform, token-less defaults.

## Compatibility

- React Native **0.74+** (examples assume the New Architecture, default since RN 0.76 / Expo SDK 52).
- Reanimated **3 or 4**. On Reanimated 4, worklets live in the separate `react-native-worklets` package — the APIs used here (`useSharedValue`, `withTiming`, `withSpring`, `useAnimatedStyle`, `useDerivedValue`, layout animations) are stable across both.
- New-Architecture renderer notes are **version-dated** where they matter (e.g. "as of `react-native-svg` 15"). Verify against your installed version — renderer behavior changes between releases.

## Installation

```bash
npx skills add andersendank/make-rn-interfaces-feel-better
```

Or clone this repo and copy `skills/make-rn-interfaces-feel-better/` into your agent's skills directory (e.g. `~/.claude/skills/`).

## Usage

Once installed, Claude applies these principles automatically when building or reviewing React Native UI. You can also invoke it manually:

```
/make-rn-interfaces-feel-better
```

## Reference docs

| Doc | Covers |
| --- | --- |
| [`SKILL.md`](skills/make-rn-interfaces-feel-better/SKILL.md) | The principle index, common mistakes, review format & checklist |
| [`surfaces.md`](skills/make-rn-interfaces-feel-better/surfaces.md) | Radius, optical alignment, shadows, hairlines, outlines, hit areas |
| [`animations.md`](skills/make-rn-interfaces-feel-better/animations.md) | Interruptible motion, enter/exit, icon animation, press feedback, spring vs timing |
| [`typography.md`](skills/make-rn-interfaces-feel-better/typography.md) | Tabular numbers, font loading, the text-wrap gap |
| [`performance.md`](skills/make-rn-interfaces-feel-better/performance.md) | UI-thread motion, the renderer matrix, New-Architecture notes |
| [`platform.md`](skills/make-rn-interfaces-feel-better/platform.md) | iOS shadow vs Android elevation, Pressable/ripple, safe-area insets |

## Credits

Concept, structure, and the original principles: **Jakub Krehel** — [_Details that make interfaces feel better_](https://jakub.kr/writing/details-that-make-interfaces-feel-better). This repository adapts that work for React Native under the MIT license.

## License

MIT — see [LICENSE](LICENSE). Original work © Jakub Krehel; React Native adaptation © 2026 Andersen Dank.
