---
description: Execute implementation following the architect's plan. Use this command to start coding tasks.
agent: architect
model: opencode/glm-5
---

Execute implementation based on the architectural plan in `.opencode/PLAN.md`.

## Process

1. **Read the Plan**: Load `.opencode/PLAN.md` from the project root
2. **Execute Steps**: Follow the implementation steps defined in the plan
3. **Create/Modify Files**: Implement each requirement from the plan
4. **Run Build**: Execute the appropriate build command (npm run build, dotnet build, cargo build, etc.)
5. **Report Status**: Update SESSION_CONTEXT.md with progress

## Constraints

- Only implement what is defined in the architectural plan
- Do not deviate from the plan without consulting the architect first
- Run builds to verify implementation before marking complete
