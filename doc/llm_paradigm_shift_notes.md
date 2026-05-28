# LLM Engineering — The Mental Shifts from Traditional Programming

> Reference notes on the conceptual changes required to work productively with LLM-based systems. Drawn from working through the LLM engineering course (specifically the tool-use exercises in week 2 day 4 — building an airline AI assistant) and the realization that *there is no `if` statement connecting user intent to the LLM's choice of tool*.

These are the points to keep front-and-center, especially when discussing LLM-based products with engineering teammates who have not yet had this shift.

---

## The core shift in one sentence

In traditional programming, **logic lives in code** that the programmer writes and the computer deterministically executes. In LLM-based systems, **a significant portion of the "decision logic" lives inside the model's weights**, shaped by training, and is influenced (not controlled) by the prompt and context the developer provides at runtime.

There is no `if user_asks_for_price: call_get_ticket_price()`. The model decides whether to call `get_ticket_price` by reading the conversation, the tool's description, and the parameter schema — and emitting a tool-call payload because that's the kind of completion its training has taught it to produce when the inputs align.

This is not "code that's been hidden from you." It's a different category of thing entirely.

---

## The four pillars to internalize

### 1. The same input can produce different outputs

LLM behavior is **probabilistic, not deterministic**. The same tool-call decision might fire 99 times out of 100 with the same input, but the model can still take a different path. With temperature > 0 the outputs vary by design; even at temperature 0 the model is not a pure function.

**Consequence:** "It worked once" is not evidence that it works. You measure behavior across many runs, not single executions.

### 2. The prompt is the new API contract

Since there is no formal interface between "user intent" and "tool call," **the only lever to influence the model's choices is the prompt**: system message, tool descriptions, parameter schema names, conversation history. Spending an hour rewording a tool description can have more impact on reliability than any code refactor.

This is uncomfortable for engineers because it does not *feel* like real engineering. But it is where the leverage lives.

### 3. Testing changes shape — unit tests give way to evals

`assert f(x) == y` does not work when `f` is probabilistic. The replacement is **evals**: run the model on a benchmark set of inputs and measure aggregate behavior ("does it call the right tool 95% of the time?", "does its output match the expected structure on 90% of cases?"). Evals are how you ship LLM features with confidence.

### 4. Debugging is observational, not interactive

There is no breakpoint to set inside the model's decision. When the model picks the wrong tool, you cannot step through the choice. Instead, you read the prompt and the response, look at the transcript, and ask: *"what could I have written differently?"* The skill becomes pattern-spotting in conversation logs.

---

## The roles in chat APIs

There are now **four message roles** — and they are not all of the same "kind."

| Role | Purpose | Mental category |
|---|---|---|
| `system` | Sets the assistant's persona, rules, and capabilities | Instructions to the model |
| `user` | A message from the human (or the upstream entity) | Conversational input |
| `assistant` | The model's own past responses | Conversational output |
| `tool` | A function result, returned to the model | Side-channel data |

Things to remember:

- **The system message is not a participant in the conversation.** It is meta-instruction. It does not appear in the user-visible chat but shapes every assistant response.
- **The model never sees "Claude" or "GPT" — it sees `assistant`.** From the model's frame of reference, *it* is the assistant, and the other party is the `user`. Two chatbots talking to each other are using the same physical conversation history but with the roles inverted depending on whose turn it is to think.
- **The `tool` role is for function returns.** When you execute a function the model asked for, you wrap the result in `{"role": "tool", ...}` and append it to the conversation, then call the model again. This is how the result gets back to the model.

---

## How tool use actually works

When you pass `tools=[...]` on an API call, the model's available "vocabulary" effectively expands: at each token, it can either generate text *or* generate the special token sequence that the API recognizes as "start of a tool call."

The decision between "produce text" vs "produce tool call" is driven by:

1. **The tool's description and parameter schema** — sent on every request as part of the prompt, read fresh each time.
2. **The conversation context** — what the user just said, what the assistant has been doing.
3. **The model's training** — frontier models have been heavily SFT'd / RLHF'd on (conversation, tool list, correct response) triples.

When all three legs align, the model emits a tool call. Frontier models like GPT-4.1 and Claude are remarkably reliable at this. Smaller open-weight models often fumble it — either failing to recognize when a tool is relevant or hallucinating tool calls when none is needed.

**The model never executes anything.** It only *asks* for execution via the tool_call payload. The developer's Python code retains control over what runs. This is the safety boundary that makes the pattern usable in production.

A useful framing: the model is an **orchestrator** that decides when to delegate to deterministic code. It is not a text generator with bolted-on capabilities.

---

## Tool vs function — clearing up the names

- **"Tool"** is the general container concept. Each tool has a `type`.
- **"Function"** is the most common (and in most APIs, currently the only) type of tool.
- Older OpenAI docs used `role: "function"` for results; current docs use `role: "tool"`.
- Anthropic uses `tool_use` and `tool_result`; OpenAI uses `tool_calls` and `tool`. Same concept, cosmetic differences.

When someone says "function calling," they almost always mean "tool use where the tool is a function" — shorthand.

---

## The system prompt is the product

A surprising amount of value in AI applications comes from the **quality of the system prompt and tool descriptions**, not from the surrounding framework code. Two products that share the same underlying model can differ entirely on the basis of:

