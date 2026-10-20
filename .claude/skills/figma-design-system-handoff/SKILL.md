---
name: "figma-design-system-handoff"
description: "Guide a designer step by step through handing off a Figma design system to developers with Claude Code: tokens, components, token export, Code Connect, DESIGN.md, testing. Asks for the Figma link and target platform (Web/Tailwind, Android/Material 3, iOS/Apple HIG) first and gets approval before every change."
---

# Figma Design System Handoff (guided)

You are guiding the user through handing off a Figma design system so developers and Claude Code can build with it accurately. Lead them through the work one step at a time, in plain language. The user is usually a designer, so explain *why* each step matters in one sentence, without jargon.

## Ground rules (apply to every step)

1. **One step at a time.** Never start the next step until the current one is finished and the user has said to continue.
2. **Read first, change later.** Every step begins with a read-only audit. Nothing in Figma or the repo changes until the user approves.
3. **Approval gates.** Before any change, show exactly what will change, then ask with `AskUserQuestion` using these options:
   - **Approve**: apply as proposed
   - **Change something**: the user edits the plan, you re-propose
   - **Skip this part**: move on without applying
   If `AskUserQuestion` is unavailable, ask in plain text and wait.
4. **Design decisions belong to the designer.** When something is a judgment call (which duplicate to keep, a prop name, whether a missing token is intentional, which platform role a token maps to), present 2–3 options with your recommendation and let them choose. Never guess silently.
5. **Never break bindings.** Rename variables and components in place (Figma links by ID), never delete-and-recreate.
6. **Verify after every change.** Re-read the file and report real before/after numbers.
7. **Track progress** in `.design-handoff/progress.md` in the project (create it at the start). Record the chosen platform(s) and framework at the top. After each step, record: what was done, decisions the user made, open items. If a session restarts, read this file first and resume from the last unfinished step.
8. **The platform decides the naming.** Code syntax, exported files, Code Connect and the rulebook all follow the conventions of the platform(s) chosen in Step 0 (see **Platform conventions** below). Never mix conventions: no CSS names in Android code syntax, no Material role names in Web code syntax. If an existing codebase already has its own names, those win over the defaults below, and you say so.

## Prerequisites check

Before Step 0, confirm the Figma MCP tools are available (e.g. `use_figma`, `get_metadata`, `get_variable_defs`). If they are not, tell the user to run:
```
claude plugin install figma@claude-plugins-official
```
then authenticate with `/mcp`, and stop until it's connected. Before any `use_figma` call, load the `figma-use` skill (and `figma-generate-library` for variable/component edits) as the Figma plugin requires.

---

## Step 0: Start

Greet the user briefly and show the roadmap:

> We'll hand off your design system in 6 steps. I'll check everything first and ask for your approval before changing anything.
> 1. Make the tokens machine-readable
> 2. Make components code-shaped
> 3. Export tokens into the codebase
> 4. Connect Figma components to code (Code Connect)
> 5. Write the rulebook (DESIGN.md)
> 6. Test with a real screen and set up syncing

Then ask, **first**, for the design system link:
> Please send the link to your Figma design system file.

Parse it: `figma.com/design/:fileKey/...?node-id=1-2` → `fileKey`, nodeId `1:2`. For `/branch/:branchKey/` use the branchKey.

**Then ask which platform the design system is for** (one `AskUserQuestion`, `multiSelect: true`):
- **Web**: code names follow Tailwind CSS
- **Android**: code names follow Material Design 3
- **iOS**: code names follow Apple's Human Interface Guidelines

Explain in one line: *"Developers on each platform name things differently, so I'll name the tokens the way your developers will actually type them."*

**Follow-up questions** (one `AskUserQuestion` call, up to 4 questions, only for the platforms chosen):
- Web: **Tailwind v4** (recommended, theme lives in CSS `@theme`) or **Tailwind v3** (`tailwind.config.js`)
- Android: **Jetpack Compose** (recommended) or **XML Views**
- iOS: **SwiftUI** (recommended) or **UIKit**
- **Is this a copy, or the live file?** Recommend working on a duplicate for the first run.

