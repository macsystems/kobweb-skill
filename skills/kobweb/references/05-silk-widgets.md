# Silk: layout primitives, widgets, icons

Silk is Kobweb's batteries-included UI layer. It ships in three artifacts:

| Artifact | Contents |
| --- | --- |
| `com.varabyte.kobweb:silk-foundation` | `CssStyle`, theming, palettes, breakpoints, `SpanText`, deferred rendering. No Kobweb-specific deps. |
| `com.varabyte.kobweb:silk-widgets` | The widget set (below). Usable from plain Compose HTML projects. |
| `com.varabyte.kobweb:silk-widgets-kobweb` | Widgets that need Kobweb routing: `Link`, `Image`, `Toc`. |
| `com.varabyte.kobweb:kobweb-silk` | Umbrella — depend on this in a Kobweb site and get all three. |

## Layout primitives (`kobweb-compose`)

These are the Compose-shaped wrappers around flexbox/grid. They come with
`kobweb-core`, not Silk.

| Composable | Renders |
| --- | --- |
| `Box(modifier, contentAlignment, ref) { }` | a `<div>` using CSS grid, stacking children |
| `Row(modifier, horizontalArrangement, verticalAlignment, ref) { }` | flex row |
| `Column(modifier, verticalArrangement, horizontalAlignment, ref) { }` | flex column |
| `Spacer()` | flex-grow filler |

`Arrangement` (`SpaceBetween`, `SpaceAround`, `SpaceEvenly`, `spacedBy(…)`, …) and
`Alignment` mirror the Compose names. `RowScope`/`ColumnScope` provide `Modifier.align(…)`
and `Modifier.weight(…)`.

Choosing `Row`/`Column` over hand-written flexbox is the point of the library: it is far
easier to reason about than remembering whether to justify items or align content.

## Widgets (`silk-widgets`)

| Area | Composables |
| --- | --- |
| Forms | `Button`, `Checkbox`, `TriCheckbox`, `Switch`, `TextInput`, `InputGroup` (+ `InputGroupScope.TextInput`), `Label` |
| Layout | `Surface`, `SimpleGrid` (+ `numColumns(…)`), `HorizontalDivider`, `VerticalDivider` |
| Disclosure | `Tabs` (+ `TabsScope.TabPanel`, `TabPanelScope.Tab`) |
| Display | `Callout` |
| Graphics | `Canvas2d`, `CanvasGl`, `CanvasGl2` (WebGL / WebGL2 contexts) |
| Overlay | `Tooltip`, `AdvancedTooltip`, `Popover`, `AdvancedPopover`, `Overlay`, plus `KeepPopupOpenStrategy` / `OpenClosePopupStrategy` for custom trigger behaviour |
| Text | `SpanText` (in `silk-foundation`) |
| Kobweb-aware | `Link`, `Image`, `Toc` (table of contents, from heading elements) |

Widgets follow a consistent parameter shape, e.g.:

```kotlin
Button(
    onClick = { … },
    modifier = Modifier,
    variant = null,                 // CssStyleVariant<ButtonKind>?
    type = ButtonType.Button,
    enabled = true,
    size = ButtonSize.MD,           // a "restricted style"
    colorPalette = null,
    focusBorderColor = null,
    ref = null,
) { /* RowScope content */ }
```

Three things generalize from that signature:

- `modifier` is applied **after** the widget's own style, so an inline modifier always
  wins.
- `variant` is typed to the widget's `ComponentKind`; you create your own with
  `ButtonStyle.addVariant { }`.
- `ref: ElementRefScope<HTMLxxxElement>?` gives you the raw DOM node.

