# AI Oversight Patterns

A practical reference for developers who want to maintain human control over AI agents.

Not a research paper. Not a policy document. Code patterns you can actually use.

---

## Why This Exists

Most AI safety writing sits at the research level. That's important work, but it doesn't help a developer who's shipping an agent next week and needs to know: what do I add to make sure this thing doesn't do something I didn't intend?

That gap between theoretical safety research and practical engineering choices is what this catalog tries to fill.

---

## What's Here

A structured catalog of software patterns. Each pattern covers one specific failure mode. Each one includes:

- A description of the problem it solves
- When to use it (and when not to)
- A Python implementation you can adapt
- The tradeoffs you're making
- Common mistakes
- Related patterns

**[See the full index →](./PATTERNS_INDEX.md)**

---

## Completed Patterns

**[01 — Human Approval Gate](./patterns/01-human-approval-gate.md)**
Require human approval before any irreversible action. The agent proposes; the human decides.

**[02 — Action Scope Limiter](./patterns/02-action-scope-limiter.md)**
Enforce a whitelist of permitted actions in code. If it's not on the list, it can't happen.

**[03 — Audit Log Checkpoint](./patterns/03-audit-log-checkpoint.md)**
Log every decision before it executes — action, reasoning, confidence, alternatives considered.

---

## How to Use This

You don't need all 20 patterns. Most agents need 3–5.

Start here:
- **What's the worst thing this agent could do by accident?** → [Human Approval Gate](./patterns/01-human-approval-gate.md)
- **Could a user manipulate this agent into doing something it shouldn't?** → [Action Scope Limiter](./patterns/02-action-scope-limiter.md)
- **How would you debug a problem if something went wrong?** → [Audit Log Checkpoint](./patterns/03-audit-log-checkpoint.md)

---

## Status

17 more patterns in progress. See the [full index](./PATTERNS_INDEX.md) for what's planned.

---

## Contributing

If you've hit a failure mode in a real agent deployment that isn't covered here, open an issue. The goal is a catalog grounded in what actually breaks.

---

## License

MIT