Then a second round (one `AskUserQuestion` call):
- **Is there a codebase yet?** (Yes, in this repo / Yes, elsewhere / Not yet). If yes, inspect it in Step 1 so names match existing code.
- **Which steps?** All 6 (recommended), or a subset.

If there's no codebase, Steps 1, 2 and 5 still work, and Steps 3, 4 and 6 wait until one exists. Say this clearly.

Write the platform(s) and framework into `.design-handoff/progress.md`. Run a quick read-only inventory (pages, variable collections with counts and modes, text/effect styles, component sets per page) and summarize it in 3–5 lines before Step 1.

---

## Platform conventions

Figma stores code syntax separately per platform: `WEB`, `ANDROID` and `iOS` (`variable.setVariableCodeSyntax('WEB' | 'ANDROID' | 'iOS', value)`). Set only the platforms the user chose, and set all of them when several were chosen.

**General rule for every platform:** map a semantic token to a built-in platform role **only when the meaning truly matches** (e.g. main text → `onSurface` / `label`). Everything else (brand colors, status colors, button-specific tokens, spacing) gets a **custom name** in that platform's style. Primitives (raw palette like `blue/500`) never map to platform roles; they get custom palette names or stay hidden. Always show the mapping as a table with a confidence note, and flag any uncertain match as a designer decision.

### Web: Tailwind CSS

Use Tailwind's theme variable namespaces so each token becomes a utility class automatically.

| Token type | Tailwind namespace | Figma `WEB` code syntax | Utility it creates |
|---|---|---|---|
| Color | `--color-*` | `var(--color-primary)` | `bg-primary`, `text-primary`, `border-primary` |
| Spacing | `--spacing-*` | `var(--spacing-4)` | `p-4`, `gap-4`, `m-4` |
| Radius | `--radius-*` | `var(--radius-md)` | `rounded-md` |
| Font size | `--text-*` | `var(--text-base)` | `text-base` |
| Line height | `--leading-*` | `var(--leading-base)` | `leading-base` |
| Letter spacing | `--tracking-*` | `var(--tracking-wide)` | `tracking-wide` |
| Font weight | `--font-weight-*` | `var(--font-weight-semibold)` | `font-semibold` |
| Font family | `--font-*` | `var(--font-sans)` | `font-sans` |
| Shadow | `--shadow-*` / `--inset-shadow-*` | `var(--shadow-md)` | `shadow-md` |

