# Skills Repository

Centralized repository for Codex skills that can be reused across projects and teams.

## Current Skills

- `xctest-to-testing-migrator` at `skills/xctest-to-testing-migrator`

## Repository Layout

```text
skills/
  <skill-name>/
    SKILL.md
    agents/
      openai.yaml
    scripts/      (optional)
    references/   (optional)
    assets/       (optional)
```

## Generic Usage Examples

Copy one skill into another repository:

```bash
cp -R <skills-repo>/skills/<skill-name> <target-repo>/.agents/skills/
```

Copy all skills into another repository:

```bash
mkdir -p <target-repo>/.agents/skills
cp -R <skills-repo>/skills/* <target-repo>/.agents/skills/
```

Use a skill in prompts by name:

```text
Use $xctest-to-testing-migrator to convert this XCTest suite to Swift Testing.
```

## Generic Sync Workflow (Manual Either-Way)

1. Decide which copy is the source of latest changes.
2. Compare folders before syncing:
   `diff -ru <source-skill-path> <destination-skill-path>`
3. Copy changed files from source to destination.
4. Validate the destination skill.
5. Commit in the repository that was updated.

Example comparison:

```bash
diff -ru \
  <product-repo>/.agents/skills/<skill-name> \
  <skills-repo>/skills/<skill-name>
```

## Validation

Validate a migrated or updated skill:

```bash
python3 <path-to-skill-creator>/scripts/quick_validate.py \
  <skills-repo>/skills/<skill-name>
```

Example with Codex system skill path:

```bash
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py \
  ./skills/<skill-name>
```

## Conventions

- Use hyphen-case folder names for skills.
- Keep execution guidance in `SKILL.md`.
- Keep repository-level docs in this root `README.md`.
- Keep `agents/openai.yaml` aligned with `SKILL.md` metadata and intent.