Known gaps: not every widget accepts `size`/`colorPalette` yet (#255), a11y labels are
still being filled in (#673, #295), and there is no file-input widget yet (#749). All
three are milestone 1.0.

### Built-in SVG icons (no extra dependency)

`ArrowBackIcon`, `ArrowDownIcon`, `ArrowForwardIcon`, `ArrowUpIcon`, `AttachmentIcon`,
`CalendarIcon`, `CheckCircleIcon`, `CheckIcon`, `ChevronDownIcon`, `ChevronLeftIcon`,
`ChevronRightIcon`, `ChevronUpIcon`, `CircleIcon`, `ClipboardIcon`, `CloseIcon`,
`CodeIcon`, `DownloadIcon`, `EditIcon`, `ExclaimIcon`, `EyeIcon`, `EyeOffIcon`,
`HamburgerIcon`, `IndeterminateIcon`, `InfoIcon`, `LightbulbIcon`, `LockIcon`,
`MinusIcon`, `MoonIcon`, `PlusIcon`, `QuestionIcon`, `QuoteIcon`, `SearchIcon`,
`SettingsIcon`, `SquareIcon`, `StopIcon`, `SunIcon`, `TrashIcon`, `UploadIcon`,
`UserIcon`, `WarningIcon`.

Enough for a nav bar and a color-mode toggle; reach for an icon pack beyond that.

## Icon packs (`com.varabyte.kobwebx:silk-icons-*`)

| Artifact | Prefix | Notes |
| --- | --- | --- |
| `silk-icons-fa` | `Fa…` e.g. `FaMoon()`, `FaSpider(Modifier.color(Colors.Red))` | Font Awesome (free set). |
| `silk-icons-ms` | `Ms…` e.g. `MsMail()`, `MsLightMode(style = MsIconStyle.SHARP)` | **Material Symbols** — Google's current set. Styles: `OUTLINED` (default), `ROUNDED`, `SHARP`. Added 0.24.1. |
| `silk-icons-lucide` | `Lucide…` e.g. `LucideMail(size = 1.em, strokeWidth = 2, color = null)` | Lucide. Added 0.24.1. |
| `silk-icons-mdi` | `Mdi…` e.g. `MdiBugReport()` | Material Design Icons — **deprecated upstream by Google**; prefer `silk-icons-ms` for new work. |

Font-based packs pull a stylesheet from a CDN by default. To self-host it at build time
(GDPR, offline, or CSP reasons):

```kotlin
kobweb {
    app {
        index { interceptUrls { enableSelfHosting() } }
    }
}
```

## Palettes and theme groups

`ctx.theme.palettes.light` / `.dark` expose site-wide roles (`background`, `color`,
`border`, `focusOutline`, `overlay`, `placeholder`, …) plus per-widget groups
(`button`, `checkbox`, `input`, `link`, `switch`, `tab`, `tooltip`, `callout`, …).

```kotlin
@InitSilk
fun initTheme(ctx: InitSilkContext) {
    ctx.theme.palettes.light.background = Color.rgb(0xFAFAFA)
    ctx.theme.palettes.dark.link.visited = Colors.Violet
}
```

Beyond those roles, define your own palette type — the templates' idiom:

```kotlin
class SitePalette(val nearBackground: Color, val brand: Brand) {
    class Brand(val primary: Color, val accent: Color)
}

object SitePalettes {
    val light = SitePalette(nearBackground = Color.rgb(0xF4F6FA), brand = …)
    val dark  = SitePalette(nearBackground = Color.rgb(0x13171F), brand = …)
}

fun ColorMode.toSitePalette() = when (this) {
    ColorMode.LIGHT -> SitePalettes.light
    ColorMode.DARK -> SitePalettes.dark
}
```

A first-class theme/design-token API is requested but unimplemented (#297, milestone 1.0),
and widget defaults are still being redesigned (#346). The `when`-based approach above is
the current best practice.

## Overriding widget styles globally

```kotlin
@InitSilk
fun initSiteStyles(ctx: InitSilkContext) {
    ctx.theme.modifyStyleBase(HorizontalDividerStyle) { Modifier.fillMaxWidth() }
    ctx.theme.replaceStyle(ButtonStyle) { /* start over */ }
}
```

Widget CSS variables (`ButtonVars.BackgroundDefaultColor`, `ButtonVars.FontSize`, …) are
often the lighter-touch option:

```kotlin
val UncoloredButtonVariant = ButtonStyle.addVariantBase {
    Modifier.setVariable(ButtonVars.BackgroundDefaultColor, Colors.Transparent)
}
```

## Deferred rendering

`silk-foundation` exposes `Deferred { … }` (hosted by `DeferringHost`, which `SilkApp`
installs for you). Content inside it is rendered at the *end* of the DOM, above
everything else — this is how popups and tooltips escape their parent's stacking context.
Reuse it for your own overlays instead of escalating `z-index` values.
