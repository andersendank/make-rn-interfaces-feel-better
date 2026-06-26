# Platform

The detail category the web simply doesn't have: **one codebase rendering on two native platforms.** Most "it looked right on my simulator, wrong on the other phone" bugs live here. These are the cross-platform mechanics; the design _principles_ that use them live in [surfaces.md](surfaces.md) and [animations.md](animations.md).

## iOS Shadow vs Android Elevation

iOS and Android draw depth through **two unrelated systems**. A style that sets only one is half-done.

| Concern | iOS | Android |
| --- | --- | --- |
| Shadow source | `shadowColor` + `shadowOffset` + `shadowOpacity` + `shadowRadius` | `elevation` (a single z-depth number) |
| Color control | Full (any `shadowColor`) | Limited — system-derived; `shadowColor` honored only on Android 9+ and roughly |
| Offset / direction | Yes (`shadowOffset`) | No — elevation shadows are symmetric, system-defined |
| Blur control | Yes (`shadowRadius`) | Indirect (larger `elevation` = larger, softer) |
| Shape | Follows alpha silhouette if background is transparent | Always the view's (rounded-)rect outline |
| Needs a background color | Not for silhouette shadows | Effectively yes — `elevation` traces the box |

### Set both, every time

```tsx
const styles = StyleSheet.create({
  card: {
    backgroundColor: '#1c1b18',
    borderRadius: 16,
    // iOS
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 8 },
    shadowOpacity: 0.25,
    shadowRadius: 16,
    // Android
    elevation: 6,
  },
});
```

Practical notes:

- There is **no exact mapping** between `shadowRadius`/`shadowOffset` and `elevation`. Tune `elevation` separately by eye on an Android device — a value that looks right on iOS won't translate numerically.
- `elevation` also affects **draw order / z-stacking** on Android (higher = drawn on top), occasionally with surprising results in overlapping layouts. If an Android shadow won't appear, check that the view has a non-transparent `backgroundColor` and isn't clipped by an ancestor's `overflow: 'hidden'`.
- Silhouette shadows (transparent background → shadow follows the alpha) are **iOS-only**; on Android the same view falls back to a rect shadow or none. Don't rely on a cut-out shape casting a shaped shadow cross-platform.

## Pressable Feedback Idioms

There is **no hover** on touch devices, so the web's `:hover`/`:active` vocabulary is replaced by `Pressable`'s props. Know the whole toolkit:

| Prop | What it does | Platform |
| --- | --- | --- |
| `style={({ pressed }) => …}` | Style by press state (opacity, scale, bg) | Both |
| `android_ripple={{ color, borderless, radius }}` | Native Material ripple | Android (ignored on iOS) |
| `hitSlop` | Expand the touch target without affecting layout | Both |
| `pressRetentionOffset` | Keep the press active if the finger drifts off | Both |
| `unstable_pressDelay` | Delay `onPressIn` (avoid flicker inside scrollables) | Both |
| `onPressIn` / `onPressOut` | Hook your own animated feedback | Both |

```tsx
// Platform-appropriate feedback in one component:
// iOS gets an opacity dim, Android gets a native ripple.
<Pressable
  android_ripple={{ color: 'rgba(0,0,0,0.12)', borderless: false }}
  style={({ pressed }) => [
    styles.row,
    pressed && Platform.OS === 'ios' && { opacity: 0.6 },
  ]}
  hitSlop={8}
>
  <Text>Settings</Text>
</Pressable>
```

Guidance:

- **Match platform expectations.** Android users expect a ripple from the touch point; iOS users expect an opacity/scale response. Shipping an iOS-style opacity dim on Android (with no ripple) feels subtly wrong, and vice versa.
- For a **borderless** ripple (icon buttons), set `android_ripple={{ borderless: true }}`.
- Inside `ScrollView`/`FlatList`, a small `unstable_pressDelay` (~80–120ms) prevents the press state from flashing during a scroll-start.

## Safe-Area Insets

Phones have notches, Dynamic Islands, home indicators, and status bars. **Never hardcode** the padding for them — the values differ across devices and orientations. Use [`react-native-safe-area-context`](https://github.com/AppAndFlow/react-native-safe-area-context).

```tsx
import { useSafeAreaInsets } from 'react-native-safe-area-context';

function Screen() {
  const insets = useSafeAreaInsets();
  return (
    <View style={{ paddingTop: insets.top, paddingBottom: insets.bottom, flex: 1 }}>
      {/* content clears the status bar and home indicator on any device */}
    </View>
  );
}
```

`useSafeAreaInsets()` vs `SafeAreaView`:

- **`useSafeAreaInsets()`** gives you the four numbers, so you can apply only the edges you need and **add** inset to existing padding (`paddingTop: insets.top + 16`). Prefer this for anything non-trivial.
- **`<SafeAreaView edges={['top']}>`** is the quick wrapper — but always pass explicit `edges`, or it pads all four and fights your layout (e.g. doubling a tab bar's bottom inset).

Common patterns:

- A bottom action bar: `paddingBottom: insets.bottom + 12` so it clears the home indicator but doesn't float too high.
- A custom header: `paddingTop: insets.top` on the header container, not the whole screen.
- Don't apply the same inset twice — if a navigator already insets the screen, don't re-add `insets.top` inside it.

## Pixel Grid & Hairlines

RN's layout unit (`dp`) is density-independent, but the screen is physical pixels — and thin lines or precise positions can land between pixels and render blurry.

- **Hairlines:** `StyleSheet.hairlineWidth` is exactly one physical pixel for the device's density — use it for dividers and 1px rules instead of `1` (which is `1dp` ≈ 2–3 physical px at 2×/3×). See [surfaces.md](surfaces.md#hairline-dividers).
- **Snapping to the pixel grid:** when a computed size or offset must be crisp, round it to device pixels with `PixelRatio`:

```tsx
import { PixelRatio } from 'react-native';

// round a dp value to the nearest physical pixel for a crisp edge
const crisp = (dp: number) => PixelRatio.roundToNearestPixel(dp);
```

- **Image sizing:** `PixelRatio.getPixelSizeForLayoutSize(dp)` converts a layout size to the physical pixel size — useful when requesting a remote image at the exact resolution a slot needs (avoids fetching a 3×-too-large bitmap, or a blurry too-small one).

These are small things, but they're the difference between hairlines and edges that look intentional versus faintly soft — the same "is this on the pixel grid" instinct the web skill applies, made explicit because RN gives you density-independent units by default.
