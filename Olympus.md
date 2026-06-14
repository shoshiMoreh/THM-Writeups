# TryHackMe CTF – Olympus | Writeup

> **Room:** https://tryhackme.com/room/olympusroom  
> **Difficulty:** Medium  
> **Goal:** Find 4 flags
---

## Step 1 – Nmap Scan

Started with an `nmap` scan against the target machine.

nmap scan results-

<img width="476" height="100" alt="image1" src="https://github.com/user-attachments/assets/a3d61fec-f71a-4352-bf4d-bb089c1609d5" />


**Open ports:**
- SSH – port 22
- HTTP – port 80

> **Note:** Add the machine's IP with the hostname `olympus.thm` to `/etc/hosts`.

---

## Step 2 – Web Enumeration

Browsed to the website – nothing interesting visible on the surface.

website main page-

<img width="1912" height="607" alt="image2" src="https://github.com/user-attachments/assets/dd32d635-c31a-41d1-a271-13fcf930d513" />


Checked page source, cookies, etc. – nothing useful. Moved on to **subdomain enumeration**.

subdomain enumeration-

