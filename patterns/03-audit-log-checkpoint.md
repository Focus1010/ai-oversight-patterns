# Pattern 3: Audit Log Checkpoint

## The Problem

Something went wrong. The agent did something it shouldn't have, or the output doesn't match what the user expected. Now you need to figure out what happened.

If you don't have a log, you're in the dark. You know the input and you know the output. Everything in between, why the agent chose a particular action, what it considered, how confident it was, is gone.

This is the most common gap in AI agent implementations. Developers log API errors and exceptions. They rarely log the agent's reasoning before each decision.

This pattern is about writing down what the agent is thinking at every decision point, before anything gets executed. Not after. Before. That distinction matters: a log written after execution can't capture a decision that led to a blocked or failed action.

---

## When to Use This

Use this in almost every agent deployment. The overhead is low and the value compounds over time. It's especially important when:

- Your agent makes multiple sequential decisions (each step depends on the last)
- You're operating in a regulated environment and need an audit trail
- You're still tuning the agent's behavior and need visibility into failure modes
- Other people or teams use the agent and you need to be able to reconstruct what happened

---

## How It Works

Before every action, the agent writes a structured log entry that includes: what action it's about to take, why it chose that action, what alternatives it considered, and its confidence level. This gets written to an append-only log, a file, a database row, whatever suits your stack, before execution starts.

"Append-only" is not an implementation detail. If the log can be edited or deleted, it's not a reliable audit trail. The point is a record that reflects what actually happened, not what someone would prefer to have happened.

---

## Implementation

```python
import anthropic
import json
import uuid
from datetime import datetime, timezone
from pathlib import Path
from typing import Any

client = anthropic.Anthropic()

LOG_FILE = Path("agent_audit.log")


def write_log_entry(entry: dict) -> None:
    """
    Append a single log entry as a JSON line.
    One entry per line makes it easy to parse later.
    """
    with LOG_FILE.open("a") as f:
        f.write(json.dumps(entry) + "\n")


def get_action_with_reasoning(task: str, context: dict | None = None) -> dict[str, Any]:
    """
    Ask the agent what it wants to do and why, in structured form.
    This gets logged before anything is executed.
    """
    context_str = json.dumps(context) if context else "none"

    prompt = f"""
Task: {task}
Context: {context_str}

Respond only with a JSON object in this exact format:
{{
  "action_type": "<what you will do>",
  "action_details": {{<parameters needed>}},
  "reason": "<why this action is the right one>",
  "alternatives_considered": ["<option 1>", "<option 2>"],
  "confidence": <float between 0.0 and 1.0>,
  "uncertainty": "<what information would change your decision, if any>"
}}
"""

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=600,
        messages=[{"role": "user", "content": prompt}]
    )

    try:
        return json.loads(response.content[0].text)
    except json.JSONDecodeError:
        return {
            "action_type": "parse_error",
            "action_details": {},
            "reason": "Agent response could not be parsed",
            "alternatives_considered": [],
            "confidence": 0.0,
            "uncertainty": "n/a"
        }


def execute_with_audit(task: str, action_runner, context: dict | None = None) -> dict[str, Any]:
    """
    Full cycle: get decision → log it → execute → log outcome.
    """
    run_id = str(uuid.uuid4())[:8]
    timestamp = datetime.now(timezone.utc).isoformat()

    # Step 1: Get the agent's decision and reasoning
    decision = get_action_with_reasoning(task, context)

    # Step 2: Write the pre-execution log entry
    pre_entry = {
        "run_id": run_id,
        "timestamp": timestamp,
        "phase": "pre_execution",
        "task": task,
        "context": context,
        "action_type": decision.get("action_type"),
        "action_details": decision.get("action_details"),
        "reason": decision.get("reason"),
        "alternatives_considered": decision.get("alternatives_considered"),
        "confidence": decision.get("confidence"),
        "uncertainty": decision.get("uncertainty"),
        "status": "pending"
    }
    write_log_entry(pre_entry)

    print(f"[LOG {run_id}] Action: {decision.get('action_type')}")
    print(f"[LOG {run_id}] Reason: {decision.get('reason')}")
    print(f"[LOG {run_id}] Confidence: {decision.get('confidence')}")

    # Step 3: Execute the action
    try:
        result = action_runner(
            decision.get("action_type"),
            decision.get("action_details", {})
        )
        outcome = "success"
        error = None
    except Exception as e:
        result = None
        outcome = "error"
        error = str(e)

    # Step 4: Write the post-execution log entry
    post_entry = {
        "run_id": run_id,
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "phase": "post_execution",
        "action_type": decision.get("action_type"),
        "outcome": outcome,
        "result_summary": str(result)[:200] if result else None,
        "error": error
    }
    write_log_entry(post_entry)

    return {
        "run_id": run_id,
        "action": decision.get("action_type"),
        "outcome": outcome,
        "result": result,
        "error": error
    }


def read_audit_log(run_id: str | None = None) -> list[dict]:
    """
    Read the full log, optionally filtered by run_id.
    """
    if not LOG_FILE.exists():
        return []

    entries = []
    with LOG_FILE.open() as f:
        for line in f:
            line = line.strip()
            if line:
                try:
                    entry = json.loads(line)
                    if run_id is None or entry.get("run_id") == run_id:
                        entries.append(entry)
                except json.JSONDecodeError:
                    continue
    return entries


# --- Example usage ---

def mock_runner(action_type, details):
    print(f"[EXECUTING] {action_type}: {details}")
    return {"status": "ok", "message": "action completed"}


if __name__ == "__main__":
    result = execute_with_audit(
        task="Fetch the latest order for customer #882 and flag it for review if total exceeds $500",
        action_runner=mock_runner,
        context={"customer_id": 882, "review_threshold": 500}
    )

    print(f"\nRun ID: {result['run_id']}")
    print(f"Outcome: {result['outcome']}")

    # Inspect the log for this run
    print("\n--- Audit log for this run ---")
    for entry in read_audit_log(result['run_id']):
        print(json.dumps(entry, indent=2))
```

---

## Tradeoffs

**What you gain:** Full reconstruction of agent behavior. When something goes wrong, you know exactly what the agent decided, why, and what it was uncertain about. Over time, the log becomes a dataset for improving the system, you can see which actions the agent is consistently uncertain about, which alternatives it keeps discarding, and where confidence correlates with errors.

**What you lose:** Some latency per step, and storage. The latency comes from the additional structured reasoning call. On most tasks this is negligible. Storage depends on how much you log and how long you keep it, for most small projects, this is not a real constraint.

**Important:** Logs can contain sensitive data. If the agent is working with user records, financial data, or anything personal, the audit log inherits that sensitivity. Treat it accordingly, access controls, retention policies, the usual.

---

## Common Mistakes

- **Logging only on errors.** The whole point is that you don't know in advance which runs will have problems. Log everything.
- **Writing the log after execution.** A post-execution log can be rationalized or skipped if the execution crashes. Pre-execution logging is the only reliable approach.
- **Storing logs in the same place as mutable application state.** Keep logs separate and append-only. If your application can delete its own logs, the audit trail is worth nothing.
- **Ignoring the confidence field.** Low confidence doesn't mean the agent is wrong, but it's a signal worth tracking. If you see consistent low confidence on a certain type of task, the agent probably needs better context for that task.

---

## Related Patterns

- [Human Approval Gate](./01-human-approval-gate.md) — log entries for pending-approval actions are especially valuable for reconstructing what a human saw before approving
- [Action Scope Limiter](./02-action-scope-limiter.md) — scope violations should always be logged, even when no execution occurs
