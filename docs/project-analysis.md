### MITRE ATT&CK Mapping

| Technique | MITRE ID |
| --- | --- |
| AS-REP Roasting | T1558.004 |
| Kerberoasting | T1558.003 |
| Password Cracking | T1110 |
| Network Share Discovery | T1135 |
| OS Credential Dumping: DCSync | T1003.006 |
| Use Alternate Authentication Material: Pass the Hash | T1550.002 |

---
### Detection

#### Kerberoasting 
 
* `Event 4769`, the Kerberos Service Ticket request ID, is generally noisy as it is not uncommon for users to routinely request access to services. However, high volume of 4769 events from a single account within a short window should be flagged. Tools such as `GetUserSPNs.py` query Active Directory via LDAP for SPN-holding accounts, then request a TGS for each in rapid succession, producing a kind of burst pattern.
* Watch for ticket requests using encryption type (RC4-HMAC): Tools will often ask for weaker/legacy encryption types because they are easier to crack offline
> Note this environment defaulted to AES256 which is already a mitigating factor against Kerberoasting's practical impact


#### AS-REP Roasting 

* `Event 4768`, the event ID for Kerberos AS-REQ, should be inspected for a pre-authentication type of `0`. A `Event ID 4768` with `Pre-Auth Type 0` shows that Kerberos preauthentication was not used, which is the necessary condition for AS-REP roasting success. 
* Since tools like `GetNPUsers.py` require only a valid username list to attempt this attack,  `Event 4768`'s `Client Address` field can be used to detect a burst of AS-REQs originating from a single-source IP within a short timeframe. This detection is a stronger signal than a single `pre-auth-type-0` event alone.


#### DCSync 

* `Event 4662` logs object operations and filtering for replication-specific GUID's can indicate a request for directory replication: 
* `{1131f6aa-9c07–11d1-f79f-00c04fc2dcd2} — DS-Replication-Get-Changes`
* `{1131f6ad-9c07–11d1-f79f-00c04fc2dcd2} — DS-Replication-Get-Changes-All`

---

### Mitigation

#### Kerberoasting 

* Fortify Service Account Passwords/Policy 
    * Passwords should be long and complex (>=25 characters, per common industry standards ). Passwords should avoid dictionary-based patterns and not just satisy character-category requirements as seen with `svc_backup` (`sunshine1!`). Although this password met AD's default complexity requirement it was still crackable. It serves as a reminder that simply meeting a complexity policy does not guarantee resistance to dictionary attacks. 
    * Passwords should periodically rotate and expire, as  reduces the window in which a compromised credential reamins valid 

* Enforce Group Managed Service Accounts (gMSA)
    * Group Managed Service Accounts (gMSA) are special domain accounts that manage passwords automatically and work across multiple servers simultaneously. gMSA's is a service account, but instead of human setting a static password that stays the same, Windows generates a cryptographically random, 240-character password that automatically rotates. The hash is computationally infeasible to crack making wordlist or brute-force unrealistic. In this lab, `svc_backup` was deliberately built with the legacy pattern to illustrate this vulnerability, and gMSA serves as the modern counter. 


* Adopt Least Privilege 
    * Service accounts should follow the principle of least privilege that is: a user, program, or system process should only have the bare minimum permissions and access needed to do its job, and nothing more. 
    * Don't assign any service accounts with domain admin or higher level permissions. In our lab, `svc_backup` authenticated successfully but was denied write access to DC01's administrative shares. Because `svc_backup` was never added to the local Administrators group, `psexec.py` was unable to achieve code execution.


#### AS-REP Roasting 

* Enforce Pre-Authentication
    * Ensuring Kerberos pre-authentication remains enabled, its default, is the root cause fix for AS-REP Roasting. The vulnerability exists only on accounts where `DONT_REQ_PREAUTH` has been explicitly set. When an account's pre-authentication is disabled, the Key Distribution Center will respond to any `AS-REQ` with a corresponding `AS-REP` without first verifying the account's actual password. This is exactly what happened on the TryHackMe room: because preauthentication was disabled on `svc-admin`, Kerbrute's `AS-REQ` was blindly met with an `AS-REP`. After receiving the `AS-REP`, Kerbrute extracted and returned the `$krb5asrep$` hash which would later be cracked offline. Tools like `GetNPUsers.py`,  used in the room to verify Kerbrute's finding, can perform the same extraction independently. Without prior knowledge of a valid password `GetNPUsers.py` can target any account with pre-authentication disabled

* Strong Password Policy 
    * Account passwords should evaluted for predicatability. `svc-admin`'s password `management2005` follows a common pattern: a dictionary word paired with a year. This predictable combination makes passwords highly susceptible to wordlist attacks. 

* Least Privilege 
    * Enforcing Least Privilege remains a relevant mitigation mechanism in the event of account compromise
    > See Kerberoasting Mitigations for a fuller discussion

#### DCSync 

* Audit and restrict who holds Replicating Directory Changes/Replicating Directory Changes
    * Any DCSync attack requires that the attacker have permissions `Replicating Directory Changes` and `Replicating Directory Changes All`. Although these permissions are typically restricted to Domain and Enterprise Admins, they can be delegated to standard accounts. In the TryHackMe room `backup` held both rights despite no admin-group membership. After compromising `backup` we were able to use `secretsdump.py` to perform the DCSync attack and retrieve all domain credentials. Directory replication rights should be restricted to domain controllers and a tightly controlled group of users that is regularly audited. 





#### Lessons Learned 



### Future Improvements 