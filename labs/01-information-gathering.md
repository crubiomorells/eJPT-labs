# Lab 01 - Information Gathering CTF

This lab focuses on web reconnaissance techniques. The objective is to discover five flags hidden across a web application by analyzing exposed files, misconfigurations, and common weak points.

## ✅ Flag 1 – robots.txt file
**Hint:** This tells search engines what to and what not to avoid.

The `robots.txt` file was accessed directly by visiting:
[http://target.ine.local](http://target.ine.local)

The file contained the first flag


## ✅ Flag 2 – Extracting metadata from the site
**Hint:** What website is running on the target, and what is its version?

The `whatweb` tool was used to analyze the target’s homepage and extract metadata:

```bash
whatweb http://target.ine.local
```

The output revealed that the site is running **Apache/2.4.41** on **Ubuntu**, and uses **WordPress 6.5.3** as the CMS.

The second flag was embedded directly in the site’s `MetaGenerator` tag, which was automatically parsed and displayed by `whatweb`.


## ✅ Flag 3 – Discovering directories via brute-force
**Hint:** Directory browsing might reveal where files are stored.

Since the target machine is not indexed and has no external exposure, Google Dorks were not applicable.

Instead, directory enumeration was performed using **DirBuster**, a GUI-based brute-force tool for discovering hidden paths.

A custom wordlist located at `/root/Words/words.txt` was used. It contained paths commonly found in WordPress installations:

```wp-content
wp-content/uploads
wp-config.php
wp-config.bak
backup
.old
.tmp
```

The scan successfully discovered the `wp-content/uploads/` directory, which was accessible. Inside it, a file named `flag.txt` was found, containing the third flag.

This highlights how default upload directories in CMS platforms like WordPress can unintentionally expose sensitive files if not properly restricted.


## ✅ Flag 4 – Exposed backup file in the web root
**Hint:** An overlooked backup file in the webroot can be problematic if it reveals sensitive configuration details.

While inspecting common WordPress-related files, a backup version of the configuration file was discovered:
`http://target.ine.local/wp-config.bak`

The file was accessed using `curl`:
```bash
curl http://target.ine.local/wp-config.bak
```
It contained sensitive information such as database credentials and internal settings. The fourth flag was found inside this file.

This reinforces the importance of not leaving raw or backup files in the web root, especially those containing sensitive data.


## ✅ Flag 5 – Discovering hidden files via site mirroring
**Hint:** Certain files may reveal something interesting when mirrored.

To explore the full structure of the website, the entire site was mirrored locally using **HTTrack**, which allowed for offline inspection of all discovered files.

Command used:

```bash
httrack http://target.ine.local
```

By navigating through the mirrored content, an unlinked file was discovered that contained the fifth flag. This file was not visible via standard browsing or listed in any directory index.

HTTrack proved useful for identifying orphaned files and for analyzing the full layout of the web application outside of a live session



---
## 📝 Summary

This lab simulated a real-world scenario where a penetration tester must enumerate a web server and uncover hidden or exposed information through passive and active techniques. The objective was to locate five distinct flags using tools and methods commonly applied during the information gathering phase.


## 🛠️ Tools Used

- Firefox – to manually browse and inspect HTTP responses
- WhatWeb – to fingerprint the web server and detect CMS/version info
- DirBuster – to brute-force common WordPress directories using a custom wordlist
- Curl – to test direct access to known files
- HTTrack – to mirror the entire website and locate unlinked or orphaned files
