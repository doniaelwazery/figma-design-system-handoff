# Figma Design System Handoff

A [Claude Code](https://code.claude.com) skill that walks a designer, step by step, through handing off a Figma design system to developers in the AI era.

It asks for your Figma link and your platform first, checks everything read-only, and **asks for your approval before every change**.

## Platform-aware naming

Tokens get the names developers on your platform actually type:

| Platform | Convention | Example code syntax |
|---|---|---|
| Web | Tailwind CSS (v4 or v3) | `var(--color-primary)` → `bg-primary` |
| Android | Material Design 3 (Compose or XML Views) | `MaterialTheme.colorScheme.onSurface` |
| iOS | Apple Human Interface Guidelines (SwiftUI or UIKit) | `Color(.label)`, `Font.body` |

Pick one or several. Tokens that match a built-in role (an M3 color role, a HIG text style) map to it; everything else gets a custom name in the platform's style. Every mapping is shown to you for approval.

## The 6 steps

| # | Step | What happens |
|---|------|--------------|
| 1 | Make the tokens machine-readable | Audits variables, normalizes names, adds platform code syntax (Tailwind / Material 3 / HIG) and scopes |
| 2 | Make components code-shaped | Real prop names instead of `Property 1`, removes duplicates, binds hardcoded colors, adds usage notes |
| 3 | Export tokens into the codebase | `tokens.json` (W3C DTCG) → Tailwind theme, Compose `MaterialTheme`, or iOS asset catalog + Swift |
| 4 | Connect Figma to code | Code Connect maps each Figma component to its code component |
| 5 | Write the rulebook | `DESIGN.md` + hard rules in `CLAUDE.md` that Claude follows every session |
| 6 | Test and keep in sync | Builds one real screen, compares it to Figma, sets up a sync routine |

Steps 1, 2 and 5 work without a codebase. Steps 3, 4 and 6 need one.

## Requirements

- Claude Code
- The Figma plugin for Claude Code:
  ```
  claude plugin install figma@claude-plugins-official
  ```
  then run `/mcp` to sign in to Figma.
- A Figma seat with access to the Figma MCP server.

## Install

**Easiest: let Claude Code do it.** Open Claude Code in your project and paste:

```
Install the Claude Code skill from https://github.com/doniaelwazery/figma-design-system-handoff
into this project: copy the .claude/skills/figma-design-system-handoff folder
into this project's .claude/skills/. Then check that the Figma plugin is installed,
and install it if not: claude plugin install figma@claude-plugins-official
```

Then run `/reload-skills` (or restart Claude Code) and `/mcp` to sign in to Figma.
Want it in every project? Replace "this project's .claude/skills/" with "~/.claude/skills/".

**Manually, in one project** (shared with your team through git): copy `.claude/skills/figma-design-system-handoff/` into your project's `.claude/skills/` folder and commit it.

**For yourself in every project:** copy the same folder into `~/.claude/skills/`.

**In the Claude app:** zip the `figma-design-system-handoff` folder and upload it under **Customize → Skills**.

## Use

In Claude Code, run:
```
/figma-design-system-handoff
```
Then paste your Figma design system link when asked.

> Tip: run it on a **duplicate** of your Figma file the first time.

## How it keeps you in control

- Every step starts with a read-only audit.
- Before any change it shows exactly what will change and asks: **Approve / Change something / Skip**.
- Design decisions (which duplicate to keep, prop names, missing tokens) are always yours.
- Renames happen in place, so variable and component bindings never break.
- Progress is saved in `.design-handoff/progress.md`, so you can stop and resume anytime.

## License

MIT
