# OpenClaw Ubuntu Server Hardening Guide

This guide outlines the recommended steps to securely set up and harden OpenClaw on an Ubuntu server. It includes SSH hardening, firewall configuration, unattended upgrades, OpenClaw security audits, and plugin/security steps. Follow the steps **in order** for maximum security.

## 1️⃣ SSH Key & Server Setup

1. Generate a secure SSH key (if you don’t have one):
ssh-keygen -t ed25519 -C "your-email@example.com"

2. Copy the public key to the server (~/.ssh/authorized_keys).

3. Harden SSH using a single sed command:
sudo sed -i -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
            -e 's/^#\?PermitRootLogin.*/PermitRootLogin no/' \
            -e 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' \
            -e 's/^#\?PermitEmptyPasswords.*/PermitEmptyPasswords no/' \
            -e 's/^#\?ChallengeResponseAuthentication.*/ChallengeResponseAuthentication no/' \
            -e 's/^#\?UsePAM.*/UsePAM yes/' /etc/ssh/sshd_config

4. Restart SSH:
sudo systemctl restart sshd

## 2️⃣ Network & Firewall (Tailscale / UFW)

1. Install Tailscale (no exposed ports):
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4

2. Install and configure UFW to restrict SSH to Tailscale only:
sudo apt install ufw -y
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port 22
sudo ufw enable

## 3️⃣ System Updates & Unattended Upgrades

1. Ensure the system is up to date:
sudo apt update && sudo apt upgrade -y

2. Enable unattended upgrades:
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades

## 4️⃣ Install Node.js & OpenClaw

1. Install NVM and Node.js:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
source ~/.bashrc
nvm install 24

2. Install OpenClaw:
curl -fsSL https://openclaw.ai/install.sh | bash

> **Note:** Do not run onboarding yet — wait until after security hardening.

## 5️⃣ Harden OpenClaw Permissions & Disable Bonjour

chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/*.json
chmod 600 ~/.openclaw/credentials/*
echo 'export OPENCLAW_DISABLE_BONJOUR=1' >> ~/.bashrc

## 6️⃣ Run OpenClaw Security Audit

1. Deep audit:
openclaw security audit --deep

2. Apply fixes if needed:
openclaw security audit --fix

## 7️⃣ Optional: Install Plugins / Security Skills

### Matrix Plugin (E2E Encryption)
openclaw plugins install @openclaw/matrix
cd ~/.openclaw/extensions/matrix
sed -i 's/"workspace:\*"/"*"/g' package.json
npm install

- Configure Matrix accounts in ~/.openclaw/openclaw.json

### Security Skills (ACIP)
Message your bot: "Install this: https://github.com/Dicklesworthstone/acip/tree/main"

- Test defenses (bot should refuse sensitive instructions)

## 8️⃣ Run Onboarding (Safe After Hardening)

openclaw onboard

- Recommended options:
  - Mode: Manual
  - Provider: Claude Code or Venice AI
  - Model: Fully private (kimi-k2-5 or your chosen secure model)
  - Gateway bind: Loopback
  - Gateway auth: Token
  - Enable all hooks (boot, command-logger, session-memory)

## ✅ Notes on Ordering

1. Steps 1–3 secure the server and network first.
2. Steps 4–6 install and harden OpenClaw before onboarding.
3. Step 7 is optional but recommended for E2E encryption and extra bot security.
4. Step 8 (onboarding) is run last, after everything is locked down.

*This guide ensures a security-first setup for running OpenClaw on Ubuntu, minimizing exposure and attack surface.*