---
sidebar_position: 10
sidebar_label: Quickly deploy a self-managed AD Forest on Amazon EC2
---

# Quickly deploy a self-managed AD Forest on Amazon EC2
by Tekena Orugbani

### Introduction
When working with customers, I often need to spin up and spin down Active Directory Forests to test customer use cases. I use the scripts below to spin up a new AD Forest called "example.com". You can run this script as userdata when launching a new Amazon Elastic Compute Cloud (Amazon EC2) instance or you can execute the script via AWS Systems Manager Session Manager after the new instance is launched. Either way works. And if you want to add a second domain controller to the AD forest, run the second script.

![IMAGE1](IMG/IMG-1.png)

### Store local admin in AWS Secrets Manager
First, store the local Administrator of your EC2 instance as a secret in Secrets Manager.

```
@'
{
    "username": "Administrator",
    "password": "YourPassword"
}
'@ | Set-Content -Path "C:\Users\$env:USERNAME\Documents\admin-cred.json"

aws secretsmanager create-secret `
--name LocalAdministrator `
--secret-string file://C:/Users/$env:USERNAME/Documents/admin-cred.json

Remove-Item -Path "C:\Users\$env:USERNAME\Documents\admin-cred.json" -Force
```

### Deploy first Domain Controller

Now, create your first Domain Controller and the new Forest:

```
#Deploy New Forest PowerShell
$domain = "example.com" 
$Username = ((Get-SECSecretValue -region us-east-1 -SecretId LocalAdministrator).SecretString | ConvertFrom-JSON).username
$FinalUserName = $domain + '\' + $Username
$Password = ((Get-SECSecretValue -region us-east-1 -SecretId LocalAdministrator).SecretString | ConvertFrom-JSON).password
$Credential = New-Object System.Management.Automation.PsCredential($FinalUserName,$Password)

Install-WindowsFeature -name AD-Domain-Services -IncludeManagementTools
Install-ADDSForest -DomainName example.com -DomainNetBIOSName EXAMPLE -InstallDNS -SafeModeAdministratorPassword $Password -confirm:$false
```

### Deploy additional domain controllers

To deploy a new domain controller to the new forest, first obtain the private IP address of the first domain controller above and replace in the script below. It also assumes the local Admin password of this new EC2 instance is the same as the first one. Run this:

```
##Build new DC in an existing domain
Get-NetAdapter | Select-Object InterfaceAlias , InterfaceIndex
$IntIndex = Get-NetAdapter | where {$_.ifDesc -notlike "TAP*"} | foreach InterfaceIndex | select -First 1
Set-DnsClientServerAddress -InterfaceIndex $IntIndex -ServerAddresses ("IP Address of first DC")

$domain = "example.com"
$Username = ((Get-SECSecretValue -region us-east-1 -SecretId LocalAdministrator).SecretString | ConvertFrom-JSON).username
$FinalUserName = $domain + '\' + $Username
$Password = ((Get-SECSecretValue -region us-east-1 -SecretId LocalAdministrator).SecretString | ConvertFrom-JSON).password
$Credential = New-Object System.Management.Automation.PsCredential($FinalUserName,$Password)

Install-WindowsFeature -name AD-Domain-Services -IncludeManagementTools
Install-ADDSDomainController -DomainName $domain -InstallDns:$true -Credential $Credential -SafeModeAdministratorPassword $Password -confirm:$false
;
```
