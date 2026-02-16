# Skills Repository

Reusable Codex skills for code projects.

## Quick Start: Use a Skill in Your Project

Copy one skill:

```bash
mkdir -p <target-repo>/.agents/skills
cp -R <skills-repo>/skills/<skill-name> <target-repo>/.agents/skills/
```

Copy all skills:

```bash
mkdir -p <target-repo>/.agents/skills
cp -R <skills-repo>/skills/* <target-repo>/.agents/skills/
```

Prompt invocation example:

```text
Use $xctest-to-testing-migrator to convert this XCTest suite to Swift Testing.
```

## Skills

### `xctest-to-testing-migrator`

- Path: `skills/xctest-to-testing-migrator`
- Overview: Migrates XCTest-based tests to Swift Testing with struct-first defaults, assertion mapping guidance, lifecycle migration rules, and unsupported-feature fallback guidance.

## Sync Between Repos (Manual)

```bash
diff -ru \
  <product-repo>/.agents/skills/<skill-name> \
  <skills-repo>/skills/<skill-name>
```

Then copy changed files, validate, and commit in the repo you updated.
