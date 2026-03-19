---
description: Execute implementation following the architect's plan. Use this command to start coding tasks.
agent: architect
model: opencode-go/glm-5
---

Execute implementation based on the architectural plan in `.opencode/plans/[dynamic-name].md`.

## Process

1. **Read the Plan**: Load the dynamically named plan from `.opencode/plans/`
2. **Execute Steps**: Follow the implementation steps defined in the plan
3. **Create/Modify Files**: Implement each requirement from the plan
4. **Run Build**: Execute the appropriate build command (npm run build, dotnet build, cargo build, etc.)
5. **Report Status**: Update `.opencode/context/SESSION_CONTEXT.md`:
```
## Last Major Change
*Timestamp*: [YYYY-MM-DD HH:MM:SS UTC]
*Agent*: coder
*Description*: [Summary of implementation progress]

## Build Status
*Status*: ✅ Passed / ❌ Failed / ⏳ Pending
*Last Check*: [YYYY-MM-DD HH:MM:SS UTC]

## Pending Tasks
1. [Remaining implementation step]
2. [Documentation updates]
```

## Constraints

- Only implement what is defined in the architectural plan
- Do not deviate from the plan without consulting the architect first
- Run builds to verify implementation before marking complete
