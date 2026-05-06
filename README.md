# My Cursor Skills

Personal skills backup. Single source lives at `~/.cursor/skills/` on each machine and is shared with Claude Code via symlinks (same `SKILL.md` format works for both).

## Setup on a new machine

```bash
# 1. Clone the repo as the cursor skills directory
mkdir -p ~/.cursor
gh repo clone EllaLiu0401/my-cursor-skills ~/.cursor/skills

# 2. Symlink each skill into ~/.claude/skills/ so Claude Code picks them up too
mkdir -p ~/.claude/skills
for d in ~/.cursor/skills/*/; do
  ln -s "$d" ~/.claude/skills/"$(basename "$d")"
done
```

Edit a skill once in `~/.cursor/skills/` — Cursor and Claude Code both see the change immediately, no drift.

## Sync changes

```bash
cd ~/.cursor/skills
git add .
git commit -m "update"
git push
```

Pull latest on another machine:

```bash
cd ~/.cursor/skills && git pull
```

If you add a brand-new skill directory, re-run the symlink loop above to expose it to Claude Code.
