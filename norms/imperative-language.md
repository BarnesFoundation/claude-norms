# Imperative Language in Claude Instructions

## The Rule

Every instruction in `CLAUDE.md` and related files that you want Claude to follow MUST use imperative language. Never use conditional phrasing.

## Why This Matters

Conditional language gives models permission to skip steps. This is not theoretical — it was discovered in production.

**The incident**: On an MCP server project (Chat Claude integration), the CLAUDE.md contained instructions like "you should read the MCP schema before implementing tools." Claude (especially smaller models like Haiku) interpreted "should" as optional and skipped the schema read entirely, resulting in tools that didn't match the schema and failed at runtime.

**The fix**: Changing "you should read" to "you MUST read" — and adding "Do NOT skip this step" — immediately resolved the issue. The same Claude model, same context, different instruction phrasing = different behavior.

**Why it happens**: Language models are trained on text where "should" genuinely means "this is recommended but not required." When under context pressure (long conversations, many instructions), soft language gets deprioritized first. Imperative language survives context pressure because the model treats it as a hard constraint rather than a soft suggestion.

## The Translation Table

| Conditional (BAD) | Imperative (GOOD) |
|---|---|
| should | MUST |
| consider | ALWAYS |
| prefer X | Use X |
| you may want to | ALWAYS |
| if possible | (remove — either it's required or it's not) |
| try to | ALWAYS |
| don't | NEVER or Do NOT |
| it's recommended that | (state the requirement directly) |
| could | (remove — state what to do, not what could be done) |

## Examples

### CLAUDE.md instructions

BAD:
```markdown
Every Claude session should read these files before starting substantive work.
```

GOOD:
```markdown
Every Claude session MUST read these files before starting substantive work.
Do NOT skip this step. Do NOT assume you know the current strategy without
reading these files.
```

### Code conventions

BAD:
```markdown
- Prefer SWR for data fetching in components
- Consider using Mantine components where possible
- You may want to check the existing types before creating new ones
```

GOOD:
```markdown
- Use SWR for data fetching in components
- Use Mantine components and theme tokens; avoid raw CSS
- Check `common/types.ts` for existing types BEFORE creating new ones
```

### Behavioral rules

BAD:
```markdown
- If the current direction contradicts another contributor's file, you should raise it.
- Try to update DECISIONS.md when meaningful choices are made.
```

GOOD:
```markdown
- If the current direction contradicts another contributor's file, raise it explicitly. Do NOT silently proceed.
- When a meaningful choice is made during a session, update DECISIONS.md before the session ends. Do NOT leave decisions unrecorded.
```

## When Conditional Language IS Appropriate

Not every sentence needs to be imperative. Use conditional language for:

- **Genuinely optional suggestions**: "You can optionally add a description to the ADR"
- **Context-dependent guidance**: "If the backend is running, you can verify against the live API"
- **Exploratory discussion**: In context docs and contributor files where the content is thinking-out-loud, not instructions

The rule applies specifically to **instructions you want followed** — the behavioral directives in CLAUDE.md and the Rules sections of shared memory docs.

## Reinforcement Patterns

For critical instructions, layer the imperative:

1. **State the requirement**: "MUST read these files"
2. **Forbid the skip**: "Do NOT skip this step"
3. **Block the assumption**: "Do NOT assume you know the current strategy without reading"
4. **State the consequence**: "If you are about to contradict this, STOP and flag it"

This is not redundant — each layer catches a different failure mode.

## Applying This to Your Project

1. Read through your `CLAUDE.md` and search for: should, consider, prefer, may, might, try, if possible, recommended
2. For each instance, decide: is this a requirement or a suggestion?
3. If it's a requirement, rewrite using the translation table above
4. If it's genuinely optional, leave it — but make sure the optional nature is intentional
5. Add "Do NOT skip" reinforcement to your most critical instructions (Required Reading, conflict detection)
