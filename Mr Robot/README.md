# Mr Robot CTF
[TryHackMe room link](https://tryhackme.com/room/mrrobot)

# Informations about the room
> Can you root this Mr. Robot styled machine? This is a virtual machine meant for beginners/intermediate users. There are 3 hidden keys located on the machine, can you find them?

# Open ports
80 http 

443 https 

# Enumeration 

With directory enumeration we find a `/0/` directory containing a wordpress site

With WPScan we discover the wordpress version and any possible plugins and themes

```
WordPress version 4.3.1

WordPress theme in use: twentyfifteen
```

In the `robots.txt` file there are two files listed:

```
fsocity.dic 
key-1-of-3.txt
```

The first file is a wordlist

The second is flag number 1

# Bruteforce

After a long time hydra finds the password for user `elliot` to access the wordpress admin panel

To get a reverse shell you can modify the 404.php file with a php reverse shell

# Privilege escalation

After getting a shell you will find flag number 2 and a file with a username and a md5 password

After decoding it you will find credentials for the user `robot`

After enumeration you find a SUID with nmap

[Link to GTFOBins](https://gtfobins.github.io/gtfobins/nmap/)

```console
robot@linux:/dev/shm$ nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
# id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
# whoami
root
```
