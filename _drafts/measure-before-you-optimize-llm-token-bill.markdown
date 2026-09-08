---
layout: post
tags: [llm, ai, cost-optimization, caching, observability, cli]
author: Nicolas Mugnier
categories: ai
title: "Measure Before You Optimize: Cutting an AI Coding Agent's Token Bill"
description: "A CLI agent burned through 8.5M input tokens in a single day. The popular fix would have saved 0.35%. Here is what the data actually said, and the three changes that mattered."
image: /assets/img/measure-before-you-optimize-llm-token-bill.webp
locale: en_US
---

I spent about €20 in a single day running an LLM coding agent locally. That is not catastrophic, but extrapolated over a month of daily use it stops being pocket change, and it was worth understanding before it became a habit.

My first instinct was the one everybody has: install an output compressor. There is a whole category of tools for this now — CLI proxies that intercept `git status`, `ls`, or `pytest`, strip the noise, and hand the model a compact summary instead of the raw output. The pitch is compelling and the numbers advertised are large: 60–90% reduction.

I decided to measure first. The measurement said the compressor would have saved me **0.35%**.

Here is how to find out where the money actually goes, and the three changes that turned out to matter.

---

## The four token classes

Before measuring anything, it helps to know that not all tokens on an LLM bill cost the same. For a cache-enabled API there are four distinct classes, and their prices differ by more than an order of magnitude:

| Class | What it is | Relative price |
|---|---|---|
| Input (fresh) | New content the model has never seen | 1× |
| Cache **write** | Content stored into the prompt cache | 1.25× |
| Cache **read** | Content served from the prompt cache | 0.10× |
| Output | What the model generates | 5× |

The interesting pair is cache write versus cache read: **12.5× apart for identical content**. Whether your conversation history is written or read is therefore a bigger lever than how large it is.

That is the part output compressors cannot help with, because it is not about volume at all.

---

## Measuring instead of guessing

My agent keeps a SQLite state database, with a `session_model_usage` table that records per-model, per-task token counts. Most agents keep something equivalent, and if yours does not, the provider dashboard usually breaks usage down the same way. One query:

```sql
SELECT model,
       SUM(api_call_count)      AS calls,
       SUM(input_tokens)        AS fresh,
       SUM(cache_write_tokens)  AS cache_write,
       SUM(cache_read_tokens)   AS cache_read,
       SUM(output_tokens)       AS output
FROM session_model_usage
WHERE last_seen > strftime('%s','now','start of day')
GROUP BY model;
```

The result for that day:

```
model            calls   fresh    cache_write   cache_read   output
--------------   -----   ------   -----------   ----------   ------
<large-model>      123   957 k       1 296 k      6 277 k     86 k
```

Two numbers jump out.

**Output is 86 k against 8.5 M of input.** The bill is ~96% input. Anything that optimizes what the model *writes* is targeting the wrong end of the pipe.

**Cache write is 1 296 k over 123 calls — about 10.5 k tokens per call.** That is the smoking gun. A healthy cache is written once and read many times. Mine was being rewritten on essentially every single call, at 1.25× while a read would have cost 0.10×.

Weighting each class by its price:

```
cache write   ~45%
fresh input   ~26%
cache read    ~17%
output        ~12%
```

Nearly half the bill was one pathology: paying premium rates to re-upload context the provider already had.

---

## What the compressor could have reached

Now the counterfactual. Same database, different question — how many bytes do tools actually contribute?

```sql
SELECT tool_name,
       COUNT(*) AS n,
       SUM(LENGTH(COALESCE(content,''))) / 1000 AS kchars
FROM messages
WHERE role = 'tool'
  AND timestamp > strftime('%s','now','start of day')
GROUP BY tool_name
ORDER BY kchars DESC;
```

```
tool_name        n    kchars
------------    --    ------
terminal        70       172
read_file       11        67
skill_view      12        47
search_files    17        35
web_search       3        31
execute_code    11        22
```

Total tool output: 396 kchars, roughly 99 k tokens of unique content. Shell commands — the only slice a CLI proxy can touch — are 172 kchars of that, about 43 k tokens.

Compress 70% of it, the optimistic end of the published range, and you save ~30 k tokens against a daily input of 8 531 k.

**0.35%.**

The arithmetic is not a criticism of these tools. Their compression is real and often elegant. The problem is the denominator: shell output is one contributor to input tokens, input tokens are one part of the bill, and the reduction dilutes at every step. A vendor claim of "90% of bash output" is perfectly honest and still nearly irrelevant to your invoice. JetBrains ran an independent benchmark on the same class of tool and landed on a ceiling around 3% — an order of magnitude above my case, still not where the money is.

File reads, search results, and web fetches — 224 kchars here, more than the shell — bypass a CLI proxy entirely.

---

## The three changes that mattered

