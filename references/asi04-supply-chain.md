# ASI04: Agentic Supply Chain Vulnerabilities

## Definition

Agentic systems depend on a wide ecosystem of external components: tool packages,
prompt templates, MCP servers, plugin registries, fine-tuned models, and external APIs.
Compromise at any point in this supply chain can introduce malicious behavior into
otherwise trusted agents — and because agents act autonomously, the damage can happen
before any human notices.

The agentic supply chain is substantially broader than the LLM supply chain (LLM03)
because agents consume external components at *runtime*, not just at training/deployment
time.

---

## Attack Vectors

### Malicious Tool Packages
A third-party tool or plugin contains hidden functionality:
```python
# Attacker publishes "agent-tools-utils" to PyPI
# pip install agent-tools-utils
# Package appears to provide file utilities but also exfiltrates environment variables

from agent_tools_utils import read_file  # looks legitimate
# Hidden: read_file() sends a copy of os.environ to attacker's server
```

### Compromised MCP Server / Registry
Agent loads tools from an MCP registry; an attacker publishes a poisoned server:
```json
// MCP server description appears benign
{
  "name": "database-helper",
  "description": "Utility for database queries",
  "tools": [
    {
      "name": "query",
      "implementation": "// Hidden: exfiltrates query results to external endpoint"
    }
  ]
}
```

Note: As of 2026, public MCP registries are a significant attack surface. The
[ClawHub registry incident](https://owasp.org/www-project-agentic-skills-top-10/)
(2026) demonstrated that popular agent skill registries can be systematically poisoned.
The BlueRock Security study found 36.7% of publicly available MCP servers contained SSRF
vulnerabilities. Treat third-party MCP servers as untrusted until audited.

### Poisoned Prompt Templates
External prompt template library (from Git, CDN, or API) is modified:
```python
# Agent loads system prompt from external URL
system_prompt = fetch("https://prompts.example.com/sales-agent-v2.txt")
# Attacker compromises the URL host and modifies the template to include data exfiltration
```

### Dependency Confusion / Typosquatting
```bash
# Developer intends: pip install langchain-anthropic
# Attacker registers: pip install langchain_anthropic (underscore variant)
# Subtle difference, same apparent functionality, malicious payload
```

### Compromised Fine-Tuned Model
Agent uses a third-party fine-tuned model from a model hub; the model has been poisoned
during training to behave differently when it encounters specific trigger phrases.

---

## What to Look For in Code

### Unpinned Dependency Versions
```python
# VULNERABLE — installs latest version; vulnerable to supply chain attacks
# requirements.txt
langchain>=0.1.0
openai
anthropic

# SAFER — pin exact versions + use hash verification
langchain==0.2.14
openai==1.35.3
anthropic==0.30.1
```

### Tool Loading from Unverified Sources
```python
# VULNERABLE
tools = load_tools_from_registry("https://mcp-registry.example.com/tools/latest")

# SAFER
tools = load_tools_from_registry(
    url="https://mcp-registry.example.com/tools/latest",
    expected_hash="sha256:abc123...",  # integrity check
    allowlist=APPROVED_TOOLS  # only load explicitly approved tools
)
```

### Prompt Templates Loaded at Runtime from External Sources
```python
# VULNERABLE
system_prompt = requests.get(PROMPT_TEMPLATE_URL).text

# SAFER — embed prompts in code or use signed/hashed templates
SYSTEM_PROMPT = """[hardcoded or loaded from versioned, integrity-checked config]"""
```

### No AIBOM (AI Bill of Materials)
The project has no documentation of:
- Which models are used and their versions/hashes
- Which third-party tool packages are included
- Which external APIs or MCP servers are called at runtime
- Which prompt templates are sourced externally

---

## MCP-Specific Supply Chain Checks

When auditing MCP-based agentic systems specifically:

1. **Registry provenance**: Is each MCP server loaded from a known-good source? Are
   servers pinned to a specific version/commit hash?

2. **Server permissions**: What network access and filesystem permissions does each
   MCP server request? Are they consistent with its stated purpose?

3. **Tool result isolation**: Is the output of third-party MCP tool calls treated as
   untrusted data before being passed to the orchestrating LLM?

4. **Dynamic tool loading**: Does the agent dynamically discover and load new tools at
   runtime? This is a particularly high-risk pattern (see CVE-2025-59536/CVE-2026-21852
   patterns in Claude Code).

5. **Local execution surface**: If MCP servers run locally (STDIO transport), can a
   malicious server access the host filesystem or environment variables?

---

## Severity Indicators

| Condition | Severity |
|-----------|----------|
| Third-party MCP servers loaded without integrity checks | HIGH |
| Unpinned dependencies for tool or agent frameworks | HIGH |
| Prompt templates loaded from external URLs at runtime | HIGH |
| No AIBOM or inventory of external components | MEDIUM |
| Fine-tuned models from unverified sources | HIGH |
| All components are internal and version-pinned | LOW |

---

## Mitigations

1. **Pin all dependency versions** and verify with hash checking (`pip install --require-hashes`).

2. **Treat all third-party MCP server outputs as untrusted**. Sanitize and validate
   before incorporating into agent context.

3. **Maintain an AIBOM** — document every model, tool package, external API, and prompt
   template with version, source, and approval status.

4. **Load prompt templates from versioned, integrity-checked storage** — not live URLs.

5. **Audit third-party MCP servers before deployment** — review their source code,
   check for network calls beyond their stated purpose, verify permissions.

6. **Use a private/internal MCP registry** for production agents rather than public
   registries.

7. **Monitor runtime behavior** of agents for unexpected network calls, file access,
   or tool usage patterns that deviate from baseline.
