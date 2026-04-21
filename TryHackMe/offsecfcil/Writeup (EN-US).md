# [OffSec - HASH](https://tryhackme.com/room/offsecfcil)

Original Capture The Flage available at [Try Hack Me](https://tryhackme.com/room/offsecfcil).

Solved at: `2026/04/20`

# Table of Contents

- [OffSec - HASH](#offsec---hash)
- [Table of Contents](#table-of-contents)
- [Writeup](#writeup)
   * [Summary](#summary)
   * [Hash Cracking](#hash-cracking)
      + [Hash 1](#hash-1)
      + [Hash 2](#hash-2)
      + [Hash 3](#hash-3)
      + [Hash 4](#hash-4)
      + [Hash 5](#hash-5)

# Writeup

## Summary

The challenge consists in breaking the given hashes.

## Hash Cracking

For this CTF, the objective consists solely on breaking hashes, without virtual machines or other pentest subjects. Each task is "obtain the original value from the hash".

In general, the solution steps will be the same for all tasks (except where specified): first I'll use `hashid`[^hashid] or [hash-analyzer](https://www.tunnelsup.com/hash-analyzer/) to identify what hash is it, then use `hashcat`[^hashcat] or [crackstation](https://crackstation.net/) for actually breaking the hash.

The command I'll use with `hashcat` uses the rockme[^rockme] wordlist:
- `hashcat -a 0 -m <hashtype> '<hash>' /usr/share/wordlists/rockyou.txt.gz`

### Hash 1

Given hash: `482c811da5d5b4bc6d497ffa98491e38`

```bash
$ hashid 482c811da5d5b4bc6d497ffa98491e38
Analyzing '482c811da5d5b4bc6d497ffa98491e38'
[+] MD2 
[+] MD5 
...
```

Trying it out with `MD5`:

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

Got the first password, <PASS_1>.


### Hash 2

Given hash: `861c4f67e887dec85292d36ab05cd7a1a7275228`

```bash
$ hashid 861c4f67e887dec85292d36ab05cd7a1a7275228
Analyzing '861c4f67e887dec85292d36ab05cd7a1a7275228'
[+] SHA-1 
[+] Double SHA-1 
...
```

Trying it out with `SHA-1`:

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

Second pass acquired, <PASS_2>.


### Hash 3

Given hash: `4149c5cc4c378444d116d65ad5ba4099`

```bash
$ hashid 4149c5cc4c378444d116d65ad5ba4099        
Analyzing '4149c5cc4c378444d116d65ad5ba4099'
[+] MD2 
[+] MD5 
[+] MD4 
...
```

Trying it out with `MD5`:

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

Exhausted. The tip says it is a password with 6 lowercase or digit characters. So, I defined the custom rule `-1 ?l?d` so to include that restriction and then used `hashcat` on the bruteforce tool, still with `MD5`:

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

Once more exhausted. I, then, tried cracking it as `MD4`:

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

Acquired the pass, <PASS_3>.


### Hash 4

Given hash: `cdeb746ec095149627348b61d4140fc58b745875`
Given salt: `satech`

```bash
$ hashid 'cdeb746ec095149627348b61d4140fc58b745875:satech'
Analyzing 'cdeb746ec095149627348b61d4140fc58b745875:satech'
[+] SHA-1 
[+] Double SHA-1 
...
```

Well, `SHA-1`, though which kind? Since I only had the salt (and not the password), I first tried `HMAC-SHA1 (key = $salt)`:

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

And, by sheer luck, it really was it. One more pass, <PASS_4>.


### Hash 5

Given hash: `362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912`

```bash
$ hashid 362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912
Analyzing '362fda2183b7ac73400a83f6ab2c359451e48adf6c3d46a2963ee2abdf852912'
[+] Snefru-256 
[+] SHA-256 
...
```

Trying it out with `SHA-256`:

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

Exhausted. Once more the tip says it to be a 6 character lowercase/digit combo, so I replicated the same strategy as earlier:

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

Getting the last password, <PASS_5>.


[^hashid]: https://psypanda.github.io/hashID/
[^hashcat]: https://hashcat.net/hashcat/
[^rockme]: https://weakpass.com/wordlists/rockyou.txt