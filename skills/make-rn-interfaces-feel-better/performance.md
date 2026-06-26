# Performance

The web skill's performance chapter is about `transition: all` and `will-change` — browser-compositor hints. React Native's equivalent concern is different and more consequential: **which thread and which renderer computes each frame.** Get that wrong and motion stutters no matter how clean the styles are.

## Animate Explicit, Compositor-Friendly Props

There's no `transition: all` footgun in RN (no transition shorthand exists), but the underlying rule is even more important here:

- **Animate `transform` and `opacity`.** Reanimated can drive these on the UI thread without touching layout.
- **Never animate `width`, `height`, `flex`, `margin`, `padding`, or `top`/`left` for motion.** These trigger layout (yoga) on the JS side every frame — the classic source of RN jank.

```tsx
// Good — transform/opacity, UI-thread, smooth
const style = useAnimatedStyle(() => ({
  opacity: opacity.value,
  transform: [{ translateY: y.value }, { scale: scale.value }],
}));

// Bad — animating layout; re-runs layout every frame, drops frames under load
const style = useAnimatedStyle(() => ({ height: h.value, marginTop: m.value }));
```

Need a size change to _feel_ animated? Animate `scaleX`/`scaleY` (a transform) and set the final layout once, rather than tweening `width`/`height`. And enumerate exactly the shared values you mean in `useAnimatedStyle` — don't bundle unrelated properties into one animated style.

## Drive Motion on the UI Thread

React Native runs your JS on one thread and the UI on another. An animation driven by JS `setState` (or `Animated` without the native driver) computes each frame in JS and ships it across the bridge — so it stutters whenever JS is busy (a list re-render, a network callback, image decoding).

Reanimated solves this by running animation **worklets on the UI thread**: `useAnimatedStyle` and `useDerivedValue` read shared values and produce frames without a JS round-trip.

```tsx
// UI-thread: this keeps animating smoothly even while JS is busy
const style = useAnimatedStyle(() => ({ transform: [{ rotate: `${angle.value}deg` }] }));
```

Rules of thumb:

- Prefer Reanimated shared values + `useAnimatedStyle` over `Animated` with JS-driven values.
- If you must use the legacy `Animated` API, set `useNativeDriver: true` (works for `transform`/`opacity`, not layout props).
- Keep per-frame work inside worklets; marshal to JS only at discrete moments (`runOnJS` on gesture end, not every frame).

## The Renderer Matrix

"Drive it on the UI thread" has a catch: **not every renderer can be driven from a worklet.** This is the RN-specific fact with no web parallel — and it changes how you animate depending on what you're animating.

| Rendering layer | Reanimated-drivable? | How to animate |
| --- | --- | --- |
| `View` / core components | ✅ Yes | `useAnimatedStyle` (transform/opacity on the UI thread) |
| `@shopify/react-native-skia` | ✅ Yes | Pass a `useDerivedValue` to a node's `transform`/props — read on the UI thread, zero JS re-renders per frame |
| `react-native-svg` | ⚠️ **No (under the New Architecture)** | `requestAnimationFrame` + `setState`, or move the node into Skia |

### Why `react-native-svg` is the odd one out

Under the New Architecture, `react-native-svg` nodes read their transform matrix from the **React commit**, not from Reanimated's worklet writes — so assigning a shared value to an SVG node's transform never reaches the render. The reliable ways to animate SVG are:

```tsx
// Option A — JS-thread rAF + state (works, but runs on the JS thread)
function SpinningSvg() {
  const [deg, setDeg] = useState(0);
  useEffect(() => {
    let raf: number;
    const tick = () => { setDeg(d => (d + 4) % 360); raf = requestAnimationFrame(tick); };
    raf = requestAnimationFrame(tick);
    return () => cancelAnimationFrame(raf);
  }, []);
  return <Svg><G rotation={deg} origin="50,50">{/* … */}</G></Svg>;
}

// Option B — render the animated piece in Skia instead, and drive it with useDerivedValue
const transform = useDerivedValue(() => [{ rotate: angle.value }]);
// <Group transform={transform}> … </Group>
```

Use Option A for occasional/low-frequency SVG motion; move to Skia (Option B) when you need genuinely smooth, continuous, or gesture-driven SVG-like animation.

## New Architecture Caveats

Renderer behavior is **version-specific** — write it down and date it, or the skill ages into being wrong.

- The "`react-native-svg` is not Reanimated-drivable" fact is true **as of `react-native-svg` 15 under the New Architecture (Fabric)**. Newer versions may change this — verify against your installed version before assuming.
- The New Architecture is the default since **RN 0.76 / Expo SDK 52**. On older RN with the legacy architecture, some of these renderer constraints differ.
- When in doubt, write a 10-line spike: assign a Reanimated shared value to the node's transform and see if it moves. If it doesn't, you're in rAF-or-Skia territory.

The durable takeaway, independent of versions: **decide where each animated frame is computed — UI thread vs JS thread, and which renderer — before you pick the API.** That single decision is the RN equivalent of the web's compositor-layer thinking, and it's the difference between 60fps and visible stutter.
