# Overpass 3

[TryHackMe room link](https://tryhackme.com/room/overpass3hosting)

# Informations about the room

> You know them, you love them, your favourite group of broke computer science students have another business venture! Show them that they probably should hire someone for security...

# Open ports

21 vsftpd 3.0.3

22 OpenSSH 8.0

80 Apache httpd 2.4.37

# Enumeration 

With directory bruteforcing we find a `backups` directory

It cointains a PGP key and a encrypted xlsx file

```console
 gpg --import priv.key 
 gpg --decrypt CustomerDetails.xlsx.gpg                                                                                                                                   
```

The excel sheet contains Usernames and passwords for potential users:
```
Customer Name	Username	Password	Credit card number	CVC	
Par. A. Doxx	paradox	REDACTED	4111 1111 4555 1142	432	
0day Montgomery	0day	REDACTED	5555 3412 4444 1115	642	
Muir Land	muirlandoracle	REDACTED	5103 2219 1119 9245	737	
```

Bruteforcing ssh did not work but ftp worked with user paradox 

We now have access to the webroot, and write access too

We upload a php reverse shell 

# Privilege escalation

After logging in we can switch to the user paradox with the password found earlier

Linpeas finds a [weak NFS permission](https://juggernaut-sec.com/nfs-no_root_squash/)

`/home/james *(rw,fsid=0,sync,no_root_squash,insecure)`

By enumerating the NFS we can see that is listening on port 2049 but we cannot see it from the attacking system

I uploaded a meterpreter shell and i forwarded the port to my system

`meterpreter > portfwd add -l 9999 -p 2049 -r 10.10.44.161`                                                                                                              

I mounted it:

`sudo mount -v -t nfs localhost:/ test`

And found the user flag

We also find james's private ssh key that we can use to log in his account

Linpeas finds a potential [privesc vector](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#writable-path-abuses) in the PATH:

```console
PATH
/home/james/.local/bin:/home/james/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
New path exported: /home/james/.local/bin:/home/james/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/sbin:/bin
```

From the attacking machine we copy `/bin/bash` we set the SUID bit `chmod +s bash` and we start it from the `james` user with `./bash -p`
