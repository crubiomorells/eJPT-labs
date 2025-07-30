# Lab 02 – Footprinting and Scanning CTF

This lab focused on footprinting and scanning the target host. Four flags were hidden behind common misconfigurations in HTTP, FTP and MySQL services.

## Flag 1 – Web‑server banner disclosure  
**Hint:** *The server proudly announces its identity in every response. Look closely; you might find something unusual.*

A quick fingerprint with **WhatWeb** revealed a custom `Server:` header:

```bash
whatweb http://target.ine.local
```
Sample output (truncated):

```
HTTPServer[Werkzeug/3.0.6 Python/3.10.12, FLAG1]
```

The flag string was embedded directly in that header. Using WhatWeb keeps the request volume minimal compared to running a full `nmap -A`, making it the cleaner option for an initial probe.


## Flag 2 – Disallowed path in robots.txt  
**Hint:** *The gatekeeper's instructions often reveal what should remain unseen. Don't forget to read between the lines.*

`robots.txt` was fetched and reviewed:

```bash
curl http://target.ine.local/robots.txt
```

```
User-agent: *
Disallow: /secret-info/
```

Navigating to the hidden directory and listing its contents:

```bash
curl -s http://target.ine.local/secret-info/
# -> ["flag.txt"]
curl -s http://target.ine.local/secret-info/flag.txt
# -> FLAG2
```

The flag sat in a plain‑text file that was never meant for indexing.


## Flag 3 – Anonymous FTP exposure  
**Hint:** *Anonymous access sometimes leads to forgotten treasures. Connect and explore the directory; you might stumble upon something valuable.*

Port 21 was confirmed open with `nmap -sS -sV`. Service‑script output showed `ftp-anon: Anonymous FTP login allowed`.

```bash
ftp target.ine.local
Name (target.ine.local:anonymous): anonymous
ftp> ls
200 PORT command successful
150 Here comes the directory listing.
-rw-r--r--   1 0        0              22 Oct 28  2024 creds.txt
-rw-r--r--   1 0        0              39 Jul 30 16:59 flag.txt
ftp> get flag.txt
```
Downloading `flag.txt` revealed the third flag.  
`creds.txt` contained database credentials used in the next step.


## Flag 4 – MySQL enumeration with leaked credentials  
**Hint:** *A well-named database can be quite revealing. Peek at the configurations to discover the hidden treasure.*

The credentials from `creds.txt` (user `db_admin`) allowed direct MySQL access:

```bash
mysql -h target.ine.local -u db_admin -p
```

Once connected, listing databases exposed a database whose name itself was the flag:

```sql
SHOW DATABASES;
+-------------------------------+
| Database                      |
+-------------------------------+
| information_schema           |
| FLAG4                        |
+-------------------------------+
```

No further queries were necessary—simply spotting the database name completed the objective.



---

### Summary

1. Start wide with banner grabbing (`WhatWeb`, `nmap -sV`) 
2. Review disclosure files (`robots.txt`) for overlooked paths.  
3. Check every open service for default or anonymous access (FTP in this case).  
4. Chain discoveries—credentials leaked via FTP unlocked MySQL and the final flag.

Even simple misconfigurations can cascade into full data exposure when services are left at their defaults.

### Tools Used

- whatweb – passive HTTP fingerprinting  
- nmap – TCP port and service enumeration  
- curl – quick HTTP requests  
- GNU ftp client – file retrieval over FTP  
- mysql 8.0 client – direct database interaction
