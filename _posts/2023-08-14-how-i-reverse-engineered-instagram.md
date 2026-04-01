---
title: "How I Reverse Engineered Instagram: A Minimal Guide"
date: 2023-08-14 18:45:00 +0530
categories: [Security, Reverse Engineering]
tags: [reverse-engineering, android, frida, ssl-pinning, fiddler, network-analysis, instagram, adb]
image:
  path: https://i.postimg.cc/7Y8NPSRN/download-(9).jpg
  alt: "Network traffic analysis on a monitor"
description: "A practical walkthrough of intercepting and analyzing Instagram's app traffic using Frida, Fiddler, and Memu Player — for educational purposes."
---

In this article I'll walk through how I reverse engineered Instagram to analyze its app traffic. This is purely for educational purposes — understanding how apps communicate with their backends, how SSL pinning works, and how tools like Frida operate is genuinely useful knowledge for anyone interested in security or app development.

> This guide is intended for educational and research purposes only. Do not use these techniques to access data you're not authorized to, violate any app's terms of service, or do anything illegal. Reverse engineering for personal learning is fine — anything beyond that is your responsibility.
{: .prompt-warning }

---

## What You'll Need

- **[Fiddler Classic](https://www.telerik.com/fiddler/fiddler-classic)** — HTTP/HTTPS debugging proxy that captures app traffic
- **[Memu Player](https://www.memuplay.com/)** — Android emulator, lightweight and ADB-friendly
- **[Frida](https://frida.re/)** — dynamic instrumentation toolkit for hooking into running processes
- **ADB (Android Debug Bridge)** — comes with Android SDK Platform Tools, used to talk to the emulator

---

## Background: Why Is This Non-Trivial?

If you just set up a proxy and pointed the emulator at it, you'd capture plain HTTP traffic easily. But Instagram (like most production apps) uses two defenses that make this harder:

**TLS/HTTPS** — all traffic is encrypted. A proxy like Fiddler can act as a man-in-the-middle, but the app needs to trust Fiddler's certificate. On modern Android (7+), apps no longer trust user-installed certificates by default.

**SSL Pinning** — even if you install a custom CA cert, Instagram's app checks that the certificate it receives matches a hardcoded expected value (the "pin"). If it doesn't match, the app drops the connection entirely rather than letting a proxy intercept it.

Frida bypasses both of these at the runtime level — it hooks into the app's process while it's running and patches out the pinning checks before they can reject your proxy's certificate.

---

## Step 1: Install Fiddler Classic

Download and install [Fiddler Classic](https://www.telerik.com/fiddler/fiddler-classic). During or after installation, enable HTTPS decryption:

Go to **Tools → Options → HTTPS** and check:
- ✅ Capture HTTPS CONNECTs
- ✅ Decrypt HTTPS traffic

Fiddler will generate its own root certificate. Note your PC's local IP address (you'll need it later to configure the emulator's proxy).

---

## Step 2: Set Up Memu Player

Install [Memu Player](https://www.memuplay.com/) and boot it up. Memu is preferred here over heavier emulators like BlueStacks because it plays nicer with ADB and Frida. Once it's running, make sure ADB can see it:

```bash
adb devices
```

You should see the emulator listed. If not, try:

```bash
adb connect 127.0.0.1:21503
```

Memu's default ADB port is `21503`.

---

## Step 3: Check the Emulator's CPU Architecture

Frida server is architecture-specific — you need the right build for your emulator. Check what Memu is running:

```bash
adb shell getprop ro.product.cpu.abi
```

Common outputs:
- `x86` — most emulators default to this for performance
- `x86_64` — 64-bit x86
- `arm64-v8a` — ARM 64-bit
- `armeabi-v7a` — ARM 32-bit

Note this down, you'll need it in the next step.

---

## Step 4: Download the Right Frida Server

Go to [Frida's GitHub releases page](https://github.com/frida/frida/releases) and download the latest `frida-server` build matching your architecture. The file will be named something like:

```
frida-server-16.x.x-android-x86.xz
```

Extract it:

```bash
xz -d frida-server-16.x.x-android-x86.xz
```

Rename it for convenience:

```bash
mv frida-server-16.x.x-android-x86 frida-server
```

---

## Step 5: Push Frida Server to the Emulator

```bash
adb push frida-server /data/local/tmp/
```

Grant execute permissions:

```bash
adb shell "chmod 755 /data/local/tmp/frida-server"
```

> Use `755` rather than `777` — it's sufficient for execution and is better practice. `777` gives write permissions to everyone, which is unnecessary.
{: .prompt-tip }

---

## Step 6: Start Frida Server

```bash
adb shell "/data/local/tmp/frida-server &"
```

The `&` runs it in the background. Verify it's working:

```bash
frida-ps -U
```

This lists all running processes on the emulator. If you see a list of apps, Frida is working correctly. If it errors, the most common cause is architecture mismatch — double-check step 3 and 4.

---

## Step 7: Configure Fiddler as the Emulator's Proxy

In Memu Player, go to:

**Settings → Wi-Fi → Long press the connected network → Modify network → Advanced options**

Set:
- **Proxy:** Manual
- **Proxy hostname:** your PC's local IP (e.g. `192.168.1.x`)
- **Proxy port:** `8888` (Fiddler's default)

Now open a browser inside the emulator and visit `http://ipv4.fiddler:8888`. Download and install Fiddler's root certificate. This lets the emulator trust Fiddler's MITM certificate for regular HTTPS — Instagram will still reject it due to pinning, but we'll handle that next.

---

## Step 8: Bypass SSL Pinning with Frida

This is the key step. Frida has a community script specifically for bypassing SSL pinning across multiple frameworks at once:

```bash
frida --codeshare akabe1/frida-multiple-unpinning -U -f com.instagram.android
```

What this does:
- `--codeshare akabe1/frida-multiple-unpinning` — pulls a community script that patches SSL pinning in OkHttp, TrustKit, Conscrypt, and several other common pinning implementations
- `-U` — targets a USB/emulator-connected device
- `-f com.instagram.android` — spawns Instagram fresh (rather than attaching to a running instance, which is less reliable)

The app will launch. Watch Fiddler — you should start seeing Instagram's HTTPS requests flowing through.

---

## Step 9: Analyze the Traffic

Once traffic is flowing into Fiddler, you can explore:

- **API endpoints** — what URLs does Instagram hit, and how are they structured?
- **Request headers** — what authentication tokens, device fingerprints, or session identifiers does it send?
- **Request/response payloads** — what data is being exchanged? Instagram's API mostly returns JSON.
- **GraphQL queries** — Instagram uses GraphQL internally for a lot of its feed and story endpoints

Use Fiddler's **Inspectors** tab to view decoded request/response bodies. The **AutoResponder** feature lets you modify responses before they reach the app — useful for testing how the app handles unexpected data.

---

## Troubleshooting

**Traffic not appearing in Fiddler:**
- Double-check the proxy IP and port in the emulator's Wi-Fi settings
- Make sure Fiddler is set to allow remote connections: **Tools → Options → Connections → Allow remote computers to connect**
- Ensure PC firewall isn't blocking port 8888

**Frida fails to attach:**
- Architecture mismatch — re-check `adb shell getprop ro.product.cpu.abi` and download the matching Frida server
- Frida server might have crashed — re-run the push and start commands
- Version mismatch between `frida` (CLI on your PC) and `frida-server` (on the emulator) — both must be the same version

**SSL pinning bypass not working:**
- Instagram updates frequently and sometimes ships new pinning implementations. Try alternative scripts: `universal-android-ssl-pinpin` or `https://codeshare.frida.re/`
- Try attaching to a running process instead of spawning: replace `-f com.instagram.android` with `-n Instagram`
- Older versions of the Instagram APK are easier targets — try pulling an older APK from APKMirror

---

## What This Teaches You

Beyond the Instagram-specific stuff, going through this process gives you hands-on experience with:

- **How TLS works in practice** — not just theoretically, but why certificate chains matter and how pinning breaks the standard trust model
- **Dynamic instrumentation** — Frida's approach of hooking into live processes is used in both security research and app debugging at scale
- **Android internals** — ADB, the `/data/local/tmp/` execution pattern, process spawning
- **API analysis** — understanding how a production app actually communicates with its backend is invaluable when building your own APIs or clients

---

## Closing Thoughts

Reverse engineering is one of the most educational things you can do as a developer. You're not reading documentation written for you — you're figuring out how something actually works from the outside. That skill transfers everywhere.

Instagram specifically is interesting because it's a production app at massive scale, with a proper security team, actively maintained defenses. Getting traffic flowing through a proxy despite all of that teaches you more about TLS and certificate trust than any tutorial will.

Use it to learn. Don't use it to scrape, abuse, or violate anyone's privacy.