- Names are lowercase kebab-case. Font weights must be numbers in code (400, 500, 600, 700).
- **Avoid doubled class names.** `color/bg/primary` → `--color-bg-primary` gives the awkward `bg-bg-primary`. Show the user the resulting class names and offer two options: (a) **short role names** like `--color-surface`, `--color-muted`, `--color-foreground` → `bg-surface`, `text-muted` (recommended when names stay unique), or (b) **keep the full path** for explicitness. Let the designer choose.
- Primitives: `--color-blue-500` (matches Tailwind's own palette style).
- Dark mode and responsive modes override the same variables (see Step 3), so the names don't change per mode.
- Tailwind v3: same names as CSS variables, referenced from `tailwind.config.js` `theme.extend`.

### Android: Material Design 3

**Compose (default):**

| Token type | Material 3 target | Figma `ANDROID` code syntax |
|---|---|---|
| Color matching an M3 role | `ColorScheme` role | `MaterialTheme.colorScheme.primary` |
| Color with no M3 role | Extended theme colors | `AppTheme.colors.success` |
| Text style | M3 type scale | `MaterialTheme.typography.bodyLarge` |
| Radius | M3 shape scale | `MaterialTheme.shapes.medium` |
| Spacing | Custom (M3 has no spacing tokens) | `AppTheme.spacing.md` |
| Elevation / shadow | Tonal elevation levels | `AppTheme.elevation.level2` |

- **M3 color roles:** `primary, onPrimary, primaryContainer, onPrimaryContainer`, the same four for `secondary`, `tertiary` and `error`, plus `background, onBackground, surface, onSurface, surfaceVariant, onSurfaceVariant, surfaceDim, surfaceBright, surfaceContainerLowest, surfaceContainerLow, surfaceContainer, surfaceContainerHigh, surfaceContainerHighest, inverseSurface, inverseOnSurface, inversePrimary, outline, outlineVariant, scrim`.
  Typical matches: main text → `onSurface`, secondary text → `onSurfaceVariant`, page background → `surface`, card → `surfaceContainer`, border → `outlineVariant` (strong border → `outline`), error text → `error`, error background → `errorContainer`.
- **M3 type scale (15 roles):** `displayLarge/Medium/Small, headlineLarge/Medium/Small, titleLarge/Medium/Small, bodyLarge/Medium/Small, labelLarge/Medium/Small`. Map each text style by size and use (e.g. 16 Regular body → `bodyLarge`, 14 Medium button → `labelLarge`). Flag text styles left over or roles left empty.
- **M3 shape scale:** `extraSmall, small, medium, large, extraLarge` (plus `RoundedCornerShape(50)` for full).
- Custom names are camelCase (`AppTheme.colors.warningContainer`, `AppTheme.spacing.lg`). Ask the user for the theme object name (default `AppTheme`).
- **M3 has no "success" or "warning" roles.** Propose extended colors following the M3 pattern: `success, onSuccess, successContainer, onSuccessContainer`.
- Primitives: `Palette.blue500` (not exposed in the theme).

**XML Views:** colors `?attr/colorPrimary` / `?attr/colorOnSurface` (custom: `?attr/colorSuccess`), text `?attr/textAppearanceBodyLarge`, spacing `@dimen/spacing_md`, radius `?attr/shapeAppearanceMediumComponent`. Resource names are snake_case.

### iOS: Apple Human Interface Guidelines

**SwiftUI (default):**

| Token type | HIG target | Figma `iOS` code syntax |
|---|---|---|
| Color matching a system role | Semantic system color | `Color(.label)` |
| Brand / custom color | Asset catalog color | `Color.textPrimary` |
| Text style | Dynamic Type text style | `Font.body` (custom font: `AppFont.body`) |
| Spacing | Custom constants | `Spacing.md` |
| Radius | Custom constants | `Radius.md` |
| Shadow | Custom modifier | `Shadow.md` |

- **HIG semantic colors:** labels `label, secondaryLabel, tertiaryLabel, quaternaryLabel, placeholderText`; backgrounds `systemBackground, secondarySystemBackground, tertiarySystemBackground` (grouped: `systemGroupedBackground`, …); `separator, opaqueSeparator, link`; `Color.accentColor` for the app tint.
- **Brand colors stay custom.** Only map to system colors when the designer wants the native iOS look (system colors adapt to dark mode, high contrast and accessibility settings automatically, which is a good reason to choose them). Otherwise use asset catalog colors with Light/Dark appearances: `Color.textPrimary`, `Color.buttonPrimaryBackground` (Xcode generates these from asset names).
- **HIG text styles (Dynamic Type):** `largeTitle, title, title2, title3, headline, subheadline, body, callout, footnote, caption, caption2`. Map each text style to the closest one (17 pt is iOS body, so a 16 px web body style still maps to `body`, and you tell the user). With a custom font like Poppins, propose `AppFont.body` defined as `.custom("Poppins-Regular", size: 17, relativeTo: .body)` so Dynamic Type still scales it. Flag that HIG expects text to scale.
- Custom names are camelCase. Primitives: `Palette.blue500`.
- Remind the user that HIG minimum tap target is 44×44 pt; note it for Step 5.

**UIKit:** `UIColor.label`, `UIColor(named: "textPrimary")`, `UIFont.preferredFont(forTextStyle: .body)`, spacing/radius as `CGFloat` constants.

---

## Step 1: Make the tokens machine-readable

**Why (tell the user):** Claude Code reads variable *code syntax* to know which name to type in code. Without it, it falls back to raw values like hex codes.

**Audit (read-only)** for every variable collection. Report in a table:
- variables with no code syntax for each chosen platform (`WEB` / `ANDROID` / `iOS`)
- inconsistent names: mixed casing, spaces, leftover suffixes like ` 2`, duplicated words (`Font Family/Font Family`)
- variables with `ALL_SCOPES`
- uneven token sets (e.g. one button type has `bg/text/border/hover/active`, another is missing some)
- primitive/semantic structure: do semantic tokens alias primitives? Are there Light/Dark modes?
- if a codebase exists: the names it already uses (Tailwind theme / `@theme` CSS, Compose theme files, Swift color extensions or asset catalog)

Keep `use_figma` return values small (summaries and counts, not full dumps); the return limit is ~20KB.

**Propose, in this order:**
1. **Clean Figma names** to lowercase kebab-case, keeping existing groups. Check for name collisions before renaming. (Figma names stay platform-neutral; the platform naming lives in code syntax.)
2. **Platform mapping table** for each chosen platform, following **Platform conventions**: Figma variable → code syntax, with a column showing whether it maps to a built-in role (Tailwind namespace / M3 role / HIG role) or a custom name. For Web, show the resulting utility classes. For Android and iOS, list unmapped built-in roles (e.g. M3 `tertiary` with no Figma token) and Figma tokens with no clear role, as designer decisions.
3. **Scopes:** primitives hidden (`[]`); backgrounds `FRAME_FILL, SHAPE_FILL`; text `TEXT_FILL`; borders `STROKE_COLOR`; spacing `GAP` (+ padding); radius `CORNER_RADIUS`; font family `FONT_FAMILY`; font weight `FONT_STYLE`. Match whatever pattern the file already uses consistently.
4. **Uneven token sets** flagged as designer decisions, not auto-fixed.

Show the full rename list and the mapping table (one per platform). **APPROVAL GATE.** For multi-platform systems you may approve each platform's table separately.

**Apply**, then verify, and report a before/after table:
| | Before | After |
|---|---|---|
| Variables with code syntax (per platform) | x / N | N / N |
| Inconsistent names | x | 0 |
| Variables on "All" scopes | x | 0 |
| Bindings broken | n/a | 0 |

Note anything code needs that Figma can't hold (e.g. font weights stored as style names map to 400/500/600/700 on Web, `FontWeight.SemiBold` on Compose, `.semibold` on SwiftUI). Update progress file. Ask: continue to Step 2?

---

## Step 2: Make components code-shaped

**Why:** Code Connect and Claude Code map Figma properties to code props. `Property 1=Variant5` becomes a useless prop.

**Audit (read-only), one page per `use_figma` call** (run page reads in parallel). For each component set report:
- generic property names (`Property 1`) and placeholder values (`Variant5`, `Status4`, typos)
- duplicate component sets with the same name
- broken sets (duplicate variant names; reading `componentPropertyDefinitions` throws, so wrap it in try/catch and report "broken")
- hardcoded fills/strokes not bound to variables, unbound radius/gap
- frames without auto layout
- missing descriptions
- inconsistent state names across components

Present a summary table of all components ranked worst first, then propose a **shared state vocabulary** for the whole system (e.g. `default, hover, focus, pressed, disabled, error`) and get it approved once. Adjust it to the platform: Web has `hover` and `focus`; Android (M3) uses `pressed`, `focused`, `dragged` and no hover on touch; iOS uses `highlighted`/`pressed` and `disabled`, with no hover.

Property names follow the platform's component API style: React-style props for Web (`variant`, `size`, `disabled`), M3 component names and parameters for Android (`FilledButton`, `OutlinedButton`, `enabled`), SwiftUI style for iOS (`.buttonStyle(.borderedProminent)`, `.controlSize(.large)`, `isEnabled`). For multi-platform systems, keep neutral Figma names (`variant`, `size`, `state`) and map them per platform in Step 4.

Then go **one component at a time**, worst first unless the user prefers otherwise. For each, propose:
- property renames (`Property 1` → `state`, `Type` → `variant`) and value cleanup
- which duplicate to keep (show screenshots of each with `get_screenshot`; this is a designer decision)
- binding hardcoded values to the closest existing token; list ones with **no close match** instead of guessing
- a one-line description: "Use for… Don't use for…"; for Android/iOS also note the native equivalent (e.g. "M3 Filled button", "iOS bordered prominent button")

**APPROVAL GATE per component.** Don't delete duplicates; propose moving the unused ones to an `Archive` page and get explicit approval. After each component, take one screenshot to confirm nothing visually changed. Update progress after each component. Let the user stop and resume anytime.

---

## Step 3: Export tokens into the codebase

Requires a repo. **Why:** the codebase needs its own copy of the values so developers and Claude use the same names.

**Inspect the repo first:** platform, framework, existing theme files, package manager or build system.

**Shared source:** `tokens/tokens.json` in W3C DTCG format, keeping primitive → semantic aliases, with every mode (Light/Dark, Desktop/iPad/Mobile). Use Style Dictionary to generate each platform's files from it. Read variables with the Figma MCP (`get_variable_defs`, or a read-only `use_figma` script for full collections and modes). The Variables REST API needs Enterprise, so don't depend on it.

**Propose a file plan per chosen platform** (paths + what each holds). Every generated name must equal that platform's Figma code syntax.

- **Web / Tailwind v4:** `src/styles/theme.css` with `@import "tailwindcss";` and an `@theme { … }` block of the token variables. Dark mode overrides the same variables under `[data-theme="dark"]` (or `.dark`, ask). Responsive modes override them in media queries (ask for breakpoints, register them as `--breakpoint-*`). If the design system replaces Tailwind's defaults, use `--color-*: initial;` to clear the default palette (ask first).
- **Web / Tailwind v3:** `src/styles/tokens.css` (CSS variables) plus `theme.extend` in `tailwind.config.js` pointing at them.
- **Android / Compose:** `ui/theme/Color.kt` (palette), `ui/theme/Theme.kt` with `lightColorScheme(...)` and `darkColorScheme(...)` mapping every M3 role, `ui/theme/Type.kt` (`Typography` with the 15 roles), `ui/theme/Shape.kt` (`Shapes`), and `ui/theme/AppTheme.kt` with extended colors, spacing and elevation via `CompositionLocal`. Responsive modes → `WindowSizeClass`.
- **Android / XML Views:** `res/values/colors.xml`, `res/values-night/colors.xml`, `themes.xml` attributes, `dimens.xml`, with `values-sw600dp` for tablet.
- **iOS / SwiftUI:** `Assets.xcassets/Colors/*.colorset` with Any/Dark appearances, `DesignSystem/Typography.swift` (`AppFont`), `DesignSystem/Spacing.swift`, `Radius.swift`, `Shadow.swift`. Responsive modes → size classes (`horizontalSizeClass`).
- **iOS / UIKit:** same asset catalog, plus `UIColor`/`UIFont` extensions.

Add one regenerate command (`npm run tokens:build`, a Gradle task, or a script).

**APPROVAL GATE** on the file plan, then write the files, install deps, run the build, and show a sample of the output. Verify every Figma code syntax name for that platform exists in the output (report any missing). For Android, confirm every M3 color role is filled in both schemes.

---

## Step 4: Connect Figma components to code (Code Connect)

Requires components in the repo. Load the `figma-code-connect` skill. Code Connect supports React/HTML (Web), Jetpack Compose (Android) and SwiftUI (iOS); use the one that matches the platform.

- List Figma components (`list_file_components_for_code_connect` / `get_code_connect_suggestions`) and code components in the repo.
- Present a mapping table: Figma component → code file, plus a property mapping in the platform's API style (Web: `state=disabled` → `disabled`; Compose: `state=disabled` → `enabled = false`; SwiftUI: `state=disabled` → `.disabled(true)`).
- Where the code uses native components directly (an M3 `Button`, a SwiftUI `Toggle`), map to those instead of a custom wrapper, and say so.
- Flag Figma components with no code match and code components with no Figma match. Don't create new components.
- Start with the most-used components; ask the user which ones matter most.

**APPROVAL GATE** on the mapping table, then publish mappings and confirm with `get_code_connect_map`.

---

## Step 5: Write the rulebook (DESIGN.md)

**Why:** rules Claude follows every session. It holds rules, not values (values live in tokens).

Draft `DESIGN.md` at the repo root (or the location the user picks) from the Figma file, the tokens, and the code. Don't invent rules. Mark anything inferred as `<!-- confirm -->`.

Sections:
1. Principles (3–5 lines; ask the user for these, they're design intent)
2. Tokens: point to `tokens/tokens.json` and the generated files, never copy values; "semantic tokens only in components"
3. Platform rules (one subsection per chosen platform):
   - Web: use Tailwind utilities from the theme (`bg-surface`, `p-4`); no arbitrary values like `bg-[#2563eb]` or `p-[13px]`
   - Android: use `MaterialTheme.colorScheme` / `typography` / `shapes` and `AppTheme.*`; no `Color(0xFF…)` or raw `.dp`/`.sp` in components; follow M3 component guidance
   - iOS: use semantic and asset colors, `AppFont` with Dynamic Type, `Spacing`/`Radius` constants; no hardcoded `Color(red:…)` or fixed font sizes; follow HIG
4. Layout & spacing: scale logic, responsive behavior (breakpoints / window size classes / size classes)
5. Components: for each, when to use, variants, required states, one do and one don't
6. Accessibility: contrast, focus states (Web), minimum tap size (44×44 pt on iOS, 48×48 dp on Android), Dynamic Type / font scaling
7. Do / Don't: top 10 mistakes

Short imperative rules an agent can follow. Under ~300 lines.

Also propose these lines for the top of `CLAUDE.md` (create it if missing, never overwrite existing content), adapted to the platform:
```
- Never hardcode colors, spacing, radius, or font sizes; use the design tokens
  (Web: Tailwind theme utilities · Android: MaterialTheme + AppTheme · iOS: asset colors, AppFont, Spacing)
- Reuse existing components before creating new ones
- Follow DESIGN.md for all UI work
```
Show both drafts. **APPROVAL GATE**, then write.

---

## Step 6: Test with a real screen, then set up syncing

Ask the user for one medium-complexity Figma frame link. Load `figma-design-to-code` before `get_design_context`.

- Build the screen following CLAUDE.md and DESIGN.md, using only existing components and tokens.
- Screenshot the result and compare it side by side with `get_screenshot` of the frame. Web: run the dev server and use Playwright if available. Android: emulator screenshot (`adb exec-out screencap -p`) or a Compose Preview. iOS: Simulator screenshot (`xcrun simctl io booted screenshot`, macOS only). If no screenshot is possible, ask the user to send one.
- Report: hardcoded values, components rebuilt instead of reused, missing states, visual differences, and platform-convention breaks (arbitrary Tailwind values, raw `Color(0xFF…)`, fixed iOS font sizes). For each problem, say where the fix belongs: **code, DESIGN.md, Code Connect, or the Figma file**.
- Offer to apply the fixes (**APPROVAL GATE**).

Then agree on a sync routine and write it into `.design-handoff/progress.md`:
- variable change in Figma → update code syntax for every platform, re-export, regenerate, PR
- new or changed component → Code Connect mapping + DESIGN.md entry
- rule change → PR to DESIGN.md reviewed by the designer

Give the user this re-sync prompt for later:
> Variables changed in Figma [link]. Compare them to tokens/tokens.json, show me what's added, renamed, or changed, check the code syntax for each platform, then regenerate the token files and list every component affected.

---

## Finishing

End with a short summary: platform(s), each step's status (done / skipped / waiting on codebase), key before/after numbers, open designer decisions, and the one next thing to do. Keep it short; the progress file has the detail.
