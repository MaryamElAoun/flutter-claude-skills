# Flutter Claude Skills

A collection of [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that teach Claude how to build, review and test Flutter apps.

| Skill | What it does |
|---|---|
| [`flutter-riverpod`](skills/flutter-riverpod/SKILL.md) | Builds features with Riverpod (v2/v3, codegen or manual) in a feature-first architecture, with ready patterns for notifiers, async state, pagination and common error fixes. |
| [`flutter-firebase`](skills/flutter-firebase/SKILL.md) | FlutterFire setup, Auth, Firestore data layers (converters, pagination, transactions), FCM push notifications, security rules, emulators and Crashlytics. |
| [`flutter-review-testing`](skills/flutter-review-testing/SKILL.md) | Structured code review checklist (bugs, performance, architecture, security, a11y) plus templates for unit, notifier, widget, golden and integration tests and a CI workflow. |

## Install

### Claude Code

```bash
git clone https://github.com/MaryamElAoun/flutter-claude-skills.git
# personal (all projects)
cp -r flutter-claude-skills/skills/* ~/.claude/skills/
# or project-only
cp -r flutter-claude-skills/skills/* .claude/skills/
```

### Claude apps (claude.ai / desktop / mobile)

Zip a single skill folder (e.g. `flutter-riverpod/`) and upload it from **Settings → Capabilities → Skills**.

## Structure

```
skills/
  <skill-name>/
    SKILL.md          # instructions + frontmatter (name, description)
    references/       # longer examples loaded only when needed
```

## Contributing

Issues and PRs are welcome. Keep `SKILL.md` short and focused; put long code samples in `references/`.

## License

[MIT](LICENSE)
