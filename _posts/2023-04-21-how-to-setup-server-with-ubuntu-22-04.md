---
title: "How to Setup a Server with Ubuntu 22.04"
date: 2023-04-21 21:32:05 +0530
categories: [Tutorials, Linux]
tags: [ubuntu, server, linux, ssh, firewall, ufw, devops]
image:
  path: https://github.com/user-attachments/assets/fffa5928-340e-42f6-b1b3-640508343191
  alt: "Server racks in a data center"
description: "A beginner-friendly walkthrough of setting up an Ubuntu 22.04 server — new user, firewall, and SSH key authentication."
---

In this article, we'll walk through the process of setting up an Ubuntu 22.04 server. Whether you're a CS student or just getting started with Linux infrastructure, this guide keeps things practical and straightforward. Let's dive in.

---

## Step 1: Logging into the Server

**For Linux / macOS users**, open a terminal and run:

```bash
ssh -i ssh_key abhishek@<server_ip_address>
```

**For Windows users:**

1. Download and install [PuTTY](https://www.putty.org/)
2. Launch PuTTY and enter:
   - **Hostname:** `<username>@<server_ip_address>`
   - Navigate to **SSH → Auth**, then click **Browse** to select your SSH private key
3. Click **Open** to connect

You're in. 🎉

---

## Step 2: Creating a New User

Running everything as root is a bad habit. Create a dedicated user instead:

```bash
sudo adduser <username>
```

Replace `<username>` with whatever you want. Then grant sudo privileges:

```bash
sudo usermod -aG sudo <username>
```

Your new user can now run admin commands with `sudo`.

---

## Step 3: Setting Up the Firewall

Ubuntu ships with UFW (Uncomplicated Firewall). Let's use it.

Check what application profiles are available:

```bash
ufw app list
```

Allow the traffic you actually need:

```bash
ufw allow OpenSSH
ufw allow http
ufw allow https
```

Enable the firewall:

```bash
ufw enable
```

Verify it's running correctly:

```bash
ufw status
```

Expected output:

```
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443                        ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
443 (v6)                   ALLOW       Anywhere (v6)
```

> Always allow OpenSSH **before** enabling UFW, otherwise you'll lock yourself out of the server.
{: .prompt-warning }

---

## Step 4: Setting Up SSH Key Authentication

Password-based SSH is convenient but weak. Key-based auth is the right way to do it.

Generate a new SSH key pair (run this on your **local machine**, not the server):

```bash
ssh-keygen
```

This creates two files — a private key (no extension, keep this secret) and a public key (`.pub` — this goes on the server).

Switch to your new user on the server:

```bash
su - <username>
```

Set up the `.ssh` directory with correct permissions:

```bash
mkdir ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Open the authorized keys file:

```bash
nano ~/.ssh/authorized_keys
```

Paste the contents of your `.pub` file in here and save.

Now test logging in with the key:

```bash
ssh -i <privatekeyfile> <username>@<server_ip_address>
```

> The permissions on `~/.ssh` (700) and `~/.ssh/authorized_keys` (600) are not optional — SSH will silently refuse to use your key if they're wrong.
{: .prompt-tip }

SSH key authentication is now enabled. 🎉

---

## Wrapping Up

By following this guide, you've:

- Logged into a fresh Ubuntu 22.04 server
- Created a non-root user with sudo privileges
- Configured UFW to allow only the traffic you need
- Set up SSH key-based authentication

This is the baseline setup for any server you'll run. Everything else — Nginx, Docker, your apps — builds on top of this foundation.

Happy exploring 🚀
