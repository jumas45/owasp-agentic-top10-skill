# ASI02: Tool Misuse and Exploitation

## Definition

Tool Misuse occurs when agents use legitimate, authorized tools in unsafe ways — either
because they're manipulated into doing so, or because tool boundaries aren't properly
enforced. This is distinct from "excessive agency" (having too many permissions) — this
risk is about the *misuse of whatever permissions exist*.

---

## Attack Vectors

### Parameter Pollution
The agent is manipulated into calling a tool with parameters beyond intended scope:
```python
# Agent authorized to search customer records by ID
# Attacker crafts input that results in:
search_customers(id="' OR 1=1 --")  # SQL injection via LLM-generated parameter
search_customers(id="../../etc/passwd")  # Path traversal
search_customers(id="*")  # Wildcard returning all records
```

### Tool Chain Manipulation
The agent is guided into a sequence of legitimate tool calls that together achieve
an unauthorized outcome:
```
Step 1: Read internal salary spreadsheet (authorized — "prepare a report")
Step 2: Write summary to external webhook (authorized — "share with stakeholders")
Combined: Data exfiltration via chained legitimate tools
```

### Automated Abuse at Scale
An agent authorized to send one notification is manipulated into sending thousands:
```python
# Intended: send one confirmation email
# Result after goal hijack: send_email() called 10,000 times in a loop
```

### Tool Schema Bypass
Agent constructs tool calls that exploit loose schema validation:
```json
// Tool schema says "name" is a string — no format validation
{"tool": "create_user", "params": {"name": "<script>alert(1)</script>", "role": "admin"}}
```

### Recursive Tool Calls
Agent authorized to call tool A, which triggers tool B, which loops back to A:
```python
# No cycle detection — agent exhausts quota or causes infinite loop
agent.call_tool("generate_report")
  → generate_report calls fetch_data
    → fetch_data calls generate_report  # undetected recursion
```

---

## What to Look For in Code

### Missing Input Validation on Tool Parameters
```python
# VULNERABLE
@tool
def delete_file(path: str):
    os.remove(path)  # No validation — agent can delete anything

# SAFER
@tool
def delete_file(path: str):
    allowed_prefix = "/tmp/agent_workspace/"
    resolved = os.path.realpath(path)
    if not resolved.startswith(allowed_prefix):
        raise ValueError(f"Path {path} is outside allowed workspace")
    os.remove(resolved)
```

### Tool Results Fed Back as Instructions
```python
# VULNERABLE
result = tools["web_search"](query)
agent.add_to_context(f"Search result: {result}")  # result may contain instructions

# SAFER
result = tools["web_search"](query)
structured = extract_facts(result)  # LLM call with locked prompt to extract only data
agent.add_to_context(structured)
```

### No Rate Limiting on Tool Calls
```python
# VULNERABLE — no bounds on how many times a tool can be called
while agent.has_more_steps():
    agent.step()

# SAFER
MAX_TOOL_CALLS_PER_RUN = 50
tool_call_count = 0
while agent.has_more_steps() and tool_call_count < MAX_TOOL_CALLS_PER_RUN:
    agent.step()
    tool_call_count += tool_calls_this_step
```

### Tool Schemas Without Strict Allowlists
```python
# VULNERABLE — accepts arbitrary strings
def send_notification(recipient: str, message: str): ...

# SAFER — recipient must be from a pre-approved list
ALLOWED_RECIPIENTS = load_from_config("notification_recipients")
def send_notification(recipient: str, message: str):
    if recipient not in ALLOWED_RECIPIENTS:
        raise PermissionError(f"Recipient {recipient} not in allowlist")
```

---

## MCP-Specific Patterns

When auditing MCP servers:

- **Tool schema strictness**: Does the MCP server define strict JSON Schema for all
  tool parameters with `enum`, `pattern`, `minimum`/`maximum`, and `maxLength`? Or
  does it accept freeform strings everywhere?
- **Tool result trust**: Does the MCP client treat tool results as trusted instructions
  or as untrusted data?
- **Tool discovery abuse**: Can an agent call a "list available tools" endpoint and use
  that information to discover and exploit tools it wasn't explicitly given?
- **Side effects documentation**: Are tool side effects (network calls, writes, emails)
  clearly documented and auditable?

---

## Severity Indicators

| Condition | Severity |
|-----------|----------|
| Tools with write/delete/send capabilities, no parameter validation | CRITICAL |
| No rate limiting on tools that cost money or generate external traffic | HIGH |
| Tool results used as context without sanitization | HIGH |
| Tool chain can achieve data exfiltration via individually-authorized steps | HIGH |
| Read-only tools with no input validation | MEDIUM |
| Loose schema validation, no destructive capabilities | LOW |

---

## Mitigations

1. **Validate all tool inputs** against a strict schema before execution — use
   `enum` values, regex patterns, length limits, and path prefix checks.

2. **Treat tool outputs as untrusted data** — never concatenate raw tool results back
   into the agent prompt as if they were trusted instructions.

3. **Implement per-tool rate limits** — max calls per minute, per session, and per task.
   Alert on anomalous call volumes.

4. **Log all tool calls** with full parameters and results to an immutable audit log.

5. **Require human confirmation** for tools with external side effects or data writes.

6. **Use tool schemas with the minimum viable parameter set** — if a tool only needs
   a customer ID, don't accept a full SQL query string as a parameter.

7. **Test tool boundary assumptions** with adversarial inputs: path traversal, SQL
   fragments, oversized inputs, recursive calls.
