---
layout: default
title: How to Set Up an AI Chief of Staff from Scratch
---

# How to Set Up an AI Chief of Staff from Scratch

**A complete, non-technical guide for setting up an always-on AI team member on a Mac mini, connected to Slack.**

This guide walks you through every step — from unboxing a Mac mini to having a fully operational AI agent that responds on Slack, handles email, runs scheduled tasks, and manages your tools.

**Time estimate:** A full day if you're doing it for the first time. Faster the second time.

---

## Table of Contents

1. [What You're Building](#what-youre-building)
2. [What You'll Need Before You Start](#what-youll-need-before-you-start)
3. [Phase 1: Set Up the Mac Mini](#phase-1-set-up-the-mac-mini)
4. [Phase 2: Install Core Software](#phase-2-install-core-software)
5. [Phase 3: Create the Agent's Online Accounts](#phase-3-create-the-agents-online-accounts)
6. [Phase 4: Set Up Claude Code](#phase-4-set-up-claude-code)
7. [Phase 5: Build the Slack Bot](#phase-5-build-the-slack-bot)
8. [Phase 6: Connect Slack to the Mac via Cloudflare Tunnel](#phase-6-connect-slack-to-the-mac-via-cloudflare-tunnel)
9. [Phase 7: Make It Always On (launchd)](#phase-7-make-it-always-on-launchd)
10. [Phase 8: Give It an Identity](#phase-8-give-it-an-identity)
11. [Phase 9: Connect Google Workspace](#phase-9-connect-google-workspace-email-calendar-drive)
12. [Phase 10: Set Up the Email Watcher](#phase-10-set-up-the-email-watcher)
13. [Phase 11: Set Up the File Browser](#phase-11-set-up-the-file-browser)
14. [Phase 12: Set Up Scheduled Jobs](#phase-12-set-up-scheduled-jobs)
15. [Phase 13: Optional Extras](#phase-13-optional-extras)
16. [Maintenance & Troubleshooting](#maintenance--troubleshooting)
17. [Architecture Overview](#architecture-overview)

---

## What You're Building

You're setting up an AI agent that:
- **Lives on a Mac mini** that's always powered on and connected to the internet
- **Talks to your team via Slack** — they DM it or @mention it in channels
- **Has its own email address** — it can receive, read, and reply to emails
- **Has access to Google Workspace** — Gmail, Calendar, Drive, Sheets, Docs
- **Runs on a schedule** — it can do things automatically (daily reports, pipeline refreshes, etc.)
- **Has a web-accessible file browser** — your team can see its files via a browser
- **Is reachable via a public URL** — Cloudflare Tunnel provides a secure connection without opening firewall ports

The brain behind all of this is **Claude Code** (Anthropic's AI coding agent), running via a **Claude Max subscription** ($200/month flat rate).

**Why Claude Code and not the API?** Anthropic's ToS prohibit using a Max subscription with third-party tools. Claude Code is Anthropic's own CLI — running it via a Slack bot is fully compliant. You get flat $200/mo for Claude Opus instead of unpredictable API bills.

> **Companion site with more context:** [nityeshaga.github.io/claude-home-base](https://nityeshaga.github.io/claude-home-base/) — explains the concepts, architecture, and philosophy behind this setup.

---

## What You'll Need Before You Start

### Hardware
- [ ] **Mac mini** (Apple Silicon — M1, M2, M3, or M4)
- [ ] **Monitor, keyboard, mouse** (for initial setup only — you can disconnect these later)
- [ ] **Ethernet cable or stable WiFi** (ethernet is more reliable for an always-on machine)
- [ ] **Power cable** (comes with the Mac mini)

### Accounts You'll Create
- [ ] **Anthropic account** with **Claude Max subscription** ($200/month) — this is the AI brain
- [ ] **Slack workspace** (you probably already have one)
- [ ] **GitHub account** — for the agent (to store and manage code)
- [ ] **Google Workspace account** — an email address for the agent (e.g., `agent@yourcompany.com`)
- [ ] **Cloudflare account** (free tier works) — for the secure tunnel
- [ ] **Tailscale account** (free for personal use) — for internal team access
- [ ] **Google Cloud project** (free tier) — needed for Gmail push notifications
- [ ] **Gemini API key** (free tier from Google AI Studio) — for image generation (optional)

### Domain
- [ ] A domain name you control (e.g., `yourcompany.com`) — you'll point a subdomain like `bot.yourcompany.com` through Cloudflare. The domain's nameservers must already point to Cloudflare (or you'll need to move them).

---

## Phase 1: Set Up the Mac Mini

### 1.1: Factory Reset (if previously used)

1. Back up anything worth keeping from the old install
2. Go to **System Settings → General → Transfer or Reset → Erase All Content and Settings**
3. This takes ~15-20 minutes. Keep it plugged in and connected to WiFi.

### 1.2: Initial macOS Setup

1. **Plug in** the Mac mini (power, monitor, keyboard, mouse, ethernet)
2. **Turn it on** — press the power button on the back
3. Walk through the setup wizard:
   - Choose your language and region
   - Connect to WiFi (if not using ethernet)
   - **Create a user account** — use the agent's name (e.g., username: `claudie`, full name: `Claudie`). Pick a short username — you'll type it every time you SSH in.
   - **Sign in with your existing Apple ID** (needed to download Tailscale from the App Store)
   - **When asked about FileVault (disk encryption): Skip it / turn it OFF.** If the Mac restarts after a power loss, FileVault shows a lock screen that needs a keyboard. Since it might restart unattended (lid closed), skip encryption so it boots straight to the desktop.
   - Skip all the "share analytics" options

### 1.3: System Settings Checklist

Open **System Settings** and configure each:

1. **Energy** (or **Energy Saver**):
   - Turn ON **"Prevent automatic sleeping when the display is off"**
   - Turn ON **"Start up automatically after a power failure"**
   - Turn ON **"Wake for network access"**
   - Turn ON **"Optimized Battery Charging"** (protects battery while plugged in 24/7)

2. **Lock Screen**:
   - Set "Turn display off when inactive" to **Never** (or a long time like 3 hours)
   - Set **"Require password after screen saver"** to **Never** (so Screen Sharing works without auth prompts)

3. **Users & Groups**:
   - Set **Automatic login** to the user account you created (so after a reboot, it goes straight to the desktop)

4. **General → Sharing**:
   - Turn ON **Remote Login (SSH)** — lets you connect from another computer
   - Turn ON **Screen Sharing** — lets you see the Mac's screen remotely
   - Note the command shown (something like `ssh claudie@192.168.1.xxx`)

### 1.4: Keep It Awake with the Lid Closed

Open **Terminal** (search for "Terminal" in Spotlight, or find it in Applications → Utilities) and run:

```bash
# Prevent sleep entirely — even with the lid closed
sudo pmset -a disablesleep 1

# Verify it worked
pmset -g

# To reverse later: sudo pmset -a disablesleep 0
```

> **What's `sudo`?** It means "run as admin." It will ask for the password you set during setup. Type it — nothing will appear on screen as you type, that's normal — and press Enter.

> **Battery note:** Running plugged in 24/7 can cause battery swelling over months/years. "Optimized Battery Charging" helps. Worst case, battery replacement is ~$100-150.

### 1.5: Grant Full Disk Access to Terminal

macOS blocks background processes (like the Slack bot) from accessing files unless explicitly allowed. Without this, you'll get constant "would like to access data" popups — and since the lid is closed, nobody can click "Allow."

1. Go to **System Settings → Privacy & Security → Full Disk Access**
2. Click the **+** button
3. Navigate to **Applications → Utilities → Terminal.app** and add it
4. Toggle it **ON**

> **Why?** launchd services inherit permissions from Terminal. This one setting prevents all file-access popups.

### 1.6: Install Tailscale

Tailscale creates a private network between your devices — so you can access the Mac from anywhere (home, coffee shop, travelling) without exposing anything to the public internet.

1. Open the **Mac App Store** on the Mac mini, search for **Tailscale**, install it
2. Open Tailscale, sign in with your account (Google, GitHub, or Apple ID)
3. In Tailscale preferences, enable **"Start Tailscale when I log in"**
4. Note the **Tailscale IP** (something like `100.x.x.x`) — you'll need this later
5. **Install Tailscale on your laptop/phone too** — same account. Now both devices are on the same private network.

```bash
# Find the Tailscale IP later if you forget
tailscale ip -4
```

> At home you can SSH using the local address (`ssh claudie@your-mac.local`). Outside the house, use the Tailscale IP (`ssh claudie@100.x.x.x`). Both always work.

### 1.7: Set It Up Physically

1. Plug into power (always — use a reliable outlet, ideally with a UPS/surge protector)
2. Connect to your home WiFi (or ethernet via USB-C adapter — more reliable)
3. Close the lid, tuck it somewhere out of the way — a shelf, behind a monitor, wherever
4. Note the local IP address or hostname (shown in System Settings → Sharing)

**From your laptop**, you can now connect:
- **Screen Sharing**: Open Finder → Go → Connect to Server → type `vnc://TAILSCALE_IP`
- **SSH**: Open Terminal → type `ssh claudie@TAILSCALE_IP`

---

## Phase 2: Install Core Software

From here on, you can do everything via SSH from your laptop — you don't need a monitor connected to the Mac mini.

```bash
ssh claudie@TAILSCALE_IP   # or ssh claudie@your-mac.local if on the same WiFi
```

### 2.1: Install Homebrew (Package Manager)

Homebrew is like an app store for command-line tools. Copy and paste this into Terminal:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

It will ask for your password. Type it (you won't see characters — that's normal) and press Enter.

After it finishes, it'll show you two commands to run. Copy and paste those too. They look something like:

```bash
echo >> ~/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 2.2: Install Required Tools

```bash
# Install everything in one go
brew install git node python3 cloudflared gh tmux mosh
```

What these are:
- **git** — version control (managing code)
- **node** — JavaScript runtime (needed for Claude Code and some tools)
- **python3** — Python runtime (the bot runs on Python)
- **cloudflared** — Cloudflare's tunnel client
- **gh** — GitHub's command-line tool
- **tmux** — lets you run terminal sessions that persist even after you disconnect. Start a task, close your SSH connection, come back later, and pick up where you left off.
- **mosh** — mobile-friendly SSH replacement. If your WiFi drops or your laptop sleeps, mosh reconnects automatically. Optional but very nice to have.

### 2.3: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Verify it installed:

```bash
claude --version
```

### 2.4: Enable Passwordless Sudo for Claude Code

Claude Code needs to run admin commands (installing packages, managing services, etc.) but can't type a password into the interactive prompt. This lets it run admin commands freely:

```bash
# Allow passwordless sudo for your user (replace YOUR_USERNAME with the actual username, e.g., claudie)
echo "YOUR_USERNAME ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/YOUR_USERNAME
```

> **Is this safe?** On a personal laptop, you wouldn't do this. But this is a dedicated AI server behind Tailscale that only you access. The bot needs full system access to be useful — installing tools, managing services, reading logs. This is the right call for this machine.

### 2.5: Install Google Workspace CLI (gws)

```bash
npm install -g @googleworkspace/cli
```

Verify:

```bash
gws --version
```

### 2.6: Install Browser Automation Tools (Optional)

```bash
npm install -g agent-browser dev-browser
dev-browser install  # Installs the browser engine
```

---

## Phase 3: Create the Agent's Online Accounts

### 3.1: Create a GitHub Account

1. Go to **github.com** in a browser
2. Click **Sign up**
3. Create an account for the agent (e.g., username: `claudie-yourcompany`)
4. Verify the email

**Set up SSH key** (so the Mac can push code to GitHub):

```bash
# Generate an SSH key
ssh-keygen -t ed25519 -C "agent@yourcompany.com"
# Press Enter for default location, no passphrase (just press Enter twice)

# Show the public key
cat ~/.ssh/id_ed25519.pub
```

Copy the output. Then:
1. Go to **github.com → Settings → SSH and GPG keys → New SSH key**
2. Paste the key and save

**Set up GitHub CLI authentication:**

```bash
gh auth login
# Choose: GitHub.com → SSH → Yes (use existing key) → Login with a web browser
# Follow the prompts
```

### 3.2: Create a Google Workspace Email

If your company uses Google Workspace:
1. Go to your **Google Admin console** (admin.google.com)
2. Create a new user (e.g., `claudie@yourcompany.com`)
3. Set a password
4. Log into Gmail with this account in Chrome on the Mac mini to verify it works

If you don't have Google Workspace, you can use a regular Gmail account — create one at gmail.com.

### 3.3: Create a Cloudflare Account

1. Go to **cloudflare.com** and sign up (free)
2. Add your domain to Cloudflare (it will walk you through updating your domain's nameservers)
3. This is needed later for the tunnel

### 3.4: Get a Claude Max Subscription

1. Go to **claude.ai** and sign up or log in
2. Subscribe to **Claude Max** ($200/month) — this gives you flat-rate access to Claude Opus via Claude Code
3. On the Mac mini, run:

```bash
claude login
```

Follow the prompts to authenticate. This stores credentials on the Mac.

> **Tip:** Claude will print a URL. Open it in any browser — on this Mac, your phone, or your main machine. It's just a login link.

### 3.5: Get a Gemini API Key (Optional — for Image Generation)

1. Go to **aistudio.google.com/apikey**
2. Click **Create API Key**
3. Copy the key — you'll add it to the `.env` file later

---

## Phase 4: Set Up Claude Code

### 4.1: Create the Directory Structure

```bash
# Create the workspace directories
mkdir -p ~/Projects
mkdir -p ~/work
mkdir -p ~/temp
mkdir -p ~/diary
mkdir -p ~/bookmarks
mkdir -p ~/discoveries
mkdir -p ~/teammates
mkdir -p ~/logs
```

### 4.2: Clone the Slack Bot Code

The bot code is open-source:

```bash
cd ~/Projects
git clone https://github.com/nityeshaga/claude-home-base.git slack-bot
cd slack-bot
```

### 4.3: Install Python Dependencies

```bash
cd ~/Projects/slack-bot
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

This creates an isolated Python environment and installs: `slack-bolt`, `slack-sdk`, `python-dotenv`, `flask`

> **What's a venv?** A virtual environment keeps your Python packages separate from the system Python. It prevents conflicts and keeps things clean.

### 4.4: Configure Claude Code Settings

```bash
mkdir -p ~/.claude
nano ~/.claude/settings.json
```

Paste this content:

```json
{
  "model": "opus[1m]",
  "skipDangerousModePermissionPrompt": true
}
```

> **What `skipDangerousModePermissionPrompt` does:** When Claude Code runs from a scheduled job (no human present), it needs to skip the "are you sure?" prompt. This enables that.

Save and exit (in nano: Ctrl+O, Enter, Ctrl+X).

---

## Phase 5: Build the Slack Bot

### 5.1: Create a Slack App

You need admin access to your Slack workspace:

1. Go to **api.slack.com/apps** → **Create New App**
2. Choose **"From an app manifest"** (faster) or **"From scratch"** (manual)

### 5.2 (Option A): Easy Way — App Manifest

Paste this JSON manifest — it configures everything in one shot. Replace the bot name and URL:

```json
{
  "display_information": {
    "name": "Your Bot Name",
    "description": "AI cofounder for your team",
    "background_color": "#0f172a"
  },
  "features": {
    "bot_user": {
      "display_name": "Your Bot Name",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read", "channels:history", "channels:join",
        "channels:read", "chat:write", "chat:write.customize",
        "commands", "files:read", "files:write",
        "groups:history", "groups:read", "im:history",
        "im:read", "im:write", "links:read",
        "mpim:history", "mpim:read", "mpim:write",
        "pins:read", "pins:write", "reactions:read",
        "reactions:write", "team:read", "users:read",
        "users:read.email", "users.profile:read",
        "bookmarks:read", "metadata.message:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://bot.yourdomain.com/slack/events",
      "bot_events": [
        "app_mention", "message.im", "message.channels",
        "message.groups", "message.mpim",
        "member_joined_channel", "reaction_added",
        "file_shared"
      ]
    },
    "org_deploy_enabled": false,
    "socket_mode_enabled": false,
    "token_rotation_enabled": false
  }
}
```

After creating the app, manually grab:
- **Bot Token**: OAuth & Permissions → Install to Workspace → copy `xoxb-...`
- **Signing Secret**: Basic Information → App Credentials → copy the secret

### 5.2 (Option B): Manual Way

1. **OAuth & Permissions** → Add all the Bot Token Scopes listed in the manifest above
2. **Event Subscriptions** → Enable Events → Request URL: `https://bot.yourdomain.com/slack/events`
3. Subscribe to bot events: `app_mention`, `message.im`, `message.channels`, `message.groups`, `message.mpim`, `member_joined_channel`, `reaction_added`, `file_shared`
4. **Install to Workspace** → Copy the Bot Token (`xoxb-...`)
5. **Basic Information** → Copy the Signing Secret

> **No Socket Mode.** We're using the HTTP Events API — the production-standard approach. Slack sends stateless HTTP POST requests to your URL. No fragile WebSocket connections to drop.

### 5.3: Find Your Slack User ID

In Slack, click on your profile → click the three dots (⋮) → **"Copy member ID"**. Do this for every team member who should be authorized to DM the bot.

### 5.4: Configure the Bot's Environment

```bash
cd ~/Projects/slack-bot
cp .env.example .env
nano .env
```

Fill in the values:

```
SLACK_BOT_TOKEN=xoxb-your-actual-bot-token-here
SLACK_SIGNING_SECRET=your-actual-signing-secret-here
AUTHORIZED_USERS=U0XXXXXXXXXX,U0YYYYYYYYYY
PROJECT_DIR=/Users/claudie
CLAUDE_TIMEOUT=1800
PORT=3000
GEMINI_API_KEY=your-gemini-key-here
```

### 5.5: Test the Bot Locally

```bash
cd ~/Projects/slack-bot
source venv/bin/activate
python bot.py
```

You should see: `Bot server starting on port 3000...`

Keep this running — you'll connect Slack to it in the next phase.

---

## Phase 6: Connect Slack to the Mac via Cloudflare Tunnel

This gives your Mac a public URL so Slack can send events to it. Free, secure, production-grade.

### 6.1: Authenticate Cloudflare

```bash
cloudflared tunnel login
```

This opens a browser. Log into Cloudflare and authorize the connection.

> **Do this before closing the lid!** It opens a browser window you need to click.

### 6.2: Create a Tunnel

```bash
cloudflared tunnel create my-bot
```

This outputs a tunnel ID (a long string like `e07e7261-4ea5-4d07-8e23-...`). Note it.

### 6.3: Create the Tunnel Config

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

Paste this (replace with your tunnel ID and domain):

```yaml
tunnel: YOUR-TUNNEL-ID-HERE
credentials-file: /Users/YOUR_USERNAME/.cloudflared/YOUR-TUNNEL-ID-HERE.json

ingress:
- hostname: bot.yourcompany.com
  service: http://localhost:3000
- service: http_status:404
```

### 6.4: Create the DNS Record

```bash
cloudflared tunnel route dns my-bot bot.yourcompany.com
```

### 6.5: Start the Tunnel and Install as a Service

```bash
# Test it first
cloudflared tunnel run my-bot

# If it works, install as a permanent system service (survives reboots)
sudo cloudflared service install
```

Now `bot.yourcompany.com` routes traffic to `localhost:3000` on your Mac. Cloudflare handles HTTPS, DDoS protection, and auto-retries. The tunnel runs as a system service — survives reboots.

### 6.6: Update the Slack Event URL

1. Go back to **api.slack.com/apps → your app → Event Subscriptions**
2. Set the Request URL to: `https://bot.yourcompany.com/slack/events`
3. Slack will send a verification request — if the bot is running, it should show "Verified" ✓
4. Save

### 6.7: Test It

Go to Slack and send a DM to your bot. It should respond!

---

## Phase 7: Make It Always On (launchd)

The bot is running, but if your Mac reboots or the process crashes, it's dead. **launchd** is macOS's built-in system for keeping programs alive forever.

### 7.1: Understand launchd

- **Plist files** go in `~/Library/LaunchAgents/` (for per-user services)
- **`RunAtLoad: true`** — starts the service when you log in
- **`KeepAlive: true`** — restarts the service if it crashes
- **`StartCalendarInterval`** — runs at specific times (like a scheduler)
- Times are in your **local timezone** (not UTC)

### 7.2: Create the Slack Bot Service

```bash
nano ~/Library/LaunchAgents/com.claude.homebase.plist
```

Paste (replace `YOUR_USERNAME` everywhere):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.homebase</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/venv/bin/python</string>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/bot.py</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/bot.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/bot-stderr.log</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

> **Why EnvironmentVariables?** launchd services don't inherit your shell's PATH. Without this, the bot can't find `claude` or brew-installed tools.

### 7.3: Load, Verify, and Test

```bash
# Start the service
launchctl load ~/Library/LaunchAgents/com.claude.homebase.plist

# Verify it's running
launchctl list | grep homebase

# Test crash recovery — kill it, watch launchd bring it back
kill $(pgrep -f bot.py)
sleep 3 && pgrep -f bot.py

# Test reboot survival
sudo reboot
# Wait 2-3 min, then DM the bot in Slack
```

Your bot now survives crashes and reboots. Close the lid, tuck it away.

---

## Phase 8: Give It an Identity

This is what makes your agent feel like a team member, not just a tool.

### 8.1: Create the CLAUDE.md File

This is the **operations manual** — Claude Code reads it at the start of every conversation. It tells the agent who it is, what tools it has, and how to behave.

Create `~/CLAUDE.md` and include:
- The agent's name and role
- Your company context
- The team roster (names, Slack user IDs, roles)
- How to communicate (Slack, email)
- Privacy rules (who can see what)
- File paths and workspace structure
- Instructions for every tool (gws, Cloudflare, Tailscale, etc.)
- Proactive messaging instructions
- Scheduling notes (launchd, not cron)

Use the [existing Claudie CLAUDE.md](https://github.com/nityeshaga/claude-home-base) as a template — it's comprehensive.

### 8.2: Create the Identity File

Create `~/identity.md` — this is the agent's personality and principles. Keep it short and genuine:
- What role does it play?
- How does it communicate? (direct? diplomatic?)
- What does it care about? (quality? speed? thoroughness?)
- What are its principles?

### 8.3: Create the Origin Story

Create `~/about-you-and-how-you-came-to-life.md` — context about why the agent exists, what problem it solves, and how it came to be.

### 8.4: Create the Teammates Directory

```bash
mkdir -p ~/teammates
```

Create `~/teammates/teammates.md` with:
- Authority levels (who's a supervisor, who's core team, who's wider org)
- Privacy rules and escalation protocol

Create individual files for each person (e.g., `~/teammates/natalia.md`):
- Name, Slack user ID, role, communication preferences

### 8.5: Set Up Memory

```bash
mkdir -p ~/.claude/projects/-Users-YOUR_USERNAME/memory
```

Create `~/.claude/projects/-Users-YOUR_USERNAME/memory/MEMORY.md` — the agent's persistent memory index. Starts empty and builds over time.

> **Or let Claude do it:** Open Claude Code on the Mac and paste: *"Clone https://github.com/nityeshaga/claude-home-base to ~/claude-home-base. Copy CLAUDE.md.example to ~/CLAUDE.md. Then ask me questions about my team, my company, and how I want you to behave — and write your own identity.md and origin story."*

---

## Phase 9: Connect Google Workspace (Email, Calendar, Drive)

This gives the agent access to Gmail, Calendar, Drive, Docs, Sheets, etc.

### 9.1: Create a Google Cloud Project

1. Go to **console.cloud.google.com** and sign in
2. Click **New Project** — name it something like "AI Agent"
3. Select the project

### 9.2: Enable APIs

In the Google Cloud Console:
1. Go to **APIs & Services → Library**
2. Search for and enable each of these:
   - Gmail API
   - Google Calendar API
   - Google Drive API
   - Google Docs API
   - Google Sheets API
   - Google Slides API (if needed)
   - Cloud Pub/Sub API (needed for email notifications)

### 9.3: Create OAuth Credentials

1. Go to **APIs & Services → Credentials**
2. Click **Create Credentials → OAuth client ID**
3. If prompted, configure the **OAuth consent screen** first:
   - User type: **Internal** (if using Google Workspace) or **External**
   - Fill in the app name, support email
   - Add scopes for Gmail, Calendar, Drive, etc.

> **Tip:** "Internal" means only people in your Google Workspace org can use it. This skips the test user and verification hassle entirely. If you're using a personal Gmail, choose "External" and add your email as a test user.

4. Back to credentials: Application type = **Desktop app**
5. Name it (e.g., "Agent CLI")
6. Copy the **Client ID** and **Client Secret**

### 9.4: Authenticate gws CLI

```bash
# Open Claude Code and tell it to do this:
claude
```

Then paste:

```
Install the Google Workspace CLI from https://github.com/googleworkspace/cli using npm.
Then authenticate using these credentials:

Client ID: <paste your client id>
Client Secret: <paste your client secret>

Run gws auth login with those set as env vars. I'll handle the browser consent screen.
After auth, verify it works by listing my Drive files.
```

> **Setting up on another machine?** You don't need to redo the Google Cloud project. Just install gws, run `gws auth login` with the same Client ID and Client Secret, and approve the consent screen.

### 9.5: Set Up Pub/Sub for Email Notifications

This lets the agent get real-time notifications when new emails arrive.

1. In **Google Cloud Console**, go to **Pub/Sub**
2. Create a topic (e.g., `gmail-notifications`)
3. Create a subscription for that topic
4. Grant `gmail-api-push@system.gserviceaccount.com` the **Pub/Sub Publisher** role on the topic
5. In the Gmail API settings, set up push notifications to this topic

The email watcher script handles the rest.

---

## Phase 10: Set Up the Email Watcher

### 10.1: Verify the Scripts Exist

```bash
ls ~/Projects/slack-bot/email-watcher.py
ls ~/Projects/slack-bot/email-sweep.py
```

These come with the slack-bot repository:
- **email-watcher.py** — Watches for new emails in real time, spawns a Claude session for each one
- **email-sweep.py** — Runs every 12 hours as a safety net to catch anything the watcher missed

### 10.2: Test the Email Watcher

```bash
cd ~/Projects/slack-bot
source venv/bin/activate
python email-watcher.py
```

Send a test email to the agent's address. You should see it get picked up in the logs.

### 10.3: Create the Email Watcher launchd Service

```bash
nano ~/Library/LaunchAgents/com.claude.email-watcher.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.email-watcher</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/venv/bin/python</string>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-watcher.py</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-watcher.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-watcher-stderr.log</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

### 10.4: Create the Email Sweep launchd Service

```bash
nano ~/Library/LaunchAgents/com.claude.email-sweep.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.email-sweep</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/venv/bin/python</string>
        <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-sweep.py</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
    </dict>

    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-sweep.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/Projects/slack-bot/email-sweep-stderr.log</string>

    <key>RunAtLoad</key>
    <true/>

    <key>StartInterval</key>
    <integer>43200</integer>
</dict>
</plist>
```

### 10.5: Load Both Services

```bash
launchctl load ~/Library/LaunchAgents/com.claude.email-watcher.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-sweep.plist
```

---

## Phase 11: Set Up the File Browser

This gives your team a web-based way to browse files on the Mac mini via Tailscale.

### 11.1: Create the File Browser

```bash
mkdir -p ~/work/file-browser
```

The file browser is a simple Python HTTP server with a web UI and markdown rendering. You can have Claude Code generate it for you:

```bash
claude -p "Create a Python HTTP server at ~/work/file-browser/server.py that serves files from my home directory. It should have a nice web UI with file browsing, markdown rendering, and a dark theme. Accept a port number as a command-line argument."
```

### 11.2: Create the launchd Service

```bash
nano ~/Library/LaunchAgents/com.claude.file-browser.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.file-browser</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/python3</string>
        <string>/Users/YOUR_USERNAME/work/file-browser/server.py</string>
        <string>8889</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME/work/file-browser</string>

    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/work/file-browser/server.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/work/file-browser/server-error.log</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.claude.file-browser.plist
```

Your team can now browse files at: `http://TAILSCALE_IP:8889`

---

## Phase 12: Set Up Scheduled Jobs

These are optional — add them as you need them.

### Example: Daily Task at 10:30 AM

```bash
nano ~/Library/LaunchAgents/com.claude.daily-task.plist
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.claude.daily-task</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.local/bin/claude</string>
        <string>-p</string>
        <string>--dangerously-skip-permissions</string>
        <string>Your prompt here — describe what the agent should do. When done, DM the team a summary using: python /Users/YOUR_USERNAME/Projects/slack-bot/bot.py --send SLACK_USER_ID "your summary"</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>10</integer>
        <key>Minute</key>
        <integer>30</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/Users/YOUR_USERNAME/logs/daily-task.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/YOUR_USERNAME/logs/daily-task.err</string>
    <key>WorkingDirectory</key>
    <string>/Users/YOUR_USERNAME</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/Users/YOUR_USERNAME/.local/bin:/Users/YOUR_USERNAME/.claude/local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>/Users/YOUR_USERNAME</string>
    </dict>
</dict>
</plist>
```

**Key points for scheduled jobs:**
- **Always use `--dangerously-skip-permissions`** — there's no human to click "approve"
- **Times are LOCAL** — if you want 10:30 AM Eastern, put `10` and `30`. Don't convert to UTC.
- **Weekday numbers:** 0 = Sunday, 1 = Monday, ..., 6 = Saturday
- The agent can notify the team via Slack using `bot.py --send`

Load it:
```bash
launchctl load ~/Library/LaunchAgents/com.claude.daily-task.plist
```

---

## Phase 13: Optional Extras

### 13.1: Install Claude Code Plugins

Plugins add specialized skills to the agent:

```bash
nano ~/.claude/settings.json
```

Add plugin marketplaces:

```json
{
  "model": "opus[1m]",
  "extraKnownMarketplaces": {
    "anthropic-agent-skills": {
      "source": {
        "source": "github",
        "repo": "anthropics/skills"
      }
    }
  },
  "enabledPlugins": {
    "example-skills@anthropic-agent-skills": true
  },
  "skipDangerousModePermissionPrompt": true
}
```

### 13.2: Set Up Browser Automation

For authenticated web browsing (sites where the agent is logged in):

1. Open Chrome on the Mac mini and log into all relevant accounts
2. The agent can use `dev-browser --connect` to control Chrome with full session access
3. See the CLAUDE.md section on browser automation for the full setup

### 13.3: Connect MCP Servers

MCP (Model Context Protocol) servers give the agent additional tool access. For example, you can connect Google Workspace MCP servers for richer integration.

### 13.4: Disable Automatic macOS Updates

Automatic updates can restart your Mac unexpectedly:

1. Go to **System Settings → General → Software Update → Automatic Updates**
2. Turn OFF **"Install macOS updates"** and **"Install Security Responses and system files"**
3. Apply updates manually when convenient

---

## Maintenance & Troubleshooting

### Check if services are running

```bash
# See all claude services
launchctl list | grep com.claude

# See if the bot process is alive
ps aux | grep bot.py

# See if the tunnel is alive
ps aux | grep cloudflared
```

### Read logs

```bash
# Bot logs
tail -50 ~/Projects/slack-bot/bot.log

# Email watcher logs
tail -50 ~/Projects/slack-bot/email-watcher.log
```

### Restart a service

```bash
# Stop it
launchctl unload ~/Library/LaunchAgents/com.claude.homebase.plist

# Start it again
launchctl load ~/Library/LaunchAgents/com.claude.homebase.plist
```

### The bot stopped responding

1. Check if it's running: `ps aux | grep bot.py`
2. Check the logs: `tail -50 ~/Projects/slack-bot/bot-stderr.log`
3. Common fixes:
   - Restart the service (see above)
   - Check that the Cloudflare tunnel is running
   - Check that your Claude Max subscription is active

### Gmail watch expired

The Gmail push notification watch expires every 7 days. The email watcher auto-renews on restart:

```bash
launchctl unload ~/Library/LaunchAgents/com.claude.email-watcher.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-watcher.plist
```

### Google auth token expired

```bash
gws auth login
# Complete the browser OAuth flow
gws auth export --unmasked > ~/.config/gws/credentials.json
```

### After a macOS update or reboot

Everything should restart automatically (thanks to `RunAtLoad: true` and `KeepAlive: true`). If not:

```bash
launchctl load ~/Library/LaunchAgents/com.claude.homebase.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-watcher.plist
launchctl load ~/Library/LaunchAgents/com.claude.email-sweep.plist
launchctl load ~/Library/LaunchAgents/com.claude.file-browser.plist
```

---

## Architecture Overview

Here's what the final setup looks like:

```
YOUR TEAM (Slack)
      |
      | Messages / @mentions
      v
CLOUDFLARE TUNNEL (bot.yourcompany.com)
      |
      | Secure HTTP -> localhost:3000
      v
+---------------------------------------------+
|            MAC MINI (always on)              |
|                                             |
|  +-------------------------------------+    |
|  |  SLACK BOT (bot.py on port 3000)    |    |
|  |  - Receives Slack events            |    |
|  |  - Spawns Claude Code sessions      |    |
|  |  - Streams responses back to Slack  |    |
|  +----------------+--------------------+    |
|                   |                          |
|                   v                          |
|  +-------------------------------------+    |
|  |  CLAUDE CODE (AI brain)             |    |
|  |  - Full filesystem access           |    |
|  |  - Can run any command              |    |
|  |  - Reads CLAUDE.md for identity     |    |
|  |  - Persistent memory across chats   |    |
|  +----------------+--------------------+    |
|                   |                          |
|       +-----------+-----------+              |
|       v           v           v              |
|  +--------+  +--------+  +----------+       |
|  | Google |  | GitHub |  | Gemini   |       |
|  | (gws)  |  | (gh)   |  | (images) |       |
|  +--------+  +--------+  +----------+       |
|                                             |
|  BACKGROUND SERVICES (launchd):             |
|  - Email watcher (always on)                |
|  - Email sweep (every 12 hours)             |
|  - File browser (port 8889, Tailscale)      |
|  - Scheduled jobs (daily reports, etc.)     |
|                                             |
|  TAILSCALE (private network):               |
|  - File browser at http://TAILSCALE_IP:8889 |
|  - Team can access any web service          |
+---------------------------------------------+
```

### Summary of Accounts & Services

| Component | Purpose | Cost |
|-----------|---------|------|
| Mac mini | The physical computer | ~$600 one-time |
| Claude Max | AI brain (Claude Code) | $200/month |
| Cloudflare | Secure tunnel (free tier) | Free |
| Tailscale | Private team network | Free |
| Google Workspace | Email, calendar, drive | Part of existing plan |
| GitHub | Code storage | Free |
| Google Cloud | Pub/Sub for email notifications | Free tier |
| Gemini API | Image generation (optional) | Free tier |

### Key Files Cheat Sheet

| File | What it does |
|------|-------------|
| `~/CLAUDE.md` | Operations manual — the agent reads this every session |
| `~/identity.md` | Personality and principles |
| `~/teammates/teammates.md` | Team roster with authority levels |
| `~/Projects/slack-bot/bot.py` | The Slack bot server |
| `~/Projects/slack-bot/.env` | Tokens and configuration |
| `~/Projects/slack-bot/email-watcher.py` | Real-time email handler |
| `~/Projects/slack-bot/email-sweep.py` | Periodic email safety net |
| `~/.claude/settings.json` | Claude Code configuration |
| `~/.cloudflared/config.yml` | Cloudflare Tunnel routing |
| `~/.config/gws/` | Google Workspace credentials |
| `~/Library/LaunchAgents/com.claude.*.plist` | Background service configs |

### What's Next: Growing Your AI Cofounder

Once the basics are running:

1. **Write a comprehensive CLAUDE.md** — your agent is only as good as the context you give it. Business model, architecture, processes, standing instructions.
2. **Set up scheduled routines** — morning briefings, email triage, daily reports. Each is a launchd job with a claude prompt.
3. **Give it more accounts** — email, GitHub, task tracker. The more tools it can access, the more it can do autonomously.
4. **Build custom skills** — write SKILL.md files that teach specific workflows (inbox management, deployment procedures, customer support templates).
5. **Point it at growth** — A/B test pages, rewrite onboarding emails, generate case studies, analyze what your best customers have in common.

---

**That's it.** Once all of this is running, you have an AI team member that's always on, always available, and gets smarter with every interaction.
