# Plan: [Your Plan Name Here]

## Objective
[1-3 sentences describing what this plan accomplishes when fully implemented]

## Relevant Files
[List key files for reference, existing files to modify, and new files to create]

### Reference
- `path/to/docs.md` — Documentation to follow
- `path/to/existing-pattern.ts` — Example of the pattern to follow

### Existing Files to Modify
- `path/to/modify.ts` — What needs to change and why

### New Files to Create
- `path/to/new-file.ts` — What this file will contain

## Packets

### Packet 1: [Foundation / Setup Task Name]
**Description**: [Detailed description of what this packet accomplishes. Include enough context that a developer seeing this for the first time can execute it without asking questions.]

**Files**:
- `path/to/file1.ts` — Create with [description of contents]
- `path/to/file2.ts` — Modify [function/section] to [change description]

**Criteria**:
- [ ] [Specific, measurable criterion — e.g., "File exports a `UserService` class with `create()` and `findById()` methods"]
- [ ] [Another criterion — e.g., "All TypeScript types are properly defined, no `any` types"]
- [ ] [Another criterion — e.g., "Follows existing repository patterns for error handling"]

**Dependencies**: none

**Context**:
[Optional: code examples, API shapes, architectural decisions, or references]
```typescript
// Example of the pattern to follow:
export class ExampleService {
  async create(data: CreateInput): Promise<Entity> { ... }
}
```

---

### Packet 2: [Core Implementation Task Name]
**Description**: [What this packet accomplishes]

**Files**:
- `path/to/file3.ts` — Create with [description]
- `path/to/file1.ts` — Add [new functionality] to the code from Packet 1

**Criteria**:
- [ ] [Criterion 1]
- [ ] [Criterion 2]
- [ ] [Criterion 3]

**Dependencies**: Packet 1

**Context**:
[Any additional context needed]

---

### Packet 3: [Integration / Wiring Task Name]
**Description**: [What this packet accomplishes]

**Files**:
- `path/to/routes.ts` — Register new endpoints
- `path/to/config.ts` — Add configuration entries

**Criteria**:
- [ ] [Criterion 1]
- [ ] [Criterion 2]

**Dependencies**: Packet 1, Packet 2

---

### Packet 4: [Testing / Validation Task Name]
**Description**: [What this packet accomplishes]

**Files**:
- `path/to/file1.test.ts` — Unit tests for Packet 1 code
- `path/to/file3.test.ts` — Unit tests for Packet 2 code
- `path/to/integration.test.ts` — Integration tests

**Criteria**:
- [ ] [All unit tests pass]
- [ ] [Integration tests pass]
- [ ] [Test coverage meets threshold]

**Dependencies**: Packet 1, Packet 2, Packet 3

---

## Validation Commands
[Commands to run after all packets are complete to verify the entire plan succeeded]
- `npm test` — Run all tests
- `npm run build` — Ensure project builds
- `npm run lint` — Ensure no lint errors

## Notes
[Any additional context, constraints, or decisions that apply to the entire plan]
