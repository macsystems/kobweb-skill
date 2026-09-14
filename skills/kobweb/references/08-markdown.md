# Markdown

Kobweb converts markdown files into `@Page` composables **at build time**. There is no
runtime markdown parser in the bundle: the plugin generates Kotlin source, so markdown
pages are as fast as hand-written ones and can embed live components.

## Enabling it

```kotlin
plugins {
    alias(libs.plugins.kobwebx.markdown)
}

kotlin {
    sourceSets {
        jsMain.dependencies { implementation(libs.kobwebx.markdown) }
    }
}
```

Files go in `src/jsMain/resources/markdown/`. The path becomes the route:
`markdown/docs/tutorial/Kobweb.md` → `/docs/tutorial/kobweb`.

The `kobwebx-markdown` runtime dependency is what gives generated pages access to
`ctx.markdown` (front-matter data at runtime).

## Front matter

```markdown
---
title: Getting Started
layout: .components.layouts.DocsLayout
routeOverride: a*-demo
imports:
  - .components.widgets.VisitorCounter
---

# Getting Started
```

| Key | Meaning |
| --- | --- |
| `layout` | FQN of a `@Layout` composable. An **empty** `layout:` means `@NoLayout`. |
| `root` | Legacy pre-`@Layout` mechanism; prefer `layout`. |
| `routeOverride` | Same semantics as `@Page(routeOverride)`, and the only way to get characters into a URL that a Kotlin file name cannot express. |
| `imports` | Extra imports for this file only. |
| anything else | Free-form; readable at runtime from `ctx.markdown.frontMatter` and at build time from `MarkdownEntry.frontMatter`. |

Project-wide default layout:

```kotlin
kobweb { markdown { defaultLayout.set(".components.layouts.MarkdownLayout") } }
```

## Embedding Kotlin — "Kobweb calls"

**Block** form, for standalone widgets:

```markdown
{{{ .components.widgets.VisitorCounter }}}
```

**Inline** form, inside a sentence:

```markdown
Press ${.components.widgets.ColorButton} to toggle the color.
```

No spaces are allowed inside the braces. A leading `.` resolves against the module's root
package.

Global imports so calls stay short:

```kotlin
kobweb { markdown { imports.add(".components.widgets.*") } }
```

## Callouts

GitHub-style blockquote alerts:

```markdown
> [!NOTE]
> Something worth knowing.
```

Built-in kinds: `NOTE`, `TIP`, `IMPORTANT`, `CAUTION`, `WARNING`, `QUESTION`, `QUOTE`.

Wire them up (and relabel or restyle them) in the build script:

```kotlin
import com.varabyte.kobwebx.gradle.markdown.handlers.SilkCalloutBlockquoteHandler

kobweb {
    markdown {
        handlers.blockquote.set(SilkCalloutBlockquoteHandler(labels = mapOf("QUOTE" to "")))
    }
}
```

Variants such as `OutlinedCalloutVariant` / `OutlinedFilledCalloutVariant` restyle them;
custom callout kinds are supported.

## Custom node handlers

Every markdown node type has an overridable handler that emits Kotlin source:

`text`, `img`, `heading`, `p`, `br`, `a`, `em`, `strong`, `hr`, `ul`, `ol`, `li`, `code`
(fenced blocks), `inlineCode`, `blockquote`, `table`, `thead`, `tbody`, `tr`, `td`, `th`,
`rawTag`, `inlineTag`, `html`.

```kotlin
kobweb {
    markdown {
        handlers.code.set { codeBlock -> "org.example.widgets.CodeBlock(\"\"\"${codeBlock.literal}\"\"\")" }
    }
}
```

Other switches: `useSilk` (emit Silk widgets rather than raw HTML elements),
`generateHeaderIds`, `idGenerator`.

HTML embedded in markdown is parsed properly and applied through `attrs` blocks as of
0.24.0 — it used to be string-spliced. A builder-style API for handlers is still open
(#287).

## Build-time processing of every file

`process` runs once with every `MarkdownEntry` (`filePath`, `frontMatter`, `route`,
`fqn`) and can generate new files:

```kotlin
kobweb {
    markdown {
        process.set { entries ->
            generateMarkdown("blog/index.md", buildString {
                appendLine("# All posts")
                entries.sortedBy { it.route }.forEach { appendLine("* [${it.filePath}](${it.route})") }
            })
        }
    }
}
```

Three generators are available in that scope: `generateMarkdown(path, content)`,
`generateKotlin(path, content)`, `generatePublicResource(path, content)`.

Because `MarkdownEntry.fqn` is the generated composable's fully-qualified name, a listing
page can `import` and render other markdown pages directly.

## Extra markdown sources

```kotlin
kobweb.markdown.addSource(someGeneratorTask)                               // task output
kobweb.markdown.addSource(layout.projectDirectory.dir("src/jsMain/resources/markdown-src"), ".")
```

Useful when markdown is fetched or generated during the build. The second parameter is
the target package (default `.pages`).

## Limitations

- Exporting arbitrary *sections* of markdown as separate reusable pieces is open (#301).
- Link previews / social cards for markdown posts are open (#178).
- `kobwebx-frontmatter` is the shared parsing library if you need front matter outside
  the plugin.
