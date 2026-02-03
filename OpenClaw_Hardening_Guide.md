# OpenClaw Ubuntu Server Hardening Guide

This guide outlines the recommended steps to securely set up and harden OpenClaw on an Ubuntu server. It includes system patching, SSH hardening, firewall configuration, unattended upgrades, OpenClaw security audits, and plugin/security steps. Follow the steps **in order** for maximum security.

> **Shared host?** If other users have accounts on this server, the official OpenClaw security docs recommend creating a dedicated OS user for the Gateway (e.g., `sudo adduser --disabled-password --gecos "OpenClaw Service" openclaw`). This limits blast radius if the agent is compromised. For single-user servers where you are the only account, running under your own user is fine — just ensure `~/.openclaw` permissions are locked down (Step 7 handles this).

## 1️⃣ System Updates & Unattended Upgrades

Always patch the OS first — before configuring anything else.

1. Ensure the system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

2. Enable unattended upgrades:

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## 2️⃣ SSH Key & Server Setup

1. Generate a secure SSH key (if you don't have one):

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

2. Copy the public key to the server (`~/.ssh/authorized_keys`).

3. **Before changing anything**, open a second SSH session using the key and confirm it works. Keep this session open as a lifeline.

4. Harden SSH using a single sed command:

```bash
sudo sed -i \
  -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
  -e 's/^#\?PermitRootLogin.*/PermitRootLogin no/' \
  -e 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' \
  -e 's/^#\?PermitEmptyPasswords.*/PermitEmptyPasswords no/' \
  -e 's/^#\?ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' \
  -e 's/^#\?UsePAM.*/UsePAM yes/' \
  -e 's/^#\?MaxAuthTries.*/MaxAuthTries 3/' \
  -e 's/^#\?LoginGraceTime.*/LoginGraceTime 30/' \
  -e 's/^#\?ClientAliveInterval.*/ClientAliveInterval 300/' \
  -e 's/^#\?ClientAliveCountMax.*/ClientAliveCountMax 2/' \
  /etc/ssh/sshd_config
```

> **Additional hardening (lines added above):**
> - `MaxAuthTries 3` — Limits authentication attempts per connection to prevent brute-force.
> - `LoginGraceTime 30` — Drops unauthenticated connections after 30 seconds.
> - `ClientAliveInterval 300` — Sends a keepalive probe every 5 minutes.
> - `ClientAliveCountMax 2` — Disconnects after 2 missed keepalives (10 min idle timeout).

5. Restart SSH:

```bash
sudo systemctl restart sshd
```

6. **Verify** you can still connect via your lifeline session before closing anything.

## 3️⃣ Install & Configure Fail2ban

Fail2ban adds brute-force protection even behind Tailscale — defense-in-depth at minimal cost.

1. Install fail2ban:

```bash
sudo apt install fail2ban -y
```

2. Create a local jail config (never edit the main config directly):

```bash
sudo tee /etc/fail2ban/jail.local > /dev/null <<EOF
[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
bantime  = 3600
findtime = 600
EOF
```

3. Enable and start fail2ban:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

4. Verify it's running:

```bash
sudo fail2ban-client status sshd
```

## 4️⃣ Network & Firewall (Tailscale / UFW)

1. Install Tailscale (no exposed ports):

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

2. **Verify** you can SSH to the server over your Tailscale IP before enabling UFW. If this doesn't work, do not proceed — you will be locked out.

```bash
# From another machine on your tailnet:
ssh your-user@<tailscale-ip>
```

3. Install and configure UFW to restrict SSH to Tailscale only:

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port 22
sudo ufw enable
```

## 5️⃣ Install Node.js & OpenClaw

1. Install NVM and Node.js:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 24
```

2. Install OpenClaw:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

## 6️⃣ Run Onboarding

Onboarding creates the `~/.openclaw` config structure, credentials directory, and gateway token. It must run before we can harden file permissions or run security audits.

### Generate your setup-token (Claude Max / Pro)

In a **separate terminal** (local machine or another session), run:

```bash
claude setup-token
```

Copy the token it prints. This routes OpenClaw through your existing Claude Max subscription instead of per-token API billing — no Anthropic API key required.

> **Note:** `claude setup-token` only requests the `user:inference` scope, so usage tracking in `openclaw status` won't work. If you need usage visibility, run `claude login` (full OAuth) instead — though that requires a browser, which is trickier on a headless server.

### Run the wizard

```bash
openclaw onboard
```

- Recommended options:
  - **Mode:** Manual
  - **Provider:** Anthropic → **setup-token** (paste the token from above)
  - **Model:** `claude-sonnet-4-5` (Sonnet 4.5 — best balance of capability and quota efficiency on Max 5x; see model notes below)
  - **Channel:** Select **Matrix** when prompted — the wizard will offer to download and install the Matrix plugin for you automatically. Accept this to set up E2E encrypted messaging.
  - **Gateway bind:** Loopback
  - **Gateway auth:** Token (the wizard generates one automatically)
  - Enable all hooks (`boot`, `command-logger`, `session-memory`)

> **Model recommendation:** Sonnet 4.5 is the recommended primary model. It delivers ~90–95% of Opus 4.5's performance while consuming significantly less of your Max subscription quota. For a failover chain, set Sonnet 4.5 as primary with Haiku 4.5 as fallback — this keeps heartbeats and background tasks from draining your quota. You can override to Opus 4.5 per-conversation when you need maximum reasoning power.

## 7️⃣ Harden OpenClaw Permissions & Disable Bonjour

Now that onboarding has created the config and credential files, lock them down.

1. Lock down OpenClaw config and credential files:

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/*.json
chmod 600 ~/.openclaw/credentials/*
```

2. Disable Bonjour/mDNS discovery (v2026.1.29+ defaults to minimal disclosure, but be explicit):

```bash
echo 'export OPENCLAW_DISABLE_BONJOUR=1' >> ~/.bashrc
source ~/.bashrc
```

3. Verify mDNS is off:

```bash
openclaw config get gateway.mdns
```

## 8️⃣ Run OpenClaw Security Audit

1. Deep audit:

```bash
openclaw security audit --deep
```

2. Apply fixes if needed:

```bash
openclaw security audit --fix
```

## 9️⃣ Optional: Install Plugins / Security Skills

### Matrix Plugin (E2E Encryption)

If you selected **Matrix** as your channel during onboarding (Step 6), the wizard already downloaded the plugin. You may still need to apply the peer dependency fix and run `npm install`:

```bash
cd ~/.openclaw/extensions/matrix
sed -i 's/"workspace:\*"/"*"/g' package.json
npm install
```

If you **skipped Matrix during onboarding**, install it manually:

```bash
openclaw plugins install @openclaw/matrix
cd ~/.openclaw/extensions/matrix
sed -i 's/"workspace:\*"/"*"/g' package.json
npm install
```

> **Why the `sed` command?** The Matrix plugin ships with a peer dependency scope of `"workspace:*"`, which restricts resolution to the OpenClaw workspace's monorepo context. When installed as a standalone extension (outside the monorepo), npm cannot resolve this. Replacing it with `"*"` tells npm to accept any compatible version from the registry instead. Verify this is still needed for your plugin version — future releases may ship with the correct scope by default.

- Configure Matrix accounts in `~/.openclaw/openclaw.json`

### Security Skills (ACIP)

Message your bot: `"Install this: https://github.com/Dicklesworthstone/acip/tree/main"`

- Test defenses (bot should refuse sensitive instructions)

## ✅ Notes on Ordering

1. **Step 1** patches the OS before any configuration — always patch first.
2. **Steps 2–4** secure the server access layer (SSH, brute-force protection, firewall).
3. **Step 5** installs Node.js and the OpenClaw CLI.
4. **Step 6** runs onboarding to create the config, credentials, gateway token, and optionally installs the Matrix plugin.
5. **Steps 7–8** harden permissions and audit the config — these require the files created by onboarding.
6. **Step 9** covers Matrix plugin fixup (if selected during onboarding) or manual installation, plus extra bot security.

*This guide ensures a security-first setup for running OpenClaw on Ubuntu, minimizing exposure and attack surface.*