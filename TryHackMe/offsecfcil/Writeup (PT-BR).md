> [!WARNING]
> This writeup is in portuguese. For the english version, please follow [this link](./Writeup%20(EN-US).md).

# [OffSec - HASH](https://tryhackme.com/room/offsecfcil)

Capture The Flag original disponível em [Try Hack Me](https://tryhackme.com/room/offsecfcil).

Resolvido em: `2026/04/20`

# Conteúdos

- [OffSec - HASH](#offsec---hash)
- [Conteúdos](#conteúdos)
- [Writeup](#writeup)
   * [Sumário](#sumário)
   * [Hash Cracking](#hash-cracking)
      + [Hash 1](#hash-1)
      + [Hash 2](#hash-2)
      + [Hash 3](#hash-3)
      + [Hash 4](#hash-4)
      + [Hash 5](#hash-5)

# Writeup

## Sumário

O desafio consiste em quebrar hashes disponibilizados.

## Hash Cracking

Para este CTF, o objetivo consiste solenemente no processo de quebrar hashes, sem máquinas virtuais envolvidas ou outros campos de pentesting. Cada tarefa é obter o valor original do hash fornecido.

Em geral, o processo de solução será o mesmo para todas as tarefas (exceto quando especificado): primeiro usa-se `hashid`[^hashid] ou [hash-analyzer](https://www.tunnelsup.com/hash-analyzer/) para identificar qual o formato da hash, e depois usar uma ferramenta como `hashcat`[^hashcat] ou [crackstation](https://crackstation.net/) para quebrar o hash.

O comando que usarei com `hashcat` usa a wordlist rockme[^rockme]:
- `hashcat -a 0 -m <hashtype> '<hash>' /usr/share/wordlists/rockyou.txt.gz`

### Hash 1

Hash fornecido: `482c811da5d5b4bc6d497ffa98491e38`

```bash
$ hashid 482c811da5d5b4bc6d497ffa98491e38
Analyzing '482c811da5d5b4bc6d497ffa98491e38'
[+] MD2 
[+] MD5 
...
```

Testando com `MD5`:

```bash
hashcat -a 0 -m 0 '482c811da5d5b4bc6d497ffa98491e38' /usr/share/wordlists/rockyou.txt.gz
...
482c811da5d5b4bc6d497ffa98491e38:<PASS_1>              
                                   
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: 482c811da5d5b4bc6d497ffa98491e38
...
Started: Mon Apr 20 12:05:53 2026
Stopped: Mon Apr 20 12:06:18 2026
```

Obtive a primeira senha, <PASS_1>.


### Hash 2

Hash fornecido: `861c4f67e887dec85292d36ab05cd7a1a7275228`

```bash
$ hashid 861c4f67e887dec85292d36ab05cd7a1a7275228
Analyzing '861c4f67e887dec85292d36ab05cd7a1a7275228'
[+] SHA-1 
[+] Double SHA-1 
...
```

Testando com `SHA-1`:

```bash
hashcat -a 0 -m 100 '861c4f67e887dec85292d36ab05cd7a1a7275228' /usr/share/wordlists/rockyou.txt.gz
...
861c4f67e887dec85292d36ab05cd7a1a7275228:<PASS_2>             
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 100 (SHA1)
Hash.Target......: 861c4f67e887dec85292d36ab05cd7a1a7275228
...
Started: Mon Apr 20 12:10:18 2026
Stopped: Mon Apr 20 12:10:29 2026
```

Obtive a segunda senha, <PASS_2>.


### Hash 3

Hash fornecido: `4149c5cc4c378444d116d65ad5ba4099`

```bash
$ hashid 4149c5cc4c378444d116d65ad5ba4099        
Analyzing '4149c5cc4c378444d116d65ad5ba4099'
[+] MD2 
[+] MD5 
[+] MD4 
...
```

Testando com `MD5`:

```bash
hashcat -a 0 -m 0 '4149c5cc4c378444d116d65ad5ba4099' /usr/share/wordlists/rockyou.txt.gz
...
Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 0 (MD5)
Hash.Target......: 4149c5cc4c378444d116d65ad5ba4099
...
Started: Mon Apr 20 12:11:50 2026
Stopped: Mon Apr 20 12:11:54 2026
```

Exausto. A dica diz ser uma senha de 6 caracteres e todos minúsculos. Defini, então, a regra `-1 ?l?d` para incluir todas as letras minúsculas e os dígitos, e depois coloquei o `hashcat` no modo força-bruta, ainda com `MD5`:

```bash
$ hashcat -a 3 -m 0 -1 ?l?d '4149c5cc4c378444d116d65ad5ba4099' '?1?1?1?1?1?1'
...
Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 0 (MD5)
Hash.Target......: 4149c5cc4c378444d116d65ad5ba4099
...
Started: Mon Apr 20 12:18:10 2026
Stopped: Mon Apr 20 12:18:17 2026
```

Novamente exausto. Tentei, então, quebrar no hash `MD4`:

```bash
$ hashcat -a 3 -m 900 -1 ?l?d '4149c5cc4c378444d116d65ad5ba4099' '?1?1?1?1?1?1'
...
4149c5cc4c378444d116d65ad5ba4099:<PASS_3>                   

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 900 (MD4)
Hash.Target......: 4149c5cc4c378444d116d65ad5ba4099
...
Started: Mon Apr 20 12:19:18 2026
Stopped: Mon Apr 20 12:19:30 2026
```

Obtive a senha, <PASS_3>.


### Hash 4

Hash fornecido: `cdeb746ec095149627348b61d4140fc58b745875`
Salt fornecido: `satech`

```bash
$ hashid 'cdeb746ec095149627348b61d4140fc58b745875:satech'
Analyzing 'cdeb746ec095149627348b61d4140fc58b745875:satech'
[+] SHA-1 
[+] Double SHA-1 
...
```

Bem, `SHA-1`, mas a dúvida é qual tipo? Já que tenho apenas o salt e não a senha, tentei primeiramente o `HMAC-SHA1 (key = $salt)`:

```bash
$ hashcat -a 0 -m 160 'cdeb746ec095149627348b61d4140fc58b745875:satech' /usr/share/wordlists/rockyou.txt.gz
...
cdeb746ec095149627348b61d4140fc58b745875:satech:<PASS_4>    
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 160 (HMAC-SHA1 (key = $salt))
Hash.Target......: cdeb746ec095149627348b61d4140fc58b745875:satech
...
Started: Mon Apr 20 12:27:37 2026
Stopped: Mon Apr 20 12:27:39 2026
```

E, por acaso, realmente era. Consegui a senha, <PASS_4>.


### Hash 5

Hash fornecido: `362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912`

```bash
$ hashid 362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912
Analyzing '362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912'
[+] Snefru-256 
[+] SHA-256 
...
```

Testando com `SHA-256`:

```bash
$ hashcat -a 0 -m 1400 '362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912' /usr/share/wordlists/rockyou.txt.gz
...
Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 1400 (SHA2-256)
Hash.Target......: 362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963...852912
...
Started: Mon Apr 20 12:29:39 2026
Stopped: Mon Apr 20 12:29:52 2026
```

Exausto. Novamente, a dica diz ser uma senha de 6 caracteres minúsculos ou dígitos. Reaplicando o modo força-bruta com a regra `-1 ?l?d`:

```bash
$ hashcat -a 3 -m 1400 -1 ?l?d '362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912' '?1?1?1?1?1?1'
...
362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912:<PASS_5>
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1400 (SHA2-256)
Hash.Target......: 362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963...852912
...
Started: Mon Apr 20 12:31:16 2026
Stopped: Mon Apr 20 12:31:37 2026
```

Obtive a última senha, <PASS_5>.


[^hashid]: https://psypanda.github.io/hashID/
[^hashcat]: https://hashcat.net/hashcat/
[^rockme]: https://weakpass.com/wordlists/rockyou.txt