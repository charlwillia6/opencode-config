# OpenCode Configuration

Forked From: https://github.com/flpbalada/my-opencode-config

My personal [OpenCode](https://opencode.ai/docs) configuration optimized for full-stack web development with autonomous agent workflows.

Feel free to:
- Browse and adapt skills for your own config
- Use agents as templates for your workflows
- Share tips and tricks you've found useful

## Structure

```
.
├── AGENTS.md          # Personal preferences & coding guidelines
├── opencode.json      # Main config (plugins, MCP servers, providers)
├── agents/            # Custom agents for specialized tasks
├── skills/            # executable knowledge notes
├── commands/          # Custom slash commands
└── scripts/           # Helper scripts 
```

## Custom Agents

| Agent | Purpose |
|-------|---------|
| [`architect`](./agents/architect.md) | High-level planning and architecture (Tier 1) |
| [`coder-monitor`](./agents/coder-monitor.md) | Autonomous build validation (Tier 3) |
| [`code-reviewer`](./agents/code-reviewer.md) | Deep quality and security audit (Tier 4) |
| [`web-researcher`](./agents/web-researcher.md) | Internet research and documentation |
| [`requirements-analyzer`](./agents/requirements-analyzer.md) | Feature requirements analysis |
| [`deep-thinker`](./agents/deep-thinker.md) | Structured thinking for complex problems |
| [`effort-estimator`](./agents/effort-estimator.md) | Estimate development effort |
| [`refactoring`](./agents/refactoring.md) | Code restructuring and improvement |
| [`commit-drafter`](./agents/commit-drafter.md) | Generate conventional commit messages |
| [`code-simplifier`](./agents/code-simplifier.md) | Simplify code for clarity |
| [`skill-creator`](./agents/skill-creator.md) | Create new skills with proper structure |
| [`chat`](./agents/chat.md) | Conversational interactions |

## Skills Library

Skills are **executable knowledge notes** — a collection of patterns, best practices, and frameworks converted into actionable guidance for the AI agent.

| Category | Skills |
|----------|--------|
| **Full-Stack Web** | [React + Vite + Redux](./skills/web-fullstack-react-vite/SKILL.md), [Node + PostgreSQL + InfluxDB](./skills/node-pg-influx-stack/SKILL.md), [Python FastAPI/Django](./skills/python-web-fastapi-django/SKILL.md) |
| **Desktop Apps** | [Tauri 2.0 + Rust](./skills/tauri-v2-rust-patterns/SKILL.md), [.NET 9 Minimal APIs](./skills/dotnet-9-minimal-apis/SKILL.md), [Electron Secure IPC](./skills/electron-secure-ipc/SKILL.md) |
| **Cloud & DevOps** | [AWS/Azure/GCP Deployment](./skills/cloud-deploy-aws-azure-gcp/SKILL.md), [Git Worktree Management](./skills/git-worktree-management/SKILL.md) |
| **GIS Development** | [PostGIS + Leaflet](./skills/gis-development-core/SKILL.md) |
| **TypeScript** | [best practices](./skills/typescript-best-practices/SKILL.md), [advanced types](./skills/typescript-advanced-types/SKILL.md), [`satisfies` operator](./skills/typescript-satisfies-operator/SKILL.md), [interface vs type](./skills/typescript-interface-vs-type/SKILL.md) |
| **React** | [`useState`](./skills/react-use-state/SKILL.md), [`useCallback`](./skills/react-use-callback/SKILL.md), [`key` prop](./skills/react-key-prop/SKILL.md), [`"use client"` boundaries](./skills/react-use-client-boundary/SKILL.md) |
| **CSS** | [container queries](./skills/css-container-queries/SKILL.md), [Tailwind v4 best practices](./skills/code-architecture-tailwind-v4-best-practices/SKILL.md) |
| **Architecture** | [naming conventions](./skills/naming-cheatsheet/SKILL.md), [project structure](./skills/project-structure/SKILL.md), [wrong abstraction patterns](./skills/code-architecture-wrong-abstraction/SKILL.md) |
| **Product Frameworks** | [Jobs-to-be-Done](./skills/jobs-to-be-done/SKILL.md), [Business Model Canvas](./skills/business-model-canvas/SKILL.md), [Hooked Model](./skills/hooked-model/SKILL.md), [Fogg Behavior Model](./skills/fogg-behavior-model/SKILL.md), [PEST analysis](./skills/pest-analysis/SKILL.md), [product decisions](./skills/making-product-decisions/SKILL.md) |
| **UX Psychology** | [cognitive load](./skills/cognitive-load/SKILL.md), [cognitive biases](./skills/cognitive-biases/SKILL.md), [cognitive fluency](./skills/cognitive-fluency-psychology/SKILL.md), [Hick's law](./skills/hicks-law/SKILL.md), [progressive disclosure](./skills/progressive-disclosure/SKILL.md), [trust signals](./skills/trust-psychology/SKILL.md), [halo effect](./skills/halo-effect-psychology/SKILL.md) |
| **Behavioral Design** | [loss aversion](./skills/loss-aversion-psychology/SKILL.md), [status quo bias](./skills/status-quo-bias/SKILL.md), [social proof](./skills/social-proof-psychology/SKILL.md), [curiosity gap](./skills/curiosity-gap/SKILL.md), [self-initiated triggers](./skills/self-initiated-triggers/SKILL.md), [visual cues & CTAs](./skills/visual-cues-cta-psychology/SKILL.md) |
| **Decision Making** | [hypothesis trees](./skills/hypothesis-tree/SKILL.md), [five whys](./skills/five-whys/SKILL.md), [graph thinking](./skills/graph-thinking/SKILL.md), [game theory (tit-for-tat)](./skills/game-theory-tit-for-tat/SKILL.md) |
| **Agile** | [Kanban](./skills/kanban/SKILL.md), [theme-epic-story hierarchy](./skills/theme-epic-story/SKILL.md), [user stories](./skills/user-story-fundamentals/SKILL.md) |
| **Product Management** | [what not to do as PM](./skills/what-not-to-do-as-product-manager/SKILL.md) |

## MCP Servers

- **Context7** — Real-time library/API documentation

## Plugins

- `@mohak34/opencode-notifier` — Desktop notifications
- `@franlol/opencode-md-table-formatter` — Markdown table formatting
- `opencode-wakatime` — Time tracking
- `@tarquinen/opencode-dcp` — Context management

## Commands

| Command | Purpose |
|--------|---------|
| `/implement` | Execute implementation following architect's plan |
| `/git-commit` | Create git commit with conventional messages |
| `/learn` | Extract patterns from session to create new skills |
| `/summarize-branch` | Generate PR description from commits |

## Resources

- [OpenCode Documentation](https://opencode.ai/docs)
- [Skills Guide](https://opencode.ai/docs/skills)
- [Agents Guide](https://opencode.ai/docs/agents)

## License

[MIT](./LICENSE)
