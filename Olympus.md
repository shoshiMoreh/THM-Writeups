# TryHackMe CTF – Olympus | Writeup

> **Room:** https://tryhackme.com/room/olympusroom  
> **Difficulty:** Medium  
> **Goal:** Find 4 flags
---

## Step 1 – Nmap Scan

Let's start by running an `nmap` scan against the target machine to see what we're dealing with.

nmap scan results-

<img width="907" height="255" alt="image" src="https://github.com/user-attachments/assets/af20087b-466e-4ea2-ac9a-341e37d80a52" />

The scan reveals two open ports:
- **Port 22** – SSH
- **Port 80** – HTTP

> **Note:** Add the machine's IP with the hostname `olympus.thm` to `/etc/hosts`.
> <img width="476" height="100" alt="image1" src="https://github.com/user-attachments/assets/d186aee6-c920-4eab-b7f4-86cd7175599e" />


---

## Step 2 – Web Enumeration

Browsed to the website – nothing interesting visible on the surface.

website main page-

<img width="1912" height="607" alt="image2" src="https://github.com/user-attachments/assets/dd32d635-c31a-41d1-a271-13fcf930d513" />

Nothing interesting on the surface. Checking the page source, cookies, and headers didn't reveal anything useful either. Time to look for **subdomains**.

subdomain enumeration-

<img width="550" height="261" alt="image" src="https://github.com/user-attachments/assets/490b75ce-cf7f-4a7d-b2bc-78501f12a67f" />

Found a suspicious subdomain!

---

## Step 3 – SQL Injection

Navigating to the subdomain, I landed on a **CMS**. Looking around the page, I noticed a search field — which immediately raises the question: could this be vulnerable to **SQL Injection**?

A quick search on ExploitDB confirms that this type of PHP-based CMS does have a known SQLi vulnerability:  
🔗 https://www.exploit-db.com/exploits/48734

Ran the `sqlmap` command listed there:

```bash
sqlmap -u "http://olympus.thm/REDACTED/search.php" --data="search=1337*&submit=" --random-agent
```

sqlmap output-

<img width="303" height="156" alt="image4" src="https://github.com/user-attachments/assets/877f15f3-a6e2-42fb-88e0-a514a64e4dbc" />

---

## Step 4 – Database Enumeration

I queried the `olympus` database and see what tables are available:

```bash
sqlmap -u "http://olympus.thm/REDACTED/search.php" --data="search=1337*&submit=" -D olympus --tables --random-agent -v 3
```

database tables-

<img width="237" height="242" alt="image6" src="https://github.com/user-attachments/assets/660a8acf-cd75-460e-9423-23ddb6023ae1" />

Spotted a suspicious table named `flag`. Dumped its contents:

```bash
sqlmap -u "http://olympus.thm/REDACTED/search.php" --data="search=1337*&submit=" -D olympus -T flag --dump --random-agent -v 3
```

flag table dump-

<img width="356" height="182" alt="image" src="https://github.com/user-attachments/assets/36654db5-91d9-4384-9f83-c575438eca69" />

### 🚩 Flag 1 – Found!

---

## Step 5 – Hash Cracking

I also noticed a users table, so I dumped it as well.
users table-

<img width="1245" height="107" alt="image" src="https://github.com/user-attachments/assets/1823a219-b3a0-444a-8ac3-d416ca2c3464" />

The table contains 3 password hashes. Let's save them to a file and crack them with `john`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

john cracking-

<img width="1240" height="200" alt="image" src="https://github.com/user-attachments/assets/f3802f97-d7c9-4feb-b4a6-f1f682042285" />

Got the password! Let's use the email and password we found to log into the admin panel.

admin login-

<img width="357" height="215" alt="image10" src="https://github.com/user-attachments/assets/d8b19a9a-744d-48b8-8b75-67f7391fab2b" />

admin panel-

<img width="1862" height="597" alt="image11" src="https://github.com/user-attachments/assets/a822d377-4386-4e76-a248-ace085cab2f9" />


---

## Step 6 – Discovering Another Subdomain

When I looked at the users in the admin panel, I noticed that two of them had a slightly different domain suffix. I added it to `/etc/hosts`.

users domain-

