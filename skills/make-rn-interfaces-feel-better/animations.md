# Animations

Interruptible motion, enter/exit transitions, contextual icon animations, press feedback, and choosing between spring and timing — in React Native, primarily with [Reanimated](https://docs.swmansion.com/react-native-reanimated/).

> **Dependency note.** Check the project's `package.json`. If `react-native-reanimated` is present, use it (every example below). If not, the built-in `Animated` API plus `Pressable`'s `style`/state callbacks cover most of this — the patterns translate, the ergonomics are just clunkier. Don't add Reanimated solely for a single press effect; do reach for it for anything interruptible or gesture-driven.

## Interruptible Animations

Users change intent mid-interaction. If an animation can't be interrupted, the interface feels broken — a half-open drawer that snaps shut, a toggle that ignores a fast double-tap.

### Reanimated retargets; keyframes don't

| | `withTiming` / `withSpring` (retargeting) | `withSequence` / one-shot chains |
| --- | --- | --- |
| **Behavior** | Animate toward the latest value you assign | Run a fixed, staged timeline |
| **Interruptible** | Yes — reassign the shared value and it redirects | No — restarts or finishes the sequence |
| **Use for** | Interactive state (open/close, toggle, press) | One-time sequences (a drop-then-settle, a reveal) |

```tsx
import Animated, {
  useSharedValue, useAnimatedStyle, withTiming, cancelAnimation, Easing,
} from 'react-native-reanimated';

function Drawer({ open }: { open: boolean }) {
  const x = useSharedValue(open ? 0 : -300);

  // Reassigning toward the latest target mid-flight smoothly reverses — no jank.
  useEffect(() => {
    x.value = withTiming(open ? 0 : -300, {
      duration: 200, easing: Easing.out(Easing.cubic),
    });
  }, [open]);

  const style = useAnimatedStyle(() => ({ transform: [{ translateX: x.value }] }));
  return <Animated.View style={[styles.drawer, style]}>{/* … */}</Animated.View>;
}
```

`cancelAnimation(x)` stops an in-flight animation where it is (e.g. when a gesture takes over). **Rule:** retargeting animations for interactive state; one-shot sequences only for staged motion that runs once.

## Enter Animations: Split and Stagger

Don't animate one large container. Break content into semantic chunks and animate each individually, slightly staggered.

1. **Split** into logical groups (title, description, actions).
2. **Stagger** ~60–100ms between groups.
3. **Combine** `opacity` and a small `translateY` (~8–12px). Skip blur — see [the blur gap](#contextual-icon-animations).

### With Reanimated layout animations (least code)

```tsx
import Animated, { FadeInDown } from 'react-native-reanimated';

function Header() {
  return (
    <>
      <Animated.Text entering={FadeInDown.duration(400)}>Welcome</Animated.Text>
      <Animated.Text entering={FadeInDown.delay(80).duration(400)}>
        A description of the screen.
      </Animated.Text>
      <Animated.View entering={FadeInDown.delay(160).duration(400)}>
        <Button>Get started</Button>
      </Animated.View>
    </>
  );
}
```

### Driving the stagger yourself (full control)

When you need precise timing or to key the stagger to something dynamic, drive shared values with `withDelay`:

```tsx
import { withDelay, withTiming, Easing } from 'react-native-reanimated';

const STAGGER = 80;
items.forEach((_, i) => {
  opacity[i].value = withDelay(i * STAGGER, withTiming(1, { duration: 400 }));
  translateY[i].value = withDelay(i * STAGGER,
    withTiming(0, { duration: 400, easing: Easing.out(Easing.cubic) }));
});
```

> **Dynamic lists:** you can't call a hook per item in a variable-length list. To stagger N-of-up-to-M rows, render a fixed pool of M animated rows (each calling its hooks unconditionally) and gate which are active by count — or just use layout `entering` with a per-index `.delay()`, which sidesteps the hook problem entirely.

## Exit Animations

Exits should be softer and shorter than enters — the user's focus is already moving on. Use a **small fixed `translateY`**, not the full height, and keep some directional motion so the element reads as _going_ somewhere.

```tsx
import Animated, { FadeOutUp } from 'react-native-reanimated';

// Subtle exit (recommended) — a short rise + fade
<Animated.View exiting={FadeOutUp.duration(150)}>{content}</Animated.View>
```

For a custom subtle exit with explicit values:

```tsx
import { FadeOut } from 'react-native-reanimated';
// translate a fixed -12, fade out, faster than the enter
<Animated.View exiting={FadeOut.duration(150).withInitialValues({ /* … */ })}>
```

**Key points**

- Small fixed `translateY` (≈ `-12`), not the container height.
- Exit duration shorter than enter (≈150ms vs ≈300–400ms).
- Don't drop exits entirely — an element that just vanishes loses spatial context.
- Use a **full** slide-out only when spatial continuity matters (a row returning to a list, a sheet dismissing).

> Reanimated layout `exiting` is the closest RN equivalent to the web's `AnimatePresence` exit. The component must be an `Animated.*` view and stay mounted long enough for the exit to play (layout animations handle the unmount timing for you).

## Contextual Icon Animations

When an icon swaps on a state change — play ↔ pause, like ↔ liked, check appearing — animate it instead of hard-cutting.

- **scale:** `0.25 → 1`
- **opacity:** `0 → 1`
- **blur:** **dropped on RN.** There is no cheap per-node blur. (`@shopify/react-native-skia` can blur, but it's a heavier, Skia-only path — treat it as optional/advanced, not the default.)

The robust pattern keeps **both** icons mounted (one absolutely positioned over the other) and cross-fades them, so both the entering and exiting icon animate without unmount timing games:

```tsx
import Animated, { useAnimatedStyle, withTiming, Easing } from 'react-native-reanimated';

// Call useAnimatedStyle once per icon at the top level — never inside a helper
// or loop (that would break the rules of hooks).
function IconSwap({ active }: { active: boolean }) {
  const pauseStyle = useAnimatedStyle(() => ({
    opacity: withTiming(active ? 1 : 0, { duration: 200 }),
    transform: [{ scale: withTiming(active ? 1 : 0.25, {
      duration: 200, easing: Easing.out(Easing.cubic) }) }],
  }));
  const playStyle = useAnimatedStyle(() => ({
    opacity: withTiming(active ? 0 : 1, { duration: 200 }),
    transform: [{ scale: withTiming(active ? 0.25 : 1, {
      duration: 200, easing: Easing.out(Easing.cubic) }) }],
  }));

  return (
    <View>
      <Animated.View style={[StyleSheet.absoluteFill, pauseStyle]}><PauseIcon /></Animated.View>
      <Animated.View style={playStyle}><PlayIcon /></Animated.View>
    </View>
  );
}
```

The non-absolute icon defines the layout size; the absolute one overlays it without affecting flow.

### When to animate icons

| Animate | Don't animate |
| --- | --- |
| State-change icons (play→pause, like→liked) | Static navigation icons |
| Icons that appear on interaction | Decorative icons |
| Loading / success indicators | Always-visible icons |
| | Icon labels (the text beside an icon) |

## Press Feedback

RN has no `:active` pseudo-class. You build press feedback explicitly — and the important thing is to **pick one idiom and use it consistently** across a surface. Three good options:

| Idiom | Feel | How |
| --- | --- | --- |
| **Scale `0.96`** | Tactile "push" | Reanimated `withTiming(0.96)` on `onPressIn` |
| **Opacity dim (~0.6–0.7)** | Quiet, flat-design | `Pressable` `style={({ pressed }) => ...}` |
| **Translate `1px`** | Physical "key press" | `translateY: 1` (often + a touch less shadow) |

Always use `0.96` for scale — never below `0.95`, which feels exaggerated.

### Interruptible scale (Reanimated)

```tsx
function PressableCard({ children, onPress, static: isStatic }) {
  const scale = useSharedValue(1);
  const style = useAnimatedStyle(() => ({ transform: [{ scale: scale.value }] }));
  return (
    <Pressable
      onPress={onPress}
      onPressIn={() => { if (!isStatic) scale.value = withTiming(0.96, { duration: 120 }); }}
      onPressOut={() => { scale.value = withTiming(1, { duration: 120 }); }}
    >
      <Animated.View style={style}>{children}</Animated.View>
    </Pressable>
  );
}
```

This is interruptible — release mid-press and it smoothly returns. Pass a `static` prop to opt buttons out where the motion would distract (a destructive confirm, a dense list).

### Snap opacity (no Reanimated)

```tsx
<Pressable style={({ pressed }) => [styles.button, pressed && { opacity: 0.65 }]}>
  <Text>Continue</Text>
</Pressable>
```

`Pressable`'s `style` callback is the dependency-free path. It snaps rather than eases, which is perfectly fine for a flat/quiet design language. (On Android you'll often pair or replace this with `android_ripple` — see [platform.md](platform.md).)

## Spring vs Timing

The central motion decision in React Native. Two coherent, equally valid houses — choose by what's moving, and stay consistent within a surface. Neither is universal.

| | Timing / cubic | Spring |
| --- | --- | --- |
| **Feel** | Mechanical, deterministic — "chrome snaps" | Physical, velocity-aware — "objects settle" |
| **API** | `withTiming(to, { duration, easing })` | `withSpring(to, { damping, stiffness, mass })` |
| **Interruptible** | Yes (retargets to new value) | Yes (carries current velocity) |
| **Deterministic duration** | Yes | No (settles when it settles) |
| **Best for** | Toggles, tabs, chrome, choreographed staggers | Drag-release, gesture-following, content cards |
| **No-overshoot variant** | inherent | `withSpring(to, { overshootClamping: true })` |

```tsx
// Timing — a tab indicator that snaps cleanly into place
indicatorX.value = withTiming(targetX, { duration: 200, easing: Easing.out(Easing.cubic) });

// Spring — a card that follows the finger and settles with momentum on release
cardY.value = withSpring(restY, { damping: 18, stiffness: 180 });
```

**The neutral rule:** match the motion to the element. Chrome and state-toggles want **no visible overshoot** — reach for `withTiming`, or a clamped spring. Gesture-following physical objects want a **spring**, so the motion inherits the finger's velocity on release.

### One tasteful overshoot without a spring

A timing house isn't limited to flat deceleration. `Easing.back` adds exactly one small controlled overshoot — useful for a "landing" or "settle" moment — while staying deterministic and spring-free:

```tsx
// A control that seats into place with a single, restrained rebound
scaleY.value = withSequence(
  withTiming(0.86, { duration: 70, easing: Easing.linear }),      // compress on contact
  withTiming(1,    { duration: 150, easing: Easing.out(Easing.back(1.3)) }), // rebound, one overshoot
);
```

> A codebase that uses **only** timing and zero springs is a disciplined, legitimate house — not a mistake, and not a universal law either. Likewise a spring-first codebase is fine if it's consistent. The failure mode isn't choosing one; it's mixing bouncy springs into chrome that should snap, or flat-timing content that should feel physical.

## Skip the Animation on First Mount

Elements already in their default state shouldn't animate in on launch — only on subsequent state changes. This matters for icon swaps, toggles, tabs, and segmented controls, which have a resting state the moment the screen mounts.

Reanimated layout `entering` animations **run on mount by default** (there's no `AnimatePresence initial={false}`). Gate them behind a "mounted" flag so the default state appears instantly and only later transitions animate:

```tsx
function Segmented({ value }) {
  const mounted = useRef(false);
  useEffect(() => { mounted.current = true; }, []);
  // Only animate the indicator after the first render settles.
  // (Drive an indicator shared value with withTiming on `value` change,
  //  and on the very first layout set it without animation.)
}
```

**When _not_ to gate:** a deliberate hero entrance or a loading→content reveal _relies_ on animating in on mount. Removing that skips the entrance entirely. Decide per element: resting chrome = skip on mount; intentional entrances = let them play. Verify on a full reload before shipping.
