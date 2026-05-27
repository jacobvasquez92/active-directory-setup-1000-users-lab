# 🖥️ Active Directory Home Lab — VMware Workstation

> **Set up Active Directory with 1000 users**  
>  **Use this lab in conjunction with [Active Directory Honeypot Lab with IDS and SIEM Monitoring](https://github.com/jacobvasquez92/active-directory-honeypot-lab/blob/main/README.md)**


---

## 📋 Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Prerequisites & Downloads](#2-prerequisites--downloads)
3. [VM 1 — Domain Controller Setup (Windows Server 2022)](#3-vm-1--domain-controller-setup)
4. [Configure Network Adapters](#4-configure-network-adapters)
5. [Install Active Directory Domain Services](#5-install-active-directory-domain-services)
6. [Create a Domain Admin Account](#6-create-a-domain-admin-account)
7. [Install RAS / NAT (Internet Routing)](#7-install-ras--nat)
8. [Configure DHCP Server](#8-configure-dhcp-server)
9. [Bulk Create Users with PowerShell](#9-bulk-create-users-with-powershell)
10. [VM 2 — Windows Client Setup](#10-vm-2--windows-client-setup)
11. [Join Client to the Domain](#11-join-client-to-the-domain)
12. [Post-Lab Admin Tasks](#12-post-lab-admin-tasks)

---

## 1. Overview & Architecture

This lab builds a fully functional Active Directory environment with two VMs communicating over a private internal network — mirroring a real enterprise setup.

```
[ Internet ]
      |
 [ VMware NAT ]
      |
 ┌────────────────────────────────┐
 │   Domain Controller (DC)       │
 │   Windows Server 2022          │
 │   ├─ NIC 1: NAT  (internet)    │
 │   └─ NIC 2: Host-Only (LAN)    │
 │   IP: 172.16.0.1               │
 │   Roles: AD DS, DNS, DHCP, NAT │
 └────────────────────────────────┘
      |  (internal network)
 ┌────────────────────────────────┐
 │   CLIENT1                      │
 │   Windows 10/11 Enterprise     │
 │   NIC: Host-Only               │
 │   IP: 172.16.0.x (via DHCP)   │
 └────────────────────────────────┘
```

**What you'll build:**
- A Windows Server 2022 Domain Controller with AD DS, DNS, DHCP, and NAT
- A Windows 10/11 client joined to the domain
- 1,000+ dummy user accounts auto-created via PowerShell script

---

## 2. Prerequisites & Downloads

### System Requirements
| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 8 GB | 16 GB |
| Disk | 80 GB free | 120 GB+ free |
| CPU | 4 cores | 6+ cores |
| Software | VMware Workstation Player (free) | VMware Workstation Pro |

### Download Links

| File | Source |
|------|--------|
| **VMware Workstation Player** | [broadcom.com](https://www.vmware.com/products/workstation-player.html) |
| **Windows Server 2022 ISO** | [Microsoft Eval Center](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022) |
| **Windows 10/11 Enterprise ISO** | [Microsoft Eval Center](https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise) |
| **PowerShell User Script** | [github.com/joshmadakor1/AD_PS](https://github.com/joshmadakor1/AD_PS) |

> ⚠️ **Note:** The ISOs are several gigabytes — start your downloads first!

---

## 3. VM 1 — Domain Controller Setup

### Create the VM in VMware

1. Open VMware → **Create a New Virtual Machine**
2. Select **"I will install the operating system later"**

   > 💡 *Do NOT use "Installer disc image file" at this step — it can cause boot errors with Server 2022. Attach the ISO after creation.*

3. Guest OS: **Microsoft Windows** → **Windows Server 2022**
4. Name the VM: `DC` (or `Domain-Controller`)
5. Disk size: **60 GB** → Store as a single file
6. Click **"Customize Hardware"** before finishing

### VMware Hardware Settings (DC)

| Setting | Value |
|---------|-------|
| **Memory** | 2–4 GB |
| **Processors** | 1 CPU, 2 Cores |
| **Network Adapter 1** | NAT *(connects to internet)* |
| **Network Adapter 2** | Host-Only *(internal LAN)* |
| **CD/DVD Drive** | Browse → select your Server 2022 ISO |

> 🔄 **VMware vs VirtualBox:** VirtualBox uses "Internal Network" for isolated LAN traffic. In VMware, use **Host-Only** adapter to achieve the same isolated internal network.

Click **Finish**.

### Install Windows Server 2022

1. Power on the VM
2. **Act fast:** When you see `"Press any key to boot from CD/DVD..."` — press a key immediately. You have ~3 seconds.
3. Follow the Windows setup wizard:
   - Select **Windows Server 2022 Standard Evaluation (Desktop Experience)**

     > ⚠️ You MUST choose **Desktop Experience** — without it, there is no GUI.

   - Choose **Custom Install**
   - Select the unallocated disk → **Next**
4. Let the installation complete and set an Administrator password when prompted.

### Install VMware Tools (Guest Additions Equivalent)

VMware Tools improves display resolution, clipboard sharing, and drag-and-drop.

1. Log in as Administrator
2. In VMware menu → **VM** → **Install VMware Tools...**
3. Inside the VM, open File Explorer → the D: drive will show the installer
4. Run **`setup64.exe`** and follow the wizard (choose **Typical** install)
5. **Restart** the VM — the screen resolution will now auto-fit the window

---

## 4. Configure Network Adapters

The DC needs two network adapters with clear roles. We'll identify and rename them first.

### Identify Which Adapter Is Which

1. Open **Network & Internet Settings** → **Change adapter options**
2. You'll see two adapters. To identify each:
   - Double-click an adapter → **Details**
   - If the IP address starts with `169.254.x.x` → this is the **internal (Host-Only)** adapter (no DHCP server assigned it an IP yet)
   - If it has a valid IP (e.g., `192.168.x.x`) → this is the **internet (NAT)** adapter

3. Rename them for clarity:
   - `Ethernet0` → **`INTERNET`**
   - `Ethernet1` → **`INTERNAL`**

### Assign a Static IP to the INTERNAL Adapter

1. Double-click **INTERNAL** → **Properties**
2. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**
3. Enter the following:

   ```
   IP Address:         172.16.0.1
   Subnet Mask:        255.255.255.0
   Default Gateway:    (leave blank)
   
   Preferred DNS:      127.0.0.1    ← DC uses itself as DNS
   Alternate DNS:      8.8.8.8
   ```

4. Click **OK** → **Close**

> 💡 Leave the INTERNET adapter alone — it gets its IP automatically from VMware's NAT DHCP.

---

## 5. Install Active Directory Domain Services

### Add the AD DS Role

1. Open **Server Manager** → **Add Roles and Features**
2. Installation type: **Role-based or feature-based installation**
3. Server selection: choose **your DC** (should be pre-selected)
4. Server Roles: check **Active Directory Domain Services** → **Add Features**
5. Click **Next** through remaining pages → **Install** → **Close**

### Promote the Server to a Domain Controller

After installation, a notification flag appears in Server Manager:

1. Click the flag → **"Promote this server to a domain controller"**
2. Deployment configuration: **Add a new forest**
3. Root domain name: `mydomain.com` *(or any name you prefer)*
4. Set a **DSRM password** (recovery mode — write it down, rarely used in labs)
5. Leave all other settings as defaults → **Next** through each page
6. Click **Install** — the server will automatically restart

After reboot, log in as: `MYDOMAIN\Administrator`

---

## 6. Create a Domain Admin Account

Using the built-in Administrator account directly is poor practice. We'll create a dedicated admin.

### Create an Organizational Unit (OU)

OUs are containers in AD — like folders — that let you apply Group Policies to specific groups of users.

1. Start → **Windows Administrative Tools** → **Active Directory Users and Computers**
   *(Pin this to the taskbar — you'll use it constantly)*
2. Right-click your domain (`mydomain.com`) → **New** → **Organizational Unit**
3. Name it: `_ADMINS`

### Create Your Admin User

1. Right-click **`_ADMINS`** → **New** → **User**
2. Fill in the details:
   - First Name / Last Name: your name
   - User logon name: `a-yourname` *(convention: `a-` prefix for admins)*
3. Set a password → uncheck **"User must change password at next logon"** → check **"Password never expires"** (lab only)
4. Click **Finish**

### Grant Domain Admin Privileges

1. Right-click your new user → **Properties**
2. **Member Of** tab → **Add...**
3. Type `Domain Admins` → **Check Names** → **OK**
4. Click **Apply** → **OK**

### Sign In as Your New Admin

1. Sign out from `Administrator`
2. Select **Other user**
3. Log in as `a-yourname` (or `MYDOMAIN\a-yourname`)

---

## 7. Install RAS / NAT

RAS (Remote Access Service) with NAT allows the Client VM — which only has a Host-Only adapter — to reach the internet through the Domain Controller.

### Add the Remote Access Role

1. Server Manager → **Add Roles and Features**
2. Server Roles: check **Remote Access** → **Next**
3. Role Services: check **Routing** → **Add Features** → **Next** → **Install**

### Enable and Configure NAT

1. Server Manager → **Tools** → **Routing and Remote Access**
2. Right-click your server → **Configure and Enable Routing and Remote Access**
3. Configuration: select **Network Address Translation (NAT)**
4. Network Interface: select the **INTERNET** adapter *(the one with your real IP)*
5. Click **Finish**

> ✅ The server icon should now show a **green arrow**. NAT is active.

---

## 8. Configure DHCP Server

The DHCP server will automatically assign IP addresses to client machines on the internal network (range: `172.16.0.100–200`).

### Add the DHCP Role

1. Server Manager → **Add Roles and Features**
2. Server Roles: check **DHCP Server** → **Add Features** → **Install**

### Configure the DHCP Scope

1. Server Manager → **Tools** → **DHCP**
2. Expand your server → right-click **IPv4** → **New Scope...**

| Field | Value |
|-------|-------|
| **Scope Name** | `172.16.0.100-200` |
| **Start IP** | `172.16.0.100` |
| **End IP** | `172.16.0.200` |
| **Subnet Mask** | `255.255.255.0` (Length: 24) |
| **Exclusions** | *(leave blank)* |
| **Lease Duration** | 8 days (default is fine for lab) |

3. When asked "Configure DHCP options now?" → **Yes**
4. **Default Gateway:** `172.16.0.1` *(the DC's internal IP)*
5. **DNS Server:** leave as default (already configured by AD)
6. Skip WINS → **Yes, activate this scope now** → **Finish**

### Authorize the DHCP Server

1. In the DHCP console, right-click your **server name** → **Authorize**
2. Right-click again → **Refresh**
3. Both **IPv4** and **IPv6** should now show a **green checkmark** ✅

---

## 9. Bulk Create Users with PowerShell

We'll auto-generate ~1,000 user accounts using Josh Madakor's PowerShell script — simulating a real enterprise user directory.

### Prepare the Environment

1. Open **Server Manager** → **Configure this local server**
2. Find **IE Enhanced Security Configuration** → click **On**
3. Turn it **Off** for both Administrators and Users

   > This allows you to download files from Internet Explorer/Edge without constant security warnings.

### Download and Customize the Script

1. Open a browser on the DC and go to: [https://github.com/joshmadakor1/AD_PS](https://github.com/joshmadakor1/AD_PS)
2. Click **Code** → **Download ZIP** → extract it
3. Open the **`names.txt`** file in the extracted folder
4. **Add your own name** at the top of the list and save

   > 📝 You'll use this account to log into CLIENT1 later!

### Run the PowerShell Script

1. Open **Windows PowerShell ISE** as **Administrator**
2. Open the file: `1_CREATE_USERS.ps1`
3. In the terminal pane, first set the execution policy:

   ```powershell
   Set-ExecutionPolicy Unrestricted
   ```

   Select **Yes to All** when prompted.

4. Navigate to the script directory:

   ```powershell
   cd C:\Users\a-yourname\Downloads\AD_PS-master\AD_PS-master
   ```

5. Press **F5** (or click the green **Run** button) to execute the script

Watch the console — you'll see users being created in real time.

### Verify the Users Were Created

1. Open **Active Directory Users and Computers**
2. You should see a new OU called **`_USERS`** populated with ~1,000 accounts

   > 🔑 All auto-created accounts use the default password: **`Password1`**

---

## 10. VM 2 — Windows Client Setup

### Create the Client VM in VMware

1. VMware → **Create a New Virtual Machine**
2. Select **"I will install the operating system later"**
3. Guest OS: **Microsoft Windows** → **Windows 10 / Windows 11**
4. Name the VM: `CLIENT1`
5. Disk: **60 GB**
6. **Customize Hardware:**

| Setting | Value |
|---------|-------|
| **Memory** | 4 GB (give more if possible — Windows 11 is demanding) |
| **Processors** | 1 CPU, 2 Cores |
| **Network Adapter** | **Host-Only** only *(no internet adapter — traffic routes through DC)* |
| **CD/DVD Drive** | Browse → your Windows 10/11 ISO |

> ⚠️ **VMware Note:** CLIENT1 only gets a **Host-Only** adapter. It has no direct internet access — all traffic will be routed through the DC via NAT. This is the correct lab architecture.

### Install Windows 10/11

1. Power on and press a key immediately at the boot prompt
2. Follow the Windows setup:
   - Region, language → **Next**
   - When prompted to sign in with Microsoft account → click **"Sign-in options"**
   - Select **"Domain join instead"** *(allows a local account)*
3. Create a local account (e.g., `User`) — this is just temporary
4. Complete the setup and let Windows finish configuring

### Verify DHCP Is Working

1. After setup, open **Command Prompt** → run:

   ```cmd
   ipconfig
   ```

2. You should see an IP in the range **`172.16.0.100–200`** — assigned by your DC's DHCP server
3. Test internet connectivity:

   ```cmd
   ping google.com
   ```

   > ✅ If you get replies, your NAT routing through the DC is working correctly!

---

## 11. Join Client to the Domain

1. Right-click **Start** → **System**
2. Scroll down → click **"Domain or workgroup"** (or **"Rename this PC (advanced)"**)
3. Under the **Computer Name** tab → click **Change**
4. Computer name: `CLIENT1`
5. Member of: select **Domain** → type `mydomain.com`
6. Click **OK** → enter domain credentials when prompted:
   - Username: `a-yourname`
   - Password: *(your admin password)*
7. You'll see **"Welcome to the mydomain.com domain"** — click **OK**
8. **Restart** the VM

### Log In as a Domain User

After restart:

1. On the login screen, click **Other user**
2. Log in with any account from the bulk-created user list:
   - Username: *(first initial + last name, e.g., `jsmith`)*
   - Password: `Password1`

> 🎉 You're now logged into a domain-joined Windows machine using an Active Directory account!

---

## 12. Post-Lab Admin Tasks

These demonstrate real-world AD administration tasks you can practice.

### Reset a User's Password

From the DC:

1. Open **Active Directory Users and Computers**
2. Find a user in `_USERS` OU
3. Right-click the user → **Reset Password**
4. Enter a new password → **OK**

### Unlock a Locked Account

1. Right-click the user → **Properties**
2. **Account** tab → check **"Unlock account"** → **Apply**

### Create a New Organizational Unit

1. Right-click your domain → **New** → **Organizational Unit**
2. Try creating: `HR`, `IT`, `Sales`
3. Move users into appropriate OUs by dragging or right-clicking → **Move**

### Verify Domain Join from the DC

On the DC, open **DHCP** console → expand **IPv4** → **Address Leases**:

You should see **CLIENT1** listed with its assigned IP address.

---

## 🗂️ Lab Summary

| Component | Details |
|-----------|---------|
| **Hypervisor** | VMware Workstation Pro / Player |
| **Domain Controller OS** | Windows Server 2022 Standard (Desktop Experience) |
| **Client OS** | Windows 10/11 Enterprise |
| **Domain Name** | mydomain.com |
| **DC Internal IP** | 172.16.0.1 |
| **DHCP Range** | 172.16.0.100 – 172.16.0.200 |
| **AD Roles Installed** | AD DS, DNS, DHCP, RAS/NAT |
| **Users Created** | ~1,000 (via PowerShell) |

---

## 🛠️ Troubleshooting

| Issue | Fix |
|-------|-----|
| VM won't boot from ISO | Confirm ISO is attached in VM Settings → CD/DVD drive; check "Connect at power on" |
| Can't press any key fast enough at boot | In VMware, go to VM Settings → Options → Boot → increase boot delay to 5000ms |
| CLIENT1 has no IP address | Ensure the DHCP scope is authorized (green check); confirm Host-Only adapter matches DC |
| Can't ping google.com from CLIENT1 | Verify NAT is enabled on DC; check that INTERNAL adapter on DC is `172.16.0.1` |
| Domain join fails | Double-check DNS on CLIENT1 points to `172.16.0.1`; run `ipconfig /all` to confirm |
| PowerShell script won't run | Run ISE as Administrator; set `Set-ExecutionPolicy Unrestricted` first |

---

## 📚 References

- 🎥 [Josh Madakor — Active Directory Home Lab (YouTube)](https://www.youtube.com/watch?v=MHsI8hJmggI)
- 📁 [PowerShell User Creation Script (GitHub)](https://github.com/joshmadakor1/AD_PS)
- 🖥️ [VMware Workstation Player (Free)](https://www.vmware.com/products/workstation-player.html)
- 🪟 [Windows Server 2022 Evaluation ISO](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- 🪟 [Windows 11 Enterprise Evaluation ISO](https://www.microsoft.com/en-us/evalcenter/download-windows-11-enterprise)

---

*Lab built as part of my cybersecurity home lab portfolio. All VMs run locally in VMware Workstation.*
