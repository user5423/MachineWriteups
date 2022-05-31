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

Accessing this vhost we are presented with an off-brand 'Dunder-Mifflin' paper company blog

... [img]

** Discovering Technologies used**

Viewing the source code of the index page, we look at the client-side dependencies 
(e.g. the src values in the <script> tags). This isn't a foolproof method, as its possible
that a web application framework has both both server-side and client-side components.

In our case, we see several scrpt dependencies to a local directories called `wp-includes`,
`wp-content`, which are used by the Wordpress system (i.e. Wordpress = "wp"). Often times, the
footer of a webpage likely has a reference/link to the technology that it uses to work. This isn't
always true, but in the case of CMS applications like Wordpress, this is a common occurrence.

...

Looking at the `<meta/>` tag in the page source-code, we find that the version of Wordpress
has been disclosed (5.2.3). This will likely be very important later if we need to look for
vulnerabilities corresponding to this version of the software.

...

Before checking for vulnerabilities with the WP software, let's explore a bit more.

The `http://office.paper.htb` index page is a blog with several posts. Take a look, and note
any details or information regarding the users, security, or interesting facts.

From the three posts, the users we find are:
- Prisonmike
- nick
- Creed Bratton

In the blog post "Feeling Alone", nick mentions that there is "secret content" in his drafts
that is **insecure**

...

This might have sensitive information that could allow us to leverage a future attack.

#### Finding the Secret Draft

I made a few different attempts to find this secret draft:

**An Empty Search**

One way a draft could be insecure is that it is accessable, but not index (e.g. via 
index.html). Running an empty search for posts can return all posts. 

This returned two more posts
1. **Test** post - This had no valuable information
2. **Warning for Michael** - This post reiterated nicks security concerns that he commented
under Prisonmike's post "Feeling Alone"

However, we did not find the drafts


**Directory Traversal**

I tried brute-forcing the directory structure for where the other posts stored. Attempts
included trying to brute-force common post names, different dates 
(http://office.paper.htb/index.php/YYYY/MM), and trying to find directories that had links
to posts.

However, none of these were successful

**Wordpress Vulnerabilities**

We previously found that the website was managed and created using Wordpress 5.2.3. We know
that CMS's like Wordpress are used by bloggers. This means that Wordpress might provide
functionality to create, store, and release drafts. 

A bit of google searching shows that wordpress provides the abilitiy to create drafts
(https://www.wpbeginner.com/glossary/draft/). So maybe this wordpress version might be
vulnerable to accidental disclosure of drafts to unprivileged users?

Searching "wordpress 5.2.3" on https://www.exploit-db.com/ returns two exploits:

...

The first result seems very related to the draft security concern that Nick mentioned.
Following the instructions we access `http://office.paper/?static=1`, and are returned
with a block of text. After reading the text, we find a interesting **registration** link to the new
"Employee Chat System". I think we've found something good!

...


http://chat.office.paper/register/8qozr226AhkCHZdyY
