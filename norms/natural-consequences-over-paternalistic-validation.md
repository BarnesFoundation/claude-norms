# Natural Consequences Over Paternalistic Validation

When the system's own feedback loop naturally punishes a bad input, let the consequences speak. Add validation only for STRUCTURAL correctness (parser-level: must have an `@`, must have a `.` after it). Do NOT add SEMANTIC validation ("are you sure you want to watch every Gmail sender?") on top.

## The Rule

If a bad-but-syntactically-valid entry produces obviously wrong results that the user can see, and the cost of those wrong results falls on the user who made the entry, do NOT add a confirmation prompt or a "did you mean?" suggestion. The natural consequences are the teaching mechanism.

## Why This Matters

Validation feels helpful. In practice it accumulates as paternalistic friction:

- **Slows valid use cases** to protect against rare misuse.
- **Cannot anticipate every shape** of misuse, so the additions creep ("we should also catch X, and Y, and Z...") and the validation logic grows unbounded.
- **Trains learned helplessness** — users stop reading the prompt, click through, and complain that the system "should have stopped them" anyway.
- **Adds bug surface area** — every validation rule is a place where a future change can introduce a regression.

The system's natural feedback loop, when it exists, is more efficient than any prompt:

- **Bad-but-syntactically-valid entries** silently fail to match anything; the user notices the absence of expected behavior and self-corrects.
- **Over-broad entries** flood the user with noise; the user removes them within minutes.
- **Cost falls on the user who made the entry**, not on anyone else, so the feedback is properly localized.

## When to Apply

- User-supplied filter expressions, watchlists, tag systems, query languages, mute rules.
- Notification preferences and subscription topics.
- Anywhere the system is "do this thing for me when X happens" and X is user-defined.

## When NOT to Apply

- Inputs that affect OTHER users (broadcasts, shared resources, group permissions). Other people's pain is not the input author's feedback signal.
- Destructive irreversible actions. "Delete production database" needs friction even if the cost falls on the actor, because the cost includes everyone downstream.
- Inputs that bypass security (allowlist of trusted domains for a security tool, IP whitelists, etc.). The asymmetric cost of a wrong allow makes confirmation worth it.

## How to Apply

Two layers of validation are appropriate, no more:

1. **Structural / parser-level**: must have one `@`; must have a `.` after the `@`; no whitespace; lowercase-normalize. Reject anything that is not parseable into the expected shape. The user's mental model of "I typed something, what happened" is preserved.
2. **Semantic / "is this a good idea"**: do NOT add. Trust the natural consequences. Document the design choice so a future maintainer who is tempted to "fix" it has to argue against it explicitly.

In code:

```typescript
// Good: structural rejection of unparseable input.
function parseWatchedSender(input: string): string | null {
  const s = input.trim().toLowerCase();
  if (!s) return null;
  if (/\s/.test(s)) return null;
  const at = s.indexOf('@');
  if (at < 0 || s.lastIndexOf('@') !== at) return null;
  const domain = s.slice(at + 1);
  if (!domain.includes('.')) return null;
  return s;
}

// BAD: semantic gating on top.
// if (s === '@gmail.com' && !confirmedFlood) {
//   return prompt('Are you sure? @gmail.com will flood your inbox.');
// }
```

## The Anti-Pattern

Adding a `confirm()` step every time an early user complains that "the system let me do something dumb." That is the user telling you the natural feedback loop worked: they did the dumb thing, they noticed, they fixed it. The fact that they are talking to you about it means the loop closed. Do not break the loop by inserting a prompt.

If you find yourself writing a list of "edge cases to validate against," that is the signal that you are sliding into paternalistic territory. Stop, document the natural feedback path instead, and ship.

## Discovered On

**Quarantine Canary** (BarnesFoundation/quarantine-canary), 2026-05. Specifically the `add @gmail.com` case: a user added `@gmail.com` to their watchlist as an over-broad domain entry, expecting to get flooded; the prediction failed because the user's specific inbox does not get much Gmail spam, but the architecture tolerated either outcome gracefully. No validation was needed, and the experience taught us more than any prompt would have. See ADR-013 in that repo for the original framing and the empirical observations from production.
