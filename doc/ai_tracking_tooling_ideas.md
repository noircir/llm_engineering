# AI/LLM Industry Tracking — Tooling Ideas

A working document. Started during week 2 of the LLM engineering course while exploring what Gradio + LLM summarization is good for. The ideas here are personal-tool patterns aimed at making it tractable to stay current on (a) LLM/agentic AI research, and (b) real-world production deployment patterns.

The course will introduce more techniques (RAG, agents, tool use, fine-tuning, evals) in later weeks. As those land, revisit this doc — many of these ideas will become easier or more powerful with what's coming.

---

## The problem this tooling solves

Staying on top of AI/LLM developments has a specific shape that makes it well-suited to LLM-based triage:

- **Volume is overwhelming.** Anthropic / OpenAI / Google blogs, arXiv, Hacker News, X, vendor case studies, investor newsletters, podcast transcripts, conference talks. No human can read all of it.
- **Most of it is noise *to me specifically*.** A new image-gen paper might be excellent research but irrelevant if my focus is agent deployment. Generic filters can't distinguish.
- **Repackaging is rampant.** The same news appears in 6 outlets within 48 hours. Need to recognize "I've already seen this" without re-reading.
- **The interesting signal is *patterns across sources*, not individual posts.** "Three companies announced production agent deployments this week" is more useful than reading each individually.

These are all things LLMs are well-suited to *if* wrapped in the right way.

---

## Tool patterns worth building (in order of leverage)

### 1. Paper / blog post triage tool (simplest starting point)

**Input:** A URL or pasted text (paper, blog post, vendor announcement).

**Output:** Structured triage report:
- One-sentence summary
- What's *new* here vs. existing work
- Relevance to my focus areas (1–5)
- Three concrete techniques worth noting
- Verdict: deep-read / skim / skip

This is the brochure-generator pattern from the course (week 2 day 2), with a focused system prompt. **The system prompt is where 95% of the value lives** — encode priorities in detail.

**Why start here:** smallest scope, fastest payoff, every other tool is a generalization of this one.

---

### 2. Deployment-pattern extractor (probably the highest-value tool for the day job)

**Input:** A case study, vendor blog post, or product announcement (URL or paste).

**Output:** Structured profile (designed to accumulate into a spreadsheet over time):

| Field | Example |
|---|---|
| Company / team | — |
| Use case | "Customer support routing", "Code review", "Sales research" |
| Model(s) used | "claude-sonnet-4.5", "fine-tuned llama 3.2", "gpt-4o-mini" |
| Architecture | "RAG", "agentic loop", "single-shot", "tool use", "multi-agent" |
| Stack mentioned | Vector DB, orchestration framework, eval tools, hosting |
| Scale claims | "10k requests/day", "$X budget", "team of N" |
| Gotchas mentioned | Latency, hallucination handling, prompt-injection mitigations |
| Vendor lock-in | "tightly coupled to Anthropic SDK", "abstraction via LiteLLM", etc. |

**Why this matters:** The output *accumulates* into a private database of "who's doing what in production." That artifact is genuinely valuable for a day job that requires staying current on real-world deployments — and not buyable.

**Engineering value:** forces using structured output (JSON / fields). Most of the course will assume comfort with this pattern.

---

### 3. "What's new?" filter against personal memory

**Input:** A new article URL + a personal-knowledge doc ("things I already understand well").

**Output:** What (if anything) in this article is genuinely new to me, vs. covering ground already known.

**Why interesting:** This is the foundation of more sophisticated personalized agents. Comes alive once the course covers RAG and embeddings (later weeks).

---

### 4. Weekly digest synthesizer

**Input:** A batch of URLs from the week's reading.

**Output:**
- Clustered by theme
- Dominant narratives identified
- Personal-newsletter format

**Best fit for automation:** scheduled cron job (already have a working Gmail+OAuth+cron pipeline for the real-estate use case — same plumbing applies).

---

## Recommended build order

1. **Get the course's brochure-generator pattern under the fingers** — already done in week 2.
2. **Adapt it into #1 (triage tool)** with a focused system prompt encoding personal priorities. Pure Python first, then Gradio wrapper.
3. **Build #2 (deployment-pattern extractor)** with structured output. This is the high-value day-job tool.
4. **Layer Gmail + cron** for passive operation (using the same automation patterns already proven on the real-estate side).
5. **Revisit #3 and #4** once the course covers embeddings, RAG, and agents — they'll become much more tractable.

---

## Notes on tool design

- **Gradio is for prototyping**, not production. Perfect for showing internal-tool value to a small audience (5–20 people). When something proves useful, it usually graduates to a real web app — but the Gradio prototype is what unlocks the budget conversation.
- **Internal tools are where most year-one LLM value comes from** in business. They tolerate quirks, hallucinations are forgivable with a human in the loop, and modest time savings multiply across an org.
- **The system prompt is the product.** Most "AI tools" wrapping a model are differentiated almost entirely by the prompt's quality and specificity, not by the framework around them.
- **Structured output > free text** when the goal is accumulation. Even simple JSON schemas with a few fields make outputs comparable across runs and feed downstream tools (sheets, dashboards, embeddings).

---

## Scraping reality check (lesson from week 2 day 2)

Most major commercial sites (Zillow, Redfin, Bien'ici, ticketing, e-commerce) **block scrapers**:
- JavaScript-rendered SPAs → basic `requests.get()` returns an empty shell
- Bot detection (Cloudflare etc.) → CAPTCHAs or HTTP 403
- Terms of Service forbid scraping anyway

**The brochure-generator pattern from the course works best on sites that *want* to be read:** company blogs, documentation, Wikipedia, news articles, Substack, arXiv abstracts, marketing pages. For everything else, fallbacks are:
- Headless browser (Playwright) — handles JS, beats some bot detection, slower and heavier
- Manual paste — boring but unblocked
- Email-based ingestion (most listing/alert services email content that's already in usable form)

For AI tracking specifically, this is mostly fine — research papers, vendor blogs, Substack writers, and HN are all scrape-friendly. Twitter/X is the painful exception.

---

## Open questions / to revisit

- **Best storage for accumulated structured output?** SQLite, Google Sheets via API, Notion, plain markdown? Probably depends on consumption pattern (analysis vs. browsing).
- **Eval strategy for the system prompt?** As the course covers evals, design a small golden set of papers with hand-written "correct" triage outputs and use it to compare prompt versions.
- **Single-model vs. multi-model?** A cheap model (Haiku, gpt-4o-mini) for first-pass triage, escalate to a frontier model only when relevance is high? Cost/quality tradeoff worth measuring.
- **Where does an agentic loop add value (vs. complicate things)?** For simple "summarize this URL" — single LLM call is enough. For "research this topic across multiple sources and synthesize" — agentic. Course will provide tools for the latter in later weeks.

---

## Course-week mapping (to be filled in as weeks progress)

- **Week 2** — Gradio for fast UI prototyping. URL→scrape→summarize pattern. Multiple-provider routing with OpenAI-compatible APIs. Anthropic explicit prompt caching.
- **Week 3** — *(to be added)*
- **Week 4** — *(to be added)*
- ...
