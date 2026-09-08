---
layout: post
tags: [llm, litellm, gateway, ai, configuration, cli, security]
author: Nicolas Mugnier
categories: ai
title: "Pointing a CLI Agent at a LiteLLM Gateway"
description: "How to configure an AI agent to talk to a self-hosted LiteLLM proxy instead of a provider directly — including the identity-proxy trap that breaks half the agent silently."
image: /assets/img/litellm-custom-provider-cli-agent.webp
locale: en_US
---

Most AI coding agents assume you talk to a provider directly: an API key for Anthropic, another for OpenAI, one per vendor. In a team setting that is rarely how it works. You get a single internal endpoint — usually [LiteLLM](https://github.com/BerriAI/litellm) — that fronts several providers, centralizes keys, enforces quotas, and gives whoever pays the bill a dashboard.

The good news is that LiteLLM speaks the OpenAI API, so any agent that accepts a custom base URL can use it. The configuration is about ten lines.

The part worth writing down is what happens when that gateway sits behind an identity proxy. My agent's main conversation worked perfectly while **every background task silently failed** for days, behind a single warning line I had learned to ignore.

---

## The shape of the configuration

A custom provider needs four things: a name, the base URL, where to find the API key, and which models to expose.

```yaml
custom_providers:
  - name: my-litellm-gateway
    base_url: https://llm-gateway.internal.example.com/v1
    key_env: MY_GATEWAY_API_KEY
    model: claude-opus-5              # default model
    models:
      claude-haiku-4-5: {}
      claude-opus-5: {}
      claude-sonnet-5: {}
      gpt-5.6-luna: {}
      smart-router: {}
```

Three details matter more than they look.

**`base_url` ends with `/v1`.** LiteLLM exposes the OpenAI-compatible surface there. Point at the bare host and every call 404s.

**`key_env` names an environment variable — it is not the key.** The value lives in your secrets file, never in the config. This matters because config files get committed, shared, and pasted into issues; a `key_env` reference is safe to show, a key is not.

**`models` is the list your agent will offer you.** LiteLLM aliases can be anything the gateway admin chose — `smart-router` here routes server-side by cost or latency, `gpt-5.6-luna` is an internal alias, not an upstream model name. There is no reliable way to derive them; you get them from whoever runs the gateway.

The recognized fields for a custom provider entry are `name`, `base_url`, `api_key`, `api_mode`, `model`, `models`, `context_length`, `rate_limit_delay`, `extra_body`, `ssl_ca_cert`, `ssl_verify`, and `key_env`. Two are worth knowing about:

- **`api_mode`** forces the transport (`chat_completions`, `anthropic_messages`, `codex_responses`). LiteLLM normalizes to the OpenAI shape, so the default is usually right — but if your gateway proxies Anthropic natively and you want cache control or extended thinking to pass through untouched, this is the switch.
- **`context_length`** overrides the assumed context window. Agents fall back to a generic default (often 200 K) for models they do not recognize, and behind a gateway alias they recognize nothing. Set it if your model's real window is smaller, or context-compression will trigger too late.

---

## Why you cannot just query the model list

The obvious way to discover models is `GET /v1/models`. Try it through an authenticated gateway and you may get this:

```bash
$ curl -H "Authorization: Bearer $MY_GATEWAY_API_KEY" \
       https://llm-gateway.internal.example.com/v1/models
HTTP 302
```

A redirect to an SSO login page. The gateway is behind an identity proxy — Cloudflare Access, Google IAP, an OAuth2 reverse proxy — that intercepts requests *before* LiteLLM sees them. Your LLM API key is irrelevant at that layer; the proxy wants its own credential.

Service tokens exist for exactly this case, and they are worth trying:

```bash
curl -H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID" \
     -H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET" \
     -H "Authorization: Bearer $MY_GATEWAY_API_KEY" \
     https://llm-gateway.internal.example.com/v1/models
```

If the policy does not include that token, this returns 302 as well. At that point stop fighting the wall — **the model list is documentation, not an API call.** Ask the person who runs the gateway, or read it from your own agent's config if a previous setup wizard already discovered it.

I burned several attempts on this, escalating from plain curl to service tokens to following redirects, before accepting that the endpoint was simply not reachable from a shell. Two of those attempts tripped my own security tooling for shovelling credentials around in one-liners, which was a fair warning that I was brute-forcing rather than thinking.

---

## The trap: headers that only reach half the agent

Here is the interesting failure.

An identity proxy needs an auth header on every request. So you add it to the provider:

```yaml
custom_providers:
  - name: my-litellm-gateway
    base_url: https://llm-gateway.internal.example.com/v1
    key_env: MY_GATEWAY_API_KEY
    extra_headers:
      cf-access-token: ${env:CF_ACCESS_TOKEN}
```

Chat works. You move on. And every session start prints one line you learn to ignore:

```
⚠ Auxiliary title generation failed: HTTP 302 — 302 Found
```

That line is not cosmetic. A modern agent makes far more API calls than the ones you see: context compression when history grows, session-title generation, approval classification, memory-query rewriting, skill curation, background self-improvement passes. These are **auxiliary calls**, and in the agent I use they build their HTTP headers from a different configuration path than the main conversation.

Reading the source made it explicit — the auxiliary client merges headers from the *global model* config, not from the custom provider block:

```python
def _apply_user_default_headers(headers: dict | None) -> dict | None:
    """Merge user ``model.default_headers`` onto resolved headers (user wins;
    ``model.extra_headers`` alias wins over both). Mirrors
    ``AIAgent._apply_user_default_headers`` so a custom endpoint behind a
    WAF rejecting ``User-Agent`` / ``X-Stainless-*`` works for aux calls."""
```

`model.default_headers` and `model.extra_headers`. Not `custom_providers[].extra_headers`.

So every auxiliary task was sending no auth header, hitting the identity proxy, and receiving an HTML login page where it expected JSON. Every one of them failed, every session, silently except for that one warning.

The fix is to declare the header at the global level as well:

```yaml
model:
  extra_headers:
    cf-access-token: ${env:CF_ACCESS_TOKEN}
```

Or in one command:

```bash
<agent> config set 'model.extra_headers.cf-access-token' '${env:CF_ACCESS_TOKEN}'
```

The `${env:VAR}` syntax keeps the token in your secrets file. The config stays shareable.

Note the docstring's other hint: the same mechanism exists because some WAFs reject the SDK's default `User-Agent` or `X-Stainless-*` headers. If a gateway works in curl but not from your agent, comparing the two header sets is the first thing to check.

---

## Two failure modes to keep apart

This episode had two distinct problems that produce nearly identical symptoms, and separating them saved me from a wrong conclusion.

**A partially-working integration.** The main path works, side paths are dead. There is no loud failure because the thing you look at is fine. One dismissible warning per session is the entire signal. This is worse than a total outage: a total outage gets fixed in ten minutes.

**A configuration key that is read by nothing.** While optimizing the same setup I set a plausible-looking top-level `cache_ttl`. The agent accepted it and warned that the key was not recognized — saved to the file, read by no code. The real key was nested one level deeper. Had the warning been slightly less clear I would have "changed" a setting and measured no effect, with nothing to explain why.

The shared lesson: **verify the plumbing before you credit the config.** I was routing auxiliary tasks to a cheaper model at the same time as fixing the header. Had I not tested, I would have reported a cost reduction that came entirely from those tasks not running at all.

---

## Summary

1. `base_url` ends in `/v1`; secrets go in `key_env`, never inline.
2. Get the model list from your gateway admin — `/v1/models` is often unreachable behind an identity proxy, and service tokens only work if the policy allows them.
3. Set `context_length` when the gateway hides the real model, or compression fires too late.
4. If the endpoint is behind an auth wall, declare the header **twice**: on the custom provider *and* under `model.extra_headers`, or every background task fails silently.
5. Treat a recurring "auxiliary … failed" warning as an outage report, not decoration.

None of this is specific to one agent. Any tool with a main request path and a set of background helpers can have them diverge in configuration — and the background ones fail quietly, because nobody is watching a session title get generated.

---

## References

- [LiteLLM documentation](https://docs.litellm.ai/)
- [LiteLLM proxy: virtual keys and model aliases](https://docs.litellm.ai/docs/proxy/virtual_keys)
- [Cloudflare Access: service tokens](https://developers.cloudflare.com/cloudflare-one/identity/service-tokens/)
- [OpenAI API: chat completions reference](https://platform.openai.com/docs/api-reference/chat)
