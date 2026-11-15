# Windows Administration: A Technician's Guide

Windows administration is controlling **who can do what** on a machine, keeping it **patched and healthy**, and knowing **where to look** when it misbehaves. This guide covers the everyday tools, then goes deeper on Remote Desktop, the difference between operator and administrator rights, and the Power Users group.

---

## 1. The 30-Second Mental Model

Every action on Windows is performed by a **security principal** (a user or group) carrying an **access token**. The token lists the user's identity, group memberships and special **privileges**. When a program touches a file, service or registry key, Windows compares the token with that object's **permissions (ACL)** and allows or denies it.

```text
User signs in
   -> Windows builds an access token (SID + group SIDs + privileges)
        -> Program starts with that token
             -> Each file / service / registry key checks its ACL against the token
                  -> Allow or Deny (and write an audit event)
```

Administration is mostly about shaping that token: put people in the **smallest group** that lets them do their job.

---

## 2. How It Works in Practice

Run these from **PowerShell**. Open it as Administrator (right-click, **Run as administrator**) for anything that changes the system.

### 2.1 Users, groups and UAC
- **Standard user:** can run apps and change their own settings. Can't change system-wide settings.
- **Administrator:** full control of the machine.
- **UAC (User Account Control):** even an administrator normally runs with a *filtered* token. Elevating (the shield prompt) switches to the full token for one program.

```powershell
whoami /groups                         # groups in my current token
whoami /priv                           # privileges in my current token
Get-LocalUser                          # local accounts
Get-LocalGroup                         # local groups
Get-LocalGroupMember Administrators    # who is an administrator
New-LocalUser -Name "labuser" -Password (Read-Host -AsSecureString "Password")
Add-LocalGroupMember -Group "Users" -Member "labuser"
```

On a domain, accounts and groups live in **Active Directory** (AD) and are managed with *Active Directory Users and Computers* or the `ActiveDirectory` PowerShell module (`Get-ADUser`, `Get-ADGroupMember`).

### 2.2 Services and processes
A **service** is a background program managed by the Service Control Manager. A **process** is any running program.

```powershell
Get-Service | Where-Object Status -eq 'Running'
Get-Service -Name Spooler | Format-List *
Restart-Service -Name Spooler
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Stop-Process -Id 1234                   # replace with a real PID
```

### 2.3 Event Viewer and logs
**Event Viewer** (`eventvwr.msc`) holds the logs. The three you use most: **System** (drivers, services, boot), **Application**, and **Security** (sign-ins and audited actions).

```powershell
Get-WinEvent -LogName System -MaxEvents 20
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2} -MaxEvents 20   # Level 2 = errors
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10 # failed sign-ins
```

| Event ID | Meaning (Security log) |
|----------|-----------------------|
| 4624 | Successful sign-in |
| 4625 | Failed sign-in |
| 4672 | Special privileges assigned (an administrator-level sign-in) |
| 4720 / 4726 | User account created / deleted |
| 4732 | Member added to a local group |

### 2.4 Files, NTFS permissions and sharing
NTFS permissions are set per file or folder. Common levels: Read, Write, Modify, Full control. Permissions are **inherited** from the parent folder unless inheritance is broken. An explicit **Deny** overrides an Allow.

```powershell
icacls C:\Data                          # show the ACL
icacls C:\Data /grant "labuser:(OI)(CI)M"   # Modify, inherited by files and subfolders
Get-SmbShare                            # list shares
```
Share permissions and NTFS permissions both apply to network access, and the **more restrictive** one wins.

### 2.5 Updates, health and repair
```powershell
Get-HotFix                              # installed updates
sfc /scannow                            # check and repair protected system files
DISM /Online /Cleanup-Image /RestoreHealth   # repair the component store
chkdsk C: /scan                         # online disk check
Get-Volume                              # disk space
```
Order matters: run `DISM` first when `sfc` can't repair files, then run `sfc` again.

### 2.6 Networking basics
```powershell
ipconfig /all
Test-NetConnection www.lukaslim.com -Port 443
Resolve-DnsName www.lukaslim.com
Get-NetTCPConnection -State Listen      # what is listening, equivalent of ss
```

### 2.7 Group Policy (domain environments)
**Group Policy** pushes settings (password rules, firewall, software restrictions, drive maps) to many computers from a central place. Settings are bundled into **GPOs** and linked to AD sites, domains or OUs.

```powershell
gpupdate /force                         # re-apply policy now
gpresult /r                             # which policies applied to me and this PC
gpresult /h report.html                 # detailed HTML report
```
Local policy on a standalone PC: `gpedit.msc` (Pro and above) or `secpol.msc`.

