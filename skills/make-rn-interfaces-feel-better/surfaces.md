# Surfaces

Border radius, optical alignment, shadows, hairline dividers, image outlines, and hit areas — in React Native `StyleSheet`.

## Concentric Border Radius

When nesting rounded elements, the outer radius must equal the inner radius plus the padding between them:

```
outerRadius = innerRadius + padding
```

RN has no radius tokens (`rounded-xl`) to lean on, so you always compute this by hand. If the padding is larger than ~24px, treat the layers as separate surfaces and choose each radius independently instead of forcing strict concentric math.

```tsx
// Good — concentric radii
const styles = StyleSheet.create({
  card: { borderRadius: 20, padding: 8 }, // 12 + 8
  cardInner: { borderRadius: 12 },
});

// Bad — same radius on both (the inner corner looks pinched)
const styles = StyleSheet.create({
  card: { borderRadius: 12, padding: 8 },
  cardInner: { borderRadius: 12 },
});
```

A real-world place this bites: a toggle/switch, where the knob sits inside the track with a few px of padding.

```tsx
// Good — knob radius accounts for the track padding
const styles = StyleSheet.create({
  track: { borderRadius: 14, padding: 2 }, // pill track
  knob: { borderRadius: 12 }, // 14 − 2 ✓ — concentric with the track
});
```

Mismatched radii on nested elements is one of the most common things that makes an interface feel off. Always calculate concentrically.

> **Per-corner control:** RN also supports `borderTopLeftRadius`, `borderTopRightRadius`, etc. for asymmetric surfaces (e.g. a bottom sheet rounded only on top). The concentric rule still applies per corner.

## Optical Alignment

When geometric centering looks off, align optically instead. There's no CSS `gap`-with-pixel-nudge sugar — you manage `margin` / `padding` / `gap` explicitly.

### Buttons with Text + Icon

Use slightly less padding on the icon side so the button reads balanced. A reliable rule of thumb is `icon-side padding = text-side padding − 2`.

```tsx
// Good — less padding on the icon side
const styles = StyleSheet.create({
  button: {
    flexDirection: 'row',
    alignItems: 'center',
    columnGap: 8,
    paddingLeft: 16,
    paddingRight: 14, // icon side = text side − 2
  },
});

// Bad — equal padding makes the icon look pushed too far out
button: { paddingHorizontal: 16 }
```

### Play Triangles

A play icon is triangular, so its geometric center is not its visual center. Shift it slightly toward the point:

```tsx
// Good — optically centered inside a round play button
playIcon: { marginLeft: 3 } // nudge right to account for the triangle

// Bad — geometrically centered but visually left-heavy
playIcon: {}
```

### Asymmetric Icons (Stars, Carets, Arrows)

The best fix is to adjust the SVG/asset directly (viewBox or path) so no component-level margin is needed. Fall back to a 1px margin only when you can't touch the asset.

## Shadows Instead of Borders

For **buttons, cards, and containers** that use a border for depth, prefer a subtle shadow. Shadows adapt to any background because they use transparency; solid borders don't — a border color tuned for one background reads wrong on another.