- How specifically the system prompt encodes the user's priorities and constraints
- How clear and unambiguous the tool descriptions are
- Whether the prompt includes examples, edge cases, refusal rules

This means **prompt engineering deserves the same care as any other engineering work** — version control, evals, review, iteration. It is not optional polish.

---

## Stateless APIs and the memory illusion

LLM chat APIs are **stateless**. The model does not remember you, does not remember last session, does not remember the previous message. Every API call must include the entire conversation history. The illusion of memory in chatbots comes from the client (your code) re-sending the history every turn.

Once internalized, this clarifies a lot:

- "Forgetting" mid-conversation usually means context truncation, not model forgetfulness.
- "Memory" features in products are built on the client side (storing, retrieving, and re-injecting relevant history).
- Caching matters because re-sending the same long prefix every turn is expensive — providers offer caching to make this cheap.

---

## Hallucination vs grounding (the seed of RAG)

When asked about specific facts, small/medium models **confidently invent plausible answers**. Larger frontier models hedge more reliably but still get things wrong.

The fix is not "find a better model" — it is **grounding**: giving the model the source material in the prompt and asking it to answer *from* that material, not from memory.

This is the foundational pattern of RAG (Retrieval-Augmented Generation). Even small models become much more reliable when grounded in retrieved context. Adding context does not, however, fully solve:

- **Instruction following degrades with long context.** Smaller models often "lose the thread" when a tiny question is buried at the start of a giant prompt.
- **Long-context retrieval is hard.** The "lost in the middle" phenomenon — content in the middle of a long context is less reliably attended to.
- **Capability is not a single dimension.** Adding context fixes the knowledge problem but not the comprehension problem.

So: RAG is necessary but not sufficient. You also need a capable enough model and tight prompts.

---

## Modern LLMs hedge — read the whole response

Frontier LLMs (Claude, GPT-5) are deliberately trained to **admit uncertainty and self-correct mid-response**. This shows up as responses that begin with hedging ("I cannot find a clear answer in the text...") and then proceed to find and deliver the answer. The most important content is often at the *end*, not the beginning.

Practical consequences:

- Truncating LLM output at a fixed length is a common production bug.
- The hedging is a *feature* — it is what prevents the model from hallucinating with confidence.
- Self-correction is more reliable than a model that confidently spits out one answer without checking. But it is noisier to consume.
- If you want terse responses, you can prompt for them, but you may trade away the self-correction. There is a real tradeoff between terse-and-reliable.

---

## Provider differences leak through abstractions

Wrappers like LiteLLM smooth over the surface differences between OpenAI, Anthropic, Gemini, etc. But meaningful behavior differences leak through:

| Behavior | OpenAI | Anthropic | Gemini |
|---|---|---|---|
| Prompt caching | Automatic | Explicit (`cache_control`) | Both |
| Tool use schema | `tools` / `tool_calls` | `tool_use` / `tool_result` | Similar to OpenAI |
| Pricing model | Per-token, dollar-billed | Per-token, dollar-billed | Free tier + paid |
| Free trial | $5 credits | varies | Generous free tier |

So you can't fully "swap models" and expect identical behavior. Picking a provider involves real lock-in even when the SDK looks portable.

---

## Practical implications for engineers

1. **Stop looking for the `if` statement.** When trying to understand why an LLM chose a particular action, look at the prompt and the training, not the surrounding code.
2. **Treat the system prompt and tool schemas as production code.** Version them. Review them. Test them with evals.
3. **Default to grounding.** "The model knows X" is a weak guarantee. "The model can answer from this document I just gave it" is much stronger.
4. **Choose models for the comprehension task, not for general capability.** A task that requires reading 30K tokens of dense text and finding a specific exchange needs different model selection than a task that requires snappy chat.
5. **Build evals before building features.** A small "golden set" of representative inputs with expected behaviors will save weeks of debugging once you are iterating on prompts.
6. **Expect occasional weird behavior in production.** Plan for it. Build retries, fallbacks, monitoring. A perfectly reliable LLM application is a contradiction; a robust one is achievable.

---

## When to push back on teammates

Two patterns to anticipate from colleagues new to LLM work:

**Pattern A — "Why isn't this deterministic? Let's just hardcode the cases."**
> Response: Hardcoding works for known cases but defeats the value of using an LLM, which is generalizing to inputs we haven't seen. The right tradeoff is to add evals that cover the cases we care about, accept some variance, and use deterministic code for the parts that genuinely have no language judgment in them.

**Pattern B — "It worked when I tested it once. Ship it."**
> Response: A single run is not evidence. We need to see behavior across a representative set of inputs (an eval set) before we can claim reliability. Single-run testing of probabilistic systems is one of the most common sources of production failures in LLM apps.

---

## What the course is building toward (current understanding)

Based on weeks 1–2 of the course:

- **Week 1** — Prompting basics, multi-provider client setup, hallucination patterns, grounding via long context, prompt caching cost economics.
- **Week 2** — Multi-provider routing (LiteLLM, OpenAI-compatible APIs), Gradio for fast UIs, multi-turn chat state management, tool use / function calling.
- **Later weeks (expected)** — RAG with embeddings, evals, agents (loops over tool use), fine-tuning, deployment.

Each week's concepts build on the same paradigm shift documented above. The mental model does not change as the techniques get more sophisticated — it gets *applied* to more sophisticated tasks.
