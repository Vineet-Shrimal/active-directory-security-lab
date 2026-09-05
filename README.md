# Active Directory Security Lab — Authentication Monitoring & Threat Detection 
 
A hands-on home lab simulating a small corporate Active Directory environment, built to practice attack simulation, log collection, and detection engineering using Splunk. This project covers standing up a domain controller, joining a client machine, simulating a real-world brute-force attack, executing MITRE ATT&CK-mapped techniques with Atomic Red Team, and building detections for both in Splunk.
 
---
 
## Objective
 
Most tutorials teach tools in isolation. This lab was built to connect the full loop that a SOC analyst actually works in:
 
1. Stand up a realistic small-business AD environment (domain, OUs, users)
2. Generate real attacker telemetry (brute force + ATT&CK-mapped techniques)
3. Forward that telemetry into a SIEM (Splunk)
4. Write detections and confirm they catch the activity
---
 
## Architecture
 
![AD Project Architecture](images/AD_Project_Architecture.png)
 
| Machine | Role | IP Address | Software Installed |
|---|---|---|---|
| Windows Server 2022 | Domain Controller | 192.168.10.7 | AD DS, DNS, Sysmon, Splunk Universal Forwarder |
| Ubuntu Server | SIEM / Log Indexer | 192.168.10.10 | Splunk Enterprise |
| Windows 10 Pro | Target / Client | DHCP (domain-joined) | Sysmon, Splunk Universal Forwarder, Atomic Red Team |
| Kali Linux | Attacker | 192.168.10.250 | Hydra |
 
**Domain:** `vineet.local`
**Network:** 192.168.10.0/24 (VirtualBox NAT Network)
 
All four machines run as VirtualBox VMs on a single isolated internal network, giving full control over traffic without touching a real production or home network.
 
---
 
## Part 1: Domain Controller Setup
 
The Windows Server 2022 VM was promoted to a domain controller to create a realistic AD environment for authentication testing and log generation.
 
