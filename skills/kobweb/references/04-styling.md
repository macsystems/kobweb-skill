# Styling: Modifier, CssStyle, Silk theming

## Modifier

`Modifier` is a type-safe CSS builder. It is immutable, chainable, freely reusable, and
composable with `.then(other)`.

```kotlin
Box(Modifier.backgroundColor(Colors.Red).padding(20.px).borderRadius(5.px))
```

**Modifier order does not matter.** Under the hood a modifier chain just sets HTML style
properties and attributes; there is no wrapping semantics as in Compose Multiplatform.
Do not carry Compose Multiplatform ordering advice into a Kobweb review.

Helpers:

| API | Purpose |
| --- | --- |
| `Modifier.thenIf(cond) { … }` / `thenUnless` | Conditional chaining; `holdsIn` contracts let the compiler smart-cast inside the block (0.24.0+). |
| `modifier.toAttrs()` | Convert to an `AttrsScope` lambda so a raw Compose HTML element can consume it. |
| `Modifier.attrsModifier { … }` | Escape hatch to raw attributes. |
| `Modifier.styleModifier { … }` | Escape hatch to raw styles. |
| `property("some-css-prop", value)` / `attr(...)` | Lowest-level escape hatch for properties Kobweb has not wrapped yet. |

The CSS surface is deep and still growing (epic #200). Search for a typed API before
falling back to a string.

## Inline vs stylesheet

An inline `Modifier` sets the element's `style` attribute. A `CssStyle` emits a real
stylesheet rule with a generated class name. **Inline wins over the stylesheet** on
conflict — so a widget's `modifier` parameter always beats its style, which is the
intended override path.

Use a stylesheet whenever you need something inline styles cannot express: pseudo-classes
(`:hover`), pseudo-elements (`::before`), and media queries (breakpoints).

## `CssStyle`

```kotlin
val CustomStyle = CssStyle {
    base { Modifier.color(Colors.Red) }
    hover { Modifier.color(Colors.Green) }
    Breakpoint.MD { Modifier.fontSize(2.cssRem) }
}

// Shorthand when there is only a base block:
val SimpleStyle = CssStyle.base { Modifier.background(Colors.Red) }
```

Apply with `.toModifier()`:

```kotlin
Box(CustomStyle.toModifier().then(Modifier.margin(1.cssRem)))
```

For a raw Compose HTML element: `Div(attrs = CustomStyle.toAttrs())`.

### Declaration rules (enforced by KSP)

1. **Public, top-level `val`** — or inside an `object` / companion object. A style
   declared inside a function or a composable is **silently not registered**; KSP logs a
   *warning*, not an error, and the style simply never applies. The same holds for
   `CssStyleVariant` and `Keyframes`.
   - To keep a non-public/local one, register it manually in an `@InitSilk` block —
     `ctx.theme.registerStyle(name, style)`, `ctx.theme.registerVariant(name, variant)`,
     `ctx.stylesheet.registerKeyframes(name, build)` — and suppress the warning with
     `@Suppress("LOCAL_CSS_STYLE")` / `@Suppress("PRIVATE_CSS_STYLE")` (or the
     `CSS_STYLE_VARIANT` / `KEYFRAMES` equivalents).
   - A property whose name starts with `_` is treated as a backing property and skipped
     without a warning — the idiom for `private val _XXL` + a public extension getter.
2. **A `ComponentKind` and its `CssStyle<K>` must be in the same file** — hard error.
3. **One style per `ComponentKind`** — hard error otherwise.

### Selectors available inside a `CssStyle` block

Pseudo-classes: `anyLink`, `link`, `visited`, `target`, `hover`, `active`, `focus`,
`focusVisible`, `focusWithin`, `autofill`, `enabled`, `disabled`, `readOnly`, `readWrite`,
`placeholderShown`, `default`, `checked`, `indeterminate`, `valid`, `invalid`, `inRange`,
`outOfRange`, `required`, `optional`, `userValid`, `userInvalid`, `root`, `empty`,
`firstChild`, `lastChild`, `onlyChild`, `firstOfType`, `lastOfType`, `onlyOfType`.

Pseudo-elements: `before`, `after`, `selection`, `firstLetter`, `firstLine`,
`placeholder`.

Also: `mediaPrint`, `not(...)`, `ariaDisabled`, `ariaInvalid`, `ariaRequired`,
and combinators `children("li")`, `descendants("a")`, `nextSiblings(...)`,
`subsequentSiblings(...)`.

Anything else: `cssRule(" > .child") { Modifier… }` or
`cssRule(CSSMediaQuery.MediaFeature("prefers-reduced-motion", StylePropertyValue("no-preference"))) { … }`.

Breakpoints combine with pseudo-selectors: `((Breakpoint.SM..Breakpoint.MD) + hover) { … }`.

## Style kinds

| Kind | Declaration | Use |
| --- | --- | --- |
| **General** | `val S = CssStyle { … }` | Ordinary reusable style. |
| **Component** | `sealed interface MyKind : ComponentKind` + `val S = CssStyle<MyKind> { … }` | A widget's style, so that *variants* can be typed against it. |
| **Restricted** | `class Size : CssStyle.Restricted.Base(Modifier) { companion object { val SM = Size(); val LG = Size() } }` | A closed set of style values passed as a widget parameter. |

### Variants

```kotlin
sealed interface ButtonKind : ComponentKind
val ButtonStyle = CssStyle<ButtonKind> { base { … } }

val OutlinedButtonVariant = ButtonStyle.addVariant { hover { … } }
val GhostButtonVariant = ButtonStyle.addVariantBase { Modifier.background(Colors.Transparent) }

// apply
ButtonStyle.toModifier(OutlinedButtonVariant)
```

A variant can only be applied to the style it was derived from — that is the entire point
of `ComponentKind`.

### Extending styles

```kotlin
val BaseTextStyle = CssStyle.base { Modifier.fontSize(1.cssRem) }
val EmphasizedStyle = BaseTextStyle.extendedBy { base { Modifier.fontWeight(FontWeight.Bold) } }
val QuietStyle = BaseTextStyle.extendedByBase { Modifier.opacity(0.7) }
```

An extended style emits both class names, so the base rules keep applying.

## Breakpoints

Five stops. Defaults (`SilkTheme.breakpoints`, overridable in `@InitSilk`):

| Breakpoint | Default min-width |
| --- | --- |
| `ZERO` | 0 |
| `SM` | 30 cssRem |
| `MD` | 48 cssRem |
| `LG` | 62 cssRem |
| `XL` | 80 cssRem |
| `XXL` | 96 cssRem (fallback: `XL`) |

Usage:

```kotlin
val Responsive = CssStyle {
    base { Modifier.fontSize(16.px) }
    Breakpoint.MD { Modifier.fontSize(24.px) }       // >= MD
    until(Breakpoint.MD) { Modifier.fontSize(12.px) } // < MD
}
```

Outside a style block, `rememberBreakpoint()` gives the current breakpoint as state —
but note it does not behave well during export (#318); prefer CSS-driven responsiveness
(`Modifier.displayIfAtLeast(Breakpoint.MD)`) for anything that must be correct in a
static snapshot.

## Color modes

```kotlin
val ThemedStyle = CssStyle.base {
    Modifier.color(if (colorMode.isLight) Colors.Black else Colors.White)
}
```

`colorMode` is available inside every `CssStyle` / `Keyframes` block. In a composable,
read it with `ColorMode.current`, or get a read/write handle:

```kotlin
var colorMode by ColorMode.currentState
Button(onClick = { colorMode = colorMode.opposite }) { … }
```

Configure and persist it in `@InitSilk`:

```kotlin
@InitSilk
fun initColorMode(ctx: InitSilkContext) {
    ctx.config.initialColorMode =
        ColorMode.loadFromLocalStorage(KEY) ?: ColorMode.systemPreference
}
```

Persisting is the documented way to stop an exported page flashing the wrong theme. The
underlying color-mode machinery was reworked in 0.20.5 specifically to make that fixable;
skip 0.20.5 / 0.21.0 themselves (known color-mode bugs) in favour of 0.20.6 / 0.21.1.

### Palettes

```kotlin
@InitSilk
fun initTheme(ctx: InitSilkContext) {
    ctx.theme.palettes.light.background = Color.rgb(0xFAFAFA)
    ctx.theme.palettes.light.color = Colors.Black
    ctx.theme.palettes.dark.background = Color.rgb(0x06080B)
    ctx.theme.palettes.dark.color = Colors.White
}
```

Widget-specific groups (`button`, `checkbox`, `input`, `link`, `tooltip`, …) hang off the
same palettes. For site-specific colors beyond Silk's vocabulary, the templates'
convention is a hand-rolled `SitePalette` data class plus
`fun ColorMode.toSitePalette(): SitePalette` — a plain Kotlin `when`, no framework
support needed.

Read the active palette from a style with `colorMode.toPalette()`.

## Global element styles

```kotlin
@InitSilk
fun initStyles(ctx: InitSilkContext) {
    ctx.stylesheet.registerStyleBase("body") {
        Modifier.fontFamily("Roboto", "sans-serif").fontSize(18.px).lineHeight(1.5)
    }
    ctx.stylesheet.registerStyle("html") {
        cssRule(CSSMediaQuery.MediaFeature("prefers-reduced-motion", StylePropertyValue("no-preference"))) {
            Modifier.scrollBehavior(ScrollBehavior.Smooth)
        }
    }
}
```

## Overriding Silk's own widget styles

```kotlin
ctx.theme.replaceStyle(ButtonStyle) { /* completely new rules */ }
ctx.theme.modifyStyle(ButtonStyle) { /* additive */ }
ctx.theme.modifyStyleBase(HorizontalDividerStyle) { Modifier.fillMaxWidth() }
```

`replaceStyle` is also the escape hatch when you would otherwise hit the
"CssStyle with a name that's already used" error from a duplicate registration.

## CSS variables — `StyleVariable`

```kotlin
val DialogWidth by StyleVariable<CSSLengthNumericValue>(defaultFallback = 600.px)

val DialogStyle = CssStyle.base { Modifier.width(DialogWidth.value()) }

// override per subtree
Box(Modifier.setVariable(DialogWidth, 800.px)) { Dialog() }
```

Three flavours by delegate: `StyleVariable<T : StylePropertyValue>()`,
`StyleVariable<T : Number>()`, and a string variant. Declaring them inside an `object`
named e.g. `ButtonVars` prefixes the generated CSS variable names automatically
(`--button-…`), with `-vars`/`-variables` stripped from the object name.

Arithmetic: `calc { num(DialogWidth) * num(0.5) }`.

Kobweb's `StyleVariable` replaces Compose HTML's `CSSStyleVariable`, which has a known
bug accepting invalid values.

> In many cases you do **not** want a CSS variable: a plain Kotlin `val` is simpler and
> compiles away. Reach for `StyleVariable` when the value must be overridable per DOM
> subtree or must react to a color-mode change without recomposition.

## Keyframes and animation

```kotlin
val ShiftKeyframes = Keyframes {
    from { Modifier.left(0.px) }
    to { Modifier.left(200.px) }
    // or: 25.percent { … }; each(50.percent, 75.percent) { … }
}

Box(Modifier.animation(ShiftKeyframes.toAnimation(duration = 2.s, iterationCount = AnimationIterationCount.Infinite)))
```

Like styles, `Keyframes` must be a public top-level (or object) `val`. Inside a
`CssStyleScope`, the color-mode-aware overload `ShiftKeyframes.toAnimation(...)` is
available directly.

0.24.1+ also allows setting animation sub-properties individually:
`Modifier.animation { delay(...); duration(...) }`.

## CSS layers

Silk emits its rules into an ordered set of `@layer`s, so user styles beat framework
styles regardless of selector specificity:

`reset` → `kobweb-compose` → `base` → `component-styles` → `component-variants` →
`restricted-styles` → `general-styles`

Override a style's layer with `@CssLayer("name")` and register the layer explicitly:

```kotlin
@CssLayer("important")
val ImportantStyle = CssStyle.base { … }

@InitSilk
fun initSilk(ctx: InitSilkContext) {
    ctx.stylesheet.cssLayers.add("important", "more-important")
}
```

Rarely worth it — an unregistered custom layer gets a best-effort high-priority slot, and
layer juggling makes style bugs hard to reason about. Note Kobweb currently produces bad
output for browsers without CSS-layer support (#601, open, milestone 1.0).

## Generated CSS names

Class names are derived from the property name, optionally prefixed by
`kobweb.app.cssPrefix`. Override per declaration:

```kotlin
@CssPrefix("site")
@CssName("hero")
val HeroStyle = CssStyle { … }
```

Both annotations also work on an enclosing `object`/`class` to name a whole group.

## Element refs — reaching the raw DOM node

```kotlin
Box(ref = ref { element -> element.focus() })                     // on first composition
Box(ref = ref(key) { element -> … })                              // re-run when key changes
Box(ref = disposableRef { element ->                              // with teardown
    registry.add(element); onDispose { registry.remove(element) }
})
Box(ref = refScope { ref { … }; disposableRef { … } })            // combine several
```

Silk widgets take a `ref` parameter; raw Compose HTML elements use
`DisposableEffect`/`ref` from `com.varabyte.kobweb.compose.dom`.

## Debugging checklist

| Symptom | Cause |
| --- | --- |
| A style has no effect | It is local or non-public — check the KSP warning in the build log. |
| "can only be associated with a single CSS style" | Two `CssStyle<K>` share a `ComponentKind`. |
| "is using a kind type defined in a different file" | Move the `ComponentKind` next to its style. |
| Hover/media rules ignored | They were written as an inline `Modifier` rather than a `CssStyle` block. |
| Widget style overridden unexpectedly | Inline modifiers beat stylesheet rules — by design. |
| Wrong theme flashes on load | Color mode is not persisted; add `loadFromLocalStorage`/`saveToLocalStorage`. |
| Layout collapses vertically after 0.22.0 | `minHeight(100.vh)` on a `Surface`; move it to a `Box` child. |
