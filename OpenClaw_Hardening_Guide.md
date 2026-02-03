# OpenClaw Ubuntu Server Hardening Guide

Security hardening steps for an Ubuntu server running OpenClaw. This guide assumes OpenClaw is already installed and onboarded. Run these steps **in order**.

> **Shared host?** If other users have accounts on this server, the official OpenClaw security docs recommend creating a dedicated OS user for the Gateway (e.g., `sudo adduser --disabled-password --gecos "OpenClaw Service" openclaw`). For single-user servers, running under your own user is fine — just ensure `~/.openclaw` permissions are locked down (Step 5).

## 1️⃣ System Updates & Unattended Upgrades

Patch the OS first — before configuring anything else.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## 2️⃣ SSH Hardening

1. Ensure you have an ed25519 key deployed to `~/.ssh/authorized_keys` on the server.

2. **Before changing anything**, open a second SSH session using the key and confirm it works. Keep this session open as a lifeline.

3. Harden SSH:

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

> **What these do:**
> - `MaxAuthTries 3` — Limits authentication attempts per connection to prevent brute-force.
> - `LoginGraceTime 30` — Drops unauthenticated connections after 30 seconds.
> - `ClientAliveInterval 300` — Sends a keepalive probe every 5 minutes.
> - `ClientAliveCountMax 2` — Disconnects after 2 missed keepalives (10 min idle timeout).

4. Restart SSH:

```bash
sudo systemctl restart sshd
```

5. **Verify** you can still connect via your lifeline session before closing anything.

## 3️⃣ Fail2ban

Brute-force protection — defense-in-depth even behind Tailscale.

```bash
sudo apt install fail2ban -y

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

sudo systemctl enable fail2ban
sudo systemctl start fail2ban
sudo fail2ban-client status sshd
```

## 4️⃣ Tailscale & UFW

1. Install Tailscale:

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

3. Restrict SSH to Tailscale only:

```bash
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port 22
sudo ufw enable
```

## 5️⃣ OpenClaw Permissions

> **Prerequisite:** Run `openclaw onboard` first — this step requires the config and credential files it creates.

Lock down config, credentials, and session files.

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/*.json
# Auth profiles contain your setup-token / API keys
find ~/.openclaw/agents -name "auth-profiles.json" -exec chmod 600 {} \;
# Session transcripts can contain private messages and tool output
find ~/.openclaw/agents -name "sessions.json" -exec chmod 600 {} \;
```

> **Where are my credentials?** With setup-token auth, credentials live in `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`. The `credentials/` directory is only created for WhatsApp sessions or legacy OAuth imports.

## 6️⃣ Disable Bonjour/mDNS

Prevents the gateway from advertising itself on the local network (v2026.1.29+ defaults to minimal disclosure, but be explicit).

```bash
echo 'export OPENCLAW_DISABLE_BONJOUR=1' >> ~/.bashrc
source ~/.bashrc
openclaw config get gateway.mdns
```

## 7️⃣ OpenClaw Security Audit

```bash
openclaw security audit --deep
openclaw security audit --fix
```

## ✅ Notes on Ordering

1. **Step 1** patches the OS before any configuration — always patch first.
2. **Steps 2–4** secure the server access layer (SSH, brute-force protection, firewall).
3. **Steps 5–6** harden OpenClaw's file permissions and network disclosure.
4. **Step 7** runs OpenClaw's built-in audit to catch anything else.

*This guide assumes OpenClaw is already installed and onboarded. It focuses exclusively on security hardening.*