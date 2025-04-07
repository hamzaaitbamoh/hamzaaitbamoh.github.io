---
layout: post
title: " Domain controller basics | Active Directory | Penetration Testing:"
subtitle: "Understand Domain controller basics in an active directory environment"
date: 2023-04-05 00:56:13 -0400
background: "/img/posts/domsec.png"
---

# Active Directory Part 1 | Domain controller:

The Domain Controller is the main computer in charge of managing a group of computers. It keeps a record of all the important information about the computers and users in the group. This computer is usually a Windows Server. The information is kept in the file C:\Windows\NTDS\ntds.dit , which is stored on the Domain Controller. It's important to keep this file safe and only allow access to the domain administrators, because if someone else gets access to it, they could see all the all the information about the objects of the domain (computers, users, group, policies, etc)

Usually, in a domain there is more than one Domain Controller, in order to distribute the workload and prevent single point of failures. Additionally, as any other database server, Domain Controllers must be synchronized with each other to keep the data up to date.

Moreover, in order to allow computers and users to access the database data, the Domain Controllers provides a series of services like DNS, Kerberos, LDAP, SMB, RPC, etc.

As a pentester, my top target is domain controllers. It's important to identify them, but it's not too hard to do, I can do it from any user that’s a member of the domain.

## How to identify a domain controller from other machines ?

dns query⇒

![Untitled](/img/posts/AD1/Untitled.png)

nktest⇒

![Untitled](/img/posts/AD1/Untitled%201.png)

Or using nmap and it looks something like this :

![Untitled](/img/posts/AD1/Untitled%202.png)

## What’s the most sensitive data that holds a domain controller?

Let’s suppose I got access to one of the administrators of the domain and I want to dump the contents of the domain controller database in order to read some sensitive, such as krbtgt in order to create golden ticket.

There are many ways to do that

you can log in on the domain controller and [dumping the NTDS.dit file](https://www.ired.team/offensive-security/credential-access-and-credential-dumping/ntds.dit-enumeration#no-credentials-ntdsutil) locally with [ntdsutil](<https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc753343(v=ws.11)>) or [vssadmin](https://docs.microsoft.com/en-gb/windows-server/administration/windows-commands/vssadmin)

![Untitled](/img/posts/AD1/Untitled%203.png)

![Untitled](/img/posts/AD1/Untitled%204.png)

Or you could perform a remote [dcsync attack](https://adsecurity.org/?p=1729) with the [mimikatz lsadump::dsync](https://github.com/gentilkiwi/mimikatz/wiki/module-~-lsadump#dcsync)
 command or the [impacket secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py) script.

![Untitled](/img/posts/AD1/Untitled%205.png)

All information that’s holding the domain controller about the domain are stored in a database, and as we said the physical location of the databse is C:\Windows\NTDS\ntds.dit

## I want to know more about this database !

And it’s not like any database Oracle,PostgreSQL… nah! it’s base on “Extensible Storage Engine (ESE)” which is an indexed and sequential access method (ISAM) database. It is uses record-oriented database architecture which provides extremely fast access to records.

### Classes:

For example: a class can be the subclass of a parent class, that allows to inherit properties. For example, the Computer class is a subclass of User class, therefore the computer objects can have the same properties of the user objects.

Object classes serves a different purpose for instance : User class, Computer class, Group class…

Classes are children of other classes, thus they inherit properties.

All the classes are subclasses of the [Top](https://docs.microsoft.com/en-us/windows/win32/adschema/c-top) class.

For example, the Computer class is a subclass of User class, therefore the computer objects can have the same properties of the user objects.

![Untitled](/img/posts/AD1/Untitled%206.png)

Each Class contains several properties. Those properties are storing a string value like `Name` .

### Properties:

There are some properties that you cannot read like [UserPassword](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/f3adda9f-89e1-4340-a3f2-1f0a6249f1f8) and [UnicodePwd](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-ada3/71e64720-be27-463f-9cc5-117f4bc849e1), you haveonly write rights because they will be written when changing passwords.

Moreover, there are that are only retrieved by authorized users because they are have the 128 flag in the SearchFlags of their definition and indicates that the attribute is "not returned by the global catalog." , therefore we can say confidential.

To read a confidential property you are required to have control access right over that specific property.

![Untitled](/img/posts/AD1/Untitled%207.png)

![Untitled](/img/posts/AD1/Untitled%208.png)

### Principals:

In Active Directory, a principal refers to a security principal, which is an object that represents a user, group, computer, or service account that is authenticated by the Active Directory domain.

to identify a principal, each one is assigned to **SID (security identifier).**

The first command is as I said to retrieve that principal’s SID (user’s name).

the second command is to retrieve the domain’s SID

Compare the two and notice that the `user SID = Domain SID + RID (Relative Identifier)`

![Untitled](/img/posts/AD1/Untitled%209.png)

Note that any domain user will have `S-1-5-21` at the start of their SID.

and any administrator will have `-500` as RID.

![Untitled](/img/posts/AD1/Untitled%2010.png)

### Distinguished Name:

It’s also very important to know how to read the **DistinguishedName property!!** because \*\*\*\*It is frequently used to identify objects in the database.

![Untitled](/img/posts/AD1/Untitled%2011.png)

When dealing with the Distinguished Name (DN) in Active Directory, it's important to know that it's read from the end to the start. This means that the most specific component of the DN is listed first,

In the above example: _CN=DC01,OU=Domain Controllers, DC=hamza, DC=lab_

The way I look at it like this _hamza.lab/Domain Controllers/DC01_

![Untitled](/img/posts/AD1/Untitled%2012.png)

### LDAP/LDAPS:

We can interact with the database via one of the following protocols **:** 389/tcp **ldap** **or 636/ssl **ldaps.\*\*

LDAP defines a query syntax that allows you to filter the objects that you want retrieve/edit of the database, and also to specify the properties you would like to retrieve for each object.

- **`&`** (ampersand) denotes logical AND operator, it means that both conditions must be met for the entry to be returned.
- **`objectclass=group`** matches all entries that have an object class of "group".
- **`members=*`** matches all entries that have at least one value for the "members" attribute.
- At the end I specified the propertiy `samaccountname`

![Untitled](/img/posts/AD1/Untitled%2013.png)

Windows tools like Powerview or ADExplorer use LDAP, and if you don't have these tools, you can always use Powershell to query LDAP using .NET.

Similarly, on Linux, you can use ldapsearch and ldapmodify tools to retrieve or modify objects in the Active Directory. When you want to fetch information from the Active Directory, LDAP should be the first protocol to come to your mind.

LDAP can also be used to modify objects. For instance, if you need to add a user to a group, LDAP can be used for this purpose.
