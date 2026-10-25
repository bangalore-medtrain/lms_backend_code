# Project instructions

MedTrain multi-tenant AI LMS backend, started from a NestJS boilerplate. Postgres (TypeORM) is the only supported database; the boilerplate's MongoDB (document) variant stays in the repository, frozen and local-only. The full project rules are in [.claude/CLAUDE.md](.claude/CLAUDE.md), with [foundation.md](.claude/foundation.md), [plan.md](.claude/plan.md) and [system-design.md](.claude/system-design.md) as the source of truth.

## When adding entities, schemas, or properties

Use the `generate` skill (auto-loaded from [.claude/skills/generate/SKILL.md](.claude/skills/generate/SKILL.md)) with the relational generators only (`npm run generate:resource:relational`, `npm run add:property:to-relational`), never `all-db` or `document`. Do not hand-write entity files.
