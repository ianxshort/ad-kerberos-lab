## Target: DC01 (self-built)
## Platform: Independent lab, Azure (personal PAYG)
## Date:
## Difficulty: N/A — self-directed
## Tools:

### Environment & Architecture

This lab environment consists of a Active Directory Environment hosted on a 2022 Windows Server in Azure. The result is a publically accessible domain controller secured via a source restricted NSG. 

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



### Building the Domain


### Creating the Vulnerability


### Kali Preparation


### Attack Execution


### Network Troubleshooting (445/5985)


### Privilege Boundary Finding