### 2.8 Other everyday tools
| Tool | Command | Use |
|------|---------|-----|
| Task Manager | `taskmgr` | Quick CPU, memory, disk, startup view |
| Services | `services.msc` | Start, stop, set startup type |
| Task Scheduler | `taskschd.msc` | Run jobs on a schedule |
| Computer Management | `compmgmt.msc` | Users, disks, shares, events in one place |
| Disk Management | `diskmgmt.msc` | Partitions and volumes |
| Registry Editor | `regedit` | System configuration database (back up before editing) |
| Performance Monitor | `perfmon` | Counters over time |
| Resource Monitor | `resmon` | Which process uses disk or network |

---

## 2A. Remote Desktop (RDP) Login

**Remote Desktop Protocol** gives you the full graphical session of another Windows machine over **TCP 3389** (and UDP 3389 for better performance).

### What must be true before an RDP login works
1. **The target runs an edition that can host RDP:** Pro, Enterprise, Education or Windows Server. Windows Home can connect *out* but can't be connected *to*.
2. **Remote Desktop is enabled** on the target (Settings → System → Remote Desktop).
3. **The network path is open:** Windows Firewall allows the Remote Desktop rule, and any network firewall allows TCP 3389.
4. **The account is allowed in:** members of **Administrators** can connect by default. Anyone else must be in the **Remote Desktop Users** group.
5. **The account has a password** (blank passwords are blocked for network sign-in by default).

```powershell
# Enable RDP (run as administrator on the TARGET machine)
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Require Network Level Authentication
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name UserAuthentication -Value 1

# Allow a non-administrator to connect
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "labuser"
```

### Connecting
```powershell
mstsc /v:SERVER01                  # open the Remote Desktop client
Test-NetConnection SERVER01 -Port 3389   # is the port reachable?
quser /server:SERVER01             # who is logged on, and session IDs
logoff 2 /server:SERVER01          # end session ID 2
```
Client sign-in format: `SERVER01\labuser` for a local account, or `DOMAIN\labuser` for a domain account.

### Sessions
- **Windows client (10/11):** one interactive user at a time. Connecting while someone is at the console locks their session.
- **Windows Server:** two concurrent administrative sessions by default. More users need Remote Desktop Services (RDS) and client access licenses.
- A **disconnected** session keeps running. A **logged off** session ends and closes its programs.

### Security
RDP is one of the most attacked services on the internet. Treat it carefully.
- **Never expose 3389 directly to the internet.** Use a **VPN**, a **Remote Desktop Gateway**, or a cloud bastion/session manager.
- **Keep Network Level Authentication (NLA) on.** The user proves who they are *before* a desktop session is created, which cuts the attack surface. NLA is not multi-factor authentication.
- Use **MFA** where you can (through the gateway or VPN).
- Use strong passwords and an **account lockout policy** to slow brute-force attempts.
- Put only the people who need it in **Remote Desktop Users**.
- Keep systems patched; serious RDP flaws such as BlueKeep (CVE-2019-0708) were fixed by updates.
- **Monitor:** Security event **4624 with Logon Type 10** is a successful RDP sign-in. **4625** is a failure. Many failures from one source is a brute-force attempt.

### RDP troubleshooting

| Symptom | Likely cause | What to check |
|---------|--------------|---------------|
| "Can't connect to the remote computer" | Port blocked, RDP disabled, machine off | `Test-NetConnection HOST -Port 3389`, firewall, `fDenyTSConnections` |
| "The connection was denied because the user account is not authorized for remote login" | User isn't an administrator or in **Remote Desktop Users** | `Get-LocalGroupMember "Remote Desktop Users"` |
| Asked to authenticate twice, or NLA error | User not allowed for NLA, or clock/domain trust issue | Group membership, "Access this computer from the network" right, time sync |
| "Credentials did not work" | Wrong format, expired password, locked account | Use `HOST\user` or `DOMAIN\user`, check lockout |
| Black screen or instant disconnect | Display or session problem, or a stuck session | `quser`, `logoff <id>`, update graphics driver |
| "Another user is signed in" | Single-session client edition | Ask the user to sign out, or use a server edition |

---

## 2B. Operator vs. Administrator Security Controls

The **Administrators** group is all-powerful. **Operator** groups are narrower roles that let a person do *one kind of job* without full control. This follows the **principle of least privilege**.

### Administrator
Members can do anything: install software and drivers, change any setting, take ownership of any file, manage all users and read other users' data. A compromised administrator account is a compromised machine. On a domain, **Domain Admins** extends that to the whole domain.

