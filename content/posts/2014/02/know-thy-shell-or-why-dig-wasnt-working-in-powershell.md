+++
title = "Know thy shell, or why dig wasn't working in PowerShell"
date = 2014-02-19T15:00:00-05:00
draft = false
tags = ['powershell', 'bind', 'dns']
+++
An application I work on utilizes DNS for important aspects of its functionality.  The other day, I wrote a quick and dirty command line application to write zone files for a static set of domains.  The app would allow a developer to double-click an .exe and set up a known set of domains in a local Bind instance for test purposes without having to know how or take any manual steps. It was a one-time-per-developer app so I didn't put a lot of thought or time into it.

Or, at least, I didn't mean to put a lot of time into it.  I ran the executable which wrote the zone files, restarted Bind, and ran the following command in an already opened PowerShell terminal to verify that my test domains were being served by my local Bind instance.

<!--more-->

```
PS C:\Bind\bin> ./dig @localhost ns1.dnstest.dev.local

; <<>> DiG 9.9.4-P2 <<>> ns1.dnstest.dev.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 45832
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1280
;; QUESTION SECTION:
;ns1.dnstest.dev.local.        IN      A

;; AUTHORITY SECTION:
dev.local.           3600    IN      SOA     dev-ad-01.dev.local. hostmaster.dev.local. 6876 900 600 86400 3600

;; Query time: 4 msec
;; SERVER: 10.0.5.2#53(10.0.5.2)
;; WHEN: Tue Feb 18 18:09:52 Central Standard Time 2014
;; MSG SIZE  rcvd: 131
```

`status: NXDOMAIN` in the answer section of the response means the DNS query failed. What gives? Now, if you are a seasoned PowerShell user, you probably already see the problem. As a PowerShell novice at best, I did not.
## Why it didn't work

It appeared Bind wasn't properly serving my domains. I double checked the generated zone files and at first glance, they looked fine. I checked the Windows event log and didn't see anything out of the ordinary. In fact, the logs that Bind generated claimed to load my domains!

Well, I did have a hunch.  I fired up gitbash, which is where I normally do DNS queries with dig anyway, and here's what happened.

```
nelson.wells@C11NWELLS /c/bind/bin
$ dig @localhost ns1.dnstest.dev.local

; <<>> DiG 9.9.4-P2 <<>> @localhost ns1.dnstest.dev.local
; (2 servers found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 44602
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;ns1.dnstest.dev.local.        IN      A

;; ANSWER SECTION:
ns1.dnstest.dev.local. 3600 IN A       127.0.0.1

;; AUTHORITY SECTION:
dnstest.dev.local. 3600 IN     NS      ns1.dnstest.dev.local.

;; Query time: 6 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Tue Feb 18 18:08:32 Central Standard Time 2014
;; MSG SIZE  rcvd: 89
```

It worked! The status of the DNS query for that domain was NOERROR and the answer section is what I expected it to be.  As it turns out, the @ symbol is a special character in PowerShell.  Instead of interpreting the @localhost as a parameter to the executable, PowerShell interpreted it as a [splat](http://technet.microsoft.com/en-us/magazine/gg675931.aspx).  Since there's no PowerShell variable by the name localhost, the parameter is empty and thus ignored.  Dig uses the machine's default DNS servers, and my fake domain clearly does not exist on the public internet.

What started out as a 15 minute throw-away tool unfortunately turned into a 30 minute one. Doh!
