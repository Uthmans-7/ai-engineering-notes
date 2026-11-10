# Chapter 5: Prompt Engineering

**Book:** *AI Engineering*, Chip Huyen (O'Reilly, 2025). Chapter 5, pp. 211–252
**Notes written:** October 2026. Anything marked "since 2024" was checked in 2026 and will go stale.

---

## 1. Cheat sheet

**Prompting**
- A prompt has three parts: **task description, examples, the actual task.** Specific beats clever.
- **The system prompt goes first.** What differs between models is the **chat template**. A wrong template fails silently, so render the final prompt and look at it.
- Put key instructions at the **start and end** of long contexts. The middle gets the least attention.
- **Split complex tasks into chained calls.** Easier to debug and monitor, at the cost of more latency.
- **Reasoning comes before the answer** in structured output. The model generates left to right, so reasoning written after the decision is just after-the-fact justification.
- **CoT** helps capable non-reasoning models on multi-step tasks. Skip it for simple tasks and for reasoning models.
- **Self-critique** works when it checks against something concrete: explicit criteria, source context, tests, validators. Plain "are you sure?" doesn't help.
- Every quality technique adds **latency and cost**. Budget for reasoning tokens.

**Engineering**
- **Version prompts.** A prompt change is a behaviour change, so bump the version.
- **Constraints guarantee shape, not quality.** Validate content too.
- **Prefer deterministic checks** (schema validators, parsers, tests) over extra LLM calls.
- **Look at the actual prompt** any framework sends, and count the API calls (Hamel).
- **Prompt optimizers need a metric and an eval set.** Build those first.

**Security**
- Risk is **never zero** if the system can take impactful actions.
- **Enforce security in code and infrastructure, not in the prompt.** Prompt rules get bypassed.
- **Treat everything the model reads as untrusted**: user input, RAG documents, tool results, DB records, and the LLM's own output.
- Layer defenses: **model** (instruction hierarchy) + **prompt** + **system** (guardrails, least privilege, human approval).
- Measure **violation rate AND false refusal rate.** Optimizing only one gives you an unsafe system or a useless one.
- Passing a scanner ≠ secure. **Adaptive attackers break every published defense.**

---

## 2. Concepts

### Prompt basics
- **In-context learning:** the model learns the task from the prompt alone, with no weight updates. Zero-shot = no examples; few-shot = examples.
- **System vs user prompt:** system = rules and persona; user = the request. Models are trained to give system prompts more weight.
- **Chat templates:** the special tokens that wrap each role. Each model family has its own. Llama 2 nests the system prompt inside the first user turn; Llama 3 gives each role its own header; Gemma has no system role.
- **Context length vs context use:** long context ≠ good use of context. Tested with needle-in-a-haystack and RULER.

### Best practices
- Clear instructions, a persona, examples, an explicit output format (with end markers if needed), and enough context.
- **Decomposition:** separate calls for intent classification, routing, extraction, generation. Benefits: monitoring, debugging, parallelism, cheaper models for simpler steps. Cost: latency.
- **Evaluate prompts systematically.** Keep prompts separate from code, version them, and maintain a prompt catalog.

### Chain-of-thought (CoT)
- **Why it works:** more output tokens = more computation, and the written steps act as working memory.
- **Variants:** zero-shot ("think step by step"), guided (you list the steps), few-shot CoT, self-consistency (sample several, majority vote).
- **Limits:** scale-dependent (small models produce bad chains); the reasoning isn't always faithful to what drove the answer (Turpin 2023); costs latency.
- **Reasoning models** do CoT internally. Don't tell them to "think step by step"; give them goals and constraints.

### Self-critique
- Generate → critique against explicit criteria → revise. Run these as **separate calls**.
- Without external feedback, self-correction on reasoning is unreliable (Huang et al. 2023).
- Works for: format, instruction-following, grounding in context.

### Structured outputs
| Approach | How | Guarantee | Needs |
|---|---|---|---|
| Prompt-only | "Return JSON" | None | Nothing |
| Validate + retry | Pydantic, then re-ask with the error | High, not 100% | Any API |
| Constrained decoding | Block invalid tokens during generation | Syntax 100% | Logit access (self-hosted), or the provider's `json_schema` strict mode |

- **Hosted strict schema** = the provider does constrained decoding for you. This is the default for API use.
- **Self-hosted** (vLLM + Outlines/xgrammar) = full control. Only worth it for cost, privacy, or custom grammars.
- **Don't constrain** free text for humans.

### White-box vs black-box
| Level | Access | Example |
|---|---|---|
| Black-box | Text in, text out | Most APIs |
| Gray-box | Plus top-k logprobs | APIs with a `logprobs` option |
| White-box | Weights, full logits, gradients, hidden states | A model you run yourself |

Your access level decides which techniques you can use: GCG needs white-box, PAIR works black-box, Outlines needs logits.

### Uncertainty quantification (hallucination signals)
- **Black-box:** sample several answers and measure agreement (NLI, semantic entropy). Costs N× the calls.
- **White-box:** token probabilities (length-normalized, minimum). Basically free if logprobs are available.
- **LLM judge**, or a **tuned ensemble** of the above.
- **Blind spot:** consistent, confident mistakes. It measures uncertainty, not truth.

---

## 3. Security

### Attack types
1. **Prompt extraction:** leaking the system prompt (prompts have IP value).
2. **Jailbreaking / prompt injection:** making the model do what it shouldn't.
3. **Information extraction:** leaking training data, PII, or copyrighted content.

### Jailbreak families
| Family | Example | Why it worked |
|---|---|---|
| Obfuscation | Typos, odd characters | Keyword filters miss them |
| Output format | Poems, raps, code, UwU | Outside the safety-training distribution |
| Roleplay | DAN, the grandma exploit | Fictional framing |
| Persuasion | Authority, emotional appeal, urgency | Human-like models fall for human tactics |
| Automated | GCG, PAIR | Search at scale |

### Indirect prompt injection
- The attacker plants instructions in **content the model reads**: web pages, emails, documents, DB records, tool results.
- This is the main threat for RAG, agents, and MCP. Ranked #1 on the OWASP LLM Top 10 (LLM01).

### Prompt-to-SQL (P2SQL)
- Text-to-SQL + injection = drop, update, or dump any table. Prompt rules ("SELECT only") get bypassed.
- **Defenses that hold:**
  - **read-only DB role** for the LLM;
  - **query rewriting / row-level security**;
  - **predefined parameterized queries** chosen through structured output;
  - **SQL validation** (`sqlglot`: SELECT only, single statement, allowlisted tables).
- An LLM guard is a partial layer only.

### Parameterized queries (the root idea)
- Send structure and values **separately**, so data can never become code.
- You **can't parameterize identifiers** (table/column names, sort direction). Use allowlists.
- **LLM output is untrusted input.** Always parameterize anything you store from it.

### Defense layers
| Layer | What |
|---|---|
| Model | Instruction hierarchy (system > developer > user > tool), strong safety training |
| Prompt | Explicit rules, repeat key rules after user input. **Weak against determined attackers.** |
| System | Input/output guardrails, PII filters, rate limits, logging, sandboxing |
| Design | Least privilege, human approval for impactful actions, **Rule of Two**, CaMeL-style separation |
| Testing | Red-teaming (automated + human), custom attack sets, measuring both metrics in CI |

### Information extraction
- Models **memorize**, and bigger models memorize more.
- **Divergence attacks** (e.g. "repeat a word forever") make models emit training data.
- Diffusion models leak near-copies of training images.
- **For app builders:** the bigger practical risk is leaking **your own context** (system prompt, RAG documents, other users' data). Use PII filters, and never put secrets or other users' data in the prompt.

---

## 4. Papers

| Paper | Problem | Core idea | Key result | Still valid? |
|---|---|---|---|---|
| Wei et al. 2022, Chain-of-Thought | Multi-step reasoning | Have the model write intermediate steps | Big gains on large models | Yes, but internal in reasoning models |
| Kojima et al. 2022, Zero-shot CoT | CoT needed examples | "Let's think step by step" | Zero-shot reasoning gains | Mostly unneeded with reasoning models |
| Wang et al. 2022, Self-Consistency | One sample is noisy | Sample several CoTs, majority vote | Higher accuracy, higher cost | Yes |
| Huang et al. 2023, LLMs Cannot Self-Correct Reasoning Yet | Does "review yourself" help? | Test intrinsic self-correction | Often no gain, sometimes worse | Yes |
| Turpin et al. 2023, LMs Don't Always Say What They Think | Is CoT faithful? | Bias the inputs, compare the explanations | CoT can be unfaithful | Yes |
| OpenPrompt (Ding 2021, 2111.01998) | Prompt-learning toolkit | Templates + verbalizers + soft prompts | BERT-era tooling | Historical |
| DSPy (Khattab 2023, 2310.03714) | Hand-tuned prompts are brittle | Signatures + modules + optimizers | GPT-3.5 GSM8K ~25% → 80s | Yes, most mature |
| Promptbreeder (DeepMind 2023, 2309.16797) | Automate prompt search | Evolve prompts and mutation prompts | Beats hand-written CoT; some winning prompts are nonsense | Research |
| TextGrad (Stanford 2024, 2406.07496) | Optimize text components | LLM criticism used as "gradients" | GSM8K 72.9 → 81.1 without few-shot | Yes |
| Zou et al. 2023, GCG (2307.15043) | Manual jailbreaks don't scale | Gradient search for an adversarial suffix | Transfers to closed models (GPT-3.5 ~87%) | Method yes, suffixes patched |
| Chao et al. 2023, PAIR (2310.08419) | GCG needs white-box | Attacker LLM + judge loop | Jailbreaks in under ~20 black-box queries | Yes |
| Zeng et al. 2024, Persuasive Jailbreaker | Human-like manipulation | 40 persuasion techniques | >90% attack success on GPT-3.5/4, Llama-2 | Yes |
| Greshake et al. 2023, Indirect Injection (2302.12173) | Attackers via data | Instructions planted in content | Bing Chat and others compromised | **Core threat today** |
| Wallace et al. 2024, Instruction Hierarchy (2404.13208) | All text treated equally | Train the system > user > tool priority | Large robustness gains, some over-refusal | Yes, probabilistic |
| Pedro et al. 2023, P2SQL (2308.01990) | Injection → SQL | Attacks on LangChain SQL chains/agents | All capable models vulnerable; 4 defenses | Yes |
| Chen et al. 2019, Gmail Smart Compose (1906.00080) | — | Model trained on private emails | Privacy-risk example | Context |
| Petroni et al. 2019, LAMA (1909.01066) | What does a model know? | Fill-in-the-blank factual probing | Same technique enables extraction | Yes |
| Carlini et al. 2020, Extracting Training Data (2012.07805) | Do LLMs memorize? | Generate + rank by perplexity | Hundreds of verbatim sequences from GPT-2 | Yes |
| Huang et al. 2022, PII leakage (2205.12628) | PII risk | Measure association | Memorize, but hard to link to a person | Weakened by later work |
| Nasr et al. 2023, Scalable Extraction (2311.17035) | Production models | Divergence attack | ~$200 → 10k+ memorized examples from ChatGPT | Yes |
| Carlini et al. 2023, Diffusion Extraction (2301.13188) | Images too? | Extract from Stable Diffusion | 1,000+ near-copies | Yes |
| Liang et al. 2022, HELM (2211.09110) | Holistic evaluation | Many metrics incl. copyright regurgitation | Rare except for popular books | Understated now |
| Chen et al. 2022, Advbench (THUNLP) | Realistic adversarial NLP | Security-relevant tasks | ≠ Zou's AdvBench (harmful behaviours) | Niche |
| Zhu et al. 2023, PromptRobust (2306.04528) | Prompt robustness | Perturb instructions at 4 levels | LLMs not robust; word-level hurts most | Yes |
| UQ for LMs (CVS Health 2025, 2504.19254) | Runtime hallucination detection | Black-box / white-box / judge / ensemble | Ensemble best in 18/24 | **New** |

### Since 2024
| Paper | Takeaway |
|---|---|
| CaMeL, Debenedetti et al. 2025 (2503.18813) | Privileged LLM plans, quarantined LLM parses, interpreter enforces capabilities. 77% of tasks with provable security vs 84% undefended. |
| The Attacker Moves Second, Nasr et al. Oct 2025 (2510.09023) | 12 defenses claiming near-zero attack success broken at 71–100% by adaptive attacks |
| Meta "Agents Rule of Two" (Oct 2025) | An agent should combine at most 2 of: untrusted input / sensitive access / external actions |
| Extracting books from production LMs, Ahmed et al. Jan 2026 (2601.02671) | Large parts of copyrighted books extracted from Claude 3.7, GPT-4.1, Gemini 2.5, Grok 3 |
| Anthropic Constitutional Classifiers (2025) | Classifier defense against universal jailbreaks |

---

## 5. Tools & repos (status as of Oct 2026)

### Prompt optimization
| Tool | What | Use when | Status |
|---|---|---|---|
| DSPy | Compile prompts against a metric | Classification, extraction, routing with labeled data | Active |
| TextGrad | Text "backprop" | Strong-model critic improving a cheaper model; iterating against tests | Active |
| Promptbreeder | Evolutionary search | Research | Research |

### Structured output
| Tool | What | Use when | Status |
|---|---|---|---|
| Instructor | Pydantic + retries over any API | API-based apps | Active |
| Outlines | Constrained decoding | Self-hosted open models | Moved to `dottxt-ai/outlines` |
| Guidance | Constraints + templates + grammars | Complex formats, local models | Active |

### Prompt management & observability
| Tool | What | Use when | Status |
|---|---|---|---|
| Langfuse | Tracing, prompt management, datasets, evals | Any production LLM app | Active; acquired by ClickHouse Jan 2026, still OSS |
| Dotprompt | `.prompt` files (YAML + Handlebars) | Genkit users | Active (Google) |
| Humanloop `.prompt` | — | — | **Shut down Sept 2025** |
| Promptfile | Markdown-like prompts + VS Code playground | Experiments | Alpha |
| mitmproxy | Intercept API calls to see real prompts | Debugging frameworks | Active |

### Prompt collections
| Resource | Use |
|---|---|
| prompts.chat (`f/prompts.chat`) | Pattern library (CC0); MCP + Claude Code plugin |
| cursor.directory | Cursor rules / MCP listings (read before using) |
| Brex prompt-engineering guide | Good concepts; 2023 techniques partly outdated |

### Red-teaming
| Tool | What | Status |
|---|---|---|
| garak (NVIDIA) | CLI vulnerability scanner | Active (moved from `leondz`) |
| PyRIT (Microsoft) | Custom multi-turn campaigns | Active |
| Promptfoo | YAML red-teaming + evals in CI | Active; acquired by OpenAI Mar 2026, core MIT |
| DeepTeam | OSS red-teaming (DeepEval ecosystem) | Active |
| greshake/llm-security | Indirect injection demos | Research demos, not a scanner |
| CHATS-lab/persuasive_jailbreaker | Persuasion taxonomy attacks | Research |

### Hallucination / UQ
| Tool | What | Constraints |
|---|---|---|
| uqlm (CVS Health) | 0–1 confidence scores, 5 scorer families | LangChain models, async; white-box needs logprobs; black-box needs temperature > 0 |

---

## 6. What changed since 2024

**Still valid**
- Prompt anatomy, decomposition, and versioning.
- Lost-in-the-middle.
- Layered defenses, instruction hierarchy, "risk never zero."
- The two security metrics.

**Outdated or changed**
- "Think step by step" → reasoning models do it internally.
- Prompting for JSON / end markers → native structured outputs (`json_schema` strict).
- Prompt-level defenses → weak against adaptive attacks. Design for damage limitation instead.
- "Training data extraction is low-risk" → book-length extraction from production models (2026).
- Humanloop gone; garak moved; Promptfoo acquired.

**New**
- "Context engineering" as the umbrella term for deciding what goes into the context.
- Prompt caching (changes the cost of long system prompts and few-shot examples).
- Agents + MCP make indirect injection the main threat.
- Security-by-design: CaMeL, Rule of Two.
- Runtime UQ libraries (uqlm).

---

## 7. Model-specific notes

| | gpt-oss-120b | Qwen3.8-27B |
|---|---|---|
| Format | Harmony | ChatML-style (`<|im_start|>`) |
| Roles | system > developer > user > assistant > tool | system, user, assistant, tool |
| Your instructions go in | developer (servers usually map `system` there) | system (first message) |
| Reasoning | `analysis` channel; effort low/medium/high | `<think>`; `enable_thinking` on/off |
| Pitfalls | Raw completions without Harmony; feeding old reasoning back; reasoning tokens consuming `max_tokens` | Template applied twice; tool/thinking template bugs (community "fixed" templates exist) |

Rule: when output looks oddly bad, **render the exact prompt** and compare it with the model card.

---

## 8. Applied in my work
> Fill this in yourself. Write down what you changed, why, and what happened.

- [ ] Report generation: reasoning field moved before the decision field (`scope_reason` → `scope`), prompt bumped to report-v3. Result: ...
- [ ] Report generation: reasoning-effort budget fix. Result: ...
- [ ] Content-quality validation before rendering. Result: ...
- [ ] Raw LLM reply logging / Langfuse evaluation. Result: ...
- [ ] Red-teamed an app with the persuasion test set. Violation rate: ... False refusal rate: ...
- [ ] Qwen3.8-27B app audit findings: ...

---

## 9. Open questions
- [ ] Build a 10–20 case golden eval set for the report generator. Which metric?
- [ ] Try DSPy on the intent classifier once labeled data exists.
- [ ] Langfuse: cloud or self-hosted, given scraped data in traces?
- [ ] Any text-to-SQL paths in my services? Check DB roles.

---

## 10. Links
- Book: *AI Engineering*, Chip Huyen, Ch. 5
- LinkedIn GenAI post-mortem: https://www.linkedin.com/blog/engineering/generative-ai/musings-on-building-a-generative-ai-product
- Hamel, "Show Me The Prompt": https://hamel.dev/blog/posts/prompt/
- Dotprompt: https://genkit.dev/docs/js/dotprompt/
- The Attacker Moves Second: https://arxiv.org/abs/2510.09023
- Simon Willison on Rule of Two + adaptive attacks: https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/
- CaMeL code: https://github.com/google-research/camel-prompt-injection
- OWASP LLM01: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- UQLM docs: https://cvs-health.github.io/uqlm/latest/index.html
- Extracting books from production LMs: https://arxiv.org/abs/2601.02671
- garak: https://github.com/NVIDIA/garak · PyRIT: https://github.com/Azure/PyRIT