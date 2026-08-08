## Target: DC01 (self-built)
## Platform: Independent lab, Azure (personal PAYG)
## Date:
## Difficulty: N/A — self-directed
## Tools:

### Environment & Architecture

This lab environment consists of a Active Directory Domain hosted on a 2022 Windows Server Virtual Machine in Microsoft Azure. The result is a publically accessible domain controller secured via a source restricted NSG. 

This section covers the details of the AD lab infastructure: 

Resource Group: rg-ad-lab, Region: East US (worked on first attempt, no fallback needed)

│

├── Virtual Network (vnet-ad-lab, 10.0.0.0/24)

│   └── Subnet "default" (10.0.0.0/24)

│       └── Windows Server 2022 Datacenter: Azure Edition (Gen2) VM — "DC01"

│              ├── Size: Standard_D2as_v7 (AMD, 2 vCPU, 8GB RAM)

│              ├── Private IP: 10.0.0.4 (static within 10.0.0.0/24)

│              ├── Public IP: DC01-public-ip, Standard SKU, static

│              │         (confirmed unchanged across full deallocate/restart cycle:

│              │         57.162.245.77)

│              ├── Security type: Trusted launch (secure boot + vTPM on)

│              └── NSG (DC01-nsg): inbound locked to home public IP only,

│                     8 custom rules (see Phase 4 — WinRM rule added later,

│                     not in original plan)

│

macOS Host (2023 MacBook Pro, Apple Silicon, 8GB RAM)

│

└── UTM (Kali-AD-Lab)

    └── Kali Linux — ARM64 build (native, no TCG emulation)

#### Firewall Rules 

To ensure the virtual network was reachable only from my machine, the NSG was configured with source-restricted inbound rules. Each rule permitted to a specific service port, with inbound access restricted only to my home IP. All other inbound traffic was denied by default.

![Initial-Firewall-Rules](../images/azure-lab/firewall-rules-redacted.jpeg)


#### Infrastructure Provisioning Challenges 

This Azure build was only reached after two earlier infrastructure build implementations failed. (1) Azure Students free tier - Failed after student account creation restrictions. (2) Local UTM - My Mac runs ARM64 architecture and Windows Server is built natively for x86_64 architecture. This added an emulation layer dedicated to translating instructions from one architecture to another in real time. This additional translation layer is inherently slow and fragile by nature, and it produced two separate failures across two different configuration attempts (Q35 chipset + UEFI firmware, i440FX chipset + legacy BIOS)

---

### Building the Domain

To build the active Directory environment we began by promoting the virtual machine to a domain controller 
forest name: `ad.lab`
DC name: `DC01`

Verified successful domain creation on Powershell running `Get-ADDomain`

![Get-ADDomain](../images/azure-lab/domain-info.jpeg)

Ran diagnostics on the domain controller and encountered `SYSVOL replication problems` which pointed to a test called `DFSREVENT` that failed. I investigated the event logs to pull the log entires associated with that failure. The first entry was an error stating the DFS replication service had failed to contact Active Directory. The second entry was a warning stating the configuration failed to update. I checked the SMB shares using the `Get-SMBShare` command and found the `SYSVOL` folder there and shared correctly.

![Diagnostics-Troubleshoot](../images/azure-lab/dcdiag-troubleshoot.jpeg)


#### User Account Creation

Three user accounts `alice`, `svc_backup`, and `bob` were created to represent target users in the forthcoming attack. 

![User-Creation](../images/azure-lab/domain-users.jpeg)

`alice`: A standard already-compromised low-privilege user who was used to initiate the kerberoasting attack

`svc_backup`: The SPN-holding account representing a fictional back up service and target of the Kerberoasting attack. 

**NOTE**
Account configured with intentionally wordlist-crackable password `sunshine1!` (present in `rockyou.txt`) that satisfies AD default complexity.

`bob`: Roleless account created to represent the existence of  irrelevant accounts in any attack path



---

### Creating the Vulnerability

Began by registering an SPN, mapping `svc_backup` to the fictional `backup-agent` service running on the domain controller (`dc01.ad.lab`).

Completed registration and verified successful verification using `setspn` commands.

![Set-Spn](../images/azure-lab/setspn.jpeg)

By registering `svc_backup` it transformed the account from a inactive account to a Kerberoastable one.




### Kali Preparation

With the target environment configured, I prepared the Kali Linux system that would be used to perform the attack. 
I started by installing the necessary tools 

```bash
sudo apt install -y krb5-user hashcat smbclient ldap-utils
```

DNS was configured via NetworkManager to avoid silent overwrites on `/etc/resolv.conf`. 

```bash
sudo nmcli con mod "Wired connection 1" ipv4.dns "57.162.245.77"
sudo nmcli con mod "Wired connection 1" ipv4.ignore-auto-dns yes
sudo nmcli con up "Wired connection 1”
```

The first command ensures that whenever Kali needs to resolve names it asks our domain controller instead of a public DNS server. The second command ensures that DHCP doesn't overwrite our manually configured DNS server. The third command applies the changes by reconnecting the network connection using the new settings.

Verified with: 

```bash
dig ad.lab @57.162.245.77
```


#### Establishing a Diagnostic Baseline

Before using Impacket for Kerberos Authentication I wanted to establish a diagnostic baseline using native Kerberos tooling. This way, if Impacket's Kerberos implementation failed I would know that the problem was specific to Impacket rather than the environment.

In order to interact with the domain controller's Kerberos service the Kali machine needs the necessary tools. `krb5-user` installs MIT Keberos client tools, which we can use to communicate with the DC01's domain controler. In our case, the tools `kinit` and `klist` allow us to request a Ticket Granting Ticket (TGT) and view the ticket respectively. 

After installing `krb5-user` I examined the contents of `/etc/krb5.conf`, a configuration file that maps a Kerberos realm name to it's authoritative KDC. A Kerberos Realm is an administrative boundary that defines the scope of authority of a KDC. The file was populated with stock MIT Kerberos defaults, such as (`Athena.MIT.EDU`). I deleted all of the irrelevant entries, definied the realm name, and mapped the KDC to my domain controller's IP address. 

![rewrite-stock](../images/azure-lab/manual-rewrite.jpeg)

With `krb5.conf` corrected, I requesting a TGT using Alice's account and the `kinit` tool. Using Alice's known password I successfully authenticated. To confirm the existence of the TGT I used the `klist` command which displays all active Kerberos tickets for the current user session. `klist` displayed a valid `krbtgt/AD.LAD@AD.LAB` entry. This confirmed not only successful DNS resolution and network reachability but also provided credentials. 
 

![kinit](../images/azure-lab/kinit.jpeg)

> It is worth noting that this TGT played no part in the actual kerberoasting attack - `GetUserSPNs.py` performed its own independant authentication using Alice's password directly.


### Attack Execution

After establishing a diagnostic baseline it was time to perform the extraction command using Impacket's `GetUserSPNs.py`. This tool 


![GetUserSPNS](../images/azure-lab/getuserspns.jpeg)




### Network Troubleshooting (445/5985)


### Privilege Boundary Finding


