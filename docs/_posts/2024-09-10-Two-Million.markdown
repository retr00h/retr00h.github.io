---
layout: post
title:  "Two Million - HackTheBox writeup"
date:   2024-09-10 11:22:00 +0200
categories: hackthebox ctf writeup
---
{:refdef: style="text-align: center;"}
[![Challenge logo](/assets/twomillion/0_title.png)](/assets/twomillion/0_title.png){:target="_blank"}
{:refdef}

**CTF Name**: [Two Million](https://www.hackthebox.com/machines/twomillion){:target="_blank"}  
**Description**:  TwoMillion is an Easy difficulty Linux box that was released to celebrate reaching 2 million users on HackTheBox. The box features an old version of the HackTheBox platform that includes the old hackable invite code. After hacking the invite code an account can be created on the platform. The account can be used to enumerate various API endpoints, one of which can be used to elevate the user to an Administrator. With administrative access the user can perform a command injection in the admin VPN generation endpoint thus gaining a system shell. An .env file is found to contain database credentials and owed to password re-use the attackers can login as user admin on the box. The system kernel is found to be outdated and CVE-2023-0386 can be used to gain a root shell.  
**Categories**: Linux, Web Application, API  
**Difficulty**: Easy  

The initial **nmap** scan reveals two open ports: port **22** and port **80**.

{:refdef: style="text-align: center;"}
[![Nmap scan](/assets/twomillion/1_nmap.png)](/assets/twomillion/1_nmap.png){:target="_blank"}
{:refdef}

Visiting the website results in an error that we can solve simply adding an entry in **/etc/hosts** with the displayed domain name. After correcting this we can visit the website. We can start enumerating its directories via **feroxbuster**:

{% highlight bash %}

feroxbuster --url http://2million.htb --depth 5 --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php txt html md

{% endhighlight %}

After a while we discover a couple of interesting things:
1. There are some **API** avaliable
2. There is an **invite page** where we can insert an invite code

{:refdef: style="text-align: center;"}
[![Invite page](/assets/twomillion/2_invite.png)](/assets/twomillion/2_invite.png){:target="_blank"}
{:refdef}

Examining the invite page's **source code** we discover that the evaluation seems to be handled by JavaScript code.

{:refdef: style="text-align: center;"}
[![Evaluation code](/assets/twomillion/3_evaluation_js.png)](/assets/twomillion/3_evaluation_js.png){:target="_blank"}
{:refdef}

Running this code through a **deobfuscator** reveals the original code, which is composed of two functions, the former takes the user-provided invitation code and makes a **POST** request to an **endpoint** to **** the code while the latter makes a **POST** request to another endpoint to **generate** an invite code. Both the functions log the result.

{:refdef: style="text-align: center;"}
[![Deobfuscated code](/assets/twomillion/4_decoded_js.png)](/assets/twomillion/4_decoded_js.png){:target="_blank"}
{:refdef}

As this function is loaded we can try making a request to see what's the output directly in the browser's console.

{:refdef: style="text-align: center;"}
[![Encryption](/assets/twomillion/5_encryption.png)](/assets/twomillion/5_encryption.png){:target="_blank"}
{:refdef}

We can see that the output is a **ROT13 encrypted** string that we can easily decrypt.

{:refdef: style="text-align: center;"}
[![Cyberchef](/assets/twomillion/6_cyberchef.png)](/assets/twomillion/6_cyberchef.png){:target="_blank"}
{:refdef}

The decrypted string tells us to contact **/api/v1/invite/generate** with a **POST** request. To do so we can use **CURL** and observe the result directly in the terminal.

{:refdef: style="text-align: center;"}
[![CURL request](/assets/twomillion/7_curl.png)](/assets/twomillion/7_curl.png){:target="_blank"}
{:refdef}

The result is another **encrypted** string. It looks **Base64** (because of the **=** at the end) and not only pasting it in cyberchef confirms that, but it also gives us a **valid code**!

{:refdef: style="text-align: center;"}
[![Valid invite code](/assets/twomillion/8_invitation_code.png)](/assets/twomillion/8_invitation_code.png){:target="_blank"}
{:refdef}

We can then use the code to register!

{:refdef: style="text-align: center;"}
[![Registration](/assets/twomillion/9_registration.png)](/assets/twomillion/9_registration.png){:target="_blank"}
{:refdef}

After that we may be able to enumerate part of the **endpoints** as many of them gave us the **HTTP 401** code. We can grab our own **session ID** through the browser's developer tools. To do so we simply use CURL with the **\-\-cookie** option. From there we learn there are several endpoints. There is one in particular that may be interesting, that is, the **admin/settings/update** endpoint.

**Note**: the **jq** command is useful to perform indentation to have more readable **JSON** objects.

{:refdef: style="text-align: center;"}
[![Endpoints](/assets/twomillion/10_api.png)](/assets/twomillion/10_api.png){:target="_blank"}
{:refdef}

This **endpoint** requires a **PUT** request. If we simply contact it with a **PUT** request it will give us an **invalid content type** error. Let's try supplying **"application/json"** as it is the one usually employed by **REST APIs**. Here we learn that we need to provide an **email** parameter.

{:refdef: style="text-align: center;"}
[![Admin API](/assets/twomillion/11_admin_api_1.png)](/assets/twomillion/11_admin_api_1.png){:target="_blank"}
{:refdef}

Let's add the **email** parameter with the option **\-d**, and now we learn that another parameter is needed, **is_admin**. We also learn that it has to be either **0** or **1**.

{:refdef: style="text-align: center;"}
[![Admin API](/assets/twomillion/12_admin_api_2.png)](/assets/twomillion/12_admin_api_2.png){:target="_blank"}
{:refdef}

Of course we want to be **administrator** so the parameter **is_admin** will be equal to **1**. After this request we have elevated our privileges to administrator level!

{:refdef: style="text-align: center;"}
[![Privilege escalation](/assets/twomillion/13_isadmin.png)](/assets/twomillion/13_isadmin.png){:target="_blank"}
{:refdef}

This can be confirmed by contacting **/admin/auth**.

{:refdef: style="text-align: center;"}
[![Privilege verification](/assets/twomillion/14_auth.png)](/assets/twomillion/14_auth.png){:target="_blank"}
{:refdef}

The website doesn't seem different from earlier, however, we may now be able to interact with the full range of endpoints! Let's try interacting with the admin **VPN** generator. We iteratively learn that a **POST** request is required (though we already knew this when we enumerated the endpoints), and that an **application/json** content type is required. After that, we learn that a **username** parameter is required. If we try generating a **VPN** file for ourselves we will see that it will work!

{:refdef: style="text-align: center;"}
[![Admin VPN](/assets/twomillion/15_vpn_gen.png)](/assets/twomillion/15_vpn_gen.png){:target="_blank"}
{:refdef}

This **endpoint** likely interacts with the **underlying system**. If no **validation** or **sanitizaiton** occurs, we may be able to run **arbitrary commands** on the system itself! Separating our username and a command (*e.g.* **whoami**) with a pipe **|** will still end in a **HTTP 200** response, but it will not yield any **VPN** file. This may indicate that what we input in this field may be **executed** by the system. Let's try adding another character, **;**.

{:refdef: style="text-align: center;"}
[![Command injection](/assets/twomillion/16_command_injection.png)](/assets/twomillion/16_command_injection.png){:target="_blank"}
{:refdef}

This proves that we can **exploit** this **command injection vulnerability**! Let's search for the **user flag** now.

We can see an interesting file relative to vulnerability **CVE-2023-0386**. We also learn that the user flag is only readable from the **admin** account.

{:refdef: style="text-align: center;"}
[![Home directory](/assets/twomillion/17_home.png)](/assets/twomillion/17_home.png){:target="_blank"}
{:refdef}

Before anything else we should look for a more comfortable way to execute commands: a **reverse shell**! We can submit the usual **netcat oneliner** and listen on the corresponding port on our machine.

{:refdef: style="text-align: center;"}
[![Reverse shell](/assets/twomillion/18_reverse_shell.png)](/assets/twomillion/18_reverse_shell.png){:target="_blank"}
{:refdef}

Then we can try exploring the current directory looking for interesting information. For example the **.env** file contains some **credentials** that we may be interested in.

{:refdef: style="text-align: center;"}
[![Credentials](/assets/twomillion/19_credentials.png)](/assets/twomillion/19_credentials.png){:target="_blank"}
{:refdef}

If we are lucky the **administrator** is using these credentials for **SSH** as well... we are lucky indeed! We can now read the **user flag** and execute the **exploit** as per instructions. We can then run commands as **root**, therefore, we can easily find and read the **root flag**!
