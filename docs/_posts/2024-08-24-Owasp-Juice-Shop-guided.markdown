---
layout: post
title:  "Owasp Juice Shop - TryHackMe writeup"
date:   2024-08-24 09:56:00 +0200
categories: tryhackme ctf writeup
---

{:refdef: style="text-align: center;"}
[![Challenge logo](/assets/owaspjuiceshopguided/0_title.png)](/assets/owaspjuiceshopguided/0_title.png){:target="_blank"}
{:refdef}

---

**CTF Name**: [Owasp Juice Shop](https://tryhackme.com/r/room/owaspjuiceshop){:target="_blank"}  
**Description**: This room uses the Juice Shop vulnerable web application to learn how to identify and exploit common web application vulnerabilities.  
**Categories**: Linux, Web Application, OWASP Top 10  
**Difficulty**: Easy  

**Owasp Juice Shop**, an insecure-by-design website made especially for practicing the **OWASP Top 10** vulnerabilities, and more. In this writeup we will simply follow TryHackMe's guide, using slightly different approaches in some cases (e.g. using ZAP instead of BurpSuite).

---

### Task 1

This task does not require any actual investigation, just read its content and check the buttons.

---

### Task 2

The first question in **Task 2** requires us to find the **administrator's email** while looking around the website. Snooping around the homepage we cannot find any email address. It's just product listings. The **Customer Feedback** section also does not reveal anything useful. However, the **About Us** section contains some blurred photos and non-anonymous ones have a partial email of the owner attached to them. The first photo actually looks interesting... As a matter of fact, it is the one we are looking for!

{:refdef: style="text-align: center;"}
[![About Us section](/assets/owaspjuiceshopguided/1_about_us.png)](/assets/owaspjuiceshopguided/1_about_us.png){:target="_blank"}
{:refdef}

Next, we must find out what is the parameter that is used to enable the searching functionality on this website. Making a quick mock search we learn that the search functionality uses a **GET** request, and we can also see the parameter being used.

{:refdef: style="text-align: center;"}
[![Search](/assets/owaspjuiceshopguided/2_search.png)](/assets/owaspjuiceshopguided/2_search.png){:target="_blank"}
{:refdef}

Next, we must find out what show did **Jim** reference in his review. To do so, we simply go back to the home page and look at the various products, until we find the one reviewed by Jim, the **Green Smoothie**. Googling the "**replicator**" he mentioned we can find out the source of its quote.

{:refdef: style="text-align: center;"}
[![Jim's Review](/assets/owaspjuiceshopguided/3_review.png)](/assets/owaspjuiceshopguided/3_review.png){:target="_blank"}
{:refdef}

---

### Task 3

The first question in this task wants us to **login as administrator**. To do so we must exploit an injectable field in the login form. Let's scroll all the way to the top of the website where we can find the login button, which brings us to the login form in turn. We know that we must use **SQL Injection**, in particular, this is called a **blind SQLi** because we cannot see the output of the query directly. This kind of SQLi also allows us to bypass insecure login forms. We simply input a crafted string in the **email** field, and anything as **password**. Our crafted string is exactly **' OR 1 = 1;--**. This works because the query will likely be something like this:

{% highlight sql %}
SELECT EXISTS(*) FROM users WHERE email = 'test@test.test' AND password = HASHFUNCTION('password123');
{% endhighlight %}

Where **HASHFUNCTION** is not necessarily applied in the database. This is an abstraction because usually (hopefully) passwords are not stored in **plain text**, so we cannot always hope that this kind of SQLi works when using the password field. When an application uses **unsanitized user input**, and a user inputs such a string, the query will become

{% highlight sql %}
SELECT EXISTS(*) FROM users WHERE email = '' OR 1 = 1;-- AND password = '';
{% endhighlight %}

The **--** comments out whatever follows (so we ignore the password checking), while **1 = 1** always evaluates to **TRUE** and since it is put in **OR** with the rest of the query, the query itself will return **TRUE**. Trying it out, we can easily bypass the authentication form, and here is the first flag of this task.

{:refdef: style="text-align: center;"}
[![Admin login](/assets/owaspjuiceshopguided/4_admin_login.png)](/assets/owaspjuiceshopguided/4_admin_login.png){:target="_blank"}
{:refdef}

The next part of this task requires us to login as **Bender**, and it gives us their email. We can logout from the administrator application and go back to the login panel. We can also avoid appending the **OR 1 = 1;--** to our query if we are sure that the email is correct (and assuming the form itself is not secure). Inputting **bender@juice-sh.op';--** in the email field will make the query become similar to this

{% highlight sql %}
SELECT EXISTS(*) FROM users WHERE email = 'bender@juice-sh.op';--
{% endhighlight %}

Once again, this query evaluates to **TRUE**, and therefore it will let us in, giving us the answer to the second and final question of this task.

{:refdef: style="text-align: center;"}
[![Bender login](/assets/owaspjuiceshopguided/5_bender_login.png)](/assets/owaspjuiceshopguided/5_bender_login.png){:target="_blank"}
{:refdef}

**Note**: we could have used this technique with the administrator's profile too since we know their email.

---

### Task 4

We are now tasked with the discovery of the **admin's password**. To do so, we have to discover the name of the parameters used by the form. We will use **ZAP**, a free and open source alternative to **BurpSuite**. After logging out from the account we breached in the previous task, we launch **ZAP** and make a mock request. Here we can see the names of the parameters used: **email** and **password**.

{:refdef: style="text-align: center;"}
[![Intercepting an HTTP request with ZAP](/assets/owaspjuiceshopguided/6_zap_request.png)](/assets/owaspjuiceshopguided/6_zap_request.png){:target="_blank"}
{:refdef}

Now we must **Fuzz** this request (**right click > Attack > Fuzz**). This will allow us to **brute force** the administrator's password. First, click **Edit** at the top on the window, then insert the correct email for the email parameter and save. Then select the **password** (do not select the quotation marks though) and click **Add** on the right. Click **Add** again, change the type to **File** and select the **best1050.txt** wordlist (it's located at **/usr/share/wordlists/SecLists/Passwords/Common-Credentials/best1050.txt**).

**Note**: if you don't have this one wordlist you can download it with **sudo apt update && sudo apt install seclists**.

Finally, click **Add** once again, ignore any charset warnings and confirm with **Ok**.

{:refdef: style="text-align: center;"}
[![Fuzzing an HTTP request with ZAP](/assets/owaspjuiceshopguided/7_zap_fuzz.png)](/assets/owaspjuiceshopguided/7_zap_fuzz.png){:target="_blank"}
{:refdef}

We can now start the fuzzer, which will take a while. However, sorting by **code** (ascending) we will notice a request that has the **200** state. We can also see the associated **password** at the end. We can now login using the administrator's credentials and find our beloved **flag**.

{:refdef: style="text-align: center;"}
[![Bruteforced password](/assets/owaspjuiceshopguided/8_bruteforce.png)](/assets/owaspjuiceshopguided/8_bruteforce.png){:target="_blank"}
{:refdef}

Now we have to reset **Jim**'s password. Using his email (from the review we found in an earlier task) we can reset his password. However, we have to answer the security question, which wants us to provide his **eldest sibling's middle name**. We know **Jim** likes **Star Trek**, so a quick google search for "**Jim Star Trek eldest brother middle name**" reveals the name we are looking for. We can now change the password to whatever we like.

{:refdef: style="text-align: center;"}
[![Changing Jim's password](/assets/owaspjuiceshopguided/9_jim_password.png)](/assets/owaspjuiceshopguided/9_jim_password.png){:target="_blank"}
{:refdef}

---

### Task 5

The next question wants us to explore a **confidential document**. Going back to the **About Us** page, we can see a link that lets us download the terms and conditions of usage. We can see that it is stored in a subdirectory, **/ftp**. Can we access that folder? It appears we can. From here we can access several important files, but the one we're interested in for the flag is **acquisitions.md**. Downloading it and go back to the home page to receive the **flag**!

{:refdef: style="text-align: center;"}
[![Downloading confidential data](/assets/owaspjuiceshopguided/10_ftp.png)](/assets/owaspjuiceshopguided/10_ftp.png){:target="_blank"}
{:refdef}

**Question 2** is really self explanatory. We can steal this user's account simply because they basically stated their credentials publicly.

Going back to the **/ftp** directory, we can try downloading the **package.json.bak** file, but an error happens stating that only **.md** and **.pdf** files can be downloaded. Our evil plan is ruined... or is it? Actually it isn't. We can add a **null byte** at a specific point of the string to trick the server, making it think that the file's name ends there even when it doesn't. Since we are exploiting an **URL**, we have to **URL-encode** the null byte before inserting it into the **URL**. We can then add the **.md** (fake) extension to finally download the file. Going back to the home page we will have our **flag**.

{:refdef: style="text-align: center;"}
[![Nullbyte poisoning](/assets/owaspjuiceshopguided/11_nullbyte.png)](/assets/owaspjuiceshopguided/11_nullbyte.png){:target="_blank"}
{:refdef}

---

### Task 6

As this task revolves around **broken access control**, we have to access the administration page... but where is it? There are several ways to do this, one could try enumerating the website, for example. However, we can also exploit the **JavaScript** code of the website itself, searching for useful information. To do so we must inspect the code via the browser's developer tools and look for interesting things around the "**admin**" keyword. We can find an interesting string, **path: 'administrator'**, that hints about a directory, **/administrator**.

{:refdef: style="text-align: center;"}
[![Searching for information](/assets/owaspjuiceshopguided/12_admin_js.png)](/assets/owaspjuiceshopguided/12_admin_js.png){:target="_blank"}
{:refdef}

Visiting this directory is still not enough. It makes sense that only an administrator could access it, but we already know how to login as administrator... Therefore, logging before visiting the page will let us find the answer to the first question of this task.

{:refdef: style="text-align: center;"}
[![Administrator dashboard](/assets/owaspjuiceshopguided/13_administration.png)](/assets/owaspjuiceshopguided/13_administration.png){:target="_blank"}
{:refdef}

For the next challenge, **access another user's basket**, we will have to use **ZAP** once again, so open it again (and make sure to use its **proxy**) and let's visit our own (the administrator's) basket. Looking at the request we can see that it simply reaches for an **API endpoint** with a number that identifies our cart. If no validation is done, we could easily access another user's basket by changing that number.

{:refdef: style="text-align: center;"}
[![Basket request](/assets/owaspjuiceshopguided/14_basket_request.png)](/assets/owaspjuiceshopguided/14_basket_request.png){:target="_blank"}
{:refdef}

To do so, let's modify the request (**right click > open in Requester tab**) and send it. The response will include data from the other user's basket, and going back to our browser we will see the answer for this question.

{:refdef: style="text-align: center;"}
[![Basket response](/assets/owaspjuiceshopguided/15_basket_response.png)](/assets/owaspjuiceshopguided/15_basket_response.png){:target="_blank"}
{:refdef}

Now we are tasked with the **removal of all 5-star reviews**. To do so, we shall visit the administrator's dashboard and simply manually remove the only **5-star review**.

{:refdef: style="text-align: center;"}
[![Removing reviews](/assets/owaspjuiceshopguided/16_removing_reviews.png)](/assets/owaspjuiceshopguided/16_removing_reviews.png){:target="_blank"}
{:refdef}

---

### Task 7

We now have to exploit **XSS** vulnerabilities, and the first question wants us to perform a **DOM XSS attack**. This kind of attack is going to modify the page's **Document Object Model**, which in turn means that, most likely, it look different. This is possible because the website lacks **input sanitization**. So let's go back to the homepage and input the following string into the search bar:

{% highlight html %}
<iframe src="javascript:alert(`xss`)">
{% endhighlight %}

When performing the search, we will see an alert popup with our text, and when closing it we will see the flag we were looking for.

{:refdef: style="text-align: center;"}
[![DOM XSS attack](/assets/owaspjuiceshopguided/17_dom_xss.png)](/assets/owaspjuiceshopguided/17_dom_xss.png){:target="_blank"}
{:refdef}

As we now know that the website is vulnerable to **DOM XSS**, it is likely vulnerable to other, more dangerous kinds of **XSS**. The next question wants us to perform a **persistent XSS attack**. It is **persistent** because it is saved by the application and it runs every time a specific page is visited or a specific action is carried out. First, let's clean **ZAP**'s request history. Now we have to perform a logout and modify the request to save the **last login IP**, which is actually going to be our malicious string. If you are not logged in already, login as administrator and then logout. If you are, simply logout. Now head on to **ZAP** and open the request in the **Requestor** tab. We are going to add a **header** to our request and send it.

{:refdef: style="text-align: center;"}
[![Persistent XSS attack](/assets/owaspjuiceshopguided/18_persistent_xss.png)](/assets/owaspjuiceshopguided/18_persistent_xss.png){:target="_blank"}
{:refdef}

Signing in as the administrator once again and visiting the last login page (**Account > Privacy & Security > Last Login IP**) will now display an alert similar to the one we've injected earlier. And we should also see the **flag**.

The last question of this task wants us to perform a **relfected XSS attack**. To do so, let's head to the order history page (**Account > Orders & Payment > Order History**) and click on the truck icon (for tracking an order). We can notice that this page's **URL** contains the parameter that represents our **order ID**. If this is not sanitized we could instead send a malicious string to perform our attack. Let's try sending our usual payload which should cause a popup to appear.

{:refdef: style="text-align: center;"}
[![Reflected XSS attack](/assets/owaspjuiceshopguided/19_reflected_xss_1.png)](/assets/owaspjuiceshopguided/19_reflected_xss_1.png){:target="_blank"}
{:refdef}

After visiting the malicious **URL** (and **refreshing** the page if necessary), the popup will appear, along with the last **flag**!

{:refdef: style="text-align: center;"}
[![Reflected XSS attack](/assets/owaspjuiceshopguided/20_reflected_xss_2.png)](/assets/owaspjuiceshopguided/20_reflected_xss_2.png){:target="_blank"}
{:refdef}

---

### Task 8

For the final task, we simply have to visit the scoreboard, which is located at **/score-board/**. Here we can see a lot more challenges, even more difficult ones, along with the ones we have completed.

{:refdef: style="text-align: center;"}
[![Scoreboard](/assets/owaspjuiceshopguided/21_scoreboard.png)](/assets/owaspjuiceshopguided/21_scoreboard.png){:target="_blank"}
{:refdef}