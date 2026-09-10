---
layout: post
title: "Allowlisting egress for Hermes Telegram gateways"
tags: [hermes, linux, nftables, security, telegram]
author: Nicolas Mugnier
categories: ai
description: "A tool-using agent is an orchestrator, not a vault. We cut local HTTP at the gateway cgroup, left web reading on the Nous tool gateway, and listed the holes that remain."
image: /assets/img/hermes-gateway-egress-allowlist.webp
locale: en_US
---

# Allowlisting egress for Hermes Telegram gateways

The threat I wanted to close is boring on paper: the agent reads credentials on disk and sends them off the box over HTTP, without the secret ever showing up in Telegram.

A Hermes agent with `terminal`, `read_file`, and web tools is an orchestrator. Prompt injection does not need a URL I pasted. A page the agent chose to fetch is enough. Chat redaction hides what comes *back* into the thread. It does not stop a POST that already left.

I did not find a third-party exfil in the transcripts I still have. That is not a proof of absence (compaction, no packet capture). It is a reason to put a boundary on the network, not on the model's manners.

This is the sequence of decisions from a host that is running it. It is not an nftables cookbook. Host addresses, allowlist IPs, and secret files stay out.

---

## 1. HTTPS will not let you allow GET and deny POST

The wish is: the bot may read the web, it may not ship a `.env` to a random host.

On TLS that wish has no packet-level shape. The firewall sees SNI and a destination. It does not see the method or the body.

So the useful split is not GET versus POST. It is *which process talks to whom*.

On this Hermes setup, reading the web is already a different pipe from `curl`:

| Path | Who talks to whom |
|---|---|
| Chat | VPS → model API |
| Search, extracts, cloud browser | VPS → Nous Tool Gateway → the page |
| `curl` / local scripts | VPS → whatever the agent picked |

`web.backend` is Nous. A Telegram message that never calls a web tool does not go through that gateway.

You can cut wild HTTP from the bot process and still let it read pages, because those pages were never fetched by local `curl` in the first place.

---

## 2. Match the gateway cgroup, not the Unix user

Three systemd user units poll Telegram, one per profile. They share one runtime uid with SSH sessions and with `hermes update`.

A uid filter would hit all of that. I did not want to discover, after a drop, that an update could no longer reach the install host.

What I shipped instead is an nftables table on the **OUTPUT** hook, `inet` family, matching only those three units via cgroup v2. Default policy on the rest of the machine stays accept. Established/related, loopback, and DNS stay open for the matched processes. Everything else is an allowlist of resolved addresses, then **drop** (timeout, not an ICMP reject that teaches the agent the difference).

nft wants the cgroup path from `/sys/fs/cgroup`, not the unit name. A rule with only `hermes-gateway.service` fails with *No such file or directory*. That was the first hour.

The allowlist is hostnames, refreshed to addresses on a timer (CDN records move). Categories, not a dump: Telegram, the model API, Nous (inference and the tool/media gateways), GitHub for the profile that needs it, X for the profile that needs it. Empty sets at the wrong moment means Telegram is dead. Load the table, fill the sets immediately, do not walk away in between.

Ubuntu's nftables config flushes the ruleset on boot. If this table is not included after that flush, the bots come back and the bridle does not. The timer is enabled. A real reboot is still on the list; a simulated load of the config plus the refresh populated the sets and left Telegram answering.

Desktop `hermes serve --isolated` is not in those cgroups. SSH as the runtime user is not either. That is the point of the match, and it is also a hole if you expected "every Hermes process".

---

## 3. What it covers, and what it does not

**Covered**

- Direct HTTP from the three Telegram gateways to a host that is not on the list
- Web exploration still going through Nous
- `hermes update` and SSH still working
- Other users on the box untouched

**Not covered**

- Local reads of `.env` / `auth.json`. `write_file` can refuse those paths. The shell on the same uid cannot be talked out of `cat`.
- Exfil through an *allowlisted* pipe: Telegram itself, the model prompt, or `web_extract` pointed at an attacker URL (VPS → Nous → fetch).
- Processes named `hermes` that are not those three units.
- Docker / iron-proxy. They are not installed. Without Docker, `hermes egress` is a no-op.
- systemd `IPAddressAllow=` on a user unit. On this host it failed with *Operation not permitted* without network delegation. nft as root plus a cgroup match is the same idea without giving the user instance a network cgroup.

The model that "refuses" is not a boundary. nft is, for the bot's network.

---

## 4. Checks I actually ran

From the gateway: a `curl` to a public site that is not on the list times out. The same `curl` to Telegram gets an HTTP response. A web question in chat still returns an extract. An SSH session as the runtime user can still update Hermes.

None of that is a red team. It is the minimum that says the table is attached to the right processes.

---

## 5. What I left out

- Copy-pasted nft, the hostname list, or resolved addresses
- Binding extra listeners or opening inbound ports (there are still none for Hermes)
- Switching the match to uid "to make the rule shorter"
- Pretending chat policy or `approvals.deny` replaces egress
- Putting Desktop SSH in the same net yet

Rollback is one table delete and the refresh timer off. The gateways become "the whole internet" again. Keep that next to the install notes, not in a blog post as a ritual.

---

## What I would keep doing

1. **Treat the agent as an orchestrator.** Redaction, SOUL, and "please don't" are not egress.
2. **Split "read the web" from "the shell has HTTP".** If those are the same pipe, an allowlist either blinds you or does nothing.
3. **Filter the gateway cgroup, not the runtime uid.** Updates and SSH should not share the bot's network jail.
4. **Refresh DNS on a timer.** A static IP set for Telegram or a model CDN will rot.
5. **Never load empty sets.** Table first, fill immediately, or the bots look down and you will disable the whole thing.
6. **Write the holes down in the same note as the win.** `.env` is still readable. Allowlisted pipes still leave. Docker plus a proxy that never mounts the Hermes home is the next real step, with nft staying as host defense.

This is a bridle on three processes. It is not "we secured the AI".
