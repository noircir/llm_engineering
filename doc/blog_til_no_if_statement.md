# TIL: I looked for the `if` statement, and it wasn't there

*Notes from my LLM engineering course — the moment a small exercise rearranged how I think about software.*

---

I am working through an LLM engineering course week by week. The exercises are small Python notebooks: read a webpage, summarize it, build a chatbot, hook up multiple providers. Standard stuff. Most of it has been *new* without being *strange* — I'm a programmer; new tools and APIs are part of the job.

This week was different. I had a small "click" moment while building a customer-service chatbot for a fictional airline, and I want to write it down before it stops feeling new.

## The exercise

The setup: a tiny Python program that pretends to be an AI assistant for an airline called FlightAI. You type "I'd like to fly to Tokyo," and the assistant replies with the ticket price.

The interesting twist is that the **price isn't in the model's training data**. It comes from a Python function:

```python
def get_ticket_price(destination_city):
    prices = {"london": "$799", "paris": "$899", "tokyo": "$1400"}
    return prices.get(destination_city.lower(), "Unknown price")
```

So how does the chatbot get the price into its reply? The course introduces a feature called **tool calling**. You tell the model, on each API request, "by the way, here's a function called `get_ticket_price` you can use, here's what it does, and here are its parameters." The model is then free to decide — turn by turn — whether to reply with text or to ask you to run the function.

The notebook walks through the implementation. You type "Tokyo," the model returns a `tool_call` payload instead of text, your Python code executes the function, you append the result back to the conversation, you call the model a second time, and now it produces a reply that knows the price.

Everything works on the first try. I should have been satisfied.

Instead I got stuck.

## What I expected

I am used to reading code and finding the conditional that connects an input to an outcome. Somewhere in this notebook, I expected to find something like:

```python
if "city" in user_message or "fly to" in user_message:
    call_get_ticket_price(extract_city(user_message))
```

That kind of rule-based glue is what *programmers* do. It's how the connection between "user typed something" and "code ran" is normally made explicit. I went looking for it, line by line.

It wasn't there.

The only thing the code does is: take the user's message, attach the tool definition, send it to the model, and inspect what comes back. If the response has `finish_reason == "tool_calls"`, the code runs the requested function. That's it. There's nothing that examines the user's text for keywords. Nothing that maps intent to action. **No `if` statement.**

I asked my tutor: *"How did the model decide to use this specific tool? What's the programmatic trace?"*

The answer, when it landed, rearranged something:

> *There is no programmatic step. The model decides by context.*

## Why this is hard to absorb

For a programmer, "decisions are made by code I can read" is so deeply assumed that it's almost not a belief — it's the ground you stand on. When you debug, you trace through branches. When you onboard onto a new codebase, you read its conditionals to understand what it does. When you optimize, you reason about which paths get taken.

In an LLM-based system, **a significant portion of the decision logic does not live in code you can read.** It lives inside the model's weights, shaped by training. The prompt and the tool descriptions you supply are *inputs* to that decision, not the decision itself.

Concretely: in the airline chatbot, the model picks `get_ticket_price` and the argument `"Tokyo"` because:

1. The tool's description said "get the price of a return ticket to the destination city."
2. The assistant just asked "what is your destination city?"
3. The user replied "Tokyo."
4. The model was trained on millions of examples of (conversation, tool list, correct response) triples, so it has learned the pattern: when these inputs align, emit a tool-call payload with the city as the argument.

That last bullet is the one I had to sit with. The "decision" is **trained pattern completion**. There is no deliberation. There is no `if`. There is no human-written rule. The model produces the most probable next output given everything it has seen, and "the most probable next output" happens to be a tool-call payload with the right arguments.

## The cascade of consequences

Once I started actually believing this — not just nodding at it — a cluster of consequences fell into place. Things people had said to me that sounded like cliches suddenly meant something.

**The same input can produce different outputs.** Of course it can. The model is sampling from a distribution. With temperature > 0, the sampling is explicitly probabilistic. Even at temperature 0, the model's behavior is shaped by floating-point quirks and inference-time non-determinism. "It worked once" is no longer evidence that it works.

