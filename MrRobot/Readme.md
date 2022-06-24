# **_MrRobot TryHackMe Writeup_**
## SneakyTurtle158
## Room Creator: ben
[Mr Robot](https://tryhackme.com/room/mrrobot)

I always like to start these boxes with a little housekeeping on my machine (making a directory for any files, adding the IP to my hosts file, and doing a quick ping to the machine to make sure I can reach it.)

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/1.png">
</p>

Next we can move into enumerating the box. I like to start off using a python tool written by dievus called threader3000. This allows for multithreaded port scanning. In a real world scenario this is a very noisy scan, so you wouldn't likely use it, but in THM boxes it's great becasue it can enumerate ports in 30-90 seconds and then will suggest an in-depth nmap scan that you can run against those specific ports. If you've ever run a nmap -T4 -p- -A scan against a target, you can probably see the benefit to this. 

To use the tool you can clone it right from the repo using git:

```bash
cd /opt && git clone https://github.com/dievus/threader3000.git
cd threader3000
chmod +x threader3000.py
./threader3000.py
```
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/2.png">
</p>

Our results came back and we can see that port 80 and port 443 are open, if you want to run the suggested nmap scan you can, but for this I think it's safe to say that we can move into enumerating the webpages. 

Using curl we see that the HTTP webpage is running some sort of javascript, so lets open up a browser and check it out.

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/3.png">
</p>

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/4.png">
</p>

And we are brought to an interactive webpage (someone definitely put some time and effort into making this box)

From here you can poke around at the different comands that it gives you, but after a while I decided to move on so I set up gobuster to run in the background and went to go check the good old robots.txt file to see if there was anything there:

```bash
gobuster dir -u http://mrrobot/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/5.png">
</p>

_Bingo!_ Looks like we found our first key along with a potential dictionary file, let's grab those:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/6.png">
</p>

Alright, 1 down, 2 more to go. Going back to our gobuster scan we can see that there are quite a few directories that are popping up and a few of them are WordPress directories. 

Let's see what we can find using wpscan (I always recommend getting wpscan's free api token as you will get better results):

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/7.png">
</p>

In hinesite this ended up pulling a lot of information in, some useful some not. Rather than parsing through all of the results, we can see that the WP version that is in use is outdated and we know that we located a dictionary file, so let's look at a common WP login vulnerability to see if we can brute force our way in.

We will start by using admin:password to see what happens:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/8.png">
</p>
<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/9.png">
</p>

Alright, now this is useful, the login page is telling us that admin is an invalid username. We can actually use this to try to figure out a correct user name by doing some brute forcing. 

First let's take a look at the request that is being made when we try to log in, if you are using Firefox you can right-click and select Inspect>Network Tab and then try to login with the previous creds again:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/10.png">
</p>

We want to right-click the POST and select _Edit and Resend_ then on the right side scroll down to the request body to see what the request syntax is, you will need this for the brute force attack.

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/11.png">
</p>

Now let's open up a terminal in the directory where we have that fsocity.dic and we will use hydra for the bruteforce attack:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/12.png">
</p>

#### For those who aren't familiar with hydra, let me explain the syntax of this command. Hydra is a great bruteforcing tool that can be used to bruteforce a variety of different protocols. Here we are using it to brute force the login page to try to get a correct username with the fsocity.dic file. The parameters are:

`-L fsocity.dic` : Capital L is used for the username (capitalized means we are using a list)
`-p test` : Lowercase p means we are supplying 1 password (test) for all login attempts made
`mrrobot http-post-form` : this points to the mrrobot address and lets hydra know we are using an HTTP POST request
`"/wp-login/` : this tells hydra the directory that the login page is at
`log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot%2Fwp-admin%2F&testcookie=1` : This should be familiar, this is the Request Body that we found earlier, we replaced the admin and password targets with ^USER^ and ^PASS^ to let hydra know where to use the supplied -L and -p input.
`:Invalid username."` : This is to let hydra know to ignore any reponses that include this text. This means that any incorrect user name attempts will be dropped and hydra will only show us results that did not reult in the “Invalid username” error.

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/13.png">
</p>

We should get a result pretty quickly. If we go back to the login page and try to log in with the username that we found we should get a new response:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/14.png">
</p>

Alright, we are half way there. We know the username and the login page is once again giving us too much information. Let's use that to our advantage. We will use the feedback to craft a new hydra request, this time to brute force the password. 

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/15.png">
</p>

Now that we have the username and password let's log in.

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/16.png">
</p>

Alright, we've got the admin dashboard, which if you aren't already aware WP admin panels are notoriously easy to get reverse shells off of. I am going to use the theme editor, but I highly encourage you to search around online for different ways of accomplishing this. 

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/17.png">
</p>

Under the 404.php template I will upload a php reverse shell that I got from pentestmonkey.

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/18.png">
</p>

Next I just need to set up a netcat listener on my machine using the port I designated and then navigate to the 404.php url:

`nc -nvlp 7890`

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/19.png">
</p>

Now we've got a shell. Let's check to see if we have python and then use pty to clean up the shell:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/20.png">
</p>

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/21.png">
</p>

Looks like we have a user named robot and the key we need is in their home dir, but we don't have access to the key. Looks like we have robot's password in raw md5 form though:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/22.png">
</p>

Let's copy that onto our local machine and crack it using John the Ripper:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/23.png">
</p>

Now we can grab that second key as robot:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/24.png">
</p>

Next step is to look for privilege escalation vectors. I usually run a few basic commands first to see if there is anything overly obvious before trying to move over a linpeas.sh script to enumerate for me:

```bash
sudo -l		#no luck here
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;	#this will help us look for SUID bits
```
Bingo, looks like we have a SUID bit set on nmap:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/25.png">
</p>

If you haven't already, bookmark [GTFObins](https://gtfobins.github.io/). It is your best friend when it comes to privesc.

Searching for nmap+SUID on GTFObins gives us our way to get root:

<p align="center">
  <img src="https://github.com/SneakyTurtle158/TryHackMe_Writeups/blob/pictures/mrrobot/26.png">
</p>

_And that's it, thanks for following along!_
