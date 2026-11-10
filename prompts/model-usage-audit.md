# Prompt: audit how an app uses a specific model (read-only)

**Use with:** a coding agent inside the repo. Replace `<MODEL>`.
**Origin:** Ch 5 study; written for an app using Qwen3.8-27B. Works for any model.

```
Analyze how this application uses the <MODEL> model. Do NOT change any code — read and report only. Cite file paths and line numbers for every finding. If something isn't in the code, say "not present" — don't assume.

1. Serving and connection
   - Hosted API (which provider) or self-hosted (vLLM, SGLang, Ollama, llama.cpp)?
   - Exact model ID, base URL, where configured.
   - If self-hosted: launch flags — reasoning parser, tool-call parser, max context, quantization, custom --chat-template.

2. Chat template ownership
   - Structured `messages` to a chat endpoint, or raw prompt strings built in code?
   - Any template applied in our code (apply_chat_template, manual special tokens, custom Jinja)?
   - Risk of the template being applied twice?

3. Prompt structure
   - Where prompts are defined; show system prompt(s) and user message construction.
   - Role order. Any unsupported roles (e.g. "developer")? System message not at position 0?

4. Thinking / reasoning mode
   - Is thinking enabled/disabled, or reasoning effort set? Where?
   - How reasoning output is handled: stripped, logged, shown, parsed?
   - In multi-turn flows, is previous reasoning removed from history?

5. Structured output and tools
   - JSON enforcement method; validation; behavior on invalid output.
   - Tool/function calling: how definitions are passed and results returned.

6. Inputs
   - Images/video sent? Encoding and sizing.
   - Untrusted data entering prompts; any injection handling.

7. Generation parameters
   - temperature, top_p, top_k, penalties, max_tokens, stop, seed; timeouts, retries, streaming.

8. Observability
   - Full prompts, raw responses, token usage (incl. reasoning), latency logged? Tests or eval sets?

Output:
- Text flow diagram: input → prompt build → model call → parsing → output.
- Table: Area | Current implementation (file:line) | Risk/weakness | Suggested improvement (necessary vs optional).
- Top 3 highest-impact issues, ranked.
- One fully rendered example of the exact request payload sent to the model (secrets redacted).
```