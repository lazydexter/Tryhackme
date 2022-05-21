Skynet TryHackMe Walkthrough

Introduction
This was an easy Linux box that involved accessing an open SMB share containing a list of credentials that could be used to bruteforce a SquirrelMail web application, finding SMB credentials on the application to access a new share which revealed a second web application, and exploiting a remote file inclusion vulnerability in Cuppa CMS to gain remote access. Privilege escalation was possible due to a misconfigured cron job running as root and using a wildcard with the tar command.

Enumeration
The first thing to do is to run a TCP Nmap scan against the 1000 most common ports, and using the following flags:

-sC to run default scripts
-sV to enumerate applications versions
-Pn to skip the host discovery phase, as some hosts will not respond to ping requests
-oA to save the output in all formats available
![image](https://user-images.githubusercontent.com/71508714/169668306-429f63c0-fa69-4b4d-b5fc-68f43121ed3d.png)

We can see that we have several open ports on this machine:

Port 22 — SSH, not worth to check it for now
Port 80 — A web page running a search website
Port 110 — POP3, prolly a mail server
Port 143 — IMAP, prolly also part of the mail server
Port 139/445 — SMB ports, this is a good starting point
![image](https://user-images.githubusercontent.com/71508714/169668382-80739b1e-7dba-48a7-a0e0-b8f03a9ca885.png)

