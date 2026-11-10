# Prompt: audit an LLM pipeline in a codebase (read-only)

**Use with:** a coding agent (Claude Code etc.) inside the repo.
**Origin:** Ch 5 study; first used on the report-generation service.

```
Analyze this codebase's LLM <FEATURE> flow. Do NOT change any code — read and report only.

Trace the full path: input data → prompt construction → LLM call → output → parsing/validation → downstream use. Cite file paths and line numbers for every finding.

1. Prompt design
   - Where are prompts defined (inline, separate file, templates)? Versioned?
   - System vs user prompt split.
   - Persona, explicit instructions, few-shot examples, output format spec?
   - One big prompt or decomposed into chained calls?

2. Context
   - What data is injected and how? Size limits or truncation?
   - Where do key instructions sit (start, middle, end)?
   - Any untrusted data (user text, scraped content)? Prompt injection risk?

3. Structured output
   - How is JSON enforced: json_schema strict / tool calling / json_object / prompt-only?
   - Validated against a schema? What happens on invalid output — retry, repair, crash, silent default?
   - Is there a reasoning field, and does it come BEFORE the answer fields?

4. Reasoning / quality
   - Chain-of-thought, self-critique, second pass?
   - Grounding checks (are claims supported by the input)?

5. Operations
   - Model, temperature, max tokens, reasoning effort, timeouts, retries.
   - Logging of full prompts and raw responses? Eval/test set?
   - Calls per request and latency/cost hotspots.

Output:
- Short text flow diagram.
- Table: Area | Current implementation | Risk/weakness | Suggested improvement (necessary vs optional).
- Top 3 highest-impact fixes, ranked.
If something isn't in the code, say "not present" — don't assume.
```