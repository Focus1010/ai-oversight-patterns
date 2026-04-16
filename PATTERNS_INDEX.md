# AI Oversight Patterns — Index

A catalog of software engineering patterns for maintaining human control over AI agents. Each pattern targets a specific failure mode that shows up when you deploy agents in real systems.

These aren't theoretical. They came out of observing what breaks when AI agents interact with real APIs, real users, and real data.

---

## Status

| # | Pattern | Status |
|---|---------|--------|
| 01 | [Human Approval Gate](./patterns/01-human-approval-gate.md) | ✅ Complete |
| 02 | [Action Scope Limiter](./patterns/02-action-scope-limiter.md) | ✅ Complete |
| 03 | [Audit Log Checkpoint](./patterns/03-audit-log-checkpoint.md) | ✅ Complete |
| 04 | Rollback Checkpoint | 🔄 In progress |
| 05 | Confidence Threshold Pause | 🔄 In progress |
| 06 | Dead Man's Switch | 📋 Planned |
| 07 | Blast Radius Limiter | 📋 Planned |
| 08 | Sandbox Execution Environment | 📋 Planned |
| 09 | Rate Limit Enforcer | 📋 Planned |
| 10 | Principal Hierarchy Check | 📋 Planned |
| 11 | Intent Verification Step | 📋 Planned |
| 12 | Output Sanitizer | 📋 Planned |
| 13 | Fallback Degradation Path | 📋 Planned |
| 14 | Multi-Agent Scope Boundary | 📋 Planned |
| 15 | Context Window Integrity Check | 📋 Planned |
| 16 | Instruction Override Detector | 📋 Planned |
| 17 | Minimal Footprint Enforcer | 📋 Planned |
| 18 | Time-Boxed Execution Guard | 📋 Planned |
| 19 | Sensitive Data Fence | 📋 Planned |
| 20 | Graceful Uncertainty Escalation | 📋 Planned |

---

## Pattern Descriptions

### Completed

**01 — Human Approval Gate**
Before executing any irreversible action, the agent generates a plain-language summary of what it's about to do and waits for a human yes/no. Covers: send email, delete record, submit payment, post to external service.

**02 — Action Scope Limiter**
A whitelist of permitted action types defined at agent initialization. Any action not on the list is blocked in code, not just in the prompt. The agent can't propose what it's not allowed to do.

**03 — Audit Log Checkpoint**
Before every action, the agent writes a structured log entry — action type, reason, alternatives considered, confidence level. Append-only. Enables full reconstruction of agent behavior after the fact.

---

### Planned

**04 — Rollback Checkpoint**
Before any state-changing action, snapshot the affected state. If the action fails or produces unexpected output, revert to the snapshot. Covers database writes, file modifications, configuration changes.

**05 — Confidence Threshold Pause**
Ask the agent to rate its confidence before acting. If confidence falls below a set threshold, the agent pauses and requests more information rather than proceeding on a guess.

**06 — Dead Man's Switch**
An agent that stops operating if it doesn't receive a periodic heartbeat signal from a human operator. Used for long-running autonomous agents where silent failure is more dangerous than forced shutdown.

**07 — Blast Radius Limiter**
Constrain the number of records, users, or API calls an agent can affect in a single run. A bug in the logic affects 10 records, not 10,000.

**08 — Sandbox Execution Environment**
Run agent-generated code or commands in an isolated environment before letting any output touch production systems. Catches bad outputs before they cause damage.

**09 — Rate Limit Enforcer**
Hard cap on how many actions an agent can take per minute, hour, or session. Prevents runaway loops from hammering downstream services.

**10 — Principal Hierarchy Check**
Before acting on an instruction, verify whether the instruction source has the authority level required for that action. Prevents privilege escalation through prompt injection.

**11 — Intent Verification Step**
After receiving a task, the agent restates its interpretation of the intent back to the user before doing anything. Catches misunderstandings before they become mistakes.

**12 — Output Sanitizer**
Before agent output reaches users or downstream systems, pass it through a validation layer that checks for known failure modes — hallucinated references, toxic content, PII leakage.

**13 — Fallback Degradation Path**
If the agent fails to complete a task or hits a blocking condition, it falls back to a defined safe state or hands off to a human rather than attempting to improvise.

**14 — Multi-Agent Scope Boundary**
When multiple agents operate in the same system, each agent's scope is defined independently and checked at handoff points. Prevents one agent from using another to perform actions it can't perform itself.

**15 — Context Window Integrity Check**
Periodically verify that the agent's active context hasn't been corrupted or injected with conflicting instructions during a long-running session.

**16 — Instruction Override Detector**
Monitor for instructions that attempt to modify the agent's core behavior, override safety rules, or claim elevated permissions. Flag or block these before they reach the model.

**17 — Minimal Footprint Enforcer**
Restrict what data the agent can read, store, or pass between steps to only what's necessary for the current task. Limits exposure in case of a compromise.

**18 — Time-Boxed Execution Guard**
Set a hard time limit on agent execution. If the agent hasn't completed the task by the deadline, it stops and reports what it did rather than running indefinitely.

**19 — Sensitive Data Fence**
Define categories of sensitive data (PII, credentials, financial records) that the agent can reference by identifier but cannot read the contents of directly.

**20 — Graceful Uncertainty Escalation**
When the agent encounters a situation outside its training distribution or defined scope, it escalates to a human with a structured summary of the uncertainty rather than guessing.

---

## How to Use This Catalog

These patterns are not a checklist. You don't need all of them. Start by identifying which of these failure modes are realistic for your specific deployment:

- **What's the worst thing this agent could do by accident?** → Start with Human Approval Gate and Rollback Checkpoint.
- **What would happen if a user tried to manipulate this agent?** → Start with Action Scope Limiter and Instruction Override Detector.
- **How would you debug a problem if something went wrong?** → Start with Audit Log Checkpoint.
- **What happens when the agent gets stuck or confused?** → Start with Confidence Threshold Pause and Graceful Uncertainty Escalation.

Most production agents need 3–5 of these. Which ones depend on what the agent is doing and what the cost of failure looks like.

---

## Contributing

If you've encountered a failure mode in a real agent deployment that isn't covered here, open an issue describing it. The goal is a catalog grounded in what actually breaks, not what sounds plausible.
