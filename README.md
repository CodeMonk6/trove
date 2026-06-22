# Trove

**An AI shopping concierge that finds the right product, prices it across real sellers, and refuses to recommend anything it can't back with a cited source.**

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js%2016-000000?logo=nextdotjs&logoColor=white)
![React 19](https://img.shields.io/badge/React%2019-20232A?logo=react&logoColor=61DAFB)
![Vercel AI SDK](https://img.shields.io/badge/Vercel%20AI%20SDK-000000?logo=vercel&logoColor=white)
![Zod](https://img.shields.io/badge/Zod-3E67B1?logo=zod&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind%20CSS%204-06B6D4?logo=tailwindcss&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-3FCF8E?logo=supabase&logoColor=white)
![Redis](https://img.shields.io/badge/Upstash%20Redis-DC382D?logo=redis&logoColor=white)
![Vitest](https://img.shields.io/badge/Vitest-6E9F18?logo=vitest&logoColor=white)

---

## What it does

Buying anything mildly considered — a carry-on, an air purifier, a pair of headphones — means stitching together a dozen review sites, half of them out of date, all of them trying to sell you something. The hard part isn't finding *a* product; it's trusting that the one you land on is actually right for *you*, and that the price you see is real.

Trove is a conversational concierge that does the legwork. You describe the need in plain language (or by voice). It asks at most a couple of sharp clarifying questions, surveys the live option space, prices the strongest candidates across real sellers, and returns a ranked top three — each justified by reasons that trace back to a specific source. Every claim it shows has been checked twice: once to confirm the source *mentions* it, and once to confirm the source actually *supports* it. If a recommendation can't survive that, it doesn't ship.

The result is a recommendation you can act on, with a "Buy" link to the cheapest in-stock seller and a transparent trail of *why*.

## How it works

Trove is a pipeline of small, single-purpose agents, each implementing a shared, strongly-typed contract. State flows one direction; agents *propose*, and a final Guardian *decides* what the user is allowed to see.

```
User turn (text / voice)
      │
      ▼
 Interpreter ──► structured requirement profile (category, constraints, soft prefs)
      │
      ▼
 Concierge ───► asks ≤2 high-value clarifying questions, or proceeds when ready
      │
      ▼
 Scout ───────► surveys the live option space (web search)
      │          then prices a shortlist across real sellers (shopping API)
      ▼
 Normalizer ──► merges noisy candidates into one clean schema per category
      │
      ▼
 Analyst ─────► reasons over candidates + real prices, drafts a ranked top 3
      │          (every reason carries a sourceId + evidence tokens)
      ▼
 Guardian ────► token-grounding gate: does the cited source actually say this?
      │
      ▼
 Entailment ──► semantic gate: does the source *support* the claim, or contradict it?
      │
      ▼
 Verified top 3  +  cheapest in-stock seller  +  full provenance trail
```

**The trust model is the product.** A claim cannot type-check without a source reference — the contracts make ungrounded recommendations impossible to construct in the first place. From there:

- **Gate one (grounding):** the Guardian checks that each claim's evidence tokens literally appear in the cited source's captured text. "Verified" means the source says it, not merely that a citation exists.
- **Gate two (entailment):** a cheap batched model call confirms the source actually supports the value — catching a "30h battery life" claim when the page says 20h. It runs only on claims that passed grounding, and fails *open* so a flaky model never silently drops a valid result.
- **Provenance everywhere:** every source is captured with a stable content hash and a fetch timestamp, so the same page or offer dedupes across a session and everything shown traces back to a real artifact.

**Cost and latency are engineered, not hoped for.** Models route through a single gateway so providers hot-swap with one key, and each stage is matched to the cheapest model that can do its job — reasoning where it counts, pennies everywhere else. Every upstream call (search, shopping, scrape, model) is bounded by a timeout with a deterministic fallback, so a stalled dependency degrades gracefully instead of hanging the request. Hot queries are cached behind a TTL interface, with volatile price data deliberately short-lived.

## Highlights

- **Multi-agent architecture with a hard trust boundary** — agents propose; a Guardian with veto power decides what renders. Nothing reaches the user un-verified.
- **Two-stage claim verification** — token-grounding plus entailment, so recommendations are defensible down to the individual reason.
- **Contracts as the source of truth** — a single set of Zod schemas defines every data shape across the pipeline; the type system itself forbids a claim without provenance.
- **Live, real-money pricing** — prices a wider shortlist *before* ranking, so the genuinely cheapest, in-stock option can win rather than a number scraped from review prose.
- **Self-extending to new categories** — when it meets a product category it doesn't know, it learns that category's defining dimensions once and registers them so the extractor and ranker share a vocabulary.
- **Conversational refinement** — follow-ups like "show me something lighter" or "cheaper" re-weight the profile and re-rank in place instead of starting over.
- **Streaming, transparent UX** — a live progress trace shows each stage as it runs; results carry a verified badge, a price comparison table, and source disclosure.
- **Voice input** — Web Speech API with a server-side speech-to-text fallback for browsers that lack it.
- **Graceful degradation by design** — every external dependency has a deterministic stub path, so the app stays demoable and testable with zero keys configured.
- **Tested behavior, not just code** — covered by an extensive suite spanning constraint enforcement, grounding, ranking quality, and end-to-end pipeline smoke tests.

## Tech stack

| Layer | Choices |
|---|---|
| **Frontend** | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4 |
| **AI orchestration** | Vercel AI SDK, multi-provider model routing via a unified gateway, structured generation with Zod |
| **Discovery & data** | web-search API for option surveys, Google Shopping API for live seller prices, URL→markdown reader for spec confirmation |
| **Backend** | Next.js API routes on a Node runtime (streaming NDJSON), Supabase, Upstash Redis (caching + rate limiting) |
| **Voice** | Web Speech API with a server-side Whisper transcription fallback |
| **Quality** | Vitest, ESLint, strict TypeScript, end-to-end deterministic fixtures |

## Status

Actively developed. The core pipeline — interpret → clarify → survey → price → reason → verify — is implemented and exercised by a broad test suite, with deterministic fallbacks at every stage so it runs end-to-end with or without live API keys. Source is kept in a private repository; this page is a high-level overview of the architecture and design.
