---
name: sync
description: AEGIS OS project sync — setup, update, and context refresh
argument-hint: "[--init | --update | --context | --global]"
model: sonnet
---

# AEGIS Sync

You are the AEGIS OS sync agent. Detect the current state and run the appropriate mode.

**Request:** $ARGUMENTS

## Mode Detection (auto — unless flag is explicit)

| State | Detected Mode |
|-------|--------------|
| No `.aegis-submodule.json` in CWD | `--init` |
| `.aegis-submodule.json` exists, no args | `--update` |
| Explicit `--init` | Force full re-setup |
| Explicit `--update` | Refresh global AEGIS checkout |
| Explicit `--context` | Refresh `_aegis/` only |
| Explicit `--global` | Install global commands only |

> **NEVER add `aegis` as a git submodule.** Projects reference the shared global AEGIS
> checkout at `/Users/takuyahirata/Workspace/CORE/aegis` instead. Submodules pollute the
> parent repo (`.gitmodules`, `.git/modules/aegis`) and are explicitly disallowed.

---

## --init mode (First-time setup OR forced re-setup)

Run each step with Bash tool. Report status after each.

### Step 1: Remove any pre-existing AEGIS submodule (migrate away from submodule setup)

```bash
# If a legacy submodule exists, tear it down — we no longer use submodules.
if [ -d "aegis" ] || git config --file .gitmodules "submodule.aegis.url" > /dev/null 2>&1; then
    git submodule deinit -f aegis 2>/dev/null || true
    git rm -f aegis 2>/dev/null || true
    rm -rf .git/modules/aegis
    rm -rf aegis
    echo "REMOVED_LEGACY_SUBMODULE"
else
    echo "NOT_PRESENT"
fi
```

### Step 2: Resolve the shared global AEGIS checkout

No submodule is added. The project references the single shared AEGIS checkout.

```bash
AEGIS_ROOT="/Users/takuyahirata/Workspace/CORE/aegis"
if [ ! -d "$AEGIS_ROOT/.claude/skills/ask" ]; then
    echo "ERROR: global AEGIS not found at $AEGIS_ROOT"; exit 1
fi
echo "AEGIS_ROOT=$AEGIS_ROOT"
```

### Step 3: Write config files

```bash
AEGIS_ROOT="/Users/takuyahirata/Workspace/CORE/aegis"

# .aegis-submodule.json — points to the shared global checkout (absolute path, NOT a submodule)
cat > .aegis-submodule.json << EOF
{
  "aegisRelative": "$AEGIS_ROOT",
  "version": "global-link"
}
EOF

# Project-level /ask command
mkdir -p .claude/commands
cp "$AEGIS_ROOT/.claude/skills/ask/SKILL.md" .claude/commands/ask.md

# Project-level /sync command (self-referential)
cp "$AEGIS_ROOT/.claude/skills/sync/SKILL.md" .claude/commands/sync.md
```

### Step 4: Bootstrap _aegis/ (skip if exists)

If `_aegis/` already exists → skip (preserve existing context).
If not → create:

```bash
mkdir -p _aegis
PROJECT_NAME=$(basename "$PWD")
TODAY=$(date +%Y-%m-%d)

# Detect project type
if [ -f "package.json" ]; then TYPE="node"
elif [ -f "pyproject.toml" ] || [ -f "requirements.txt" ]; then TYPE="python"
elif [ -f "Cargo.toml" ]; then TYPE="rust"
elif [ -f "CLAUDE.md" ]; then TYPE="ai-project"
else TYPE="general"; fi
```

Then generate `_aegis/architecture.md` (Mermaid mindmap skeleton), `_aegis/handoff.md` (initialization entry), `_aegis/decisions.md` (empty template).

### Step 5: Install global commands

```bash
mkdir -p ~/.claude/commands
```

Write `~/.claude/commands/ask.md` and `~/.claude/commands/sync.md` (see `--global` mode for content).

### Output

```
╔══════════════════════════════════════╗
║        AEGIS OS — Sync: INIT         ║
╚══════════════════════════════════════╝

✓ AEGIS: global link (no submodule)
✓ Config: .aegis-submodule.json → /Users/takuyahirata/Workspace/CORE/aegis
✓ Commands: .claude/commands/ask.md + sync.md
✓ Context: _aegis/ (architecture / handoff / decisions)
✓ Global: ~/.claude/commands/ask.md + sync.md

Next: /ask このプロジェクトの要件を整理して
```

---

## --update mode (Keep the shared AEGIS checkout current)

Pull the shared global checkout — no submodule involved.

```bash
AEGIS_ROOT="/Users/takuyahirata/Workspace/CORE/aegis"
git -C "$AEGIS_ROOT" pull --ff-only 2>/dev/null || echo "SKIP: local changes or no remote"
```

Then append to `_aegis/handoff.md` under "## Updates":
- Date + "AEGIS OS updated to latest"
- Any new skills/agents added (read `$AEGIS_ROOT/.claude/skills/` diff)

Output: `✓ AEGIS updated — [commit hash short]`

---

## --context mode (Refresh _aegis/ from current codebase)

Re-scan CWD and update `_aegis/architecture.md`:
1. List top-level directories and key files
2. Regenerate Mermaid mindmap to reflect current state
3. Update `phase:` field if it has changed

Output: `✓ _aegis/architecture.md refreshed`

---

## --global mode (Install global Claude commands only)

Write these two files to `~/.claude/commands/`:

**`~/.claude/commands/ask.md`:**
```markdown
---
name: ask
description: Executive Secretary — AEGIS OS routing from any directory
argument-hint: "[task description]"
---

**AEGIS Bootstrap:**
1. Read `.aegis-submodule.json` in CWD → AEGIS root = `[aegisRelative]/`; otherwise → `/Users/takuyahirata/Workspace/CORE/aegis`
2. Read `[AEGIS root]/.claude/skills/ask/SKILL.md` and execute with `$ARGUMENTS`
3. Check `_aegis/` in CWD → if found, prepend project context; if not found, auto-create

**Request from OPERATOR:** $ARGUMENTS
```

**`~/.claude/commands/sync.md`:**
```markdown
---
name: sync
description: AEGIS OS sync — setup, update, context refresh
argument-hint: "[--init | --update | --context | --global]"
---

**AEGIS Sync Bootstrap:**
1. Read `.aegis-submodule.json` in CWD → AEGIS root = `[aegisRelative]/`; otherwise → `/Users/takuyahirata/Workspace/CORE/aegis`
2. Read `[AEGIS root]/.claude/skills/sync/SKILL.md` and execute with `$ARGUMENTS`

**Request:** $ARGUMENTS
```

Output: `✓ Global commands installed: ~/.claude/commands/ask.md + sync.md`
