---
layout: "post"
title: "HTB - Cicada"
description: "Short summary of the post."
date: 2024-12-19
categories: [Write-ups]
tags: [web]
image:
  path: /assets/screenshots/cicada/cicada0.png
  alt: cicada0
---

## Portscan
<img src="/assets/screenshots/cicada/cicada1.png" alt="cicada1" width="600" />
>[!info] Observations :
> - It is a Windows machine.
> - It is an Active Directory (AD), with the domain *cicada.htb*.
> - RPC is enabled.
> - Port 5985 (WinRM) is active.

# Initial Access
## SMB Share: HR
- Guest access is allowed:

<img src="/assets/screenshots/cicada/cicada2.png" alt="cicada2" width="600" />
- We successfully list the shares with guest access. Two shares are found: *DEV* and *HR*. We access the *HR* share and find a file containing a default password for new accounts:

<img src="/assets/screenshots/cicada/cicada3.png" alt="cicada3" width="600" />

>[!done] Password: `Cicada$M6Corpb*@Lp#nZp!8`

- We do not have permission to access the *DEV* share.

## RID Bruteforce
- With a password but no valid usernames, we attempt an RID brute-force as RPC is active. This provides us with a list of users:

<img src="/assets/screenshots/cicada/cicada4.png" alt="cicada4" width="600" />

## Password Spray
- We spray the obtained password across the user list and find that the credentials `michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8` are valid:

<img src="/assets/screenshots/cicada/cicada5.png" alt="cicada5" width="600" />

## LDAP Description
- However, WinRM is not accessible. Using LDAP, we extract the user's description field and find their updated password:

<img src="/assets/screenshots/cicada/cicada6.png" alt="cicada6" width="600" />

>[!done] Creds: `david.orelious:aRt$Lp#7t*VQ!3`

## SMB Share: DEV
- Using these credentials, we access the *DEV* share and discover a PowerShell script containing plaintext credentials for `emily.oscars`:

<img src="/assets/screenshots/cicada/cicada7.png" alt="cicada7" width="600" />

>[!done] Creds: `emily.oscars:Q!3@Lp#M6b*7t*Vt`

## WinRM
- These credentials allow us to connect via WinRM. Using [[Evil-WinRM]], we access the machine and retrieve the user flag:

<img src="/assets/screenshots/cicada/cicada8.png" alt="cicada8" width="600" />

>[!done] User flag: `b4c04e7ba66381c3fe8717ffb0549ef3`

# Privilege Escalation
## SeBackupPrivilege
- Checking privileges with `whoami /all`, we find that the user is part of the `Backup Operators` group and has `SeBackupPrivilege` and `SeRestorePrivilege`:

<img src="/assets/screenshots/cicada/cicada9.png" alt="cicada9" width="600" />

- These privileges allow the user to back up sensitive system files. We exploit this to copy the *SAM* and *SYSTEM* hives and extract their hashes.

### Steps
1. Navigate to a writable directory:
   ```bash
   mkdir c:\users\public\temp
   cd c:\users\public\temp
   ```
2. Dump the hives:
   ```bash
   reg save hklm\sam C:\Users\Public\tmp\sam.hive
   reg save hklm\system C:\Users\Public\tmp\system.hive
   ```
3. Download the files to the attacking machine using Evil-WinRM's `download` function.
4. Extract hashes:
   ```bash
   secretsdump -sam sam.hive -system system.hive LOCAL
   ```

<img src="/assets/screenshots/cicada/cicada10.png" alt="cicada10" width="600" />

- Using the extracted hash, we connect as `Administrator` and retrieve the root flag:

<img src="/assets/screenshots/cicada/cicada11.png" alt="cicada11" width="600" />

>[!done] Root flag: `c04d7ed700525ec7beef05dcb62c4f21`

### Alternative Method for SeBackupPrivilege
- We can use prebuilt DLLs from this [repository](https://github.com/giuliano108/SeBackupPrivilege) that exploit the `SeBackupPrivilege`.
- Steps:
  1. Upload the DLLs to the machine.
  2. Import and load them from a writable directory.
  3. Use `Copy-FileSeBackupPrivilege` to directly copy the flag:

<img src="/assets/screenshots/cicada/cicada12.png" alt="cicada12" width="600" />
