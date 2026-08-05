## Target: Attacktive Directory (Windows Active Directory Domain)
## Platform: TryHackMe
## Date: 
## Difficulty: Medium
## Tools: 
- Nmap
- enum4linux-ng
- Kerbrute
- Impacket
    - GetNPUsers.py
    - smbclient.py
    - secretsdump.py
- Evil-WinRM
- Hashcat

### Unauthenticated Enumeration

We begin by enumerating the target machine using `nmap`, a network scanning utility used to detect open ports and, with the `-sV` flag, the services running on those ports 


![nmap-scan](../images/attacktive-directory/nmap-scan.jpeg)

The scan returned several notable ports and services, namely ldap and Kerberos, which heavily indicate the target machine is a domain controller. The `-sC` flag, which queries services for additional information using NSE scripts, gathered important metadata. The scripts revealed the NetBIOS domain name (`THM-AD`) and DNS domain name (`spookysec.local`), both of which proved useful during subsequent enumeration.

We then performed further enumeration using enum4linux-ng, a common tool used to gather details from Windows and Samba system over SMB, NetBIOS and LDAP.

![enum4linux-ng](../images/attacktive-directory/enum4linux.jpeg)

The tool provided further confirmation that the target machine was a Domain Controller


#### Enumeration Using Kerberos 




### Exploitation (AS-REP Roasting)



### Authenticated Enumeration


### Privilege Escalation


### Post Exploitation (Flag Capture)









