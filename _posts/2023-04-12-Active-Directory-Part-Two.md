---
layout: post
title: " Windows Computer basics | Active Directory | Red Team:"
subtitle: "Understand Windows Computers security"
date: 2023-04-12 00:56:13 -0400
background: "/img/posts/windowssec.png"
---

# Active Directory Part 2 | Windows Computers:

We’ve talked about domain controllers last time, because it would be a top asset to hack as a pentester, move on now to windows computers which we should encounter a lot when doing a pentest on a domain.

## How can I identify windows computers ?

You can enumerate the domain with the tool ldapsearch

![Capture d’écran 2023-04-06 100205.png](/img/posts/AD2/Capture_dcran_2023-04-06_100205.png)

In case, we have no credentials we can scan for open port, but for most windows computer, they have several open ports by default like `Netbios name service` in the port 137 thus it can allow us to even resolve the NetBIOS name from the IP.

I’ll use `nbtscan` for scanning:

![Untitled](/img/posts/AD2/Untitled.png)

SMB also is a way to go, it is used to create shares which can contain valuable information.

![Untitled](/img/posts/AD2/Untitled%201.png)

## How can I connect to windows computers ?

For me, the most important pentesting tool for windows is impacket, it is a collection of python classes and functions working with various windows network protocols.

psexec.py,wmiexec.py and so many use the protocole RPC over SMB.

Furthermore, those tools perform pass-the-hash attacks easily.

![Untitled](/img/posts/AD2/Untitled%202.png)

This way we are using NTLM authentication mechanism, but in Active directory kerberos is used by default.

To use Kerberos you need to provide a Kerberos ticket to impacket. you can just set a ccache file to being used by impacket.

In order to get a kerberos ticket to use, you can request one by using the user password, the NT hash (Overpass-the-Hash) or the Kerberos keys (Pass-The-Key).

![Untitled](/img/posts/AD2/Untitled%203.png)

## LSASS.exe credentials :

The LSASS (Local Security Authority Subsystem Service) process (lsass.exe) is a common location on a Windows machine for storing credentials. This process manages the security-related operations of the computer, including user authentication.

When a user logs in interactively to a computer, either by physically accessing the computer or via RDP, their credentials are cached in the LSASS process. This caching allows for Single Sign-On (SSO) when network logon is required to access other domain computers.

LSASS uses various Security Support Providers (SSPs) to cache credentials and provide different authentication methods. These SSPs include:

- Kerberos SSP: responsible for managing Kerberos authentication and storing tickets and Kerberos keys for current logged-in users.
- NTLMSSP or MSV SSP: responsible for handling NTLM authentication and storing NT hashes for current logged-in users. It does not cache the credentials used.
- Digest SSP: responsible for implementing the Digest Access protocol used by HTTP applications. This SSP stores the clear-text user password to calculate the digest. Although password caching is disabled by default since Windows 2008 R2, it can still be enabled by setting the HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest\UseLogonCredential registry entry to 1.

Accessing the LSASS process memory requires SeDebugPrivilege held by administrators. Retrieving cached credentials from LSASS memory can reveal NT hash, Kerberos keys and tickets, and plaintext passwords in older or misconfigured machines.

To extract credentials using mimikatz, several commands can be used to retrieve different secrets from logged-in users:

- sekurlsa::logonpasswords: Extracts NT hashes and passwords.
- sekurlsa::ekeys: Retrieves Kerberos keys.
- sekurlsa::tickets: Gets Kerberos tickets stored on the machine.

![Untitled](/img/posts/AD2/Untitled%204.png)

In case, LSA protection is enabled it can be bypassed using :

[https://github.com/RedCursorSecurityConsulting/PPLKiller](https://github.com/RedCursorSecurityConsulting/PPLKiller)

## Registry credentials:

LSA secrets are a protected storage area for important data used by the Local Security Authority in Windows. They contain various types of sensitive data, including cached domain records, passwords for various services, private user data, and more. Accessing to this data is restricted to the system.

In the disk, the LSA secrets are saved in the `SECURITY [hive](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry-hives)` file, that is encrypted with the BootKey/SysKey (stored in the SYSTEM hive file).

Moreover, in the SECURITY hive file, there are also stored the credentials from the last domain users logged in the machine, known as the Domain cached credentials (DCC). Thus, the computer can authenticate the domain user even if the connection with the domain controllers is lost. This cached credentials are MSCACHEV2/MSCASH hashes, different from the NT hashes, so they cannot be used to perform a Pass-The-Hash, but you can still try to crack them in order to retrieve the user password.

And the other place where there are credentials is the `SAM hive` file, that contains the NT hashes of the local users of the computer. This could be useful since sometimes organizations set the same local Administrator password in the domain computers.

## How to dump those credentials ?

We can dump credentials from SAM and SECURITY hives using mimikatz.

- `lsadump::secrets`: Get the LSA secrets.
- `lsadump::cache`: Retrieve the cached domain logons.
- `lsadump::sam`: Fetch the local account credentials.

But first we need to acquire SYSTEM session executing the following command: `token::elevate` , and anable SeDebugPrivilege `privilege::debug`

![Untitled](/img/posts/AD2/Untitled.jpeg)

![Untitled](/img/posts/AD2/Untitled%201.jpeg)

![Untitled](/img/posts/AD2/Untitled%202.jpeg)

![Untitled](/img/posts/AD2/Untitled%203.jpeg)

Another way to dump those credentials is to simply save those hives that contains credentials with `reg save` command and move them to our attacker machine and dump them using `secretsdump` from impacket.

We will need the **SECURITY** and **SAM** hive files but also **SYSTEM** hive because it contains the system Boot Key (or System Key) that allows to decrypt the SECURITY and SAM hives.

![Untitled](/img/posts/AD2/Untitled%205.png)

After moving them to our kali machine as we said here’s what we got:

![Untitled](/img/posts/AD2/Untitled%206.png)

- The section $MACHINE.ACC contains the computer account password (encoded in hexadecimal), as well the NT hash.
- The DPAPI_SYSTEM section contains the master DPAPI keys of the system. These keys allow to decrypt the user files.
- The Dumping cached domain logon information section contains the Domain Cached Credentials. Let’s crack one of them using hashcat.

As we can see we must use -m 2100 as our hash type is DCC2.

![Untitled](/img/posts/AD2/Untitled%207.png)

![Untitled](/img/posts/AD2/Untitled%208.png)

![Untitled](/img/posts/AD2/Untitled%204.jpeg)