**Do not** apply this to dividers or any border whose job is layout separation (see [Hairline Dividers](#hairline-dividers)). Those stay borders.

### The RN shadow model (read this once)

RN shadows are **two separate systems**:

- **iOS** reads `shadowColor`, `shadowOffset`, `shadowOpacity`, `shadowRadius`.
- **Android** ignores all of those and draws a shadow only from `elevation` (a single z-depth number with limited color/offset control).

So a shadow that exists on both platforms must set **both**. (Full parity details live in [platform.md](platform.md); the design recipe is here.)

```tsx
// Good — a card that has depth on iOS AND Android
const styles = StyleSheet.create({
  card: {
    backgroundColor: '#1c1b18',
    borderRadius: 16,
    // iOS
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 14 },
    shadowOpacity: 0.28,
    shadowRadius: 22,
    // Android
    elevation: 8,
  },
});

// Bad — looks lifted on iOS, completely flat on Android
card: {
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 14 },
  shadowOpacity: 0.28,
  shadowRadius: 22,
}
```

### No layered or inset shadows

The web stacks several `box-shadow` values (a 1px ring + a soft lift + ambient depth). RN allows **one** shadow per `View` and has **no inset shadow**. To approximate a layered shadow, nest `View`s — an outer view for the ambient shadow, an inner for the tight ring — or render the shadow in Skia/SVG where you have full control.

```tsx
// Approximating a two-layer shadow with nested Views
<View style={styles.ambient}>      {/* big, soft, low opacity */}
  <View style={styles.ring}>       {/* tight, near-1px, slightly stronger */}
    {children}
  </View>
</View>
```

### Silhouette shadows

Set `backgroundColor: 'transparent'` on the shadow-casting `View` and the iOS shadow follows the content's **alpha silhouette** instead of the bounding box. This is how an irregular shape — artwork, a cut-out, an icon with transparency — casts a true-to-shape shadow rather than a rectangle.

```tsx
// The shadow traces the PNG/Skia content's edges, not a box
const styles = StyleSheet.create({
  artShadow: {
    backgroundColor: 'transparent', // <- the whole trick
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 16 },
    shadowOpacity: 0.3,
    shadowRadius: 24,
  },
});
```

Caveats: silhouette shadows are an **iOS** behavior (Android `elevation` always traces the view's rounded-rect outline); and an opaque `backgroundColor` defeats it, so apply the shadow to a transparent wrapper, not the filled surface.

### When to use shadows vs. borders

| Use shadows | Use borders |
| --- | --- |
| Cards, sheets, popovers with depth | Dividers between list rows |
| Floating / elevated controls | Input outlines (for affordance/accessibility) |
| Irregular shapes casting real shadows | Hairline separators in dense UI |
| Elements over varied backgrounds | Table-like cell boundaries |

## Hairline Dividers

For thin separators, use `StyleSheet.hairlineWidth`, not `1`. It resolves to a single **physical** pixel for the device's density — so the line is crisp on every screen instead of looking chunky on 2×/3× displays.

```tsx
// Good — one device pixel, crisp everywhere
row: {
  borderBottomWidth: StyleSheet.hairlineWidth,
  borderBottomColor: 'rgba(255,255,255,0.08)',
}

// Bad — 1dp reads as ~2–3 physical px on high-density screens
row: { borderBottomWidth: 1, borderBottomColor: '#333' }
```

Keep dividers as **borders**, not shadows — a divider's job is separation, not depth. Match the color to a low-opacity neutral so it disappears into the surface.

## Image Outlines

Add a subtle `1px` outline to images for consistent depth, especially in a system where other elements use shadows or borders.

### Color rules (non-negotiable)

- **Light mode:** pure black — `rgba(0, 0, 0, 0.1)`.
- **Dark mode:** pure white — `rgba(255, 255, 255, 0.1)`.
- Never a near-black/near-white from your palette (slate-900, zinc-900, `#111827`, `#f5f5f7`). A tinted outline picks up the surface color and reads as dirt on the image edge.
- Never the accent or ink color. The outline is a neutral separator, not a themed element.

### The RN catch: no `outline`

The web uses `outline` with `outline-offset: -1px` so the ring costs **no layout**. RN has no `outline` — only `borderWidth`, which **does** add to the box. Two options:

```tsx
// Option A — borderWidth, and account for the 1px it adds to the size
<Image source={src} style={{ width: 120, height: 120, borderRadius: 8,
  borderWidth: 1, borderColor: 'rgba(255,255,255,0.1)' }} />

// Option B — an absolutely-positioned overlay ring (image keeps its exact size)
<View>
  <Image source={src} style={{ width: 120, height: 120, borderRadius: 8 }} />
  <View pointerEvents="none" style={[StyleSheet.absoluteFill, {
    borderRadius: 8, borderWidth: 1, borderColor: 'rgba(255,255,255,0.1)' }]} />
</View>
```

Use Option B when the image must stay a fixed size (e.g. a grid cell) and you can't absorb the extra pixel.

## Minimum Hit Area

Interactive elements should have a hit area of at least 44×44 (iOS HIG) / 48dp (Android Material). If the visible element is smaller (a 20×20 icon, a 24×24 close button), extend the touch target with `hitSlop` — which, unlike padding, **does not affect layout**.

```tsx
// Good — a 24×24 icon button with a 48×48 effective touch target
<Pressable
  hitSlop={{ top: 12, right: 12, bottom: 12, left: 12 }}
  onPress={onPress}
>
  <CloseIcon width={24} height={24} />
</Pressable>

// hitSlop also accepts a single number for symmetric expansion
<Pressable hitSlop={12} onPress={onPress}>{/* … */}</Pressable>
```

`pressRetentionOffset` is the companion prop: it keeps the press active if the finger drifts slightly off the element before release (useful for small controls in scrollviews).

### Collision rule

If an extended hit area would overlap an adjacent control, shrink it — but keep it as large as it can be without colliding. **Two interactive elements must never have overlapping hit areas**, or taps land on the wrong one.
