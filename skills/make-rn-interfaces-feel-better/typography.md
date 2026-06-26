# Typography

Numeric rendering, font loading, and an honest accounting of the web typography details that don't exist on React Native.

## Tabular Numbers

When numbers update in place — timers, counters, prices, scoreboards, table columns — give them equal-width digits so the layout doesn't jitter as values change. RN exposes this as `fontVariant`:

```tsx
const styles = StyleSheet.create({
  timer: {
    fontVariant: ['tabular-nums'],
    fontFamily: 'Menlo', // any font whose tabular figures you've verified
    fontSize: 17,
  },
});

// A countdown that won't shift width as digits change
<Text style={styles.timer}>{mmss(remainingMs)}</Text>;

function mmss(ms: number) {
  const total = Math.max(0, Math.floor(ms / 1000));
  const m = Math.floor(total / 60);
  const s = total % 60;
  return `${m}:${String(s).padStart(2, '0')}`;
}
```

### When to use

| Use tabular-nums | Don't |
| --- | --- |
| Counters and timers | Static display numbers |
| Prices that update | Decorative large numerals |
| Table / list columns of numbers | Phone numbers, zip codes |
| Animated number transitions | Version strings (v2.1.0) |
| Scoreboards, live dashboards | |

### Caveats

- **Font support varies.** The system font (San Francisco on iOS, Roboto on Android) supports tabular figures. A custom font may not — verify before relying on it, or the prop silently does nothing.
- **`fontVariant` is an array** and accepts more than tabular figures (`'oldstyle-nums'`, `'lining-nums'`, `'small-caps'`, etc.), but support is font-dependent.
- Some fonts visibly change the `1` glyph (wider/centered) under tabular figures. That's expected and usually desirable for alignment — just confirm it looks right.

## Custom Font Loading

Custom fonts are a **setup step** on RN, not a CSS `@font-face` drop-in — and getting it wrong shows up as an unstyled-text flash on launch. There are two paths:

### Expo

```tsx
import { useFonts } from 'expo-font';

export default function App() {
  const [loaded] = useFonts({
    'Inter-Regular': require('./assets/fonts/Inter-Regular.ttf'),
    'Inter-Medium': require('./assets/fonts/Inter-Medium.ttf'),
  });
  if (!loaded) return null; // gate the first render — no flash of unstyled text
  return <RootNavigator />;
}
```

(With `@expo-google-fonts/*` packages you import the font and `useFonts({ Inter_400Regular })` directly.)

### Bare React Native

Drop the font files into the native projects and register them via `react-native.config.js` + `npx react-native-asset` (links fonts into iOS `Info.plist` and Android `assets/fonts`). Then reference the font by its PostScript name in `fontFamily`.

**Either way:** hold the first render until fonts are ready, and centralize `fontFamily` constants so a font swap is one edit, not a find-and-replace.

## Font Smoothing — N/A on React Native

The web applies `-webkit-font-smoothing: antialiased` to crisp up text on macOS. **There is no React Native equivalent** — text rendering is owned by the OS (Core Text on iOS, the Android text stack), and you don't get a knob for it. Don't look for one; don't port it. This slot in the web skill is simply replaced by "load your fonts deliberately" above.

## The Text-Wrap Gap

This is the biggest honest gap from the web original. React Native has **no** `text-wrap: balance` and **no** `text-wrap: pretty`. You cannot evenly balance heading lines, and you cannot automatically prevent a one-word orphan on the last line. Don't ship a fragile JS line-balancer to fake it — set expectations and use the tools RN does give you.

| You want… | Web | React Native |
| --- | --- | --- |
| Even heading lines | `text-wrap: balance` | **No equivalent** — control width or insert a manual `\n` for known strings |
| No last-line orphans | `text-wrap: pretty` | **No equivalent** — accept it, or constrain width |
| Hard line cap | `line-clamp` | `numberOfLines={2}` (with `ellipsizeMode`) |
| Shrink-to-fit one line | (various) | `adjustsFontSizeToFit` + `numberOfLines={1}` |
| Better hyphenation/breaks | browser default | Android only: `textBreakStrategy="balanced"` |

```tsx
// Cap lines and truncate cleanly
<Text numberOfLines={2} ellipsizeMode="tail">{title}</Text>

// Fit a single line into a fixed width (badges, chips, stat tiles)
<Text numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.8}>{label}</Text>

// Android-only: nudges line breaks toward even lengths (no iOS effect)
<Text textBreakStrategy="balanced">{paragraph}</Text>
```

For a small set of known, important strings (an onboarding headline, a paywall title), a hand-placed `\n` at the intended break is a legitimate, low-tech "balance." For dynamic/user content, constrain the container width and accept the browser-grade niceties simply aren't there.

> **Accessibility aside:** RN `Text` scales with the OS font-size setting by default (`allowFontScaling`). Leave it on for body text. Only set `allowFontScaling={false}` for chrome that genuinely must not grow (a fixed-width numeric badge), and even then prefer `maxFontSizeMultiplier` to cap rather than disable scaling.
