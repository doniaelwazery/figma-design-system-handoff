---
name: "figma-design-system-handoff"
description: "Guide a designer step by step through handing off a Figma design system to developers with Claude Code: tokens, components, token export, Code Connect, DESIGN.md, testing. Asks for the Figma link first and gets approval before every change."
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
4. **Design decisions belong to the designer.** When something is a judgment call (which duplicate to keep, a prop name, whether a missing token is intentional), present 2–3 options with your recommendation and let them choose. Never guess silently.
5. **Never break bindings.** Rename variables and components in place (Figma links by ID), never delete-and-recreate.
6. **Verify after every change.** Re-read the file and report real before/after numbers.
7. **Track progress** in `.design-handoff/progress.md` in the project (create it at the start). After each step, record: what was done, decisions the user made, open items. If a session restarts, read this file first and resume from the last unfinished step.

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

Then ask (one `AskUserQuestion` call, up to 3 questions):
- **Is this a copy, or the live file?** Recommend working on a duplicate for the first run.
- **Is there a codebase yet?** (Yes, in this repo / Yes, elsewhere / Not yet). If yes, ask for its token format (CSS variables, Tailwind, other), because code names must match existing code.
- **Which steps?** All 6 (recommended), or a subset.

If there's no codebase, Steps 1, 2 and 5 still work, and Steps 3, 4 and 6 wait until one exists. Say this clearly.

Run a quick read-only inventory (pages, variable collections with counts and modes, text/effect styles, component sets per page) and summarize it in 3–5 lines before Step 1.

---

## Step 1: Make the tokens machine-readable

**Why (tell the user):** Claude Code reads variable *code syntax* to know which CSS name to use. Without it, it falls back to hex values.

**Audit (read-only)** for every variable collection. Report in a table:
- variables with no Web code syntax
- inconsistent names: mixed casing, spaces, leftover suffixes like ` 2`, duplicated words (`Font Family/Font Family`)
- variables with `ALL_SCOPES`
- uneven token sets (e.g. one button type has `bg/text/border/hover/active`, another is missing some)
- primitive/semantic structure: do semantic tokens alias primitives? Are there Light/Dark modes?

Keep `use_figma` return values small (summaries and counts, not full dumps); the return limit is ~20KB.

**Propose:**
- Rename to lowercase kebab-case, keeping existing groups. Check for name collisions before renaming.
- Web code syntax on **every** variable, using the `var()` wrapper: `color/text/primary` → `var(--color-text-primary)`. Prefix primitives with `color-` if their names don't already say color. If the codebase has existing CSS names, match those instead.
- Scopes: primitives hidden (`[]`); backgrounds `FRAME_FILL, SHAPE_FILL`; text `TEXT_FILL`; borders `STROKE_COLOR`; spacing `GAP` (+ padding); radius `CORNER_RADIUS`; font family `FONT_FAMILY`; font weight `FONT_STYLE`. Match whatever pattern the file already uses consistently.
- Flag uneven token sets as **designer decisions**, not auto-fixes.

Show the full rename list and a sample of code syntax. **APPROVAL GATE.**

**Apply**, then verify, and report a before/after table:
| | Before | After |
|---|---|---|
| Variables with code syntax | x / N | N / N |
| Inconsistent names | x | 0 |
| Variables on "All" scopes | x | 0 |
| Bindings broken | n/a | 0 |

Note anything code needs that Figma can't hold (e.g. font weights stored as style names: code must map regular 400, medium 500, semibold 600, bold 700). Update progress file. Ask: continue to Step 2?

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

Present a summary table of all components ranked worst first, then propose a **shared state vocabulary** for the whole system (e.g. `default, hover, focus, active, disabled, error`) and get it approved once.

Then go **one component at a time**, worst first unless the user prefers otherwise. For each, propose:
- property renames (`Property 1` → `state`, `Type` → `variant`) and value cleanup
- which duplicate to keep (show screenshots of each with `get_screenshot`; this is a designer decision)
- binding hardcoded values to the closest existing token; list ones with **no close match** instead of guessing
- a one-line description: "Use for… Don't use for…"

