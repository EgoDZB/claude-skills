# claude-skills

Personal [Claude Code](https://claude.com/claude-code) skills, packaged as a plugin marketplace.

## Install

```sh
claude plugin marketplace add EgoDZB/claude-skills
claude plugin install egodzb-skills@egodzb
```

Or from inside Claude Code:

```
/plugin marketplace add EgoDZB/claude-skills
/plugin install egodzb-skills@egodzb
```

## Update

```sh
claude plugin marketplace update egodzb
claude plugin update egodzb-skills@egodzb
```

## Skills

| Skill                                                          | Description                                                                                                       |
|----------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| [`postgres-conventions`](skills/postgres-conventions/SKILL.md) | PostgreSQL naming and schema conventions for .NET/EF Core, Node.js (Drizzle, Prisma, TypeORM, Knex) and Go (sqlc) |
| [`union-types-net11`](skills/union-types-net11/SKILL.md)       | C# 15 union types in .NET 11: declarations, conversions, exhaustive matching, custom non-boxing unions             |
| [`async-validation-net11`](skills/async-validation-net11/SKILL.md) | .NET 11 async validation: DataAnnotations, Minimal APIs, Blazor and the Options pattern                       |

## Adding a skill

1. Create `skills/<skill-name>/SKILL.md` with `name` and `description` frontmatter.
2. Bump `version` in `.claude-plugin/plugin.json`.

## License

[MIT](LICENSE)