**The prompt is the new API contract.** Because the model's behavior is not specified in code, the only way to *influence* it is through what you put in front of it: system prompt, tool descriptions, examples, parameter names. Rewording a tool description by one sentence can change reliability by ten percentage points. This is weird and uncomfortable. It feels like persuasion, not engineering. But it's where the leverage is.

**Testing changes shape.** `assert f(x) == y` is meaningless when `f` is a stochastic function. The replacement is evals: a benchmark set of inputs with expected behaviors, and you measure aggregate performance ("calls the right tool 95% of the time"). This is how every serious LLM team ships features. Without evals, you are flying blind.

**Debugging is observational.** You can't set a breakpoint inside the model's decision. When something goes wrong, you look at the prompt, look at the response, look at the conversation transcript, and you ask: *"what could I have written differently?"* The skill becomes pattern-spotting in conversation logs. There are no stack traces from the model's interior.

Each of these would have sounded abstract a week ago. After staring at the airline notebook for an hour, they feel obvious.

## Why I'm writing this down

I have a day job that requires me to stay current on how LLMs are being deployed in real products. I talk to engineers about LLM-based features all the time. And I'm pretty sure that the conversations I have *after* this insight are going to be different from the ones I had *before*.

The single most common mistake I anticipate hearing from engineers new to this space is some version of:

> *"Why isn't this deterministic? Let's just hardcode the cases."*

There's a real answer to that question, and it's worth being ready to give it: hardcoding works for the cases you can enumerate, but the entire reason to use an LLM is to handle the cases you *cannot* enumerate. If you could write out all the user intents in advance, you wouldn't need a language model. The right tradeoff isn't determinism — it's covering the cases you care about with evals, accepting some variance, and using deterministic code where there's no language judgment involved.

The other one I anticipate:

> *"It worked when I tested it. Ship it."*

That's not testing. That's a single sample from a distribution. Anyone shipping LLM features on the basis of single runs is going to be surprised in production, and the surprises will not be small.

## The shape of the new job

I think there's a real and important shift happening in what "software engineering" means when LLMs are in the loop. It's not that the old skills go away — you still need to write code, design APIs, manage state, think about performance. It's that a new layer is added on top, where the unit of work is **shaping a context** rather than writing logic. The context includes system prompts, tool descriptions, examples, retrieved documents, conversation history. The "code" you write at this layer is text, but it's text that's read by something that produces behavior, not just text that's displayed.

Some people call this "programming with natural language." That phrase made me roll my eyes a year ago — it sounded like the kind of thing you'd put on a startup landing page. After this week's exercise, it has acquired a specific technical meaning for me: it's what you do when the system's branching logic is not in your code but is influenced by your inputs.

I think the engineers who will be good at this will be ones who get comfortable with that influence-not-control relationship. The ones who keep trying to make the LLM deterministic by piling on rules will end up with brittle systems that are worse than either pure-code solutions or pure-LLM solutions. The ones who learn to trust the model's pattern-matching while bounding it with good evals, monitoring, and human-in-the-loop will build the systems that actually work.

## What I'm taking with me

A short list, for me to come back to when I forget:

1. **There is no `if` statement.** When something surprising happens in an LLM system, the answer is in the prompt and the training, not in the surrounding code.
2. **The prompt is the product.** Two apps using the same model can differ entirely on the basis of their prompt and tool design.
3. **Evals replace unit tests.** Anyone who tells you "I tested it" without a benchmark set has not actually tested it.
4. **Read the whole response.** Modern models hedge before they commit. The most important content is often at the end.
5. **Grounding beats memory.** "The model knows X" is a weak guarantee. Give it the source material in context.
6. **Models don't run code; they request it.** The execution boundary is yours. The safety boundary is yours.

I've spent a year hearing this stuff in podcasts and blog posts. Reading is not the same as understanding. The understanding came from staring at a six-line function definition and realizing that *the actual decision wasn't in the code at all*.

It's a strange feeling. Mostly a good one. I think I just got promoted into a category of engineering that I was already pretending to be in.

---

*Posted from the middle of week 2 of the course. More to come as later weeks land.*
