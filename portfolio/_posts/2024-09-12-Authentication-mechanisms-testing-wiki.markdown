---
layout: post
title:  "Authentication mechanisms testing wiki"
date:   2023-09-12 14:21:00 +0200
categories: wiki
sticky: true
permalink: /portfolio/wiki/auth
---

* TOC
{:toc}

---

## Password-based authentication

### Username enumeration

Usernames may be enumerable if there are **this username does not exist** type warnings. If a username exists the warning message may refer to the **incorrect password**.

If there is only **one message** that does not specify whether the **username** or the **password** are **invalid**, there may still be **subtle differences** in the messages being displayed.

Also a different **response time** may indicate that a username exists.

### Evading brute-force countermeasures

#### IP-based countermeasures

**IP-based** countermeasures may block an **IP** address after too many invalid attempts. To evade this block an attacker can test whether the **X\-Forwarded\-For HTTP header** is supported. If it is, the attacker can **spoof** their **IP** address to evade the block.

Another way to evade the block is for the attacker to provide **valid credentials** (*i.e.* **their own**) once every **X** attempts, **resetting the counter**.

#### Account locking

Brute-force attacks may be prevented by **locking** an account after too many invalid login attempts. This countermeasure may be prone to **username enumeration**. Furthermore, is the countermeasure is **not properly implemented**, the valid login may not display any **error** despite the block.

## Multi-factor authentication (MFA)

If the **MFA** is poorly implemented the attacker may **circumvent** it by manually accessing other parts of the application.

If the **MFA** is executed in a separate location than the **authentication process** and the account to be authenticated is determined by a **cookie** the attacker may:
1. alter the cookie to trick the application into generating a **valid code** for the **victim**
2. provide **their own credentials** and an **incorrect token**
3. modify the request to **brute-force** the victim's token

## Other authentication mechanisms

### Remember me

If the application provides a **remember me** functionality it may be implemented through the use of a **cookie**. If the cookie's value is **guessable** or **easily predictable**, the attacker may **brute-force** it by studying **their own cookie**. Sometimes the cookie may even reveal a user's **password**.

The cookie may also be stolen via **XSS vulnerabilities**.

When obtained, the **remember me cookie** can be used to be authenticated as **its owner**.

### Password reset

If the **password reset** functionality uses a **URL** that includes a token, its **check** may be **poorly implemented** or **absent**.

If the **X\-Forwarded\-Host** is supported, its value may point to an **attacker-controlled server** the token may also be **stolen** as it would be visible to the **middleware server**.

This functionality may also be vulnerable to **brute-force attacks**.