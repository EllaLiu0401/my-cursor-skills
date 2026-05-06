---
name: update-ds-dependency
description: Update design system dependencies. Covers two workflows - (A) modify a dependency INSIDE the local DS package (aetheron-design-system/packages/ui), and (B) upgrade @aetheronhq/ui to a published version in the consumer app (apps/web). Use when the user wants to update/add/remove a package in the design system, OR bump @aetheronhq/ui to a new version like "ui-v3.24.0".
---

# Update Design System Dependency

## Context

Two distinct workflows:
- **Workflow A (DS-internal)**: change a dependency inside `../aetheron-design-system/packages/ui/`, then rebuild DS so the updated `dist/` is picked up. Use when the user wants to add/update/remove a package that the DS itself depends on.
- **Workflow B (consumer upgrade)**: bump `@aetheronhq/ui` in `apps/web/package.json` to a version already published to GitHub Packages. Use when the user says things like "update DS to ui-v3.24.0" — they want the app to consume a new release, not to modify the DS source.

## Workflow A — DS-internal dependency changes

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

## Workflow B — Upgrade `@aetheronhq/ui` in the app

Use when the user asks to bump the DS version consumed by `apps/web` (e.g. "update to ui-v3.24.0"). This is the most common case.

1. **Verify you're on the intended branch first.** `git status && git branch --show-current`. If the user switched branches mid-task (which happens), re-read `apps/web/package.json` — don't trust prior state.
2. **Check current version and confirm the target exists on GitHub Packages:**
   ```bash
   grep '@aetheronhq/ui' apps/web/package.json
   npm view @aetheronhq/ui versions --registry https://npm.pkg.github.com | tail
   ```
3. **Bump the version** in `apps/web/package.json` (prefer `^X.Y.Z` caret range, consistent with other deps). `@aetheronhq/ui` is only consumed by `apps/web` — no other package.json needs updating.
4. **Refresh the lockfile** (do NOT run `task ds:build` — that's for Workflow A):
   ```bash
   pnpm install --filter @aetheronhq/web
   ```
5. **Type-check for breaking changes** (DS minor bumps often remove exports):
   ```bash
   pnpm --filter @aetheronhq/web exec tsc --noEmit
   ```
6. **Migrate any removed APIs in the same commit.** Look up the replacement in `node_modules/@aetheronhq/ui/dist/index.d.ts`:
   ```bash
   grep -nE 'export \{|declare const <NewName>' node_modules/@aetheronhq/ui/dist/index.d.ts
   ```
   Examples of past breaking changes:
   - `ChatSkeleton` → `ChatLoading` (props changed: `lines={N}` → `label`/`size`).
7. **Commit package.json + pnpm-lock.yaml + migration edits together** so the repo is never in a broken state. Suggested subject: `chore(deps): upgrade @aetheronhq/ui to ^X.Y.Z`.

Ignore pre-existing TS errors unrelated to the upgrade (e.g. stale `@aetheron/api-client` exports waiting on codegen) — confirm they exist at HEAD before the bump with `git stash && tsc --noEmit; git stash pop`.

## Notes

- `dependencies` = runtime packages included in the built output.
- `devDependencies` = Storybook, Vite, TypeScript tooling — not bundled.
- Do NOT touch `peerDependencies` (react, react-dom) unless explicitly asked.
- Workflow A: confirm build output ends with `Build success` before finishing.
- Workflow B: `ReadLints` may show stale errors from TS Server cache after an upgrade — trust the CLI `tsc --noEmit` result.