<img width="1625" height="351" alt="image12" src="https://github.com/user-attachments/assets/39d338d0-a885-414c-a71c-9cb4d2d06af4" />

Navigating to `http://chat.olympus.thm`, I found another login page. Trying the same credentials from before — it works! I landed on what looks like a chat between two users.

chat page-

<img width="837" height="656" alt="image13" src="https://github.com/user-attachments/assets/98cdc1cd-c761-4f09-a40c-036fd2101cd9" />

---

## Step 7 – File Upload → Reverse Shell

The chat feature allows file uploads. Uploaded a PHP reverse shell:  
🔗 https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php

Set up a `netcat` listener, uploaded the file, and triggered it:

file upload-

<img width="262" height="93" alt="image14" src="https://github.com/user-attachments/assets/baecbd1c-c761-4380-ba4b-ea46ef360011" />

---

## Step 8 – Finding the Uploaded Filename

The chat mentions that uploaded files are **renamed** — so I couldn't just guess the filename. Let's go back to `sqlmap` and dump the `chats` table to find it:

```bash
sqlmap -u "http://olympus.thm/REACTED/search.php" --data="search=1337*&submit=" -D olympus -T chats --dump --random-agent
```

chats table-

<img width="1242" height="292" alt="image" src="https://github.com/user-attachments/assets/bcf0cfde-de37-420a-9376-a067e7fc6b3a" />

Browsed to the file directly:

```
http://chat.olympus.thm/uploads/<filename>.php
```

Got a shell! Explored hidden files and directories to find the next flag.

<img width="737" height="146" alt="image" src="https://github.com/user-attachments/assets/a2f7ec84-b3d9-4027-a90d-43c847de72b0" />

### 🚩 Flag 2 – Found!

---

## Step 9 – SUID Enumeration

I searched for SUID binaries on the system:

```bash
find / -perm -u=s -type f 2>/dev/null
```

Found an interesting binary:

```
-rwsr-xr-x 1 zeus zeus 17728 Apr 18 2022 /usr/bin/cputils
```

I ran the binary to understand what it does — it turned out to be a simple file copy utility, asking for a source and target path.

I used it to copy zeus's SSH private key to a location I could read.

cputils usage-

<img width="870" height="532" alt="image18" src="https://github.com/user-attachments/assets/b17aadab-6f5d-4a90-ac4a-30caed6c68a6" />

---

## Step 10 – Cracking the SSH Key

The private key is passphrase-protected. Let's convert it to a hash and crack it with `john`:

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

ssh key cracking-

<img width="1237" height="223" alt="image" src="https://github.com/user-attachments/assets/fa9a8e76-abcb-447f-a1b6-611dac038f88" />

Connected as `zeus`:

```bash
ssh -i id_rsa zeus@olympus.thm
```

I logged in as `zeus` — but the flag isn't immediately visible. Time to dig deeper.

---

## Step 11 – Privilege Escalation to Root

Checking which groups `zeus` belongs to, we notice access to `/var/www/html`. Inside, there's a suspicious PHP file with a hardcoded password in it.

Navigating to the file in the browser and entering the password, the server responded with an error asking for an IP and port. I opened a listener, submitted my **IP and port** — and caught a **root shell**!

Searching under `/root`, I found the next flag:

<img width="1240" height="182" alt="image" src="https://github.com/user-attachments/assets/47ecd03d-d715-42d5-a929-cbce4f6b6125" />

### 🚩 Flag 3 – Found!

---

## Step 12 – Final Flag

One more flag to go. Searching under `/etc` leads us to the last one:

final flag-

<img width="1193" height="271" alt="image" src="https://github.com/user-attachments/assets/a9a494e2-bc8d-479a-8d95-d0808fabd421" />


### 🚩 Flag 4 – Found!

---

## Summary

| Step | Technique |
|------|-----------|
| Reconnaissance | Nmap, Subdomain Enumeration |
| Initial Access | SQL Injection via sqlmap |
| Credential Access | Hash Cracking with john |
| Lateral Movement | File Upload → Reverse Shell |
| Privilege Escalation | SUID binary (`cputils`), SSH Key Cracking |
| Root | PHP Backdoor → Root Shell |

**Tools used:** `nmap`, `sqlmap`, `john`, `netcat`, `ssh`