### Operator groups
| Group | Where | What it's for | Notes |
|-------|-------|---------------|-------|
| **Backup Operators** | Local and domain | Back up and restore files regardless of file permissions | Uses the backup/restore **privileges**, so can read nearly any file. Treat as high-risk. |
| **Account Operators** | Domain (AD built-in) | Create and manage ordinary user, group and computer accounts | Can't manage administrators or other privileged groups |
| **Server Operators** | Domain controllers | Maintain servers: sign in locally, start/stop services, back up, shut down | Can reach powerful system functions on a DC. High-risk. |
| **Print Operators** | Domain controllers | Manage printers and print queues | Can sign in to a DC and load drivers. High-risk. |
| **Network Configuration Operators** | Local | Change TCP/IP settings | Limited, no other admin rights |
| **Performance Monitor / Log Users** | Local | Read or log performance counters | Read-only monitoring |
| **Event Log Readers** | Local | Read event logs | Read-only |
| **Remote Management Users** | Local | Connect with WinRM/PowerShell remoting | Needed for remote management without full admin |

Check what a group can do before using it:
```powershell
Get-LocalGroupMember "Backup Operators"
whoami /priv                      # look for SeBackupPrivilege, SeRestorePrivilege
```

### Operator vs. administrator, side by side

| | Operator | Administrator |
|---|---|---|
| Scope | One job (backup, printing, accounts) | Everything |
| Risk if compromised | Limited to that job, but some operator roles can be escalated | Full compromise |
| Day-to-day use | Appropriate for delegated tasks | Should be rare and separate from daily use |
| Audit trail | Easier to trace to a task | Broad, harder to scope |

**Important caveat:** "operator" does not mean "safe". Backup, Server and Print Operators hold privileges (read any file, load drivers, sign in to a domain controller) that attackers use to become full administrators. Keep them **empty by default**, add people only for a defined reason, and monitor membership changes (event **4732**).

### Controls that make privilege safer
- **Separate accounts:** use a normal account for email and browsing, and a different admin account for admin tasks.
- **UAC:** keep it on so administrator tasks need deliberate elevation.
- **Least privilege:** grant the narrowest group, or a specific right through **Group Policy → User Rights Assignment**.
- **Just-in-time access:** grant admin rights temporarily, then remove them.
- **LAPS (Local Administrator Password Solution):** gives each PC a unique, rotating local administrator password.
- **Auditing:** log privileged sign-ins (4672) and group changes.
- **Delegation in AD:** give an OU administrator control over *that OU only*, instead of making them a Domain Admin.

---

## 2C. Power Users

**Power Users** is a legacy local group. It was introduced in the Windows NT/2000/XP era so that someone could do more than a standard user (install some programs, change some system settings, manage shares and printers) without being a full administrator. Legacy software often demanded administrator rights, and this group was a middle ground.

**What changed:** starting with **Windows Vista**, standard users became the default and **UAC** was introduced. Microsoft stated that the Power Users group was **effectively deprecated**. The group still exists on modern Windows so old applications and settings don't break, but it is not given meaningful extra rights by default, and it is **not a supported way to delegate** administration today.

**Why you shouldn't rely on it:**
- On older systems and some legacy installers, Power Users could write to protected places such as system folders or the registry. That allowed a member to **escalate to full administrator**.
- It gives a false sense of "limited" access.
- Modern Windows offers clearer tools.

**What to use instead:**

| You want to let someone... | Use |
|----------------------------|-----|
| Remote in | **Remote Desktop Users** |
| Back up files | **Backup Operators** (carefully) |
| Change network settings | **Network Configuration Operators** |
| Read logs | **Event Log Readers** |
| Monitor performance | **Performance Monitor Users** |
| Manage Hyper-V | **Hyper-V Administrators** |
| Run one admin task | Specific permission on that object, or a delegated scheduled task / service |
| Run a legacy app that needs rights | Fix the app's folder permissions (`icacls`), or compatibility settings, instead of adding to Power Users |

Check whether it's in use on a machine:
```powershell
Get-LocalGroupMember "Power Users"
```
If it lists anyone, find out why. Usually the answer is a leftover from an old setup, and the member can be removed.

---

## 3. What Happens When It Breaks