### 1. Cache TTL: 5 minutes → 1 hour

The default prompt-cache TTL was five minutes. My working rhythm is: ask a question, read the answer, think, read some code, come back. Regularly more than five minutes between turns.

Every time that gap exceeded the TTL, the cache entry expired and the entire conversation prefix was rewritten at 1.25× instead of being read at 0.10×.

```bash
# 5m → 1h
<agent> config set prompt_caching.cache_ttl 1h
```

One line, and it addresses the ~45% slice. Longer TTLs sometimes carry a small storage premium — still trivial compared to a factor of 12.5.

**Check the key name.** I first set a plausible-looking top-level `cache_ttl` and got a warning that it was not a recognized key: saved to the config, read by nothing. The real key was nested under `prompt_caching`. A silently-ignored setting looks exactly like a setting that did not help.

### 2. Route auxiliary tasks to a small model

Modern agents do not make one API call per turn. Alongside the main conversation they run side tasks, each with its own call: context compression, session-title generation, approval classification, memory-query rewriting, skill curation, background self-improvement passes.

Every one of those was configured as "inherit the main model" — so a session title, three words long, was being generated by the most expensive model available. A limousine to fetch bread.

```bash
for task in background_review title_generation approval compression \
            memory_query_rewrite skills_hub curator monitor; do
  <agent> config set auxiliary.$task.model  "<small-model>"
  <agent> config set auxiliary.$task.provider "<provider>"
done
```

The background self-improvement task alone was ~8% of the day's tokens. Its own documentation noted that running it on a non-default model replays a compact digest rather than the full conversation — 3–5× cheaper before the per-token price difference even applies.

### 3. Fix the auxiliary calls that were silently failing

This one was a genuine bug, and I only found it because I tested the previous change instead of trusting it.

Every session start printed:

```
⚠ Auxiliary title generation failed: HTTP 302 — 302 Found
```

My endpoint sits behind an identity proxy that requires an authentication header. That header was configured on the main provider block, so the main conversation worked and the warning was easy to dismiss as cosmetic.

Reading the source resolved it: auxiliary calls build their headers from a *different* configuration path than the main agent. They were not sending the header at all, so **every auxiliary task was hitting the auth wall** — a redirect to a login page, unparseable, task abandoned.

```bash
<agent> config set 'model.extra_headers.<auth-header>' '${env:TOKEN_VAR}'
```

The warning disappeared. Two lessons, and the second is the one I keep:

- A partially-working integration is worse than a broken one. Main path fine, side paths dead, one dismissible warning per session.
- **Routing a task to a cheaper model is worthless if the task cannot reach the API.** Had I not tested, I would have "optimized" eleven tasks that were failing 100% of the time, then reported a cost reduction caused entirely by work not happening.

---

## Verifying it worked

The metric to watch is not total tokens, it is the **cache write / cache read ratio**:

```sql
SELECT date(last_seen,'unixepoch','localtime') AS day,
       model,
       SUM(cache_write_tokens)/1000 AS cw_k,
       SUM(cache_read_tokens)/1000  AS cr_k
FROM session_model_usage
GROUP BY day, model
ORDER BY day DESC;
```

Cache write should collapse while cache read grows — the same content, now on the 0.10× lane instead of the 1.25× one. If write stays flat, the TTL is not being honoured somewhere in the chain and the gateway is the next place to look.

One caveat on my own numbers: my gateway does not return costs to the client, so `estimated_cost_usd` was zero across the board and I priced the classes manually at public rates. That produced a figure well above what I was actually charged, which tells me the real rate differs. **The ranking of the cost centres holds regardless of the unit price; the absolute amounts do not.** Rank first, then price.

---

## Summary

1. Query your usage table before installing anything. Split input from output, and cache write from cache read.
2. If output is a small fraction of input, output-side optimizations cannot move your bill.
3. Cache write per call is the number to stare at. High and repeated means your TTL is shorter than your thinking time.
4. Audit which model your agent's *side tasks* use. The default is usually "the expensive one".
5. Verify the plumbing actually works before crediting a config change for a saving.

The pattern generalizes past LLM bills. The compressor was the tool everyone recommends, its benchmarks were real, and it addressed 0.35% of my problem — because the advertised metric ("bash output") and my actual metric (input tokens, weighted by cache class) were not the same thing. Publishable percentages beat unmeasured intuition, but they lose to your own data.

Fifteen minutes of SQL, three config lines. I never installed the compressor.

---

## References

- [Anthropic: prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [OpenAI: prompt caching](https://platform.openai.com/docs/guides/prompt-caching)
- [JetBrains AI blog: benchmarking a token-reduction CLI proxy](https://blog.jetbrains.com/ai/2026/07/rtk-claude-code-token-savings/)
- [rtk — CLI output compressor](https://github.com/rtk-ai/rtk)
