# Cheat sheet: commands from the AD and Splunk lab

The commands I used in the project, with explanations. There are no passwords here. Passwords are entered at runtime (`Read-Host -AsSecureString`).

## PowerShell: networking and DNS (Windows)

| Command | What it does |
|---|---|
| `Get-NetAdapter` | Lists network adapters (name, status, MAC address) |
| `Get-NetAdapter \| ft Name,MacAddress` | Shows only names and MAC addresses (to tell the NAT NIC from the Host-only NIC) |
| `New-NetIPAddress -InterfaceAlias "Ethernet 2" -IPAddress 192.168.56.10 -PrefixLength 24` | Sets a static address on an adapter |
| `Remove-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.56.10 -Confirm:$false` | Removes an address from an adapter |
| `Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled` | Enables DHCP on an adapter |
| `Set-DnsClientServerAddress -InterfaceAlias "Ethernet 2" -ServerAddresses 127.0.0.1` | Sets the DNS server of an adapter |
| `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses` | Restores automatic DNS |
| `Set-DnsClient -InterfaceAlias "Ethernet" -RegisterThisConnectionsAddress $false` | Do not register this adapter's address in DNS (NAT adapter) |
| `ipconfig /all` | Full network configuration |
| `ipconfig /flushdns` | Clears the client DNS cache |
| `ipconfig /registerdns` | Refreshes this computer's DNS records |
| `nslookup github.com` | Checks name resolution |
| `Test-NetConnection github.com -Port 443` | Tests connectivity to a host and port |
| `Test-NetConnection 192.168.56.20 -Port 9997` | Checks whether Splunk is reachable on port 9997 |
| `Rename-Computer -NewName DC01 -Restart` | Renames the computer and restarts |

## PowerShell: DNS server

| Command | What it does |
|---|---|
| `Add-DnsServerForwarder -IPAddress 1.1.1.1,8.8.8.8` | Adds forwarders so the DNS server can resolve external names |
| `Clear-DnsServerCache -Force` | Clears the DNS server cache |

## PowerShell: Active Directory

| Command | What it does |
|---|---|
| `Install-WindowsFeature AD-Domain-Services -IncludeManagementTools` | Installs the AD DS role |
| `Install-ADDSForest -DomainName "lab.local" -DomainNetbiosName "LAB" -InstallDns` | Creates a new domain (asks for the Safe Mode password) |
| `Get-ADDomain` | Domain information |
| `New-ADOrganizationalUnit -Name "LabUsers" -Path "DC=lab,DC=local"` | Creates an organizational unit |
| `New-ADUser -Name "Jan Kowalski" -SamAccountName jkowalski -Path "OU=LabUsers,DC=lab,DC=local" -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true` | Creates a user (password typed at runtime) |
| `New-ADUser ... -WhatIf` | Dry run, creates nothing |
| `Get-ADUser -Filter * \| Select Name,SamAccountName` | Lists all users (including built-in accounts) |
| `Get-ADUser -Filter * -SearchBase "OU=LabUsers,DC=lab,DC=local" \| Select Name,SamAccountName` | Lists only users in a given OU |
| `Get-ADUser jkowalski -Properties *` | All properties of an account |
| `Set-ADAccountPassword -Identity asmith -Reset -NewPassword (Read-Host -AsSecureString "New password")` | Resets a password |
| `Enable-ADAccount -Identity asmith` | Enables an account |
| `Remove-ADUser asmith` | Deletes an account |
| `Add-Computer -DomainName lab.local -Credential LAB\Administrator -Restart` | Joins a computer to the domain |

**How to read an object name (DN):** right to left, `CN=Jan Kowalski,OU=LabUsers,DC=lab,DC=local` = a user in the LabUsers OU of the lab.local domain.

## PowerShell: services, downloads, help

| Command | What it does |
|---|---|
| `Get-Service Sysmon64, SplunkForwarder` | Service status |
| `Restart-Service SplunkForwarder` | Restarts the forwarder |
| `Get-Date` | Current date and time (to match events later) |
| `Invoke-WebRequest -Uri "<url>" -OutFile "<file>"` | Downloads a file from the internet |
| `$ProgressPreference = 'SilentlyContinue'` | Disables the progress bar, speeds up downloads |
| `Get-Help New-ADUser -Examples` | Usage examples for a cmdlet |

## Sysmon

```
cd $env:USERPROFILE\Downloads
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
Get-Service Sysmon64
```

The configuration comes from the SwiftOnSecurity/sysmon-config repository. Log location: Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational.

## Universal Forwarder (`inputs.conf`)

File: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

```
[WinEventLog://Security]
index = windows
disabled = 0

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = windows
disabled = 0
```

After a change: `Restart-Service SplunkForwarder`. Forwarder log: `C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log`.

## Linux (Ubuntu Server)

| Command | What it does |
|---|---|
| `ip a` | Addresses and state of network interfaces |
| `sudo nano /etc/netplan/01-lab.yaml` | Edit the network configuration |
| `sudo chmod 600 /etc/netplan/01-lab.yaml` | Restricts access to the config file |
| `sudo netplan apply` | Applies the network configuration |
| `sudo ss -tlnp \| grep 8000` | Checks whether something is listening on port 8000 |
| `curl -I http://127.0.0.1:8000` | Checks the web server response |

Host-only NIC configuration (`01-lab.yaml`):

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses: [192.168.56.20/24]
```

## Splunk (install and operation)

| Command | What it does |
|---|---|
| `sudo dpkg -i splunk.deb` | Installs the Splunk package |
| `sudo useradd -r -s /bin/bash -d /opt/splunk splunk` | Creates the `splunk` user |
| `sudo chown -R splunk:splunk /opt/splunk` | Gives it the Splunk directory |
| `sudo -u splunk /opt/splunk/bin/splunk start --accept-license` | Starts Splunk as the `splunk` user |
| `sudo -u splunk /opt/splunk/bin/splunk status` | Splunk status |

After installation, in the web UI: Settings → Indexes → New Index `windows`, Settings → Forwarding and receiving → Configure receiving → port 9997.

## Splunk: searches (SPL)

| Search | What it does |
|---|---|
| `index=windows \| stats count by host, source` | Event count per host and source |
| `index=windows EventCode=4625 \| stats count by host` | Failed logons |
| `index=windows EventCode=4720 \| table _time host Account_Name` | Created accounts |
| `index=windows EventCode=4625 \| timechart span=1h count` | Failed logons over time |
| `index=windows source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 \| stats count by host` | Process creations per host |
| `index=windows "An account failed to log on"` | Search by phrase instead of by number |

A search with names instead of numbers:

```
index=windows source="WinEventLog:Security"
| eval event=case(EventCode==4624,"Logon succeeded", EventCode==4625,"Logon failed", EventCode==4720,"Account created", true(),"other")
| stats count by event
```

The reconnaissance simulation search:

```
index=windows source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 (CommandLine="*net user*" OR CommandLine="*net localgroup*" OR CommandLine="*net1*")
| table _time host User ParentImage Image CommandLine
```

## Atomic Red Team (lab only, after taking a snapshot)

```
Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"
Install-AtomicRedTeam -getAtomics
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Invoke-AtomicTest T1087.001 -ShowDetailsBrief
Invoke-AtomicTest T1087.001 -TestNumbers 8
Invoke-AtomicTest T1087.001 -TestNumbers 8 -Cleanup
Remove-MpPreference -ExclusionPath "C:\AtomicRedTeam"
```

Pick the test number from the Windows entries in the list (`-ShowDetailsBrief`). In my lab, test 8 is "Enumerate all accounts on Windows (Local)".