| Symptom | Likely cause | First steps |
|---------|--------------|-------------|
| "Access is denied" | Missing NTFS or share permission, or not elevated | `icacls <path>`, `whoami /groups`, run as administrator |
| User can't sign in | Locked, disabled, or expired account | `Get-LocalUser`; in AD `Get-ADUser -Properties LockedOut` |
| Change doesn't apply to the user | Token is stale | Sign out and back in (group membership loads at sign-in) |
| Slow PC | CPU, memory or disk saturated | Task Manager, Resource Monitor, `Get-Process` |
| Service fails to start | Dependency, bad credentials, config | `Get-Service`, System log, service **Recovery** tab |
| Windows Update stuck | Corrupt component store | `DISM /Online /Cleanup-Image /RestoreHealth`, then `sfc /scannow` |
| Policy not applied | GPO scope, replication, DNS | `gpresult /r`, `gpupdate /force`, check DNS to the domain controller |
| Disk full | Logs, temp, updates | `Get-Volume`, WinDirStat or `Storage Sense` |
| Blue screen | Driver, memory or disk fault | Event Viewer System log, minidump in `C:\Windows\Minidump` |
| Unexpected admin account | Misuse or compromise | `Get-LocalGroupMember Administrators`, event 4732 |

---

## 4. Hands-On Lab

Use a **virtual machine or a spare PC** running Windows Pro or Enterprise. Don't run this on your only work computer. Open PowerShell **as administrator**.

**Goal:** create a limited user, observe what groups change, and see an RDP sign-in in the logs.

1. **Baseline.**
   ```powershell
   whoami /groups
   Get-LocalGroupMember Administrators
   Get-LocalGroupMember "Remote Desktop Users"
   Get-LocalGroupMember "Power Users"
   ```
2. **Create a test user.**
   ```powershell
   New-LocalUser -Name "labuser" -Password (Read-Host -AsSecureString "Password") -FullName "Lab User"
   Add-LocalGroupMember -Group "Users" -Member "labuser"
   ```
3. **Test denial.** Sign in as `labuser` (Win+L, then switch user). Try to open `regedit` and edit something under `HKLM`, or install software. You should be blocked or prompted for administrator credentials. Sign out.
4. **Grant narrowly.** Add the user to a single operator-style group:
   ```powershell
   Add-LocalGroupMember -Group "Event Log Readers" -Member "labuser"
   ```
   Sign in as `labuser` again, and confirm the user can read the Security event log in Event Viewer but still can't change system settings. Compare with `whoami /groups`.
5. **Set up RDP.** Enable RDP and NLA using the commands in section 2A, and add `labuser` to **Remote Desktop Users**.
6. **Test it.** From a second machine, run `Test-NetConnection <lab-pc> -Port 3389`, then `mstsc /v:<lab-pc>` and sign in as `<lab-pc>\labuser`.
7. **Read the evidence.** On the lab PC, as administrator:
   ```powershell
   Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 10 |
     Select-Object TimeCreated, Id
   ```
   Find your RDP sign-in. For a failed attempt, enter a wrong password and look for event **4625**.
8. **Compare roles.** Run `whoami /priv` as an administrator and as `labuser`. Note which privileges only the administrator has.
9. **Clean up.**
   ```powershell
   Remove-LocalGroupMember -Group "Remote Desktop Users" -Member "labuser"
   Remove-LocalGroupMember -Group "Event Log Readers" -Member "labuser"
   Remove-LocalUser -Name "labuser"
   Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 1
   Disable-NetFirewallRule -DisplayGroup "Remote Desktop"
   ```

**Check yourself:** why can an Event Log Reader read logs but not install software, and why is Backup Operators riskier than Event Log Readers even though neither is "Administrators"?

---

## Further Reading
- [Microsoft: Built-in Administrator Account Disabled (Power Users group deprecated)](https://learn.microsoft.com/en-us/archive/blogs/uac/built-in-administrator-account-disabled)
- [Microsoft: User can't authenticate or must authenticate twice (RDP and NLA)](https://learn.microsoft.com/en-us/troubleshoot/windows-server/remote/cannot-authenticate-must-authenticate-twice)
- [Windows Built-in Users, Default Groups and Special Identities (SS64)](https://ss64.com/nt/syntax-security_groups.html)
- [Default Local Groups (ITFreeTraining)](https://itfreetraining.com/lesson/default-local-groups/)
- [Best Practices to Secure Remote Desktop Access (NinjaOne)](https://www.ninjaone.com/blog/best-practices-to-secure-remote-desktop-access/)
- [How to Enable Remote Administration in Windows Safely (NinjaOne)](https://www.ninjaone.com/blog/how-to-enable-remote-administration-in-windows-safely/)
- [Microsoft Learn: Understand Active Directory Group Policy Security Settings](https://learn.microsoft.com/en-us/training/modules/understand-active-directory-security-policies/)
- [Microsoft Learn: ActiveDirectory PowerShell module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/?view=windowsserver2022-ps)
