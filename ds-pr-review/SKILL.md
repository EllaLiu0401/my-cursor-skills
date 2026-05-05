---
name: ds-pr-review
description: >-
  Review pull requests for the Aetheron Design System
  (aetheron-design-system). Checks component code, DaisyUI usage,
  TypeScript patterns, Storybook stories, a11y compliance, and theming
  correctness. Use when reviewing a PR in the design system repo, or
  when the user shares a DS PR URL and asks to review it.
---

# DS PR Review — Aetheron Design System

## Step 1: Gather Context (parallel)

```bash
gh pr view <number> --json title,body,headRefName,files
gh api repos/aetheronhq/aetheron-design-system/pulls/<number>/comments
gh api repos/aetheronhq/aetheron-design-system/pulls/<number>/reviews
gh api repos/aetheronhq/aetheron-design-system/issues/<number>/comments
gh pr diff <number>
```

Switch to the PR branch before reading code:
```bash
git checkout <headRefName> && git pull origin <headRefName>
```

Filter out bots: `coderabbitai[bot]`, `github-actions[bot]`, any `"type": "Bot"`.

## Step 2: Read Project Rules

**Always read `agent.md`** — single source of truth for:
- Core philosophy (minimal wrapper DS, DaisyUI v5 CSS classes, no react-daisyui)
- Approved component list
- TypeScript strictness requirements
- Color architecture (Primitive → Semantic → Component)
- File naming conventions
- Accessibility requirements (WCAG 2.1 AA)

**Then based on scope**, read relevant docs:

| PR touches... | Read |
|---------------|------|
| Colors / theming | `docs/foundations/color-system.md`, `docs/guides/multi-brand-theming.md` |
| Typography | `docs/foundations/typography.md` |
| Layout / spacing | `docs/foundations/spacing-layout.md` |
| New component | `docs/CONTRIBUTING.md` |
| CSS classes | `docs/foundations/css-class-reference.md` |
| Form components | `docs/patterns/form-patterns.md` |

## Step 3: Review Checklist

### Component Code (`*.tsx`)

| Check | What to look for |
|-------|-----------------|
| **Prop types** | Discriminated unions for coupled props (e.g., `isStreaming: true` requires `onStop`). No flat optional footguns. Use `undefined` not `never` for excluded branches. |
| **No `any`** | Strict TypeScript — no `any`, `ts-ignore`, `eslint-disable`. |
| **HTML passthrough** | `extends ComponentPropsWithoutRef<'element'>`, spread `...rest` last. |
| **Class merging** | `clsx('base', variantClass, className)` — consumer `className` always wins. |
| **DaisyUI specificity** | `btn-outline` forces primary border color; `border-base-300` cannot override. Use `btn-ghost border border-base-300` for grey borders. |
| **Cursor bleed** | `cursor-not-allowed` on parent `<label>` bleeds to child buttons. Add `cursor-pointer` on nested clickable elements if needed. |
| **forwardRef** | Components wrapping a single DOM element should forward ref. |
| **Minimal logic** | Components are thin wrappers. No heavy business logic. |
| **YAGNI** | Don't add defensive checks for hypothetical edge cases. Only add guards for observed production issues. |

### DaisyUI & Theming

| Check | What to look for |
|-------|-----------------|
| **No react-daisyui** | All components use native HTML + DaisyUI CSS classes. |
| **Color layers** | Never use `--primitive-*` directly. Use `--semantic-*` or DaisyUI class names. |
| **Shared CSS vars** | Changes to `--radius-*`, `--size-*` affect multiple DaisyUI components. Verify cross-component impact. |
| **Dark mode** | If adding colours, ensure semantic tokens flip correctly in dark mode. |

### Storybook Stories (`*.stories.tsx`)

| Check | What to look for |
|-------|-----------------|
| **Timer cleanup** | `setTimeout`/`setInterval` must be cleared — `useRef` + `clearTimeout` on stop, re-trigger, and `useEffect` cleanup. Stale timers cause state updates on unmounted components. |
| **Play functions** | Interactive guarantees (e.g., "Enter is blocked during streaming") need `play` functions, not just visual stories. |
| **Import path** | Use `storybook/test` (Storybook v9), not `@storybook/test`. |
| **Module-level mocks** | `fn()` at module level; `mockClear()` inside `play`. |
| **Union narrowing** | Discriminated union props can't be spread via args — use conditional render or `as const`. |

### Icons

| Check | What to look for |
|-------|-----------------|
| **Short form** | `name="ArrowUp"` not `name="IconArrowUp"`. |
| **Existence** | `Icon` silently returns `null` for unknown names. Verify against `@tabler/icons-react` exports. |

### Accessibility

| Check | What to look for |
|-------|-----------------|
| **WCAG 2.1 AA** | All components must pass Level AA. |
| **Touch targets** | ≥44px for all interactive elements. |
| **aria-label** | Icon-only buttons must have `aria-label`. |
| **Keyboard** | All interactive elements reachable via keyboard. Focus visible. |
| **Screen reader** | State changes (e.g., send → stop button) need discoverable feedback. |

### General Quality

| Check | What to look for |
|-------|-----------------|
| **File naming** | PascalCase for components, camelCase for utils, kebab-case for folders. |
| **Barrel exports** | New components exported from category `index.ts` and root `index.ts`. |
| **No dead code** | Remove unused imports, unreachable branches, commented-out code. |
| **Backward compat** | New optional props should not break existing consumers. |

## Step 4: Severity Classification

| Category | Criteria | Action |
|----------|----------|--------|
| **Bug** | Incorrect behaviour, type unsafety, stale timer | Must fix |
| **Contract mismatch** | Props/docs contradiction, silent footgun | Must fix |
| **Valid improvement** | DRY, specificity, consistency — simple fix | Recommend fix |
| **A11y gap** | Missing aria, keyboard trap, no feedback | Recommend fix |
| **Test gap** | Key behaviour not covered by play/test | Recommend fix |
| **Nitpick** | Style preference, hypothetical concern | Skip or note |

## Step 5: Present Summary

Present analysis before making any changes. Format per comment:

```markdown
**Comment N: {title}**
> {quote}
- Category: {Bug / Valid improvement / Nitpick}
- Analysis: {assessment}
- Recommendation: {Fix / Skip}
```

Summary table at the end. Ask for confirmation before implementing fixes.

## Step 6: Post Review to GitHub

Use `gh api` to post as a single review with inline comments:

```bash
COMMIT=$(gh api repos/aetheronhq/aetheron-design-system/pulls/<number> --jq '.head.sha')

gh api repos/aetheronhq/aetheron-design-system/pulls/<number>/reviews \
  --method POST \
  --field commit_id="$COMMIT" \
  --field event="COMMENT" \
  --field 'body=Review summary' \
  --input - <<'PAYLOAD'
{ "comments": [{ "path": "...", "line": N, "side": "RIGHT", "body": "..." }] }
PAYLOAD
```

### Reply Conventions

- **Before / After** format for fixed items.
- Concise, polite, logical. Reference `agent.md` rules when relevant.
- For skipped items, explain reasoning clearly.

## Verification Commands

```bash
pnpm lint          # ESLint + Prettier
pnpm build         # TypeScript build
pnpm test:a11y     # Accessibility tests (requires Storybook running)
```
