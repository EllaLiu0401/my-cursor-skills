---
name: ds-code-authoring
description: >-
  Conventions for writing component code in the Aetheron Design System
  (aetheron-design-system/packages/ui). Covers TypeScript prop patterns,
  DaisyUI class pitfalls, Storybook story patterns, and common review
  feedback. Use when creating or editing components, stories, or hooks inside
  the design system repo, or when the user mentions DS components, ChatComposer,
  DaisyUI classes, or Storybook stories.
---

# DS Code Authoring — Aetheron Design System

## Before Writing Code

1. Read `agent.md` in the DS repo root — it is the single source of truth.
2. Check existing components for patterns (`packages/ui/src/components/`).
3. Run `pnpm lint` before and after changes.

## TypeScript Prop Patterns

### Discriminated Unions for Coupled Props

When two props are logically coupled (one requires the other), use a discriminated union instead of flat optionals. This catches misuse at compile time.

```tsx
// BAD — silent footgun: isStreaming=true without onStop compiles fine
interface Props {
  isStreaming?: boolean;
  onStop?: () => void;
}

// GOOD — isStreaming: true forces onStop to be present
type StreamingProps =
  | { isStreaming?: false; onStop?: undefined; stopLabel?: undefined }
  | { isStreaming: true; onStop: () => void; stopLabel?: string };

export type ChatComposerProps = BaseProps & StreamingProps;
```

Use `undefined` (not `never`) for the excluded branch — `never` breaks destructuring defaults.

### Stories with Discriminated Unions

TypeScript cannot narrow a union via JSX spread. When a prop is state-driven (`boolean`), use conditional rendering to give TS a literal `true`:

```tsx
{streaming ? (
  <ChatComposer isStreaming onStop={stop} ... />
) : (
  <ChatComposer ... />
)}
```

### Standard Prop Conventions

| Pattern | Convention |
|---------|-----------|
| HTML passthrough | `extends ComponentPropsWithoutRef<'element'>` |
| Tone / color | `tone?: 'primary' \| 'secondary' \| 'danger'` |
| Size | `size?: 'sm' \| 'md' \| 'lg'` |
| Defaults | Destructure with defaults, not `defaultProps` |
| Class merging | `clsx('base-classes', className)` — consumer className always last |

## DaisyUI Class Pitfalls

### `btn-outline` Forces Primary Border Color

`btn-outline` applies the primary (blue) border color with higher specificity. Adding `border-base-300` does NOT override it.

```tsx
// BAD — border will be blue, not grey
className='btn btn-outline border-base-300'

// GOOD — btn-ghost has no border; manual border wins
className='btn btn-ghost border border-base-300'
```

### `cursor-not-allowed` on Parent Labels

When a `<label>` sets `cursor-not-allowed` (for disabled state), it bleeds onto all children, including nested clickable buttons. If a child button should remain clickable, add `cursor-pointer` directly on it.

### DaisyUI Shared CSS Variables

Changing `--radius-*`, `--size-*`, `--border`, `--depth`, or `--noise` in theme files affects multiple DaisyUI components. Always check consumers in `node_modules/daisyui/components/` before modifying.

## Storybook Stories

### Timer Cleanup

Always clear `setTimeout` / `setInterval` in stories:

```tsx
const timeoutRef = useRef<ReturnType<typeof setTimeout> | null>(null);

useEffect(() => {
  return () => { if (timeoutRef.current) clearTimeout(timeoutRef.current); };
}, []);

const start = () => {
  if (timeoutRef.current) clearTimeout(timeoutRef.current);
  timeoutRef.current = setTimeout(() => { /* ... */ }, 4000);
};

const stop = () => {
  if (timeoutRef.current) { clearTimeout(timeoutRef.current); timeoutRef.current = null; }
  setState(false);
};
```

### Play Functions for Behavioural Guarantees

Use Storybook play functions to verify interactive behaviour (not just visuals):

```tsx
import { expect, fn, userEvent, within } from 'storybook/test';

const mockSend = fn();

export const StreamingStatic: Story = {
  render: () => <ChatComposer onSend={mockSend} isStreaming onStop={...} />,
  play: async ({ canvasElement }) => {
    mockSend.mockClear();
    const canvas = within(canvasElement);
    const textarea = canvas.getByRole('textbox');
    await userEvent.type(textarea, 'hello');
    await userEvent.keyboard('{Enter}');
    expect(mockSend).not.toHaveBeenCalled();
  },
};
```

### Story Conventions

- Import `storybook/test` (not `@storybook/test`) — Storybook v9 re-export.
- Module-level `fn()` for mocks; call `mockClear()` inside `play`.
- One render function component per interactive story (`function StoryName() { ... }`).
- Static stories can use arrow functions directly.

## Icon Usage

Use short PascalCase form without the `Icon` prefix:

```tsx
<Icon name='ArrowUp' size='sm' />     // resolves to IconArrowUp
<Icon name='PlayerStopFilled' size='sm' /> // resolves to IconPlayerStopFilled
```

The `Icon` component silently returns `null` for unknown names — no runtime warning. Double-check icon names against `@tabler/icons-react` exports.

## Pre-Commit Checklist

- [ ] No `any`, no `ts-ignore`, no `eslint-disable`
- [ ] Props typed with discriminated unions where appropriate
- [ ] Stories clean up timers on unmount
- [ ] Interactive stories have play functions for key behaviours
- [ ] `pnpm lint` passes with zero warnings
- [ ] `pnpm build` succeeds
- [ ] DaisyUI class specificity verified (especially `btn-outline` vs `btn-ghost`)
