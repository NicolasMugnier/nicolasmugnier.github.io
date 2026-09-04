---
layout: post
title: "Self-hosting Hermes Agent on a VPS"
tags: [hermes, linux, systemd, telegram, security]
author: Nicolas Mugnier
categories: ai
description: "Ubuntu 24.04, systemd user units, a sudo-less runtime user, Telegram-only gateways: a real Hermes Agent deploy, not a step-by-step tutorial."
image: /assets/img/self-host-hermes-agent-on-a-vps.webp
locale: en_US
---

# Self-hosting Hermes Agent on a VPS

Hermes Agent on a laptop is fine until you close the lid. A Telegram bot that should answer at 2am, a specialised profile, a cron that must fire without you: those need a process that outlives the notebook.

I had three shapes: a managed host, [Hermes Cloud](https://hermes-agent.nousresearch.com/docs/){:target="_blank"}, or a VPS I administer. Cloud is a disposable trial, not the house for profiles I care about. I took a small Ubuntu 24.04 box (2 vCPU, 4 GB RAM, 40 GB disk) and ran the LLM over an API. One agent at a time, no local Chromium: 4 GB is enough, and RAM on this host upgrades in place.

The WAN surface I wanted was **Telegram only**. No public dashboard, no `hermes serve` on the internet, no mesh VPN on day one. Telegram bots do not need an inbound port. The gateway polls over outbound HTTPS.

This is a report of a deploy that is running, not a distro-agnostic tutorial. Commands with placeholders live in the appendix.

---

## Two processes, two Unix users

Hermes mixes two runtimes that people collapse into one:

- `hermes gateway`: Telegram, cron, messaging. **Yes.** One systemd user unit per profile.
- `hermes serve`: backend for Desktop and the web dashboard. **Not yet.**

The OS split mirrors that:

- an **admin** user: `sudo`, SSH, `apt`, file copies
- a **runtime** user `hermes`: **no sudo, no SSH key**. You enter with `sudo -u hermes -i`

The agent has a terminal. Give it sudo, a Git deploy key, or an admin port, and a prompt can spend those. Hardening SSH to key-only does not fix that threat model. It only kills password brute-force.

Runtime is **native** plus systemd `--user`, not Docker wrapping Hermes. Install with `--skip-browser`: Chromium on 4 GB is a second machine. Web search and Browser Use in the cloud still work.

---

## Harden the box before installing the agent

Order was deliberate: OS first, agent second.

**SSH.** Ed25519, KDF `-a 100`, passphrase on the key. Password auth cut with a drop-in `/etc/ssh/sshd_config.d/10-hardening.conf`. The `10-` prefix is not cosmetics. On Ubuntu 24.04 **the first matching value wins**. A `99-` file loses to `50-cloud-init.conf`. Confirm with `sshd -T`, not by reading `sshd_config` by eye. Root login off, `AuthenticationMethods publickey`. Keep a **second SSH session open** before you reload `sshd`.

**UFW.** Deny incoming, allow outgoing, OpenSSH only. `ss -lntup`: the only public listener is `:22`. No 80, 443, 9119, 8642.

**fail2ban.** Jail `sshd`, `backend = systemd`. Ubuntu 24.04 logs to the journal, not always to `auth.log`. `backend = auto` can sit there and never ban.

**Swap.** 2 GB at `/swapfile`. At rest, about 3 GB available of 3.7 GB. Headroom for a spike, not for a second browser.

Build packages (`build-essential`, `ripgrep`, `ffmpeg`, …) go in as **admin**. The Hermes installer offers `sudo`. Answer `n`. The runtime user must not grow sudo through the installer.

Linger (`loginctl enable-linger hermes`) keeps user services alive after you disconnect SSH. Linger **does not enable** a `disabled` unit. You need both.

---

## Canary: the `default` profile

The first agent on the box is throwaway. New Telegram bot, not the tokens from the laptop. Allowlist: one account. Never `GATEWAY_ALLOW_ALL_USERS=true`.

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --skip-browser
```

Wizard, in short: Nous Portal (device-code, URL opened on the laptop), a free model until setup is done, Tool Gateway Web + Whisper (Image and Browser unchecked), **local** terminal, messaging = the test bot, home channel = yes, gateway service = yes.

Then xAI OAuth headless:

```bash
hermes auth add xai-oauth --no-browser
hermes config set model.provider xai-oauth
hermes config set model.default grok-4.6
hermes gateway restart
```

The bot answered with Grok. `/whoami` shows `unrestricted`. That is not a hole. The allowlist decides **who** talks. Without `allow_admin_from`, every allowlisted user gets every slash command. Here there is one user.

systemd trap: `sudo -u hermes -i` is not a user session. `systemctl --user` fails until:

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
```

The gateway was **running** and **disabled**. Reboot would kill it. `enable` after the export, not linger alone.

`hermes update` always rebuilds the web UI. There is no `--skip-build` on update.

---

## The work profile is not the canary

The canary proved the machine. The second profile is the real use: a specialised agent and a private working tree.

### A Hermes profile is not the work directory

`hermes profile export` takes SOUL, MEMORY, config, skills. It does **not** take `.env` or `auth.json` (wanted). The work files are not in that archive. Two transfers, SCP, not a shared drive.

Import on the VPS with `hermes profile import`. Do not `hermes import` a full laptop backup: that would overwrite the `default` you just hardened.

Redo auth **with** `-p <profile>`. Without the flag, tokens land in the `default` home.

### Git stays on the laptop

The work tree is a private git repo. Putting it on the VPS with a key owned by `hermes`, even read-only, puts that key inside the `terminal` perimeter. The agent can `cat` the key and clone the private repo from anywhere. Read-only blocks push. It does not block exfil.

What I did instead:

- Git (stable docs) stays on the laptop. Commits are mine, never the agent's.
- Watch outputs and drafts: not in git, they live on the VPS.
- Copy to the VPS: `rsync` **without** `.git`, as admin, into `/tmp`, then `sudo rsync` + `chown hermes`. Zero Git secrets on the box.
- Workspace **per profile**: `/home/hermes/workspaces/<profile>/`, not a bucket the next bot would read.

Admin cannot write into `/home/hermes/` (mode 700). That is why `/tmp` exists. A direct `rsync` into the `hermes` home fails with *Permission denied*, even after `mkdir`.

### Paths

The export still contains laptop paths (`/Users/<you>/…`). Leave them and the agent hunts a ghost machine. Rewrite SOUL / USER / MEMORY toward `/home/hermes/workspaces/<profile>`. Separately:

```bash
hermes -p <profile> config set terminal.cwd /home/hermes/workspaces/<profile>
```

`terminal.cwd` is not the Hermes home. It is the starting directory of the agent's **shell**. Under a systemd gateway, `.` is in practice `/home/hermes`, not the work tree. SOUL does not change cwd by itself.

### Second Telegram bot

One Telegram token is one poller. Reusing the `default` bot on the second profile makes the second gateway fail.

New bot, distinct token, same allowlist. Unit `hermes-gateway-<profile>.service`, enable + linger. Both processes run (about 100-150 MB each).

---

## Left on purpose

- Mesh VPN, `hermes serve`, public HTTPS, OAuth dashboard
- Changing the SSH port
- An SSH key (or deploy key) on the runtime user
- Docker as the Hermes runtime
- One process multiplexing N profiles
- Importing the other laptop profiles
- Cron plus an automatic Telegram report
- The hoster's edge firewall, automatic snapshots, off-box `hermes backup`
- `allow_admin_from` (useless while a single user is allowlisted)

---

## When both bots answer

- **Machine:** Ubuntu 24.04, 4 GB, 2 GB swap, SSH key-only, UFW on 22, fail2ban on systemd
- **Users:** admin (`sudo`) / runtime `hermes` (linger, no sudo, no SSH)
- **`default`:** test Telegram bot, Grok (`xai-oauth` / `grok-4.6`), gateway enabled
- **Work profile:** second bot, same model, cwd + tree under `workspaces/<profile>`, separate gateway
- **WAN:** `:22` only
- **Git:** no key on the VPS

Check the chat model with `hermes -p <profile> config get model.provider` / `model.default`, or `/model` in Telegram. Images: `hermes -p <profile> config get image_gen`. That is not the chat model.

---

## What this actually taught

1. **SSH key-only is not agent security.** The real perimeter is the terminal, the allowlist, and secrets in the runtime home.
2. **One Linux user per runtime, not per bot.** Isolation between profiles is one gateway each and one workspace each, not three UIDs.
3. **Linger is not enabled.** A process can run today and die on reboot.
4. **The first sshd value wins.** Name the drop-in `10-`, confirm with `sshd -T`.
5. **Exporting a profile does not export the work.** Hermes archive on one side, files on the other, paths rewritten, auth redone.
6. **Do not give the agent git "to make it simpler".** Auto-pull means a key in the perimeter. Copying without `.git` is more verbose and more correct.

---

## Appendix: commands (placeholders)

`ADMIN` is the SSH admin user (often `ubuntu`). `AGENT` is the runtime user `hermes`. `VPS_IP` is the VPS IPv4. `PROFILE` is the work profile name. `WORKSPACE` is the work tree on the laptop.

### 1. SSH key (laptop)

```bash
ssh-keygen -t ed25519 -a 100 -C "laptop@hermes-vps" -f ~/.ssh/id_ed25519_vps
# macOS: ssh-add --apple-use-keychain ~/.ssh/id_ed25519_vps
```

Paste the `.pub` in the hoster's SSH-key panel at provision time, and/or into `/home/ADMIN/.ssh/authorized_keys`. **Never** the private key.

```
Host hermes-vps
    HostName VPS_IP
    User ADMIN
    IdentityFile ~/.ssh/id_ed25519_vps
```

### 2. First login, updates, reboot

```bash
ssh hermes-vps
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 3. Cut SSH passwords

Second SSH terminal already open.

```bash
sudo tee /etc/ssh/sshd_config.d/10-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
AuthenticationMethods publickey
EOF

sudo sshd -t
sudo sshd -T | grep -E 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin|authenticationmethods|pubkeyauthentication'
sudo systemctl reload ssh
```

The file is named **`10-`**, not `99-`.

### 4. UFW

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
sudo ss -lntup
```

### 5. fail2ban

```bash
sudo apt install -y fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
# in [sshd]: backend = systemd
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

### 6. Runtime user (no sudo, no SSH)

```bash
sudo adduser --disabled-password --gecos "Hermes Agent" hermes
# do NOT: usermod -aG sudo hermes
sudo loginctl enable-linger hermes
sudo -u hermes -i
```

### 7-8. Packages as admin, Hermes as runtime

```bash
# admin
sudo apt install -y build-essential python3-dev libffi-dev ripgrep ffmpeg

# runtime
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash -s -- --skip-browser
# sudo prompts → n
```

### 9. Default gateway + Grok

```bash
export XDG_RUNTIME_DIR=/run/user/$(id -u)
systemctl --user enable hermes-gateway.service
hermes auth add xai-oauth --no-browser
hermes config set model.provider xai-oauth
hermes config set model.default grok-4.6
hermes gateway restart
```

### 10. 2 GB swap

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 11-13. Work tree, profile import, second bot

```bash
# laptop → /tmp on the VPS (admin cannot write /home/hermes)
rsync -avz --exclude '.git' --exclude '.DS_Store' \
  -e 'ssh -i ~/.ssh/id_ed25519_vps' \
  /path/to/WORKSPACE/ \
  ADMIN@VPS_IP:/tmp/workspace/

ssh hermes-vps
sudo mkdir -p /home/hermes/workspaces/PROFILE
sudo rsync -a /tmp/workspace/ /home/hermes/workspaces/PROFILE/
sudo chown -R hermes:hermes /home/hermes/workspaces
sudo chmod 700 /home/hermes/workspaces /home/hermes/workspaces/PROFILE

# export profile (archive without secrets)
hermes profile export PROFILE -o ./PROFILE.tar.gz
scp -i ~/.ssh/id_ed25519_vps ./PROFILE.tar.gz ADMIN@VPS_IP:/tmp/
ssh hermes-vps
sudo mv /tmp/PROFILE.tar.gz /home/hermes/PROFILE.tar.gz
sudo chown hermes:hermes /home/hermes/PROFILE.tar.gz
# then as runtime:
hermes profile import /home/hermes/PROFILE.tar.gz
hermes -p PROFILE config set terminal.cwd /home/hermes/workspaces/PROFILE
# rewrite laptop paths → Linux in SOUL.md / USER.md / MEMORY.md
hermes -p PROFILE auth add xai-oauth --no-browser
hermes -p PROFILE gateway setup   # new Telegram token
hermes -p PROFILE gateway install --start-now --start-on-login
```
