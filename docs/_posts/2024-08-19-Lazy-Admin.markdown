---
layout: post
title:  "Lazy Admin - TryHackMe writeup"
date:   2024-08-19 10:00:00 +0200
categories: tryhackme ctf writeup
---
{:refdef: style="text-align: center;"}
[![Challenge logo](/assets/lazyadmin/0_title.png)](/assets/lazyadmin/0_title.png){:target="_blank"}
{:refdef}

**CTF Name**: [Lazy Admin](https://tryhackme.com/r/room/lazyadmin){:target="_blank"}  
**Description**: Easy linux machine to practice your skills.  
**Categories**: Linux, Web Application, CMS, Privilege Escalation  
**Difficulty**: Easy  

We begin the challenge by port scanning the target with **nmap**. We are going over each port (**-p-**) with a verbose (**-v**), aggressive scan (**-A**, which equals **-O** to detect the operating system the target is running, **-sC** to use the default set of scripts and **-sV** to detect the active services' version) scan. We are also saving nmap's output to a file (**-oN initial_scan.txt**).

{:refdef: style="text-align: center;"}
[![Nmap initial scan](/assets/lazyadmin/1_nmap.png)](/assets/lazyadmin/1_nmap.png){:target="_blank"}
{:refdef}

As the scanning continues, nmap tells us that there is something running on port **80**. It is a standard port so it is very likely that our target is hosting a website on that port. We can confirm this by visiting it, and we are met by Apache's default page.

{:refdef: style="text-align: center;"}
[![Website](/assets/lazyadmin/2_website.png)](/assets/lazyadmin/2_website.png){:target="_blank"}
{:refdef}

Now we should search for interesting directories, this is where **gobuster** comes to save the day. We can look for directories (**dir** option), but some files may also be of interest, hence we append **txt**, **php** and **html** as extensions (**-x "txt,php,html"**) to each entry in the wordlist, which is provided with the **-w** flag. As gobuster scans it finds an interesting directory, **/content**, we can now stop it and make it scan that directory, while also visiting it in the browser.

{:refdef: style="text-align: center;"}
[![Gobuster first scan](/assets/lazyadmin/3_gobuster_1.png)](/assets/lazyadmin/3_gobuster_1.png){:target="_blank"}
{:refdef}

While visiting **/content** may seem interesting, it actually comes out as a placeholder page. Going back to gobuster we can see it has found some more directories.

{:refdef: style="text-align: center;"}
[![Gobuster second scan](/assets/lazyadmin/4_gobuster_2.png)](/assets/lazyadmin/4_gobuster_2.png){:target="_blank"}
{:refdef}

Visiting them one after the other, two specific directories seem very appealing... **/as** is a login page, while **/inc** contains several files and other directories. Snooping around **/inc**, we can see that the file latest.txt contains some numbers, **1.5.1**. This is very likely a version number, and given its location it may indicate the version of the CMS used to manage this website. There is also one folder that may be of interest, can you see it?

{:refdef: style="text-align: center;"}
[![Inc directory](/assets/lazyadmin/5_inc.png)](/assets/lazyadmin/5_inc.png){:target="_blank"}
{:refdef}

It is the **mysql_backup** folder, which in turn contains a **.sql** file. It is always worth to check these files as they may contain sensitive information.
After downloading and opening the interesting file we can read what looks like a messy pile of SQL queries. Searching for "**password**" doesn't yield anything, while searching for "**pass**" does return something useful. Indeed, we found some interesting keywords, such as "**admin**", "**manager**", "**pass**", and what looks like a hash: **42f749ade7f9e195bf475f37a44cafcb**.

{:refdef: style="text-align: center;"}
[![SQL file](/assets/lazyadmin/6_sql_file.png)](/assets/lazyadmin/6_sql_file.png){:target="_blank"}
{:refdef}

We can now try cracking this hash. There are several options available but I'm sticking with **hashcat**. Let's save it in a file with **echo "42f749ade7f9e195bf475f37a44cafcb" > hash** and try attacking it with hashcat:

**hashcat -a 0 hash /usr/share/wordlists/rockyou.txt**

This runs hashcat in **dictionary attack** mode (**-a 0**), trying to crack the hashes that are in the file named "**hash**", with the **rockyou** wordlist. You may have to unzip the wordlist first with **gunzip /usr/share/wordlists/rockyou.txt.gz**.

Hashcat lists several hash types that our hash may match, but it cannot identify which one.

{:refdef: style="text-align: center;"}
[![Hashcat cracking the password](/assets/lazyadmin/7_hashcat_1.png)](/assets/lazyadmin/7_hashcat_1.png){:target="_blank"}
{:refdef}

There are several hash types but as **MD5** is the most commonly used let's try this one first. Now let's crack it with

**hashcat -a 0 -m 0 hash /usr/share/wordlists/rockyou.txt**

It shouldn't take much before it finds a match. Indeed, running **hashcat -m 0 hash --show** reveals the administrator's password.

{:refdef: style="text-align: center;"}
[![Hashcat showing the password](/assets/lazyadmin/8_hashcat_2.png)](/assets/lazyadmin/8_hashcat_2.png){:target="_blank"}
{:refdef}

We can now login into the CMS's admin dashboard at **/as**. Trying the keywords we found earlier we find the correct one being **manager**. Using **searchsploit** we can understand if this particular version of the CMS, **SweetRice 1.5.1**, has some vulnerabilities... indeed, it has quite some vulnerabilities.

{:refdef: style="text-align: center;"}
[![Searchsploit](/assets/lazyadmin/9_searchsploit.png)](/assets/lazyadmin/9_searchsploit.png){:target="_blank"}
{:refdef}

I chose to exploit the **arbitrary file upload vulnerability**, but the other ones may be viable as well. After reading the exploit's code we know that we need:

* the path to the target CMS, **TARGET_IP/content**
* the admin's credentials
* a file to be uploaded

As we want to obtain access to the target's machine we may upload a file that causes the opening of a reverse shell to our attacking machine once it is executed by the target machine. The webserver likely runs **PHP** (this can also be confirmed by using a browser extension such as **Wappalyzer**), so let's download the usual [PHP reverse shell script](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php){:target="_blank"} and edit it to connect to our own IP address.

{:refdef: style="text-align: center;"}
[![PHP reverse shell](/assets/lazyadmin/10_php_reverse_shell.png)](/assets/lazyadmin/10_php_reverse_shell.png){:target="_blank"}
{:refdef}

Before visiting the URL that now holds our malicious file, let's run a listener on the port we specified in the reverse shell. The following command uses **netcat** with the flags **-l** for listening, **-v** for verbose, **-n** to avoid name resolution with DNS and **-p** to specify the port to listen on:

**nc -lvnp PORT**

We can now launch the script with **python3 ./exploit.py** and provide all relevant information.

{:refdef: style="text-align: center;"}
[![Exploiting the vulnerable CMS](/assets/lazyadmin/11_exploit.png)](/assets/lazyadmin/11_exploit.png){:target="_blank"}
{:refdef}

Now when visiting the URL the page should look as if it’s loading, while switching to our terminal we should meet our reverse shell!

Let’s convert it into a fully interactive shell with the following commands:

1. **python3 -c 'import pty; pty.spawn("/bin/bash")'**: this runs the python code we specified, which opens a bash shell
2. press **CTRL + Z** in order to put the process in background and return to our attacking machine
3. running **stty raw -echo; fg** will set the current STTY type to raw (reads characters one at a time) and disables echo, that is, we won’t be echoed back the characters we typed; finally, **fg** foregrounds the process we put in background earlier
4. running export **TERM=xterm** sets the terminal emulator to xterm
5. finally, pressing **Enter** will give us our fully interactive shell

{:refdef: style="text-align: center;"}
[![Upgraded shell](/assets/lazyadmin/12_interactive_shell.png)](/assets/lazyadmin/12_interactive_shell.png){:target="_blank"}
{:refdef}

Running **whoami** we can see that we infiltrated as a regular, non privileged user. However, we can still look around the system for interesting information. First of all, running **ls /home** reveals that there is a user, **itguy**. Looking in this user’s home directory we can find the **first flag** at **/home/itguy/user.txt**.

{:refdef: style="text-align: center;"}
[![User flag](/assets/lazyadmin/13_user_flag.png)](/assets/lazyadmin/13_user_flag.png){:target="_blank"}
{:refdef}

However, to find the **root flag** we will have to **escalate our privileges**. Running **sudo -l** reveals that there is a command that our current user can run without entering their password

{:refdef: style="text-align: center;"}
[![Checking sudo permissions](/assets/lazyadmin/14_sudo_l.png)](/assets/lazyadmin/14_sudo_l.png){:target="_blank"}
{:refdef}

Investigating this file we can see that it is owned by root, but it can be read and executed by anyone. Reading it reveals that it runs another script, **/etc/copy.sh**. Investigating further we can see that this one script is owned by root as well, but not only everyone can read and execute it, it is also **world-writable**. This means that we can modify this script to do whatever we want, and since we can run this via **sudo** without using any password, we will have **root privileges** while doing so. Reading this file should feel familiar... what we are reading is a bash script that opens a reverse shell.

{:refdef: style="text-align: center;"}
[![Netcat reverse shell](/assets/lazyadmin/15_nc_reverse_shell.png)](/assets/lazyadmin/15_nc_reverse_shell.png){:target="_blank"}
{:refdef}

There are fancier solutions, but changing the IP with our own IP will do just fine. After modifying the file and running yet another listener on our attacking machine, we can run the script with **sudo /usr/bin/perl /home/itguy/backup.pl**. When switching to the terminal that was listening, we will be met by our **root shell**, hooray! We can now do anything we want with this machine, but to complete the challenge we just need to find the root flag. Running **ls /root** reveals the second and final flag for this CTF. We can then read it with **cat /root/root.txt**.

{:refdef: style="text-align: center;"}
[![Root flag](/assets/lazyadmin/16_root_flag.png)](/assets/lazyadmin/16_root_flag.png){:target="_blank"}
{:refdef}