**APPROVAL GATE per component.** Don't delete duplicates; propose moving the unused ones to an `Archive` page and get explicit approval. After each component, take one screenshot to confirm nothing visually changed. Update progress after each component. Let the user stop and resume anytime.

---

## Step 3: Export tokens into the codebase

Requires a repo. **Why:** the codebase needs its own copy of the values so developers and Claude use the same names.

**Inspect the repo first:** framework, existing styles, Tailwind config, existing CSS variables, package manager.

**Propose a file plan** (paths + what each holds):
- `tokens/tokens.json` in W3C DTCG format, keeping primitive → semantic aliases
- a Style Dictionary config producing `src/styles/tokens.css`: names exactly match each variable's Figma code syntax; Light on `:root`, Dark under `[data-theme="dark"]`; responsive modes (Desktop/iPad/Mobile) as media queries (ask for breakpoints)
- optional Tailwind theme mapping to the CSS variables
- npm script `tokens:build`

Read variables with the Figma MCP (`get_variable_defs`, or a read-only `use_figma` script for full collections and modes). The Variables REST API needs Enterprise, so don't depend on it.

**APPROVAL GATE** on the file plan, then write the files, install deps, run the build, and show a sample of the generated CSS. Verify every Figma code syntax name exists in the output (report any missing).

---

## Step 4: Connect Figma components to code (Code Connect)

Requires components in the repo. Load the `figma-code-connect` skill.

- List Figma components (`list_file_components_for_code_connect` / `get_code_connect_suggestions`) and code components in the repo.
- Present a mapping table: Figma component → code file, plus a property mapping (Figma `state=disabled` → `disabled` prop).
- Flag Figma components with no code match and code components with no Figma match. Don't create new components.
- Start with the most-used components; ask the user which ones matter most.

**APPROVAL GATE** on the mapping table, then publish mappings and confirm with `get_code_connect_map`.

---

## Step 5: Write the rulebook (DESIGN.md)

**Why:** rules Claude follows every session. It holds rules, not values (values live in tokens).

Draft `DESIGN.md` at the repo root (or the location the user picks) from the Figma file, the tokens, and the code. Don't invent rules. Mark anything inferred as `<!-- confirm -->`.

Sections:
1. Principles (3–5 lines; ask the user for these, they're design intent)
2. Tokens: point to `tokens/tokens.json`, never copy values; "semantic tokens only in components"
3. Layout & spacing: scale logic, responsive behavior
4. Components: for each, when to use, variants, required states, one do and one don't
5. Accessibility: contrast, focus states, minimum tap size
6. Do / Don't: top 10 mistakes

Short imperative rules an agent can follow. Under ~300 lines.

Also propose these lines for the top of `CLAUDE.md` (create it if missing, never overwrite existing content):
```
- Never hardcode colors, spacing, radius, or font sizes; use tokens from src/styles/tokens.css
- Reuse components from src/components before creating new ones
- Follow DESIGN.md for all UI work
```
Show both drafts. **APPROVAL GATE**, then write.

---

## Step 6: Test with a real screen, then set up syncing

Ask the user for one medium-complexity Figma frame link. Load `figma-design-to-code` before `get_design_context`.

- Build the screen following CLAUDE.md and DESIGN.md, using only existing components and tokens.
- Screenshot the result (run the dev server; use Playwright if available) and compare it side by side with `get_screenshot` of the frame.
- Report: hardcoded values, components rebuilt instead of reused, missing states, visual differences. For each problem, say where the fix belongs: **code, DESIGN.md, Code Connect, or the Figma file**.
- Offer to apply the fixes (**APPROVAL GATE**).

Then agree on a sync routine and write it into `.design-handoff/progress.md`:
- variable change in Figma → re-export, `tokens:build`, PR
- new or changed component → Code Connect mapping + DESIGN.md entry
- rule change → PR to DESIGN.md reviewed by the designer

Give the user this re-sync prompt for later:
> Variables changed in Figma [link]. Compare them to tokens/tokens.json, show me what's added, renamed, or changed, then run tokens:build and list every component affected.

---

## Finishing

End with a short summary: each step's status (done / skipped / waiting on codebase), key before/after numbers, open designer decisions, and the one next thing to do. Keep it short; the progress file has the detail.