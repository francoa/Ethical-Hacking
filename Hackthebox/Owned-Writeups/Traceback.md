# Traceback

Details:
- Traceback IP: 10.10.10.181
- Kali IP: 10.10.14.160
- Date of writeup: 4 Oct 2020

This box was a pretty easy one!

Sections:
- [Recon](#recon)
- [Getting to user](#getting-to-user)
- [Achieving root](#achieving-root)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

## Recon
Run nmap
```kali$  nmap -sC -sV 10.10.10.181```

![](images/Traceback_1.png?raw=true)

Nmap only showed port 80 and 22.

In index.html, port 80, there was a message.

![](images/Traceback_2.png?raw=true)

By inspecting the source, I saw a comment and the name of the author.
- Some of the best web shells that you might need
- Xh4H

![](images/Traceback_3.png?raw=true)

A simple internet search led me to a github page with several webshells. I added those files to a wordlist and ran dirbuster with it.
```
10.10.10.181/smevk.php found
```

![](images/Traceback_4.png?raw=true)

Lokking in github, default config for ```smevk.php``` was ```admin admin```. Using that I logged in and launched a reverse shell.

![](images/Traceback_5.png?raw=true)

## Getting to User
User is ```webadmin```, but it is not the one we are looking for. ```sysadmin``` is the real user.

![](images/Traceback_6.png?raw=true)

I searched in the webserver home folder (```/var/www/```) and found a note from ```sysadmin```:

![](images/Traceback_7.png?raw=true)

I run ```sudo -l``` to know if there was something we could run as root (or another user), and, effectively, there was.
```
(sysadimn) NOPASSWD: /home/sysadmin/luvit
```

If you see the help in luvit, it's a lua interpreter. Searching at [GTFObins](https://gtfobins.github.io/gtfobins/lua/), we can see that we can use ```lua``` to escalate privileges. So, we can use that script to move laterally to sysadmin:
```
sudo -u sysadmin /home/sysadmin/luvit -e 'os.execute("/bin/sh")' opens a a shell as sysadmin
```

## Achieving root

From [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS), there are some ```sysadmin``` writeable files in ```/etc/update-motd.d```. These are the MOTD (message of the day), the text that appears when a user logs in.

![](images/Traceback_8.png?raw=true)

We can modify that file to run some code, for example, open up a reverse shell.
```
echo "{reverse shell command}" >> /etc/update-motd.d/00-header 
```

Then log in with a user and we'll obtain root.

With ```webadmin```, run ```ssh-keygen```. Copy the resulting key-pair to the local environment and add the public key to ```authorized_keys``` in ```webadmin/.ssh```. Then run
```
ssh -i id_rsa webadmin@10.10.10.181
```


## Learnings from ippsec walkthrough

TODO
