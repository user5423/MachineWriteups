### Paper (Easy)


There is a relatively high amount of enumeration that needs to be performed on this machine


#### Enumeration 

We'll first start with a simple nmap scan to find what services are running publicly

`nmap -Pn -sC -sV -p- paper.htb -oN paper.scan`

- `-Pn` : Skip host discovery
- `-sC` : Perform default nmap scripts
- `-sV` : Perform versioning detection
- `-p-` : We scan all ports
- `-oN filename.txt` : Write normally to the file `filename.txt`

Our scan reveals that there are three ports open
- Port 22: SSH
- Port 80: HTTP (Apache)
- Port 443: HTTPS (Apache)
And that we might be running a CentOS service

We focus on HTTP and HTTPS since these services normally have larger attack surface areas
than SSH.

Accessing the both services through `https://paper` and `http://paper` we are responded
with an 'HTTP Test Page'. This coroborates from our scan that both of these services are
Apache running on CentOS Linux distribution.

... [img]

The most common reasons for receiving a HTTP Server Test Page is:
1. The site is under development, and the default page hasn't been changed
2. The server is uneccessarily running an apache service (less common)


If the site is under development, then it's possible there are resources we can still access
, but not through the default apache index.html page. We perform a directory brute-force attack
to see if we can find any files that won't be linked from the default page

`gobuster dir --url http://paper.htb --wordlist /usr/share/wordlists/dirb/common.txt`

... [img]

No Luck. All we see are files associated with the apache manual and other apache resources.
These are unlikley to be an attack vector.

It's possible that there may be other subdomains being provided on the same target server (vHosts)
Therefore, let's perform a subdomain enumeration attack:

`wfuzz -c -f scans/subdomains.scan -w top5000subdomains.txt -H "HOST: FUZZ.paper.htb" -u "http://paper" --hc 403`

- `-c` : print output in color
- `-f filename.txt` : output results to `filename.txt`
- `-w wordlist.txt` : use the wordlist `wordlist.txt`
- `-u http://domain` : Fuzz the url targetting the user
- `-H header:value` : Use the header in requests - In our case, we are fuzzing this header
so that we can look for vHOSTS. Doing this in the `-u` flag wouldn't work since the dns service
would try and fail to resolve the IP address for it. With vHOSTS all the subdomains are on the
same server.
- `-hc XXX` : hide responses that returned the HTTP Code `XXX`

... [img]

Success !!! We found a "200" HTTP OK response for the subdomain office.paper.htb


#### Enumerating http://office.paper.htb

Accessing this page we are presented with an off-brand 'Dunder-Mifflin' paper company blog

... [img]


