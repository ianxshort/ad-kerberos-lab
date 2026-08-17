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
> Note that this environment defaulted to AES256 which is already a mitigating factor against Kerberoasting's practical impact


#### AS-REP Roasting 

* `Event 4768`, the event ID for Kerberos AS-REQ, should be inspected for a pre-authentication type of `0`. A `Event ID 4768` with `Pre-Auth Type 0` shows that Kerberos preauthentication was not used, which is the necessary condition for AS-REP roasting success. 
* Since tools like `GetNPUsers.py` require only a valid username list to attempt this attack,  `Event 4768`'s `Client Address` field can be used to detect a burst of AS-REQs originating from a single-source IP within a short timeframe. This detection is a stronger signal than a single `pre-auth-type-0` event alone.


#### DCSync 

* `Event 4662` logs object operations and filtering for replication-specific GUIDs can indicate a request for directory replication: 
* `{1131f6aa-9c07–11d1-f79f-00c04fc2dcd2} — DS-Replication-Get-Changes`
* `{1131f6ad-9c07–11d1-f79f-00c04fc2dcd2} — DS-Replication-Get-Changes-All`

---

### Mitigation

#### Kerberoasting 

* Fortify Service Account Passwords/Policy 
    * Passwords should be long and complex (>=25 characters, per common industry standards ). Passwords should avoid dictionary-based patterns and not just satisy character-category requirements as seen with `svc_backup` (`sunshine1!`). Although this password met AD's default complexity requirement it was still crackable. It serves as a reminder that simply meeting a complexity policy does not guarantee resistance to dictionary attacks. 
    * Passwords should periodically rotate and expire, as this reduces the window in which a compromised credential remains valid 

* Enforce Group Managed Service Accounts (gMSA)
    * Group Managed Service Accounts (gMSA) are special domain accounts that manage passwords automatically and work across multiple servers simultaneously. A gMSA is a service account, but instead of human setting a static password that stays the same, Windows generates a cryptographically random, 240-character password that automatically rotates. The hash is computationally infeasible to crack making wordlist or brute-force unrealistic. In this lab, `svc_backup` was deliberately built with the legacy pattern to illustrate this vulnerability, and gMSA serves as the modern counter. 


* Adopt Least Privilege 
    * Service accounts should follow the principle of least privilege that is: a user, program, or system process should only have the bare minimum permissions and access needed to do its job, and nothing more. 
    * Don't assign any service accounts with domain admin or higher level permissions. In our lab, `svc_backup` authenticated successfully but was denied write access to DC01's administrative shares. Because `svc_backup` was never added to the local Administrators group, `psexec.py` was unable to achieve code execution.


#### AS-REP Roasting 

* Enforce Pre-Authentication
    * Ensuring Kerberos pre-authentication remains enabled, its default, is the root cause fix for AS-REP Roasting. The vulnerability exists only on accounts where `DONT_REQ_PREAUTH` has been explicitly set. When an account's pre-authentication is disabled, the Key Distribution Center will respond to any `AS-REQ` with a corresponding `AS-REP` without first verifying the account's actual password. This is exactly what happened on the TryHackMe room: because preauthentication was disabled on `svc-admin`, Kerbrute's `AS-REQ` was blindly met with an `AS-REP`. After receiving the `AS-REP`, Kerbrute extracted and returned the `$krb5asrep$` hash which would later be cracked offline. Tools like `GetNPUsers.py`,  used in the room to verify Kerbrute's finding, can perform the same extraction independently. Without prior knowledge of a valid password `GetNPUsers.py` can target any account with pre-authentication disabled

* Strong Password Policy 
    * Account passwords should evaluted for predicatability. `svc-admin`'s password `management2005` follows a common pattern: a dictionary word paired with a year. This predictable combination makes passwords highly susceptible to wordlist attacks. 

* Least Privilege 
    * Enforcing least privilege remains a relevant mitigation mechanism in the event of account compromise
    > See Kerberoasting Mitigations for a fuller discussion

#### DCSync 

* Audit and restrict who holds Replicating Directory Changes/Replicating Directory Changes
    * Any DCSync attack requires that the attacker have permissions `Replicating Directory Changes` and `Replicating Directory Changes All`. Although these permissions are typically restricted to Domain and Enterprise Admins, they can be delegated to standard accounts. In the TryHackMe room `backup` held both rights despite no admin-group membership. After compromising `backup` we were able to use `secretsdump.py` to perform the DCSync attack and retrieve all domain credentials. Directory replication rights should be restricted to domain controllers and a tightly controlled group of users that is regularly audited. 





