### MITRE ATT&CK Mapping

| Technique | MITRE ID |
| --- | --- |
| AS-REP Roasting | T1558.004 |
| Kerberoasting | T1558.003 |
| Password Cracking | T1110 |
| Network Share Discovery | T1135 |


### Detection


### Mitigation

#### Kerberoasting 

* Foritfy Service Account Passwords/Policy 
    * Passwords should be long and complex (>=25 characters, per common industry standards ). Passwords should avoid dictionary-based patterns and not just satisy character-category requirements as seen with `svc_backup` (`sunshine1!`). Although this password met AD's default complexity requirement it was still crackable. It serves as a reminder that simply meeting a complexity policy does not guarantee resistance to dictionary attacks. 
    * Passwords should periodically rotate and expire, as  reduces the window in which a compromised credential reamins valid 


* Monitor Active Directory 
    * `Event 4769`, the Kerberos Service Ticket request ID, is generally noisy as it is not uncommon for users to routinely request access to services. However, high volume of 4769 events from a single account within a short window should be flagged. Tools such as `GetUserSPNs.py` query Active Directory via LDAP for SPN-holding accounts, then request a TGS for each in rapid succession, producing a kind of burst pattern.
    * Watch for ticket requests using encryption type (RC4-HMAC): Tools will often ask for weaker/legacy encryption types as they are easier to crack offline


* Enforce Group Managed Service Accounts (gMSA)
    * Group Managed Service Accounts (gMSA) are special domain accounts that manage passwords automatically and work across multiple servers simultaneously. gMSA's is still a service account, but instead of human setting a static password that stays the same, Windows generates a cryptographically random, 240-character password that automatically rotates. The hash is computationally infeasible to crack making wordlist or brute-force unrealistic. In this lab, `svc_backup` was deliberately built with the legacy pattern to illustrate this vulnerability, and gMSA serves as the modern counter. 


* Adopt Least Privilege 


#### AS-REP Roasting 


 



#### Lessons Learned 



### Future Improvements 