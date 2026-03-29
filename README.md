# Offensive Security Home Lab Setup

## Overview

This lab serves as the foundation for hands-on offensive security 
practice including network reconnaissance, exploitation, and credential 
harvesting simulations, etc...performed in an air-gapped virtual 
environment with no impact to production systems or real networks.

---

## Objective

Build an isolated attacker/target lab environment that allows:
- Bidirectional communication between attacker and target VMs
- Outbound internet access from both VMs
- Host-to-VM access via port forwarding
---
## Environment

| Component        | Details                        |
|------------------|--------------------------------|
| Hypervisor       | Oracle VirtualBox              |
| Attacker Machine | Kali Linux                     |
| Target Machine   | Windows 11                     |
| Network Type     | VirtualBox NAT Network         |
| Subnet           | 10.0.2.0/24                    |

---
## Lab Architecture
```
┌─────────────────────────────────────┐
│         VirtualBox Host             │
│                                     │
│  ┌─────────────┐  ┌──────────────┐  │
│  │  Kali Linux │  │  Windows 11  │  │
│  │ 10.0.2.15   │  │  10.0.2.3    │  │
│  └──────┬──────┘  └──────┬───────┘  │
│         │                │          │
│         └────────────────┘          │
│              NAT Network            │
│            10.0.2.0/24              │
│                  │                  │
│             10.0.2.1                │
│           (VBox Gateway)            │
└─────────────────────────────────────┘
                   │
            Internet Access
            (via host NAT)
```
---

## Part 1: Install VirtualBox

