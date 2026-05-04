# 🛡️ Microsoft Defender SmartScreen with Group Policy Configuration

A hands-on lab walkthrough for configuring **Microsoft Defender SmartScreen** across domain-joined Windows machines using **Active Directory Group Policy** on Microsoft Azure.

---

## 📋 Overview

This project demonstrates how to protect Windows Server environments against phishing and malware from websites, applications, and downloaded files by enforcing SmartScreen via Group Policy Objects (GPOs). The policy is applied to all domain-joined machines in an Active Directory environment hosted on Azure.

---

## 🏗️ Architecture

```
Azure Virtual Network (10.0.0.0/24)
├── dc  (Domain Controller VM)  — Private IP: 10.0.0.4 (Static)
│   ├── Active Directory Domain Services (ADDS)
│   ├── DNS Server
│   └── Group Policy Management
└── vm  (Client VM)
    ├── Joined to: awesome.com domain
    └── SmartScreen enforced via GPO
```

---

## ✅ Prerequisites

- An active **Microsoft Azure** subscription
- Two provisioned Azure Virtual Machines (`dc` and `vm`) running **Windows Server 2022 Datacenter**
- Remote Desktop access to both VMs
- Basic familiarity with **PowerShell** and **Active Directory**

---

## 🚀 Getting Started

### 1. Configure the Domain Controller VM

#### Set the Private IP Address to Static

1. In the Azure portal, open the **dc** virtual machine.
2. Navigate to **Settings → Networking** and click the **dc-NetworkInterface** link.
3. Under **Settings**, click **IP configurations**.
4. Confirm the private IP is `10.0.0.4` and set to **Static**. If not, toggle the Assignment field to **Static** and save.

---

### 2. Install Active Directory Domain Services

On the `dc` VM, open **Windows PowerShell ISE** and run the following script:

```powershell
# Install AD DS and configure forest
$ProgressPreference = "SilentlyContinue"
$WarningPreference = "SilentlyContinue"
Install-WindowsFeature "AD-Domain-Services" -IncludeManagementTools | Out-Null
$pw = ConvertTo-SecureString "p@55w0rd" -AsPlainText -Force
Install-ADDSForest -DomainName "awesome.com" -SafeModeAdministratorPassword $pw `
  -DomainNetBIOSName 'CORP' -InstallDns -Force -NoRebootOnCompletion
```

> **Note:** Server Manager launches automatically on VM connection. After the script completes, restart the VM from the Azure portal.

---

### 3. Create a Domain Admin User

After the `dc` VM restarts, re-connect via Remote Desktop and run:

```powershell
# Create Domain Admin
$pw = ConvertTo-SecureString "p@55w0rd" -AsPlainText -Force
New-ADUser -Name "awesomeadmin" -Description "lab domain admin" -Enabled $true -AccountPassword $pw
Add-ADGroupMember -Identity "Domain Admins" -Members awesomeadmin
```

---

### 4. Configure DNS for the Client VM

1. In the Azure portal, open the **vm** virtual machine.
2. Navigate to **Settings → Networking → vm-NetworkInterface → DNS servers**.
3. Set the DNS server to **Custom** and enter `10.0.0.4` (the domain controller's private IP).

---

### 5. Join the Client VM to the Domain

On the `vm` virtual machine, open **Windows PowerShell ISE** and run:

```powershell
# Join computer to corp.awesome.com domain
$pw = 'p@55w0rd'
$joinCred = New-Object pscredential -ArgumentList ([pscustomobject]@{
    UserName = "CORP\awesomeadmin"
    Password = (ConvertTo-SecureString -String $pw -AsPlainText -Force)[0]
})
Add-Computer -Domain "awesome.com" -Credential $joinCred
```

Restart the `vm` VM from the Azure portal and verify the domain join in **Server Manager → Local Server → Properties**.

---

### 6. Configure Group Policy for SmartScreen

On the `dc` VM:

1. In **Server Manager**, click **Tools → Group Policy Management**.
2. Expand **Forest: awesome.com → Domains → awesome.com**.
3. Click **Action → Create a GPO in this domain, and Link it here…**
4. Name the GPO `smartscreen` and click **OK**.
5. Select the new GPO, click **Action → Edit**.
6. In the Group Policy Management Editor, navigate to:
   ```
   Computer Configuration → Policies → Administrative Templates → Windows Components → File Explorer
   ```
7. Select **Configure Windows Defender SmartScreen**.
8. Click the **Edit policy setting** link.
9. Set the policy to **Enabled**, choose **Warn and prevent bypass**, then click **Apply → OK**.

---

### 7. Confirm SmartScreen Policy Has Been Applied

On the `vm` virtual machine:

1. Open **Command Prompt** and run:
   ```cmd
   gpupdate /force
   ```
2. Open **Windows Security → App & browser control → Reputation-based protection settings**.
3. Confirm that **Check apps and files** is turned **On** and **greyed out** (managed by administrator).

This confirms that the Group Policy has been successfully applied and SmartScreen is enforced domain-wide. ✅

---

## 📁 Project Structure

```
├── README.md                  # This file
└── scripts/
    ├── install-adds.ps1       # Install AD Domain Services & configure forest
    ├── create-domain-admin.ps1 # Create domain admin user
    └── join-domain.ps1        # Join client VM to domain
```

---

## 🔐 Security Notes

- The passwords used in this lab (`p@55w0rd`) are for **demonstration purposes only**. Always use strong, unique passwords in production environments.
- SmartScreen with **Warn and prevent bypass** is the most restrictive setting — users cannot override warnings.
- GPO changes propagate automatically to all domain-joined machines; `gpupdate /force` manually triggers immediate application.

---

## 🧰 Technologies Used

| Technology | Purpose |
|---|---|
| Microsoft Azure | Cloud infrastructure hosting |
| Windows Server 2019 | Operating system for both VMs |
| Active Directory Domain Services | Domain management |
| Group Policy Management | Policy enforcement |
| Microsoft Defender SmartScreen | Phishing & malware protection |
| Windows PowerShell ISE | Automation & scripting |

---

## 👤 Author

**Gerald Mwanza**

---

## 📄 License

This project is intended for educational and lab purposes.
