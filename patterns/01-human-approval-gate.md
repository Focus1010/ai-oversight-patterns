# Pattern 1: Human Approval Gate

## The Problem

When you give an AI agent a task, it will try to complete it. That's the point. But some actions can't be undone, a sent email, a deleted file, a submitted form, an API call that charges a card. If the agent misunderstands the task, or the user's instructions were ambiguous, there's no coming back.

Most developers don't add any checkpoint between "agent decides to do X" and "agent does X." The assumption is that the agent understood correctly. That assumption fails more often than you'd expect.

This pattern fixes that for irreversible actions specifically. You don't need human approval for every step, that would defeat the purpose of an agent. You need it for the ones that matter.

---

## When to Use This

Use it when your agent can take actions that are:

- **Irreversible** — sends a message, deletes data, makes a purchase, submits a form
- **External** — touches a third-party service or API outside your system
- **High consequence** — the cost of a mistake is significant (money, reputation, data)

You probably don't need it for read-only operations, internal state changes you can roll back, or low-stakes formatting tasks.

---

## How It Works

Before executing any flagged action, the agent generates a plain-language summary of what it's about to do and why. That summary gets shown to a human. The agent waits. If the human approves, it proceeds. If not, it either stops or asks for clarification.

The key design decision: **the agent proposes, the human disposes.**

---

## Implementation

```python
import anthropic
from typing import Callable

client = anthropic.Anthropic()

# Define which action types need human review
REQUIRES_APPROVAL = {"send_email", "delete_record", "submit_payment", "post_message"}


def request_approval(action_type: str, action_details: dict) -> bool:
    """
    Show the proposed action to a human and wait for a yes/no.
    In production, this could be a Slack message, a UI modal, or a CLI prompt.
    """
    print("\n--- APPROVAL REQUIRED ---")
    print(f"Action type : {action_type}")
    print(f"Details     : {action_details}")
    response = input("Approve? (yes/no): ").strip().lower()
    return response == "yes"


def agent_with_approval_gate(task: str, execute_action: Callable):
    """
    Ask the agent what it wants to do, then gate execution on human approval
    if the action type is in REQUIRES_APPROVAL.
    """

    # Step 1: Ask the agent to plan the action in structured form
    planning_prompt = f"""
You need to complete this task: {task}

Respond with a JSON object describing your next action.
Format:
{{
  "action_type": "<type of action>",
  "action_details": {{<relevant parameters>}},
  "reason": "<one sentence explaining why>"
}}

Only describe one action. Do not execute anything.
"""

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=500,
        messages=[{"role": "user", "content": planning_prompt}]
    )

    import json
    plan = json.loads(response.content[0].text)

    action_type = plan.get("action_type")
    action_details = plan.get("action_details", {})
    reason = plan.get("reason", "")

    print(f"\nAgent wants to: {action_type}")
    print(f"Reason: {reason}")

    # Step 2: Gate on approval if this action type requires it
    if action_type in REQUIRES_APPROVAL:
        approved = request_approval(action_type, action_details)
        if not approved:
            print("Action rejected by user. Agent will not proceed.")
            return {"status": "rejected", "action": action_type}

    # Step 3: Execute only after approval
    result = execute_action(action_type, action_details)
    return {"status": "completed", "action": action_type, "result": result}


# --- Example usage ---

def mock_execute(action_type, details):
    print(f"[EXECUTING] {action_type} with {details}")
    return "done"


if __name__ == "__main__":
    agent_with_approval_gate(
        task="Send a follow-up email to the client about the invoice",
        execute_action=mock_execute
    )
```

---

## Tradeoffs

**What you gain:** You catch mistakes before they cost you. The agent's reasoning is visible, you're not just seeing the output, you're seeing the decision.

**What you lose:** Speed. If the human isn't watching, the agent stalls. This isn't suitable for fully automated pipelines running at 3am.

**The middle ground:** You can make approval async — push the request to Slack or email, set a timeout, and either wait or fail safe if no response arrives. That way it doesn't block indefinitely.

---

## Common Mistakes

- **Approving too many actions.** If everything needs approval, people start clicking "yes" without reading. Reserve gates for genuinely irreversible operations.
- **Vague summaries.** "Agent wants to send email" is useless. The summary needs to include who, what, and why, enough for a human to evaluate it in ten seconds.
- **No rejection path.** If the human says no, the agent needs to know what to do next. Either stop, ask for clarification, or suggest alternatives. Silently stopping with no feedback is confusing.

---

## Related Patterns

- [Action Scope Limiter](./02-action-scope-limiter.md) — prevents the agent from even proposing actions outside a defined boundary
- [Audit Log Checkpoint](./03-audit-log-checkpoint.md) — records what the agent decided and why, approved or not
