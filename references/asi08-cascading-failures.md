# ASI08: Cascading Failures

## Definition

In multi-step agentic pipelines, small errors — hallucinations, wrong assumptions,
subtly poisoned data — don't stay small. They propagate forward through planning steps,
tool calls, and memory writes, compounding at each stage. By the time the failure is
visible, it may have caused irreversible real-world effects across many downstream systems.

This is fundamentally a **system design risk**: individual agents can behave correctly
while the overall system fails badly because error propagation is not contained.

---

## How Cascading Failures Manifest

### Hallucinated Data Propagation
```
Step 1: Agent retrieves product price → hallucinates $149 (actual: $14.99)
Step 2: Calculates bulk discount → $134.10
Step 3: Generates invoice → $134.10
Step 4: Sends invoice to customer
Step 5: Customer pays $134.10
Step 6: Reconciliation finds $13,410 discrepancy across 100 orders
```

### False State Assumptions
```
Step 1: Agent checks if record exists → incorrectly returns "not found"
Step 2: Agent creates new record (duplicate)
Step 3: Downstream agent sees two records → picks one arbitrarily
Step 4: Reports generated from both → conflicting data in dashboards
Step 5: Decision made on corrupted data
```

### Memory-Amplified Cascades
```
Step 1: Attacker injects false "policy" into agent memory
Step 2: Agent reads memory → incorporates false policy into decisions
Step 3: Agent writes its decisions back to memory (compounding)
Step 4: Other agents read the corrupted memory
Step 5: False policy now influences the entire multi-agent system
```

### Error Handling That Masks Failures
```python
# VULNERABLE — errors silently continue
try:
    result = fetch_customer_data(customer_id)
except:
    result = {}  # empty dict — downstream treats as valid empty response
# Agent proceeds as if data was found, makes decisions on empty state
```

### Retry Storms
```python
# VULNERABLE — unbounded retries without backoff
while not success:
    result = call_external_api(payload)
    success = result.ok
# On API failure: infinite loop, quota exhaustion, or DDoS of downstream service
```

---

## What to Look For in Code

### No Output Validation Between Pipeline Stages
```python
# VULNERABLE
step1_result = agent_1.run(task)
step2_result = agent_2.run(step1_result)  # no validation between steps

# SAFER
step1_result = agent_1.run(task)
if not validate_output(step1_result, schema=STEP1_OUTPUT_SCHEMA):
    raise PipelineError(f"Step 1 output invalid: {step1_result}")
if not passes_confidence_threshold(step1_result, min_score=0.85):
    await request_human_review(step1_result)
step2_result = agent_2.run(step1_result)
```

### Irreversible Actions Without Confirmation Gates
```python
# VULNERABLE — delete happens without any checkpoint
for record in records_to_delete:
    database.delete(record)

# SAFER — checkpoint before irreversible action
deletion_plan = generate_deletion_plan(records_to_delete)
human_approval = await request_approval(deletion_plan, timeout=300)
if not human_approval:
    raise AbortedByPolicy("Deletion requires human approval")
for record in records_to_delete:
    database.delete(record)
    audit_log.write(f"Deleted {record} per approval {human_approval.id}")
```

### Missing Circuit Breakers
```python
# VULNERABLE — no protection against cascading failures to external services
def call_downstream(data):
    return requests.post("https://downstream-service/api", json=data)

# SAFER — circuit breaker pattern
from circuitbreaker import circuit

@circuit(failure_threshold=5, recovery_timeout=30)
def call_downstream(data):
    return requests.post("https://downstream-service/api", json=data)
# After 5 failures in a row, opens circuit and returns fast failure
```

### No Retry Limits with Backoff
```python
# VULNERABLE
while not success:
    result = api_call()

# SAFER
for attempt in range(MAX_RETRIES):
    try:
        result = api_call()
        break
    except TransientError:
        wait = (2 ** attempt) + random.uniform(0, 1)  # exponential backoff with jitter
        time.sleep(wait)
else:
    raise PermanentFailure("Max retries exceeded")
```

### No Intermediate State Snapshots
```python
# VULNERABLE — long pipeline with no recovery points
agent.run_full_pipeline(steps=[step1, step2, step3, step4, step5])
# If step4 fails, must restart from beginning; steps 1-3 may have already modified state

# SAFER — checkpoint after each stage
for step in [step1, step2, step3, step4, step5]:
    result = step.execute(current_state)
    checkpoint.save(step.name, result)  # resumable from last checkpoint
    current_state = result
```

---

## Pipeline Resilience Audit Checklist

- [ ] Is each pipeline stage's output validated before being passed to the next stage?
- [ ] Are there confidence/quality thresholds that trigger human review?
- [ ] Are irreversible actions (delete, send, publish, write) gated behind confirmation?
- [ ] Is there a circuit breaker on calls to external APIs or downstream services?
- [ ] Are retries bounded with exponential backoff?
- [ ] Are intermediate pipeline states checkpointed for recovery?
- [ ] Do error handlers fail loudly (raise exceptions) rather than silently continuing?
- [ ] Is there a rollback mechanism for multi-step operations?
- [ ] Are all output values range-checked (prices, quantities, dates) before use?
- [ ] Is there end-to-end monitoring with anomaly detection across pipeline stages?

---

## Severity Indicators

| Condition | Severity |
|-----------|----------|
| Irreversible actions (delete, send, financial) without any confirmation gate | CRITICAL |
| No output validation between pipeline stages | HIGH |
| Errors silently swallowed — empty/default values used as fallback | HIGH |
| No circuit breakers on external dependencies | HIGH |
| Unbounded retries | HIGH |
| Human review checkpoint before irreversible actions | LOW |
| Full validation, circuit breakers, and checkpointing in place | INFO |

---

## Mitigations

1. **Validate every inter-stage output** — define output schemas for each pipeline stage
   and enforce them. Reject and escalate invalid outputs rather than passing them forward.

2. **Implement confidence thresholds** — if an LLM's output confidence is below a
   threshold, route to human review rather than proceeding autonomously.

3. **Gate irreversible actions** — any write, delete, send, or financial transaction
   requires explicit confirmation — either programmatic (re-read + compare) or human.

4. **Add circuit breakers** on all external dependencies to prevent cascading failures
   from propagating upstream.

5. **Use bounded retries with exponential backoff** — never retry indefinitely.

6. **Checkpoint pipeline state** after each stage — enable recovery without re-running
   stages that already succeeded.

7. **Fail loudly** — prefer exceptions over default values on errors. Silent fallbacks
   are one of the most common sources of cascading failures.

8. **Monitor for anomalies across pipeline stages** — alert when output values are
   outside expected ranges, even if no exception was raised.
