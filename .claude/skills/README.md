# Skills

Skills are reusable prompts that teach Claude specific patterns for NestJS and TypeScript development.

## Structure Convention

Each skill folder contains:

| File | Purpose | Audience |
|------|---------|----------|
| `SKILL.md` | Instructions for Claude | AI (loaded on demand) |
| `references/*.md` | Detailed patterns loaded only when relevant | AI (optional) |

## Available Skills

### Framework & Data
| Skill | Description |
|-------|-------------|
| [nestjs](nestjs/) | NestJS 12 development: REST APIs, TypeORM, security, testing, cloud-native patterns |
| [typeorm-patterns](typeorm-patterns/) | TypeORM pitfalls: N+1, relation loading, transactions, locking, pagination |
| [logging-patterns](logging-patterns/) | Structured JSON logging, request IDs, AI-friendly log formats |

### Code Quality
| Skill | Description |
|-------|-------------|
| [code-quality](code-quality/) | TypeScript/NestJS code review: clean code, API contracts, type/null safety, async, performance |

### Architecture & Design
| Skill | Description |
|-------|-------------|
| [design-patterns](design-patterns/) | Factory, Builder, Strategy, Observer, Decorator, Adapter with TypeScript and NestJS DI |
| [hexagonal-architecture](hexagonal-architecture/) | Ports & Adapters for NestJS: bounded contexts, pure domain, use cases, adapters, enforced dependency rules |

## Adding a New Skill

### Before You Start

Validate your skill idea against existing skills:

- [ ] **No significant overlap** - Check the tables above for similar skills
- [ ] **Clear level** - Micro (functions) / Meso (classes) / Macro (modules) / Framework / Cross-cutting
- [ ] **Clear type** - Audit (review existing code) or Template (show how to write)
- [ ] **Unique value** - What does it add that doesn't exist?
- [ ] **Focused scope** - Can be applied in one session (<15 checklist items)

### Implementation Steps

1. Create folder: `.claude/skills/<skill-name>/`
2. Create `SKILL.md` with `name` and `description` frontmatter and instructions for Claude
3. Move long, topic-specific content into `references/` and link it from `SKILL.md`
4. Update this table
5. Update main README.md

## Usage

Skills are automatically loaded by Claude Code based on context. You can also invoke them directly:

```bash
# Automatic - Claude detects when to use skills
> "Add a products REST API"          # Loads nestjs
> "This endpoint runs 200 queries"   # Loads typeorm-patterns
> "Review this PR"                   # Loads code-quality
> "Add a transfer use case"         # Loads hexagonal-architecture (hexagonal projects)

# Manual - invoke with slash command
> /nestjs
> /code-quality
```

## Learn More

- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills) - Official guide on creating and using skills
- [NestJS Documentation](https://docs.nestjs.com) - Official NestJS docs
