---
title: "Your Spare Android Tablet Can Host Git. Here's How."
date: 2026-04-01 10:00:00 +0530
categories: [Projects, Self-Hosting]
tags: [android, termux, forgejo, git, self-hosting, cloudflare, tailscale, linux]
image:
  path: https://github.com/user-attachments/assets/96993b82-a0ce-4526-bb8a-68cdf12ac6f3
  alt: "Self Hosted Git Server Meme"
description: "I had a tablet collecting dust. I had a weekend. Bad combination. Here's how I ran a full Git server on an Android tablet using Termux, Forgejo, and Cloudflare Tunnel."
---

I didn't want to spin up another $5/month VPS for something I was just experimenting with. I had a spare Android tablet doing nothing. And I wanted to understand what it actually takes to run your own Git server before committing to doing it "properly" on real infrastructure.

So the goal became: run [Forgejo](https://forgejo.org/) on the tablet. Give it a real domain. Make it accessible from anywhere. Push commits to it. See where it breaks.

Turns out it works surprisingly well.

Here's the full stack:

- **Termux** — Linux environment on Android
- **Arch Linux via proot** — for a real package manager and userland
- **Forgejo** — the Git hosting software (Gitea fork, cleaner)
- **Cloudflare Tunnel** — public HTTPS without port forwarding
- **tmux** — keeping everything alive when you walk away
- **Tailscale** — private network access (optional but nice)

---

## 1. Termux and Getting Arch Running

Install [Termux from F-Droid](https://f-droid.org/packages/com.termux/) — not the Play Store version, that one is outdated and abandoned.

<img width="680" height="328" alt="image" src="https://github.com/user-attachments/assets/ccc25cd9-54db-4057-9330-3c1a7d24e0b5" />


```bash
pkg update && pkg upgrade
pkg install proot-distro git
proot-distro install archlinux
```

Log into Arch:

```bash
proot-distro login archlinux
```

You're now in a pretty functional Arch Linux environment running on your Android tablet. Wild.

---

## 2. Basic Setup Inside Arch

Update and grab some basics:

```bash
pacman -Syu
pacman -S git wget curl tmux
```

Create a user for running Forgejo. Don't run it as root, even in proot — it's good practice and some things behave differently:

```bash
useradd -m git
passwd git
su - git
```

---

## 3. Installing Forgejo (The Right Way)

There are a few moving parts: detecting your device's architecture, fetching the latest release version from Codeberg, and constructing the right download URL. Let's put it in a small script instead of dumping a wall of bash inline.

```bash
nano install-forgejo.sh
```

Paste this in:

```bash
#!/bin/bash
set -e

# Detect CPU architecture
ARCH=$(uname -m)

if [ "$ARCH" = "aarch64" ]; then
    GOARCH="arm64"
elif [ "$ARCH" = "armv7l" ]; then
    GOARCH="arm-6"
else
    echo "unsupported architecture: $ARCH"
    exit 1
fi

# Fetch latest Forgejo release tag from Codeberg API
LATEST=$(curl -s "https://codeberg.org/api/v1/repos/forgejo/forgejo/releases?limit=1" \
    | grep -o '"tag_name":"[^"]*"' \
    | head -1 \
    | cut -d'"' -f4)

# Remove leading 'v' from tag (e.g., v1.4.0 -> 1.4.0)
VERSION=${LATEST#v}

echo "Installing Forgejo $VERSION for $ARCH ($GOARCH)..."

# Download binary
wget "https://codeberg.org/forgejo/forgejo/releases/download/${LATEST}/forgejo-${VERSION}-linux-${GOARCH}"

# Make executable
chmod +x "forgejo-${VERSION}-linux-${GOARCH}"

# Rename for convenience
mv "forgejo-${VERSION}-linux-${GOARCH}" forgejo

echo "Done. Run with: ./forgejo web"
```

Make it executable and run it:

```bash
chmod +x install-forgejo.sh
./install-forgejo.sh
```

What this is doing:

- `uname -m` returns your CPU arch. Android tablets are almost always `aarch64` (arm64). Older 32-bit devices report `armv7l`. The script maps this to what Forgejo's release filenames actually use.
- The Codeberg API returns JSON for the latest release. The `grep | cut` chain pulls the `tag_name` field out without needing `jq` installed.
- Forgejo's release tags use a `v` prefix (`v14.0.3`) but the filename doesn't (`forgejo-14.0.3-linux-arm64`), so we strip it with `${LATEST#v}`.
- The result is a single binary called `forgejo` in your current directory. No install step, no daemon setup, nothing else needed.

---

## 4. First Run and Initial Setup

Before doing any domain stuff, get Forgejo running locally and go through the setup wizard:

```bash
./forgejo web
```

Open `http://localhost:3000` in your tablet's browser (or any browser on the same network). You'll land on the Forgejo installer. Here's what to set:

**Database:** Pick SQLite. For a personal setup this is completely fine — zero extra dependencies.

**General Settings:**
- Site Title — give it something, you can change it later
- Repository Root Path — leave default (`~/forgejo-repositories`)
- Run User — should be `git`

**Server Settings:**
- Base URL — set to `http://localhost:3000` for now. We'll update this after the tunnel is live.

Hit "Install Forgejo". It'll write a config file and redirect you to the homepage. Create your admin account when prompted.

---

## 5. Setting Up tmux Before Going Further

Android will kill background processes. tmux keeps sessions alive even when you switch apps or the screen turns off.

Install if you haven't already (inside Arch):

```bash
pacman -S tmux
```

Start a session for Forgejo:

```bash
tmux new -s forgejo
```

Run Forgejo inside this session:

```bash
./forgejo web
```

Detach with `Ctrl + B`, then `D`. You're back in your normal shell. Forgejo is still running. This is the pattern we'll use for everything.

Useful tmux commands:

```bash
tmux ls                       # see all sessions
tmux attach -t forgejo        # jump back into forgejo session
tmux new -s tunnel            # new session for cloudflare tunnel
tmux kill-session -t forgejo  # stop a session
```

---

## 6. Cloudflare Tunnel (Public Domain, No Port Forwarding)

This is the good part. Cloudflare Tunnel lets you expose a local service to the internet without opening any ports on your router. All traffic goes through Cloudflare's network.

You'll need:
- A Cloudflare account
- A domain added to Cloudflare (nameservers pointed to CF)

Install `cloudflared` in Termux (outside proot, not inside Arch):

```bash
pkg install cloudflared
```

> **Why outside proot?** cloudflared works cleanly in Termux directly and has no special dependencies we need from Arch.
{: .prompt-tip }

Log in to Cloudflare:

```bash
cloudflared tunnel login
```

This opens a browser link. Authorize it. A credentials file gets saved to `~/.cloudflared/`.

Create the tunnel:

```bash
cloudflared tunnel create forgejo
```

This outputs a tunnel ID — save it.

Route your domain to the tunnel:

```bash
cloudflared tunnel route dns forgejo git.yourdomain.com
```

Replace `git.yourdomain.com` with whatever subdomain you want. This creates a CNAME record in your Cloudflare DNS automatically.

Write the tunnel config at `~/.cloudflared/config.yml`:

```yaml
tunnel: forgejo
credentials-file: /data/data/com.termux/files/home/.cloudflared/<YOUR_TUNNEL_ID>.json

ingress:
  - hostname: git.yourdomain.com
    service: http://localhost:3000
  - service: http_status:404
```

Start the tunnel in a tmux session:

```bash
tmux new -s tunnel
cloudflared tunnel run forgejo
```

Detach with `Ctrl + B`, `D`. Your Forgejo instance is now accessible at `https://git.yourdomain.com`. Cloudflare handles the TLS certificate automatically.

<img width="680" height="348" alt="image" src="https://github.com/user-attachments/assets/87368b64-52a6-450a-b995-3716c56d28c5" />


---

## 7. Update Forgejo Config to Use Your Domain

Forgejo generates URLs for clone links, webhooks, and emails based on `app.ini`. Right now it still thinks it's on localhost — fix that.

Find your `app.ini`:

```bash
find ~ -name "app.ini" 2>/dev/null
```

Usually at `~/custom/conf/app.ini`. Open it:

```bash
nano ~/custom/conf/app.ini
```

Find or add the `[server]` section:

```ini
[server]
DOMAIN           = git.yourdomain.com
ROOT_URL         = https://git.yourdomain.com/
PROTOCOL         = http
HTTP_ADDR        = 127.0.0.1
HTTP_PORT        = 3000
```

> `PROTOCOL = http` is correct here even though the public URL is HTTPS. Cloudflare handles TLS termination — Forgejo itself just speaks plain HTTP internally.
{: .prompt-tip }

Restart Forgejo:

```bash
tmux attach -t forgejo
# Ctrl+C to stop
./forgejo web
# Ctrl+B, D to detach
```

Clone links on your repos will now show your actual domain.

---

## 8. Customizing the Look

Forgejo lets you override templates and static assets without touching the binary. Everything goes in the `custom/` directory.

**Custom landing page:**

```bash
mkdir -p ~/custom/templates/home
wget -O ~/custom/templates/home.tmpl \
  "https://codeberg.org/forgejo/forgejo/raw/branch/forgejo/templates/home.tmpl"
```

Edit it — it's Go HTML templating. Remove the default taglines, add your own intro.

**Custom CSS** at `~/custom/public/css/user.css`:

```bash
mkdir -p ~/custom/public/css
nano ~/custom/public/css/user.css
```

```css
:root {
  --color-primary: #your-brand-color;
}

.dashboard .repositories .item {
  border-radius: 8px;
}
```

Forgejo automatically loads `user.css` — no config changes needed.

**Site logo:** Drop a `logo.svg` and `logo.png` in `~/custom/public/img/` and Forgejo picks them up automatically.

---

## 9. Tailscale (Optional but Useful)

If you want access to your Forgejo without going through the public internet:

```bash
pkg install tailscale  # in Termux
tailscale up
```

Get your device's Tailscale IP:

```bash
tailscale ip
```

From any other device on your Tailscale network you can hit `http://100.x.x.x:3000` directly. Good for pushing from your laptop over LAN without touching Cloudflare.

---

## 10. Making Processes Survive Reboots

tmux sessions don't survive a Termux restart. Two options:

**Option A: Termux:Boot**

Install [Termux:Boot](https://f-droid.org/packages/com.termux.boot/) from F-Droid. Create a startup script:

```bash
mkdir -p ~/.termux/boot
nano ~/.termux/boot/start-forgejo.sh
```

```bash
#!/data/data/com.termux/files/usr/bin/bash
tmux new-session -d -s forgejo -c /data/data/com.termux/files/home \
  "proot-distro login archlinux -- su - git -c './forgejo web'"

sleep 5

tmux new-session -d -s tunnel \
  "cloudflared tunnel run forgejo"
```

```bash
chmod +x ~/.termux/boot/start-forgejo.sh
```

**Option B: Just rerun it manually**

For a low-usage personal setup, manually restarting after a reboot is completely fine. You know the commands, it takes 30 seconds.

---

## What Actually Works Well

- Personal repos, private projects, anything at small scale
- SSH and HTTP clone both work
- Webhooks work (useful for triggering deploys)
- The web interface is genuinely good
- Battery impact is lower than expected when the tablet is plugged in

## What Doesn't Work Well

- CI/CD — Forgejo Actions needs more resources than a tablet wants to give
- Large repos or large file storage
- Anything that needs to be truly "always on" without babysitting

> This isn't production infra. It's a personal tool. Treat it like one.
{: .prompt-warning }

---

## Things I Learned Doing This

**"localhost" means different things in different contexts.** Inside proot, localhost is proot's network. In Termux, it's Android's network. In the Cloudflare config, it needs to reach the port Forgejo actually bound to. Getting these to line up took longer than it should have.

**Docker inside proot is not worth the pain.** The kernel is still Android's kernel. Some things that need specific kernel features just won't work. Forgejo as a single binary with SQLite sidesteps all of this cleanly.

**SQLite is underrated.** For a personal Git host with maybe 50 repos and a handful of users, SQLite is completely fine. No Postgres setup, no connection pooling, no nothing. It just works.

**tmux is mandatory.** Android's process management is aggressive. Without tmux, Forgejo dies the moment you switch apps. With tmux, it's been running for days.

**Tunnels beat port forwarding every time.** No need to touch router settings, no dynamic DNS headaches, no dealing with your ISP. Cloudflare Tunnel just works.

---

## What's Next

- Add a simple webhook receiver to auto-pull on push
- Host some internal tools behind Tailscale only (no public tunnel)
- Try Woodpecker CI on a slightly beefier device

The tablet thing is a starting point. The real point is: you can own this stack completely. No GitHub, no GitLab SaaS, no AWS. Just a device you control, running software you understand.

---

You don't need a homelab rack. You don't need Kubernetes. Sometimes the most interesting thing is seeing how far you can push what you already have.

Spare tablet + Termux + one weekend = your own Git infrastructure. That's kind of cool.
