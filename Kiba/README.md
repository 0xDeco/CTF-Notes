# Kiba
[TryHackMe room link](https://tryhackme.com/room/kiba)

# Informations about the room 

> Identify the critical security flaw in the data visualization dashboard, that allows execute remote code execution. 

# Open ports 

22 OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)

80 Apache httpd 2.4.18 ((Ubuntu))

5044

5601 Kibana 6.5.4

# Enumeration 

This version of Kibana is vulnerable to [RCE](https://slides.com/securitymb/prototype-pollution-in-kibana#/57): `CVE-2019-7609`

There is also a [python script](https://github.com/LandGrey/CVE-2019-7609/) to automate the process

After using it we have a shell with the user `kiba`

# Privilege escalation

The room tells us to enumerate capabilities

We find: `/home/kiba/.hackmeplease/python3 = cap_setuid+ep` [GTFOBINS link](https://gtfobins.github.io/gtfobins/python/)

`./python -c 'import os; os.setuid(0); os.system("/bin/sh")'`
