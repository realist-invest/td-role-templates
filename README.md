# td Role Templates

Role prompt templates for [td](https://github.com/realist-invest/td-dev), the task management CLI for AI coding agents.

## Usage

Templates are fetched automatically by `td role define` or explicitly with `td role seed`:

```bash
# Auto-seed latest template (if no local file exists)
td role define architect

# Fetch specific version
td role seed architect --version v1

# Pin to a release tag
td role seed architect --ref v0.1.0
```

Fetched templates are saved to `.todos/roles/<role>.md` in your project. Once saved, the local file is authoritative — td never overwrites it.

## Repository Structure

```
manifest.yaml          # Registry catalog — roles, versions, descriptions
architect/
  v1.md                # Initial architect prompt
  v2.md                # Stronger constraints, handoff-aware
implementer/
  v1.md                # Initial implementer prompt
reviewer/
  v1.md                # Initial reviewer prompt
tester/
  v1.md                # Initial tester prompt
```

### manifest.yaml

Single source of truth. Maps role names to available versions:

```yaml
schema: 1
roles:
  architect:
    description: "System design, no implementation"
    latest: v2
    versions:
      v1: { summary: "Initial architect prompt" }
      v2: { summary: "Stronger constraints, handoff-aware" }
```

- `schema` must be `1` (validated by td before parsing)
- `latest` points to the recommended version
- Each version has a `summary` for display in `td role templates`

## Adding a New Role

1. Create a folder: `myrole/`
2. Add the prompt file: `myrole/v1.md`
3. Update `manifest.yaml`:
   ```yaml
   roles:
     myrole:
       description: "What this role does"
       latest: v1
       versions:
         v1: { summary: "Initial myrole prompt" }
   ```
4. Commit and push

## Adding a New Version

1. Add the new file: `architect/v3.md`
2. Update `manifest.yaml`:
   ```yaml
   architect:
     latest: v3          # Update latest pointer
     versions:
       v1: { summary: "Initial architect prompt" }
       v2: { summary: "Stronger constraints, handoff-aware" }
       v3: { summary: "Description of what changed" }
   ```
3. Commit and push
4. Optionally tag for pinning: `git tag v0.2.0`

## Prompt Guidelines

- Keep prompts under 4 KB (td enforces this limit)
- Use these sections: Identity, Responsibilities, Constraints, Workflow, Handoff Behavior
- Reference td commands (`td start`, `td log`, `td handoff`, etc.) — the prompt is injected during `td usage`
- Be concrete and actionable — "never write code" is better than "focus on design"
- Role names must match `^[a-z0-9_-]{1,32}$`

## Releases

Tag releases so users can pin to a specific version:

```bash
git tag -a v0.1.0 -m "Initial role templates"
git push origin v0.1.0
```

Users pin with: `td role seed architect --ref v0.1.0`

