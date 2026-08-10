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

* Fortify Service Account Passwords/Policy 
    * Passwords should be long and complex (>=25 characters, per common industry standards ). Passwords should avoid dictionary-based patterns and not just satisy character-category requirements as seen with `svc_backup` (`sunshine1!`). Although this password met AD's default complexity requirement it was still crackable. It serves as a reminder that simply meeting a complexity policy does not guarantee resistance to dictionary attacks. 
    * Passwords should periodically rotate and expire, as  reduces the window in which a compromised credential reamins valid 


* Monitor Active Directory 
    * `Event 4769`, the Kerberos Service Ticket request ID, is generally noisy as it is not uncommon for users to routinely request access to services. However, high volume of 4769 events from a single account within a short window should be flagged. Tools such as `GetUserSPNs.py` query Active Directory via LDAP for SPN-holding accounts, then request a TGS for each in rapid succession, producing a kind of burst pattern.
    * Watch for ticket requests using encryption type (RC4-HMAC): Tools will often ask for weaker/legacy encryption types because they are easier to crack offline
    > Note this environment defaulted to AES256 which is already a mitigating factor against Kerberoasting's practical impact 


* Enforce Group Managed Service Accounts (gMSA)
    * Group Managed Service Accounts (gMSA) are special domain accounts that manage passwords automatically and work across multiple servers simultaneously. gMSA's is a service account, but instead of human setting a static password that stays the same, Windows generates a cryptographically random, 240-character password that automatically rotates. The hash is computationally infeasible to crack making wordlist or brute-force unrealistic. In this lab, `svc_backup` was deliberately built with the legacy pattern to illustrate this vulnerability, and gMSA serves as the modern counter. 


* Adopt Least Privilege 
    * Service accounts should follow the principle of least privilege that is: a user, program, or system process should only have the bare minimum permissions and access needed to do its job, and nothing more. 
    * Don't assign any service accounts with domain admin or higher level permissions. In our lab, `svc_backup` authenticated successfully but was denied write access to DC01's administrative shares. Because `svc_backup` was never added to the local Administrators group, `psexec.py` was unable to achieve code execution.


#### AS-REP Roasting 

* Enforce Pre-Authentication
    * Ensuring Kerberos pre-authentication remains enabled, its default, is the root cause fix for AS-REP Roasting. The vulnerability exists only on accounts where `DONT_REQ_PREAUTH` has been explicitly set. When an account's pre-authentication is disabled, the Key Distribution Center will respond to any `AS-REQ` with a corresponding `AS-REP` without first verifying the account's actual password. This is exactly what happened on the TryHackMe room: because preauthentication was disabled on `svc-admin`, Kerbrute's `AS-REQ` was blindly met with an `AS-REP`. After receiving the `AS-REP`, Kerbrute extracted and returned the `$krb5asrep$` hash which would later be cracked offline. Tools like `GetNPUsers.py`,  used in the room to verify Kerbrute's finding, can perform the same extraction with pre-autentication disabled.

* Strong Password Policy 

* Monitor 

* Least Privilege 


#### DCSync 

* Enforce Kerberos Pre-Authentication


* Strong Password Policy 


* Monitor

* Least Privilege 

 



#### Lessons Learned 



### Future Improvements 