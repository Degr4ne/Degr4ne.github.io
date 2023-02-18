---
title: Active Directory - Basic Concepts
date: 2023-02-18 12:00:00 
categories: [Active Directory]
tags: [active directory]
img_path: /assets/img/AD/
image: 
  path: 00-banner.png
---
It is a `directory service` created by Microsoft that allow us to control and represent a physical network structure, facilitating the configuration and administration of users, computers, servers, and other resources, using for its operation different protocols or services like LDAP, DNS, DHCP, Kerberos, etc.
# Domain Controller(DC)
Domain Controller hosts the server which responds to security authentication requests (log in, checking permissions, etc.) and decide whether allow or deny the access of those, due to its importance is the primary target for attackers.
Also DC is responsible for keep the domain database updated, located in `C:\\Windows\NTDS\ntds.dit` which contains the definition of objects like users, computers, groups, GPO's, etc.
![01-addomain](01-addomain.png)
# Hierarchy structure

- **Object:** Basic unit logic of Active Directory.
- **Organizational units:** Collection of multiple objects, represent the areas of the organisation like marketing, sale, IT.
- **Domain:** Represent an entity, is managed by the domain Controller, a domain is always referred to by its unique name and has a proper domain name structure.
- **Tree:** Represent a series of domains connected together in hierarchical order, it is created when we add a child domain to a parent domain.
- **Forest:** Collection of multiple domain trees that share a common schema and all domains are connected together through a trust relation.

![02-structure](02-structure.png){: width="400" height="100" }
# User Identifiers
## SID
The security identifier is used to identify any entity in the Active Directory, SID's are unique in their domain and are never reused, consists of two parts: The SID of the domain and the relative identifier(RID).
## RID
Is a variable length number added at the final part of the SID which helps to identify a entinty in a domain.

The `DistinguishedName` is used by LDAP to identify objects.
![03-aduser](03-aduser.png)

Each person of the organitation has its own user, also each computer or service of the domain has its own user, because they need to perform their own actions. For example in `Kerberoast attack`,  we aim obtain the TGS, and if we crack it, can get the service user password.

# Group Policies

Group Policy is a mechanism that allow us manage users and computers applying a set of rules.
![04-policy](04-policy.png)
## Group Policy Management Editor

Administrators manage policy settings using the Group Policy Object Editor.

- **Computer configuration:** This type of GPO is applied on a particular computer object, used for example to execute scripts when the computer startup or shutdown, irrespective of the user who logs on.
- **User configuration:** In this case, the policies defined under user configuration defines the user specific system behavior,  it will be applicable on any system where that user logs in.

# SYSVOL
This is one of the most important files in AD, because it stores the Group policy and all the domain access to it. The default location is `%SYSTEMROOT%\SYSVOL\sysvol`.
SYSVOL is made up by the next folders:
-   **Group Policy templates (GPTs)**, which are replicated via SYSVOL replication.
-   **Scripts**, such as startup scripts that are referenced in a GPO.
-   **Junction points**. Junction points work like a shortcut. One directory can point to a different directory.
SYSVOL replication is done by Distributed File System Replication(DFSR) which is the default replication mechanism for SYSVOL folder replication domain-wide.
![05-sysvol](05-sysvol.png)

# Resources
- https://zer1t0.gitlab.io/posts/attacking_ad/
- https://www.youtube.com/watch?v=85-bp7XxWDQ&ab_channel=AndyMaloneMVP
- http://inforleon.blogspot.com/2008/04/active-directory.html
- https://www.wanstor.com/documents/white-papers/whitepaper-wanstor-manageengine-activedirectory-quick-guide-for-IT-professionals.pdf
- https://rootdse.org/posts/active-directory-basics-1/
