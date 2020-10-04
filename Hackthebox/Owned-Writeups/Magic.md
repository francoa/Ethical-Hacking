# Magic

Details:
- Magic IP: 10.10.10.185
- Kali IP: 10.10.14.157
- Date of writeup: 4 Oct 2020

Sections:
- [Recon](#recon)
- [Unrestricted File Upload](#unrestricted-file-upload)
- [Exploring for Credentials](#exploring-for-credentials)
- [Achieving root](#achieving-root)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

Tools/commands used
- nmap
- dirbuster
- Burp
- [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
- [LinEnum](https://github.com/rebootuser/LinEnum)

## Recon
Run nmap
```kali$  nmap -sC -sV 10.10.10.185```

![](images/Magic_1.png?raw=true)

Ports 22 and 80 are open. Not much we can do other than start enumerating the webserver at port 80. ```Dirbuster``` shows there's a ```login.php```, ```upload.php``` and ```logout.php```.

The login page looks something like:

![](images/Magic_2.png?raw=true)

Common username/password pairs do not work.

When I try to get to ```upload.php```, the webserver redirects me to ```login.php```. I really want to see that upload page, so I go to ```Burp``` and I intercept the response from the ```GET``` request to ```upload.php```. Just removing the redirection to ```login.php``` (highlighted in the image below) let's us get into the upload page.

![](images/Magic_3.png?raw=true)

Now we can upload images. Just trying with a random image we can see it works and that it appears in the main page.

![](images/Magic_4.png?raw=true)

## Unrestricted File Upload

Using burp, we can upload a file and use this technique: [Unrestricted File Upload : Weak Protections and Bypassing Methods](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload).

The steps I followed are the following:
- Intercept a file upload using Burp
- Change the extension of the file to ```name.php.jpg```
- Add commands to the end of the file data 
- Upload the file
- Send a ```GET``` request for the uploaded file (at ```10.10.10.185/images/uploads/name.php.jpg```)
- This makes Apache run the command, and we have RCE

We can now get a reverse shell by uploading laudanum reverse shell (see pentest monkey) for php and running it with the vulnerability found above.

An example of a wget command at the end of the file upload:

![](images/Magic_6.png?raw=true)

Uploading laudanum reverse shell:

![](images/Magic_7.png?raw=true)

Getting a shell:

![](images/Magic_8.png?raw=true)

## Exploring for Credentials

By taking a look at ```/home``` and ```/etc/passwd```, the user we need to crack is ```theseus```.

First thing to explore: the web server folder. If we take a look at ```/var/www```, we can see a bunch of ```php``` files in the ```Magic``` folder:

![](images/Magic_9.png?raw=true)

There is an interesting one called ```db.php5```. This file looks like it makes the connection to the Database which holds the users for the ```login.php``` page.

If we ```cat``` that file:

![](images/Magic_10.png?raw=true)

We have credentials! However, these are not credentials for the user ```theseus``` in the machine. Running ```su - theseus``` and putting in the password doesn't work. They must be credentials for the DB.

To know if there is a DB present, run ```ss -antup | grep 127.0.0.1```. We can see a ```mysql``` DB at port 3306:

![](images/Magic_11.png?raw=true)

I poked around the DB for a while and found a table called ```login```. To dump this table, run the next command, using the credentials found before:
```
mysqldump --user=user --password=password Magic login
```

![](images/Magic_12.png?raw=true)

We get another set of credentials! If we use this password with the ```su - theseus``` command, we get user access.

## Achieving root

So far so good, pretty straight forwards. The issue now comes with achieving root.

I tried linPEAS and LinEnum.

LinPEAS showed a database in ```.local``` folder which was suspicious, but dumping it showed no useful information.

I also tried attacking ```logrotate```, but it didn't work because it wasn't writing in local writable folders (just as FYI, ```systemctl list-timers --all``` lists the time for all cron jobs to launch.

Eventually, this line in LinPEAS caught my eye:

![](images/Magic_13.png?raw=true)

We can perform a ```sysinfo``` SUID privilege escalation.

I performed some reverse engineering with Ghidra, and observed that this ```sysinfo``` runs ```lshw -short```, ```fdisk -l```, ```cat /proc/cpuinfo``` and ```free -h```.

![](images/Magic_14.png?raw=true)

Thankfully, ```sysinfo``` runs as root by default, so it really looks like the way to achieve privesc.

First I analyzed all the binaries for sideloading capabilities: ```strace /bin/sysinfo 2>&1 | grep -i -E "open|access|no such file" | grep "\.so"```. This led me nowhere.

Eventually, I realized that what was being run in the ```sysinfo``` command (seen in the Ghidra analysis) was ```exec(lshw -short)```, ```exec(fdisk -l)```, and so on. So, what I did was to add a folder I control to the ```PATH``` environment variable and created a binary called ```lshw``` that had the code to open a reverse shell. This way, running ```sysinfo``` you can achieve root.

![](images/Magic_15.png?raw=true)


## Learnings from ippsec walkthrough

TODO

