---
layout: post
tags: [hermes, llm, ai, cost-optimization, caching, observability]
author: Nicolas Mugnier
categories: ai
title: "Measure Before You Optimize: Cutting a Hermes Agent Token Bill"
description: "Hermes burned through 8.5M input tokens in a single day. The popular fix would have saved 0.35%. Here is what the data actually said, and the three changes that mattered."
image: /assets/img/measure-before-you-optimize-llm-token-bill.webp
locale: en_US
---

I spent about €20 in a single day running [Hermes Agent](https://hermes-agent.nousresearch.com/) locally. That is not catastrophic, but extrapolated over a month of daily use it stops being pocket change, and it was worth understanding before it became a habit.

My first instinct was the one everybody has: install an output compressor. There is a whole category of tools for this now: CLI proxies that intercept `git status`, `ls`, or `pytest`, strip the noise, and hand the model a compact summary instead of the raw output. The pitch is compelling and the numbers advertised are large: 60-90% reduction.

I decided to measure first. The measurement said the compressor would have saved me **0.35%**.

Here is how to find out where the money actually goes, and the three changes that turned out to matter.

---

## The four token classes

Before measuring anything, it helps to know that not all tokens on an LLM bill cost the same. For a cache-enabled API there are four distinct classes, and their prices differ by more than an order of magnitude. Anthropic-style prompt cache, relative to fresh input. Writes are the same multipliers on every Claude. **Reads are not.**

| Class | What it is | Relative price |
|---|---|---|
| Input (fresh) | New content the model has never seen | 1× |
| Cache **write** (5m TTL) | Content stored into the prompt cache | 1.25× |
| Cache **write** (1h TTL) | Same write, longer retention | 2× |
| Cache **read** (Haiku, Sonnet, Opus 5) | Hit on those families | 0.10× |
| Cache **read** (Opus 5.5) | Hit on Opus 5.5 | 0.05× |
| Cache **read** (Fable 5.1) | Hit on Fable 5.1 / Mythos 5.1 | 0.025× |
| Output | What the model generates | ~5× |

I weight the rest of this post at **Opus 5** (0.10× reads). That is the model I actually run. Same multiplier as Haiku and Sonnet. The trap is copying 0.10× onto every Claude: Opus 5.5 is 0.05×, Fable 5.1 is 0.025×.

The interesting pair is cache write versus cache read. At the 5-minute tier, on Opus 5, they are **12.5× apart for identical content** (1.25 / 0.10). On Opus 5.5 that gap is 25×. On Fable 5.1 it is 50×. Whether your conversation history is written or read is therefore a bigger lever than how large it is.

That is the part output compressors cannot help with, because it is not about volume at all.

---

## Measuring instead of guessing

Hermes keeps a SQLite state database (`state.db`) with a `session_model_usage` table: per-model, per-task token counts. One query:

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

**Cache write is 1 296 k over 123 calls: about 10.5 k tokens per call.** That is the smoking gun. A healthy cache is written once and read many times. Mine was being rewritten on essentially every single call, at 1.25× while an Opus 5 read would have cost 0.10×.

Weighting each class at Opus 5 public rates, 5-minute TTL:

```
cache write   ~45%
fresh input   ~26%
cache read    ~17%
output        ~12%
```

Nearly half the bill was one pathology: paying premium rates to re-upload context the provider already had. Move the same tokens onto Opus 5.5 (0.05× reads) and the write share goes up, not down: every wasted write is then twenty-five fresh tokens, not twelve.

---

## What the compressor could have reached

Now the counterfactual. Same database, different question: how many bytes do tools actually contribute?

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

Total tool output: 396 kchars, roughly 99 k tokens of unique content. Shell commands, the only slice a CLI proxy can touch, are 172 kchars of that, about 43 k tokens.

Compress 70% of it, the optimistic end of the published range, and you save ~30 k tokens against a daily input of 8 531 k.

**0.35%.**

The arithmetic is not a criticism of these tools. Their compression is real and often elegant. The problem is the denominator: shell output is one contributor to input tokens, input tokens are one part of the bill, and the reduction dilutes at every step. A vendor claim of "90% of bash output" is perfectly honest and still nearly irrelevant to your invoice. JetBrains ran an independent benchmark on the same class of tool and landed on a ceiling around 3%, an order of magnitude above my case, still not where the money is.

File reads, search results, and web fetches (224 kchars here, more than the shell) bypass a CLI proxy entirely.

---

## The three changes that mattered

### 1. Cache TTL: 5 minutes to 1 hour

Hermes default prompt-cache TTL was five minutes. My working rhythm is: ask a question, read the answer, think, read some code, come back. Regularly more than five minutes between turns.

Every time that gap exceeded the TTL, the cache entry expired and the entire conversation prefix was rewritten at 1.25× instead of being read at 0.10×.

```bash
# 5m -> 1h  (Anthropic only accepts these two tiers, plus auto)
hermes config set prompt_caching.cache_ttl 1h
```

One line, and it addresses the ~45% slice.

Two caveats, both from the [Hermes docs](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching), not from folklore:

- The 1h tier **writes at 2×**, not 1.25×. It only pays if you actually miss the five-minute window. If you hammer the agent, you just made remaining writes dearer.
- `auto` now picks `1h` for interactive sessions (CLI, TUI, Telegram) and `5m` for machine-paced ones (subagents, cron, one-shots). If I were setting this today I would try `auto` first.

**Check the key name.** I first set a plausible-looking top-level `cache_ttl` and got a warning that it was not a recognized key: saved to the config, read by nothing. The real key is nested under `prompt_caching`. A silently-ignored setting looks exactly like a setting that did not help.

### 2. Route auxiliary tasks to a small model

Hermes does not make one API call per turn. Alongside the main conversation it runs side tasks, each with its own call: context compression, session-title generation, approval classification, curator, background review.

By default (`auxiliary.*.provider: auto`) every one of those **inherits the main model**. A session title, three words long, was being generated by the most expensive model available. A limousine to fetch bread.

The interactive path is `hermes model` then "Configure auxiliary models". Same thing from the CLI:

```bash
hermes config set auxiliary.title_generation.model "<small-model>"
hermes config set auxiliary.title_generation.provider "<provider>"
# same pattern: compression, approval, curator, background_review, vision
```

The background self-improvement task alone was ~8% of the day's tokens. Its own documentation noted that running it on a non-default model replays a compact digest rather than the full conversation: 3-5× cheaper before the per-token price difference even applies.

### 3. Fix the auxiliary calls that were silently failing

This one was a genuine bug, and I only found it because I tested the previous change instead of trusting it.

Every session start printed:

```
⚠ Auxiliary title generation failed: HTTP 302 - 302 Found
```

The endpoint sits behind an identity proxy that requires an authentication header. That header was configured on the main provider block, so the main conversation worked and the warning was easy to dismiss as cosmetic.

Reading the source resolved it: auxiliary calls build their headers from a *different* configuration path than the main agent. They were not sending the header at all, so **every auxiliary task was hitting the auth wall**: a redirect to a login page, unparseable, task abandoned.

```bash
hermes config set 'model.extra_headers.<auth-header>' '${env:TOKEN_VAR}'
```

The warning disappeared. Two lessons, and the second is the one I keep:

- A partially-working integration is worse than a broken one. Main path fine, side paths dead, one dismissible warning per session.
- **Routing a task to a cheaper model is worthless if the task cannot reach the API.** Had I not tested, I would have "optimized" auxiliary tasks that were failing 100% of the time, then reported a cost reduction caused entirely by work not happening.

The plumbing (custom provider vs `model.extra_headers`) is a separate post.

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

Cache write should collapse while cache read grows: the same content, now on the 0.10× lane instead of the 1.25× one. If write stays flat, the TTL is not being honoured somewhere in the chain and the gateway is the next place to look.

Do not mix families when you convert tokens to money. Opus 5, Haiku and Sonnet share 0.10×. Opus 5.5 is 0.05×. Fable 5.1 is 0.025×. They are not the same line item.

One caveat on my own numbers: the gateway does not return costs to the client, so `estimated_cost_usd` was zero across the board and I priced the classes manually at public rates. That produced a figure well above what I was actually charged, which tells me the real rate differs. **The ranking of the cost centres holds regardless of the unit price; the absolute amounts do not.** Rank first, then price.

---

## Summary

1. Query `session_model_usage` before installing anything. Split input from output, and cache write from cache read.
2. If output is a small fraction of input, output-side optimizations cannot move your bill.
3. Cache write per call is the number to stare at. High and repeated means your TTL is shorter than your thinking time.
4. Audit which model Hermes *side tasks* use. Default is the expensive one. `hermes model` has a picker for that.
5. Verify the plumbing actually works before crediting a config change for a saving.

The pattern generalizes past LLM bills. The compressor was the tool everyone recommends, its benchmarks were real, and it addressed 0.35% of my problem, because the advertised metric ("bash output") and my actual metric (input tokens, weighted by cache class) were not the same thing. Publishable percentages beat unmeasured intuition, but they lose to your own data.

Fifteen minutes of SQL, three config lines. I never installed the compressor.

---

## References

- [Hermes Agent](https://hermes-agent.nousresearch.com/)
- [Hermes: context compression and prompt caching](https://hermes-agent.nousresearch.com/docs/developer-guide/context-compression-and-caching)
- [Hermes: auxiliary models](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#auxiliary-models)
- [Anthropic: prompt caching (per-model cache-hit rates)](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [OpenAI: prompt caching](https://platform.openai.com/docs/guides/prompt-caching)
- [JetBrains AI blog: benchmarking a token-reduction CLI proxy](https://blog.jetbrains.com/ai/2026/07/rtk-claude-code-token-savings/)
- [rtk, CLI output compressor](https://github.com/rtk-ai/rtk)
