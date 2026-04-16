# Pattern 2: Action Scope Limiter

## The Problem

You build an agent to help with customer support tickets. You give it access to your internal tools. Six months later, someone asks it a leading question and it starts touching parts of the system it was never supposed to touch.

Agents don't have intentions. They follow instructions, and instructions can be gamed, misinterpreted, or simply underspecified. The more capable the model, the more creative it gets when filling in gaps. That's useful until it isn't.

The Human Approval Gate handles irreversible actions after they're proposed. This pattern handles something earlier in the chain: it stops the agent from being able to propose certain actions in the first place.

The idea is simple. At startup, you define exactly what the agent is allowed to do. That list is fixed. If an action isn't on it, the agent can't take it, not because you trust the agent to self-limit, but because the architecture won't let it through.

---

## When to Use This

Use this when:

- Your agent has access to multiple tools or APIs, only some of which are relevant to its task
- The cost of an out-of-scope action is high enough that "I'll review it afterward" isn't good enough
- You're deploying an agent in a context where users might try to expand its behavior through prompting
- You want to enforce separation of concerns between agents in a multi-agent system

---

## How It Works

You define a whitelist of permitted action types before the agent runs. Every proposed action gets checked against that list before execution. If it's not on the list, it gets blocked and logged, the agent is told it can't do that here, and asked to work within its defined scope.

This is different from telling the agent in its system prompt "only do X." System prompt instructions can be overridden, confused, or forgotten across long conversations. A whitelist enforced in code cannot.

---

## Implementation

```python
import anthropic
import json
from dataclasses import dataclass
from typing import Any

client = anthropic.Anthropic()


@dataclass
class ScopeViolation:
    attempted_action: str
    permitted_actions: list[str]
    task: str


class ScopeLimitedAgent:
    def __init__(self, permitted_actions: list[str], agent_role: str):
        """
        permitted_actions: explicit list of what this agent can do
        agent_role: plain-language description, used in the system prompt
        """
        self.permitted_actions = permitted_actions
        self.agent_role = agent_role
        self.violations: list[ScopeViolation] = []

    def _build_system_prompt(self) -> str:
        action_list = "\n".join(f"- {a}" for a in self.permitted_actions)
        return f"""You are {self.agent_role}.

You can only take actions from this list:
{action_list}

When responding, always output a JSON object:
{{
  "action_type": "<action from the list above>",
  "action_details": {{<parameters>}},
  "reason": "<why this action>"
}}

If the task requires an action not on your list, respond with:
{{
  "action_type": "out_of_scope",
  "action_details": {{}},
  "reason": "<what was requested and why you can't do it>"
}}
"""

    def propose_action(self, task: str) -> dict[str, Any]:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=500,
            system=self._build_system_prompt(),
            messages=[{"role": "user", "content": task}]
        )

        try:
            proposal = json.loads(response.content[0].text)
        except json.JSONDecodeError:
            return {"action_type": "parse_error", "action_details": {}, "reason": "Could not parse agent response"}

        return proposal

    def execute(self, task: str, action_runner) -> dict[str, Any]:
        proposal = self.propose_action(task)
        action_type = proposal.get("action_type")

        # Hard block: action not in permitted list
        if action_type not in self.permitted_actions:
            violation = ScopeViolation(
                attempted_action=action_type,
                permitted_actions=self.permitted_actions,
                task=task
            )
            self.violations.append(violation)

            print(f"[SCOPE BLOCK] Agent tried '{action_type}' — not permitted.")
            return {
                "status": "blocked",
                "attempted_action": action_type,
                "permitted_actions": self.permitted_actions
            }

        # Action is in scope — proceed
        result = action_runner(action_type, proposal.get("action_details", {}))
        return {"status": "completed", "action": action_type, "result": result}

    def get_violations(self) -> list[ScopeViolation]:
        return self.violations


# --- Example usage ---

def mock_runner(action_type, details):
    print(f"[RUNNING] {action_type}: {details}")
    return "success"


if __name__ == "__main__":
    # This agent can only read tickets and add internal notes.
    # It cannot close tickets, send emails, or access billing.
    support_agent = ScopeLimitedAgent(
        permitted_actions=["read_ticket", "add_internal_note", "escalate_ticket"],
        agent_role="a read-only customer support triage agent"
    )

    # Normal task — within scope
    support_agent.execute(
        task="Read ticket #4421 and add an internal note summarizing the issue",
        action_runner=mock_runner
    )

    # Task that tries to go outside scope
    support_agent.execute(
        task="Close ticket #4421 and send a resolution email to the customer",
        action_runner=mock_runner
    )

    print(f"\nScope violations recorded: {len(support_agent.get_violations())}")
```

---

## Tradeoffs

**What you gain:** A hard architectural boundary. No amount of clever prompting can get the agent to take an action that isn't on the list. You also get a record of violation attempts, which is useful for spotting patterns in how users interact with the system.

**What you lose:** Flexibility. If the permitted list is too narrow, the agent becomes useless for edge cases. You'll need to tune it over time as you learn which actions are actually needed.

**The key tension:** There's a temptation to make the permitted list broad "just in case." Resist it. Start narrow and expand only when you have a specific reason. A wider scope means a larger attack surface.

---

## Common Mistakes

- **Relying on the system prompt alone.** Telling the agent "only do X" in the prompt is not the same as enforcing it in code. Both are useful — but only the code version is guaranteed.
- **Not logging violations.** A blocked action is a data point. If the same action keeps getting attempted, maybe it should be permitted. If a weird action gets attempted once, that's worth investigating.
- **Overlapping scopes in multi-agent systems.** If two agents have overlapping permitted action sets, you can get into a situation where one agent hands off to another to perform an action it couldn't do itself. Define scopes to be non-overlapping where possible.

---

## Related Patterns

- [Human Approval Gate](./01-human-approval-gate.md) — adds a human checkpoint for actions that are permitted but still sensitive
- [Audit Log Checkpoint](./03-audit-log-checkpoint.md) — records everything, including blocked attempts
