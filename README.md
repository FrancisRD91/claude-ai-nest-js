# Claude Code Template for NestJS Application

This template provides a structured starting point for NestJS applications, optimized for Claude AI's code generation capabilities. It includes essential configurations, agents, skills and best practices to streamline development and enhance productivity.

The idea behind this template is that you can just clone this repository and use it to generate the app you want with Claude Code.

```shell
git clone https://github.com/FrancisRD91/claude-ai-nest-js.git my-service
cd my-service
claude
> Create a REST API for managing products and orders with PostgreSQL
```

## Default Stack

| Concern | Choice |
|---------|--------|
| Runtime | Node.js Active LTS (24+) |
| Framework | NestJS 12 (ES modules) |
| Language | TypeScript (`strict`) |
| Package manager | npm |
| Data access | TypeORM 1.x + PostgreSQL |
| Tests | Vitest, supertest, Testcontainers |
| CI | CircleCI |
| Local run | Docker Compose |

## Structure

```shell
.
├── .claude
│   ├── agents
│   │   ├── code-reviewer.md
│   │   ├── devops-engineer.md
│   │   ├── docker-expert.md
│   │   ├── kubernetes-specialist.md
│   │   ├── nestjs-engineer.md
│   │   └── security-engineer.md
│   ├── settings.local.json
│   └── skills
│       ├── README.md
│       ├── code-quality
│       │   └── SKILL.md
│       ├── design-patterns
│       │   └── SKILL.md
│       ├── hexagonal-architecture
│       │   ├── SKILL.md
│       │   └── references
│       │       ├── adapters.md
│       │       ├── consistency.md
│       │       ├── enforcement.md
│       │       ├── orchestration.md
│       │       ├── pitfalls.md
│       │       ├── shared-kernel.md
│       │       └── testing.md
│       ├── logging-patterns
│       │   └── SKILL.md
│       ├── nestjs
│       │   ├── SKILL.md
│       │   └── references
│       │       ├── cloud.md
│       │       ├── data.md
│       │       ├── security.md
│       │       ├── testing.md
│       │       └── web.md
│       └── typeorm-patterns
│           └── SKILL.md
├── .claude-plugin
│   └── plugin.json
├── CLAUDE.md
├── LICENSE
├── README.md
└── package.json
```

- **CLAUDE.md** - workflow rules and project conventions Claude follows on every task
- **agents** - specialized sub-agents (NestJS implementation, code review, security, Docker, Kubernetes, DevOps)
- **skills** - on-demand knowledge for NestJS, hexagonal architecture, TypeORM, logging, code quality and design patterns
- **package.json** - project name and version seed; Claude extends it when generating the app

## Credits

Adapted from [claude-ai-spring-boot](https://github.com/piomin/claude-ai-spring-boot) by Piotr Mińkowski, licensed under the [Apache License 2.0](LICENSE).