### Lessons Learned 


#### Understanding the Mechanism Behind the Tool

* `psexec.py`
    * Intially, I viewed `psexec.py` as just a tool to achieve remote execution. However, after experiencing multiple failures through connection timeouts, I had to understand what it actually requires. `psexec.py` works by connecting over SMB, meaning anytime access to port 445 is restricted it will ultimately fail. After fixing the SMB issue and successfully authenticating to DC01, I failed to achieve remote code execution using `svc_backup`. Further investigation of `psexec.py` revealed that the tool's actual mechanism: it uploads an executable, registers it as a Windows service, and starts the service. In order to achieve this it requires both write access to an administrative share and the privilege to create and start a Windows service. Understanding this mechanism would've allowed me to compare the privileges of `svc_backup` against the mechanism's requirements, recognize the gap, and pivot to another attack path. 

* `secretsdump.py`
    * Understanding `secretsdump.py`'s mechanism, unlike `psexec.py`, didn't aid the attack itself. The value it provided came after when explaining mitigations. After investigating the tool further, I came to understand that the tool performs a DCSYNC attack by invoking `DRSUAPI`'s replication function `DRSGETNCChanges`. This function is responsible for replicating data between Domain Controllers when any change occurs. Upon further investigation, I realized that Active Directory's actual criteria for granting replication isn't tied to Domain Controller identity, but rather Domain Controller permissions. For instance, the ordinary user account`svc_backup`, was able to successfully use `secretsdump.py` because it held the two required permissions: (1) `Replicating Directory Changes`, (2) `Replicating Directory Changes All`. Understanding the mechanism of `secretsdump.py`, more precisely that it invokes `DRSGetNCChanges` replication function, allowed me to identify the specific permissions to restrict and audit, as opposed to assuming the fix was tied to identity or group membership

---

#### Infrastructure & Virtualization Constraints

* My original plan for this Active Directory project included local hosting for both the attacking machine and the active directory environment. After hours of troubleshooting, it became evidently clear that the architecture mismatch (ARM64 vs. Windows Server's native x86_64) made local hosting impractical. This is because the architecture mismatch required a slow, fragile emulation layer. The added layer coupled with my hardware memory limits (8GB) led to several failures. Eventually I shifted the Active Directory Environment to the cloud. The bigger lesson is recognizing when a failure is a structural mismatch rather than a fixable misconfiguration. Realizing this earlier, and pivoting, could've saved me signifigant amount of time


#### Cross-Verification

* `dcdiag` vs `Get-SmbShare`
    * While running diagnostics on my initial Active Directory Environment, I ran `dcdiag /q`. The diagnostic tool reported a failure, indicating that SYSVOL replication was broken. If I had stopped there instead of investigating further I would've concluded the domain controller had a real, unresolved health problem. Instead, I pulled the log entries,cross checked using `Get-SmbShare`, which confimred that SYSVOL was live and healthy. The lesson here is that tools fail to accruately describe the entire picture and it is worth checking other sources to verify. 
* `Get-NetTCPConnection` vs `netstat`
    * Another instance of this occured when `Get-NetTCPConnection` showed that only an IPv6 listener existed on port 445. The tool indicated that there was no IPv4 listener at all. At the time, I was troubleshooting a network filtering problem. Instead of trusting `Get-NetTCPConnection` and chasing an absent-IPv4-listener theory, I ran a second check using `netstat`. The second check confirmed that both IPv4 and IPv6 listeners were active, showing that `Get-NetTCPConnection` had ultimately under-reported the result. The second check kept the investigation focused on the right path and saved me a lot of time.

---

### Future Improvements 

* Full Domain Compromise 
    * By artifically granting `backup` replication rights I could fully demonstrate the DCSync attack chain.

* BloodHound Enumeration
    * My Privilege Boundary Finding established that `svc_backup` lacked the permissions to properly execute `psexec.py`. Yet, it was never explored whether or not `svc_backup` had any path to further access within the domain, whether it be group memberships, ACLs, or delegation. Using `Bloodhound` against the environment would map these relationships directly, instead of stopping after one manually tested negative result. 

* AS-REP Roasting against `bob`
    * In the Independant Azure lab, one created user, `bob`, served no purpose. The lab started from an already-authenticated position, as I used Alice's known credentials to launch the Kerberoasting request. There was no step within the Azure lab that explored going from zero domain access to an authenticated foothold. In the future, we could close this gap by performing an AS-REP Roasting attack, enabling `DONT_REQ_PREAUTH` on `bob` and running `GetNPUsers.py`.


