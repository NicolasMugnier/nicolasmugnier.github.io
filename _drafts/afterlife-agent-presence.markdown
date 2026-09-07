---
layout: post
title: "A read-only Afterlife for Hermes agent presence"
tags: [hermes, presence, systemd, cyberpunk]
author: Nicolas Mugnier
categories: ai
description: "Three Hermes profiles as characters in a Cyberpunk bar: idle, busy, or offline, driven by a loopback JSON collector. The page never commands an agent."
image: /assets/img/afterlife-agent-presence.webp
locale: en_US
---

# A read-only Afterlife for Hermes agent presence

Several Hermes agents live in separate chats and gateways. You can ask each one what it is doing. You cannot see, in one glance, who is there, who is in a turn, and who is down.

I wanted a room, not a dashboard. One page, read only, three characters in a Cyberpunk 2077 Afterlife bar. The character state is the process state. A click opens a fiche. It does not send `/stop`, it does not spawn a turn, it does not write to the agent.

This is the sequence from a first version that is running on a VPS loopback, viewed from a laptop through an SSH tunnel.

---

## 1. Three levels, not a green LED

The product contract is three levels. Collapsing them into one dot was the thing to refuse.

1. **In the bar** (`gateway_up`): the profile gateway is up (systemd user unit active **and** `gateway_state.json` says `running`).
2. **On a job** (`turn_busy`): a turn is in progress.
3. **Trace** (`last_event_at` / `last_event_kind`): last message or last cron, even when idle.

Visual mapping:

- Down: empty stool. `gateway_up: false`.
- Idle: at the counter. Up, not busy.
- Busy: on a job. Up and `turn_busy`.
- Trace: a quiet label ("12 min ago"), not a siren.

The game does not invent this. It reads a JSON document built outside the page.

Three characters, one per profile. No extra NPC in v1.

| Character | `profile_id` | Gateway unit |
|---|---|---|
| Panam | `default` | `hermes-gateway.service` |
| Alt | `alt` | `hermes-gateway-alt.service` |
| Rogue | `rogue` | `hermes-gateway-rogue.service` |

---

## 2. A collector on loopback, not `hermes serve`

I already run Hermes on a VPS with Telegram-only WAN. Reusing `hermes serve` on `:9119` would have been the lazy option. That process is a full backend with auth. Presence does not need it.

What I shipped instead is a small Python HTTP server, systemd user unit `afterlife-presence.service`, bind `127.0.0.1:8787`, `GET /presence.json`. Nothing listens on the public IP. CORS is `*` because the UI is another localhost port. `Cache-Control: no-store`. The JSON is rebuilt on every GET.

Schema:

```
profile_id        default | alt | rogue
gateway_up        bool
turn_busy         bool
last_event_at     ISO-8601 UTC | null
last_event_kind   message | cron | null
```

Wrapped as `{ updated_at, agents: [...] }`. Timestamps are UTC. The UI converts (`Date` / `Europe/Paris`). Displaying the UTC hour as local is a two-hour lie in summer.

Example payload:

```json
{
  "updated_at": "2026-09-07T12:29:05.965344+00:00",
  "agents": [
    {
      "profile_id": "default",
      "gateway_up": true,
      "turn_busy": true,
      "last_event_at": "2026-09-07T12:28:42.994266+00:00",
      "last_event_kind": "message"
    }
  ]
}
```

---

## 3. Where each field comes from

Homes: default is `~/.hermes`, named profiles are `~/.hermes/profiles/<id>/`.

**`gateway_up`.** `systemctl --user is-active` on the unit **and** `gateway_state == running` in `gateway_state.json`. One without the other is not "in the bar".

**`turn_busy`.** Any of:

- `active_agents > 0` in `gateway_state.json`
- an unexpired row in `session_turn_leases`
- `sessions.last_activity_at` within 25 seconds, Telegram **or** Desktop

Telegram-only `active_agents` misses Desktop group rooms. That is why the 25-second window exists.

**`last_event_*`.** `state.db`, table `sessions`, max `last_activity_at`, opened `file:...?mode=ro` because of WAL. `source == cron` becomes kind `cron`, anything else `message`.

`systemctl --user` needs `XDG_RUNTIME_DIR=/run/user/<uid>` and the user bus. Linger is already on for the gateways. Without the runtime dir, every agent looks down.

Busy is short. `active_agents` is 1 during a Telegram turn, then 0. A slow poll never shows EN MISSION. The UI polls every 2–3 seconds and sticky-holds busy for 30–90 seconds after the last busy/`last_event`.

---

## 4. The laptop only sees a tunnel

v1 has no Tailscale, no phone, no public hostname.

```bash
ssh -N -L 8787:127.0.0.1:8787 <ssh-alias>
```

`-N` is tunnel only. The `Host` alias in `~/.ssh/config` is enough. The forward targets **VPS loopback**, so `ubuntu@vps` still works.

On the laptop:

```bash
curl -sS http://127.0.0.1:8787/presence.json
```

- Connection refused: no tunnel, or the wrong alias.
- JSON with a frozen `updated_at`: something else is bound to local 8787.
- `default.turn_busy: true` while you message Panam: the pipe is live.

The page fetches that URL. Fetch fails: fixtures, mock dropdown. Fetch works: live. Do not commit a copied `presence.json` into the repo. Presence says who is busy.

If the tunnel is up and `turn_busy` is true but the character does not move, the UI is still on mock. Check that laptop `curl` matches VPS `updated_at`, then switch the dropdown to live.

---

## 5. What I left out

- Commanding an agent from the bar (chat, `/stop`, spawn)
- Pathfinding, inventory, combat, XP trees
- Auth, player accounts, a fourth character
- Binding `:8787` on the public IP
- Reusing `hermes serve :9119`
- Websockets (poll is enough)
- Tailscale / iPhone (same JSON, different fetch URL, later)

---

## What I would keep doing

1. **Read-only is a product rule, not a missing feature.** The page must not grow a "talk to them" button that goes through the game.
2. **Down, idle, and busy are three states.** A single LED hides the interesting case (gateway up, nobody in a turn).
3. **Keep presence off the WAN.** Loopback plus SSH `-L`. A public JSON leaks who is in a turn.
4. **Rebuild on GET.** A file on disk goes stale the moment you look at it.
5. **Sticky busy on the client.** The collector is honest and brief. The badge has to linger or you never see it.
6. **UTC in the payload, local in the UI.** Do not print the ISO hour as if it were Paris.

v1 is a room you can look at. That was the point.
