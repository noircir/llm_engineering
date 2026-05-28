# Daily Update - Key Mental Models for LLM-Based Systems

> A short reference to ground conversations with engineering teammates about why LLM-driven products behave the way they do. Drawn from working through hands-on tool-calling exercises and the realization that **the model's "decision" of which tool to call is not made by code - it is made by the model itself, based on context and training.**

## Context (one line)

Working through an LLM engineering course; the tool-use / function-calling pattern revealed a fundamental difference between LLM-based systems and traditional software that I want the team to be aligned on before we ship.

## The core insight

In traditional code, decision logic is written by the developer and executed deterministically. In LLM systems, **a significant portion of the decision logic lives inside the model and is influenced — not controlled — by the prompt and tool descriptions we supply at runtime.** There is no `if` statement that connects "user said X" to "call function Y." The model emits the tool call because its training has taught it that this is the most probable response given the inputs.

## What this means in practice

- **LLM behavior is probabilistic.** The same input can produce different outputs. "It worked when I tested it once" is not evidence of reliability.
- **The prompt is the API contract.** System prompts and tool descriptions are the primary lever for influencing behavior. They deserve the same care as production code: version control, review, evals.
- **Evals replace unit tests.** Single-run assertion-based tests do not capture LLM behavior. We need benchmark sets that measure aggregate performance ("calls the right tool 95% of the time").
- **Debugging is observational.** No breakpoints inside the model. We look at prompts, responses, and conversation transcripts.
- **Models request execution; they do not perform it.** The Python code retains control of what runs. This is both the safety boundary and the integration point.

## What I expect to push back on, from the team

These are the patterns I anticipate hearing — and what I want us aligned on as responses:

**"Why isn't this deterministic? Let's hardcode the cases."**
Hardcoding works for known cases but defeats the value of using an LLM, which is generalizing to inputs we cannot enumerate. If we could write out all the cases, we wouldn't need a language model. The right tradeoff is evals for the cases we care about, plus deterministic code for the parts that have no language judgment.

**"It worked when I tested it. Ship it."**
A single run is one sample from a distribution. We need behavior measured across a representative set of inputs before claiming reliability. Single-run testing of probabilistic systems is one of the most common sources of production failures in LLM apps.

**"The model knows X, just ask it."**
For factual questions, the model's training-time knowledge is unreliable and out of date. Grounding the model in retrieved source material (RAG) is the foundational pattern for factual reliability. "Ask the model" is acceptable for general knowledge tasks; it is not acceptable for anything where being wrong has a cost.

## Implications for how we build

1. **Treat prompts and tool descriptions as production code.** Version them, review them, test them.
2. **Build a small evals set early.** Before optimizing, before iterating on prompts — a golden set of representative inputs with expected outputs saves weeks downstream.
3. **Default to grounding for factual tasks.** Do not rely on the model's memory when source material is available.
4. **Choose models for the comprehension task, not for general capability.** Long-document reading, agentic reasoning, and code generation are different skills.
5. **Plan for variance in production.** Build retries, fallbacks, monitoring. Robustness is achievable; perfection is not.

## References

- Tool-calling implementation (week 2 day 4 of the course): airline AI assistant exercise.
- For deeper notes: see `llm_paradigm_shift_notes.md` in the same directory.

----------------


that's the shift, and it's one of the hardest mental moves for engineers coming from traditional codebases. You looked for the if/else/switch and there isn't one. The "branching logic" lives inside the model's weights, shaped by what it saw during training and what you happen to put in this single API call's context.

This realization has some downstream consequences:

1. **The same input can produce different outputs.** Tool selection is probabilistic, not deterministic. Run "Tokyo" 100 times, you might get 99 tool calls and 1 weird text reply, or some other distribution depending on temperature and model. Traditional programs are deterministic; LLM-based systems are statistical. Most production bugs in LLM apps come from people not internalizing this.

2. **Prompt engineering is the new "API contract."** Since there's no formal interface between user intent and tool call, the only lever you have to influence the decision is the prompt — system message, tool descriptions, examples, schema names. Spending an hour rewording a tool description can have a bigger impact on reliability than any amount of code refactoring. This is uncomfortable for engineers because it doesn't feel like "real" engineering work, but it's where the leverage actually is.

3. **Testing changes shape.** Unit tests of the form "f(x) == y" don't really fit LLM behavior. You move toward evals — running the model on a benchmark set of inputs and measuring behavior aggregate-style ("does it call the right tool 95% of the time?"). The course will get into this. It's a different mindset.

4. **Debugging is observational.** When the model picks the wrong tool, you can't step through with a debugger. Instead you look at the prompt and the response and ask "what could I have written differently?" The skill becomes pattern-spotting in conversation transcripts.

This is fundamentally a new programming paradigm — sometimes called "programming with natural language" or "prompt programming." It feels weird at first (especially the part where the system can disagree with your intent), but once you stop fighting it, designing for LLM-driven control flow becomes natural. The tool-use pattern you just saw is the simplest example; in later weeks you'll see this generalized into agents that pick from many tools and chain calls together, and the same mental model applies — soft, contextual, trained behavior, not hard-coded logic.