1. Go to [virtualbox.org](https://www.virtualbox.org) and download the 
   installer for your host OS (Windows, macOS, or Linux)
2. Run the installer and accept defaults
3. Also download and install the **VirtualBox Extension Pack** from the 
   same page — this adds USB 2.0/3.0 support and other quality-of-life 
   features
4. Launch VirtualBox before proceeding

---

## Part 2: Set Up the Kali Linux VM

### Download Kali

1. Go to [kali.org/get-kali](https://www.kali.org/get-kali/)
2. Select **Virtual Machines** → download the **VirtualBox** pre-built 
   image (`.ova` or `.vbox` format)
   - This is the fastest option — Kali is pre-installed and ready to import
   - Avoid the ISO unless you want to go through a manual install

### Import into VirtualBox

1. In VirtualBox, go to **File → Import Appliance**
2. Browse to the downloaded `.ova` file and click **Next**
3. Review the VM settings (defaults are fine) and click **Finish**
4. Wait for the import to complete — the Kali VM will appear in your 
   VM list

### Default Credentials

The pre-built Kali image uses:
- **Username:** `kali`
- **Password:** `kali`

Change these after first boot.

---

## Part 3: Set Up the Windows 11 VM

### Download Windows 11

Microsoft provides a free evaluation ISO for Windows 11 Enterprise:

1. Go to 
   [microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise)
2. Fill out the form and select **ISO — Enterprise** as the download format
3. Select your language and download the ISO

> **Note:** This is a 90-day evaluation license. It is sufficient for 
> lab use. No product key is required during installation — skip that 
> screen when prompted.

### Create the VM in VirtualBox

1. In VirtualBox, click **New**
2. Configure:
   - **Name:** `Windows 11`
   - **Type:** Microsoft Windows
   - **Version:** Windows 11 (64-bit)
3. Click **Next** and set:
   - **RAM:** 4096 MB (4GB minimum, 8GB recommended)
   - **CPU:** 2 cores minimum
4. Create a virtual hard disk:
   - **Size:** 60 GB minimum
   - **Type:** VDI, dynamically allocated
5. Click **Finish**

### Mount the ISO and Install

1. Select the Windows 11 VM → **Settings → Storage**
2. Click the empty optical drive → click the disc icon → 
   **Choose a disk file**
3. Browse to your downloaded Windows 11 ISO and select it
4. Click **OK** and start the VM
5. Follow the Windows installer:
   - Select language and region → **Next**
   - Click **Install Now**
   - When prompted for a product key, click **I don't have a product key**
   - Select **Windows 11 Enterprise** as the edition
   - Accept the license terms
   - Choose **Custom: Install Windows only**
   - Select the unallocated virtual disk → **Next**
   - Wait for installation to complete (~15–20 minutes)
6. On the OOBE setup screen:
   - When asked to connect to a network, click 
     **I don't have internet** (bottom left) to skip Microsoft account 
     requirement and create a local account instead
   - Set a username and password

### Install VirtualBox Guest Additions

Guest Additions improves display resolution, clipboard sharing, and 
performance:

1. Boot into Windows 11
2. In the VirtualBox menu bar: **Devices → Insert Guest Additions CD Image**
3. Open File Explorer inside the VM → navigate to the CD drive
4. Run `VBoxWindowsAdditions.exe` and follow the prompts
5. Reboot when prompted

---

## Part 4: Configure the NAT Network

### Why NAT Network Instead of NAT

VirtualBox offers two similar-sounding network modes that behave very 
differently:

**NAT** gives each VM its own isolated NAT instance. The VM can reach 
the internet, but has no layer 2 visibility to other VMs. ARP broadcasts 
do not cross between VMs in this mode.

**NAT Network** places multiple VMs behind a shared NAT instance on the 
same virtual subnet. VMs can communicate with each other directly and 
still reach the internet through the host.

For an attacker/target lab, NAT Network is required. Using plain NAT on 
either VM breaks VM-to-VM communication at the ARP layer — a subtle 
misconfiguration that is not immediately obvious from IP configuration 
alone.

### Create the NAT Network

1. In VirtualBox, navigate to **File → Tools → Network Manager**
2. Select the **NAT Networks** tab
3. Click **Create**
4. Configure:
   - **Name:** `NatNetwork`
   - **IPv4 Prefix:** `10.0.2.0/24`
   - **DHCP:** Enabled
5. Click **Apply**

### Attach Both VMs to the NAT Network

For each VM (Kali and Windows 11):

1. **Settings → Network → Adapter 1**
2. Set **Attached to:** `NAT Network`
3. Set **Name:** `NatNetwork`
4. Click **OK**

> **Important:** Both VMs must be attached to the same named NAT Network. 
> Attaching one VM to `NAT` and the other to `NAT Network`, or attaching 
> them to two differently named NAT Networks, will result in no layer 2 
> connectivity between them even if they appear to be on the same subnet.

### Configure Port Forwarding

Port forwarding on a NAT Network is managed at the network level rather 
than per VM:

1. **File → Tools → Network Manager → NAT Networks**
2. Select the network → **Port Forwarding**
3. Add rules as needed, specifying the guest IP for each target VM

Example rule to SSH into Kali from the host:

| Name     | Protocol | Host IP   | Host Port | Guest IP   | Guest Port |
|----------|----------|-----------|-----------|------------|------------|
| Kali SSH | TCP      | 127.0.0.1 | 2222      | 10.0.2.15  | 22         |

---

## Part 5: Verify Connectivity

### Confirm IP Addresses

**On Kali** (`ip a`):
```
eth0: 10.0.2.15/24
```

**On Windows 11** (`ipconfig`):
```
IPv4 Address: 10.0.2.3
Subnet Mask:  255.255.255.0
Default Gateway: 10.0.2.1
```

Both VMs on `10.0.2.0/24` — confirmed same subnet.

### Run Ping Tests

**Kali → Windows (ping 10.0.2.3):**
```
64 bytes from 10.0.2.3: icmp_seq=1 ttl=255 time=1.53 ms
64 bytes from 10.0.2.3: icmp_seq=2 ttl=255 time=0.400 ms
0% packet loss
```

**Windows → Kali (ping 10.0.2.15):**
```
Reply from 10.0.2.15: bytes=32 time<1ms TTL=64
0% packet loss
```

Bidirectional communication confirmed.

---

## Troubleshooting: ARP Resolution Failure

During initial setup, Kali → Windows ping succeeded but Windows → Kali 
failed with `Destination host unreachable`. Standard firewall 
troubleshooting was ruled out first:
```cmd
netsh advfirewall set allprofiles state off
```

Disabling the Windows firewall had no effect, which shifted focus to 
layer 2.

### Diagnosis

Running `arp -a` on Windows after a failed ping attempt showed that 
`10.0.2.15` never appeared in the ARP cache — Windows was sending ARP 
broadcasts that received no response:
```
Interface: 10.0.2.3 --- 0xa
  Internet Address      Physical Address      Type
  10.0.2.1              52-55-0a-00-02-01     dynamic
  10.0.2.255            ff-ff-ff-ff-ff-ff     static
```

`10.0.2.15` was absent entirely. To confirm Kali was not receiving the 
ARP traffic, `tcpdump` was run on Kali while triggering pings from 
Windows:
```bash
sudo tcpdump -i eth0 arp
```

Result: **0 packets captured.** Kali's `eth0` was not receiving any ARP 
traffic whatsoever, confirming the two VMs were not on the same layer 2 
segment despite having IPs in the same subnet.

---
### Root Cause

Kali was attached to plain **NAT** while Windows was attached to 
**NAT Network**. These are isolated network segments in VirtualBox. 
ARP broadcasts from Windows never reached Kali because they were on 
different virtual switches.

---
### Resolution

Changed Kali's adapter setting from **NAT** to **NAT Network** and 
matched the network name to Windows. After reboot, ARP resolved 
correctly and bidirectional ping succeeded.

---
### Key Takeaway

IP-layer connectivity (same subnet, correct mask) does not guarantee 
layer 2 connectivity. When `Destination host unreachable` originates 
from the sender itself, the issue is ARP failure, not firewall or 
routing. Always verify both VMs are on the identical named network 
in VirtualBox, not just the same network *type*.

---

## Current Lab State

- [x] VirtualBox installed
- [x] Kali Linux VM imported and configured
- [x] Windows 11 VM installed and configured
- [x] NAT Network created
- [x] Both VMs attached to NAT Network
- [x] Bidirectional VM communication verified
- [x] Port forwarding configured