**Steps performed:**
1. Installed the **Active Directory Domain Services (AD DS)** role via Server Manager
2. Promoted the server to a **new forest**, establishing the root domain `vineet.local`
3. Configured DNS as part of the promotion (DC also serves as the domain's DNS server)
4. Verified successful promotion via Server Manager, confirming AD DS and DNS roles active
![AD DS Role Selection](images/image1.png)
![Installation Confirmation](images/image2.png)
![Installation Results](images/image3.png)
![Deployment Configuration - New Forest](images/image4.png)
![Domain Controller Options](images/image5.png)
![Prerequisites Check Passed](images/image6.png)
![AD DS and DNS Active in Server Manager](images/image7.png)
 
### Organizational Structure
 
To simulate a small business, two departmental Organizational Units (OUs) were created in Active Directory Users and Computers:
 
- **IT** — containing user `pparker` (Peter Parker)
- **HR** — containing user `mjane` (Mary Jane)
This mirrors how a real organization segments users by department for permissions and Group Policy scoping.
 
---
 
## Part 2: Domain Join (Windows 10 Client)
 
The Windows 10 VM was joined to the `vineet.local` domain to act as a client/target machine.
 
**Note:** Windows 10 Home does not support domain join — the machine had to be running **Windows 10 Pro** for this step to be available at all.
 
**Steps performed:**
1. Opened System Properties → Change → selected "Domain" and entered `vineet.local`
2. Authenticated with a domain administrator account
3. Confirmed successful join and rebooted
4. Verified on the DC side, in **Active Directory Users and Computers**, that the machine (`TARGET-PC`) now appears under the Computers container — proof of a successful join from the server's perspective
![Domain Join Dialog](images/image8.png)
![Domain Credentials Prompt](images/image9.png)
![System Properties Showing Domain Membership](images/image10.png)
![TARGET-PC Listed in ADUC](images/image11.png)
 
---

## Part 3: Ubuntu Server & Splunk Installation
 
The Ubuntu Server VM was configured as the central Splunk indexer — the machine every other VM in the lab forwards its logs to.
 
**Steps performed:**
1. Installed Ubuntu Server and configured networking with a static IP (`192.168.10.10/24`) via netplan (`/etc/netplan/00-installer-config.yaml`)
2. Verified connectivity with `ip a` and confirmed internet access with `ping google.com`
3. Set up a shared folder between the host machine and the VM (VirtualBox Guest Additions / `vboxsf`) to transfer the Splunk installer package into the VM without needing a separate download inside the VM
4. Installed Splunk Enterprise from the `.deb` package:
```
   sudo dpkg -i splunk-10.4.1-5a009d941268-linux-amd64.deb
```
5. Switched to the `splunk` service user and started Splunk for the first time:
```
   sudo -u splunk bash
   cd /opt/splunk/bin
   ./splunk start
```
6. Splunk completed its first-run setup (generating a self-signed cert, starting `splunkd`) and confirmed the web interface was reachable at `http://splunk:8000`
![Static IP Configuration and Connectivity Check](images/image22.png)
![Internet Connectivity Confirmed](images/image23.png)
![Splunk Package Installation](images/image26.png)
![Splunk First Start — Web Interface Ready](images/image27.png)
 
This gave every other VM in the lab a live, central endpoint (`192.168.10.10:8000`) to forward telemetry to and search against.
 
---
 
## Part 4: Log Collection Pipeline
 
To detect anything happening on the endpoints, telemetry needs to reach Splunk. The pipeline set up was:
 
**Windows Server 2022 & Windows 10:**
- **Sysmon** installed on both — provides much richer process, network, and registry event logging than default Windows Event Logs
- **Splunk Universal Forwarder** installed on both — forwards Sysmon and Windows Security event logs to the central Splunk indexer
**Ubuntu Server:**
- **Splunk Enterprise** installed as the central indexer/search head, listening for forwarded logs from both Windows machines
This mirrors a standard enterprise logging architecture: lightweight forwarders on endpoints, centralized indexing and search on a dedicated log server.
 
---
 
## Part 5: Attack Simulation #1 — RDP Brute Force
 
**Objective:** simulate a credential-based attack against a domain user account and detect it in Splunk.
 
**Tool:** Hydra (switched from Crowbar after diagnosing that Crowbar's RDP module cannot handle the self-signed certificate handshake on modern RDP — it fails silently on every attempt without ever completing a real login. This was confirmed by testing manually with `xfreerdp`, which prompted a certificate trust dialog Crowbar could never answer non-interactively.)
 
**Command used:**
```
hydra -l pparker -P passwords.txt rdp://192.168.10.100
```
 
**Result:** Hydra successfully identified valid credentials — `pparker : Admin@123` — among the tested password list.
 
![Hydra Brute Force Attack](images/image12.png)
 
### Detection in Splunk
 
Two searches were used to correlate the attack with what actually hit the domain controller / target's Security event log:
 
**Failed login attempts (Event ID 4625):**
```
index=endpoint pparker EventCode=4625
```
Returned 20 failed logon events for the `pparker` account in a tight time window — matching the password list Hydra worked through before finding the correct one.
 
![Splunk Search - 20 Failed Logon Events](images/image15.png)
 
**Successful login (Event ID 4624) immediately after:**
```
index=endpoint pparker EventCode=4624
```
Returned a single successful logon event, timestamped right after the failed attempts — the exact "burst of failures, then one success" pattern that signals a successful brute-force attack.
 
![Splunk Search - Successful Logon Event](images/image13.png)
 
**All authentication activity for the account:**
```
index=endpoint pparker
```
![Splunk Search - All pparker Events](images/image14.png)
 
This failed→success pattern (many 4625s clustered in seconds, followed by one 4624 for the same account and source) is the exact signature a SOC analyst would build an alert around in a real environment.
 
---
 
## Part 6: Attack Simulation #2 — Atomic Red Team
 
**Objective:** execute MITRE ATT&CK-mapped adversary techniques directly on the endpoint and confirm the resulting telemetry is captured and detectable in Splunk.
 
**Setup:**
```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing);
Install-AtomicRedTeam -getAtomics
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```
 
![Atomic Red Team Installation](images/image16.png)
 
### Technique 1 — T1136.001: Create a Local Account
 
```powershell
Invoke-AtomicTest T1136.001
```
 
This technique (MITRE ATT&CK **T1136.001 – Create Account: Local Account**) simulates an attacker establishing persistence by creating a new local user and adding it to the local Administrators group — a common step after initial compromise.
 
**Result:** the test successfully created a local admin account (`NewLocalUser`), added it to the Administrators group, then cleaned up by deleting it — all steps completed successfully.
 
![Atomic Test T1136.001 Execution](images/image17.png)
 
**Detection in Splunk:**
```
index=endpoint NewLocalUser
```
Returned 12 matching events, capturing the full account lifecycle (creation, group membership change, deletion) via Windows Security event logs.
 
![Splunk Detection - NewLocalUser Account Activity](images/image18.png)
 
### Technique 2 — T1059.001: PowerShell Execution
 
```powershell
Invoke-AtomicTest T1059.001
```
 
This technique (MITRE ATT&CK **T1059.001 – Command and Scripting Interpreter: PowerShell**) tests multiple sub-techniques including credential dumping tool execution (Mimikatz) and BloodHound reconnaissance via PowerShell.
 
**Result:** the test executed successfully. The screenshot below shows the start of execution — the full run required scrolling beyond what a single screenshot could capture, so only the initial output was captured for documentation.
 
![Atomic Test T1059.001 Execution](images/image20.png)
 
**Detection in Splunk** — even the attempted execution generated detectable PowerShell telemetry via Sysmon:
```
index=endpoint powershell.exe -exec bypass -noprofile
```
Captured the PowerShell process launch with suspicious flags (`-exec bypass -noprofile`) commonly associated with adversary tooling — a strong detection signature regardless of whether the payload itself succeeded.
 
![Splunk Detection - Suspicious PowerShell Execution](images/image21.png)
 
---
 
## Key Troubleshooting & Lessons Learned
 
Real-world lab work rarely goes smoothly, and documenting the problems solved is as valuable as the final result:
 
- **Crowbar vs. Hydra for RDP brute force:** diagnosed that Crowbar's RDP backend cannot pass the self-signed certificate trust prompt non-interactively, causing every attempt to fail silently in under a second. Confirmed via manual `xfreerdp` testing, then switched to Hydra, which handled the target successfully.
- **Windows Firewall blocking ICMP by default:** both Windows 10 and Windows Server 2022 block inbound ping by default on the Domain/Private profiles, which broke basic connectivity testing until the "File and Printer Sharing (Echo Request - ICMPv4-In)" rule was explicitly enabled on both machines.
- **DHCP IP conflicts on VirtualBox NAT Network:** a statically-assigned DC IP collided with a DHCP-issued address for the Windows 10 client, causing intermittent connectivity failures — resolved by assigning static IPs to all lab machines instead of mixing static and DHCP.
- **DNS forwarding on the DC:** the domain controller's DNS server had no forwarder configured for external resolution, causing internal name resolution to work while `google.com` and other external names failed — fixed by adding public DNS forwarders (8.8.8.8) to the DC's DNS server.
- **Windows 10 Home cannot join a domain:** confirmed this is a hard licensing restriction, not a config issue — required reinstalling with Windows 10 Pro.
- **VirtualBox NAT Network instability:** experienced recurring scenarios where multiple VMs simultaneously lost DHCP-assigned connectivity, resolved by restarting VirtualBox's background networking services or, when that failed, a full host reboot.
---
 
## Skills Demonstrated
 
- Active Directory deployment and administration (forest/domain creation, OU structure, user management)
- Windows domain join and client configuration
- Log source configuration: Sysmon + Splunk Universal Forwarder architecture
- SIEM search and detection engineering (SPL queries against Windows Security and Sysmon event data)
- Offensive tooling: Hydra for credential brute-forcing
- MITRE ATT&CK-mapped adversary simulation with Atomic Red Team
- Network troubleshooting in a virtualized lab environment (DHCP, DNS, firewall, VirtualBox networking)
---
 
## Tools Used
 
| Category | Tool |
|---|---|
| Virtualization | Oracle VirtualBox |
| Domain Services | Windows Server 2022 (Active Directory Domain Services, DNS) |
| SIEM | Splunk Enterprise |
| Log Forwarding | Splunk Universal Forwarder |
| Endpoint Telemetry | Sysmon |
| Brute Force Attack | Hydra |
| Adversary Simulation | Atomic Red Team |
| Attacker OS | Kali Linux |
