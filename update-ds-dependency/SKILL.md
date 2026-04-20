---
name: update-ds-dependency
description: Update a dependency in the local design system UI package (aetheron-design-system/packages/ui). Updates package.json, refreshes the pnpm lockfile, then rebuilds the DS so the app picks up the change. Use when the user wants to update, add, or remove a package in the design system.
---

# Update Design System Dependency

## Context

The local design system is a sibling repo at `../aetheron-design-system/`. The UI package lives at `../aetheron-design-system/packages/ui/`. After any dependency change, rebuild the DS so the updated `dist/` is picked up by this app.

## Workflow

### Update an existing dependency

1. Edit the version in `../aetheron-design-system/packages/ui/package.json`.
2. Run from the DS workspace root to update the lockfile:
   ```bash
   cd ../aetheron-design-system && pnpm install
   ```
3. Rebuild from this repo:
   ```bash
   task ds:build
   ```

### Add a new dependency

```bash
cd ../aetheron-design-system && pnpm --filter @aetheronhq/ui add <pkg>
```
Then `task ds:build`.

### Remove a dependency

```bash
cd ../aetheron-design-system && pnpm --filter @aetheronhq/ui remove <pkg>
```
Then `task ds:build`.

## Notes

- `dependencies` = runtime packages included in the built output.
- `devDependencies` = Storybook, Vite, TypeScript tooling — not bundled.
- Do NOT touch `peerDependencies` (react, react-dom) unless explicitly asked.
- Confirm build output ends with `Build success` before finishing.
