# My Cursor Skills

Personal Cursor skills backup. Lives at `~/.cursor/skills/` on each machine.

## Setup on a new machine

```bash
mkdir -p ~/.cursor
gh repo clone EllaLiu0401/my-cursor-skills ~/.cursor/skills
```

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
