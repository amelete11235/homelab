# Chapter 5:  
##### *October 12th, 2023*
* SANS CTF from SEC504
* Website Recon
* DNS Vulnerability
* Insecure Camera Subdomain
* SQLi 
* Command Injection
* Reverse Shells
* msfvenom and msfconsole
* meterpreter
* Process Migration / Privilege Escalation
---
# Intro

I bricked my host and started a part-time job, so I had to take a break from my home lab while I rebuilt everything and got adjusted.

I got some advice on my home lab that I ought to try out Kali Purple in order to streamline my home lab and practice war games. While I enjoy messing around with networking and configuring various new machines, my goal here is to practice offensive and defensive operations. In addition, I'm tired of running into memory limitations, so I'll be setting up my other laptop to run it as the offensive box using Kali.

In the mean-time I decided to complete the capstone CTF from SANS's SEC504, which I have yet to complete since it wasn't required for course completion. That's what this chapter will be about. I'm not sure how much I can share here without breaking the rules. I'll have to be a little bit cryptic in this journal entry.

## Website Reconnaissance

I've been tasked by the company, ISS Playlist to help them with some security investigations.

To start out, I was given a website where I was tasked with finding different pieces of information about a target company. I used my browser to discover email formats, some employee PII, a music playlist creation app, and take in the environment.

![](a/11c5118b703dbc4d758b597f6308c733.png)

![](a/2dc241fa0b5212eddb668062e1fd29ec.png)

![](a/0bd8ac78d14d49c4ecdb6500e907e564.png)

![](a/d329746d6f77ef858c77fcabe4a6be7e.png)

![](a/5294048699720564733b2b60f7078b7c.png)

![](a/abc286bef2d89d5ffaf20d78566b47e2.png)

Now I've got some email, usernames, and full names to start with if I need to perform some security testing later.

I've been directed to investigate a webapp bulletin called clippedbin[.]com

Right off the bat, I found an apache struts CVE: 2018-11776 vulnerability, a reference to a webmin CVE-2019-15107 vuln, and several recently leaked passwords from the ISS Playlist website. I also found a subdomain pointing to a security camera remote administration page on the ISS Playlist website, along with default credentials. They've all been posted by someone who's going by *BSH*. This site is being used as a drop zone for hackers! I'll add these passwords to a wordlist as well.

I also found a mysterious list of what appear to be hashes, prefixed with "bnb".

![](a/49473eebaec707b1c4825efa4232470c.png)

My customers let me know that there was some proprietary info (a flag) that they're looking for on the site. I quickly found the flag by performing a basic search, but I was curious to see if I could find anything else.

![](a/8138bea90c44525c4a804098ec7678d1.png)

## Mystery Cipher

I found some suspicious communications on the web app:

![](a/4483b7087dbfbdbca2ac9b1d0f52efeb.png)

![](a/0057f7c6ea8e7427b6d0f44608805103.png)

Looks like a salted password. I'll come back to this one later.

## DNS Enumeration

I've been directed to investigate a vulnerability at one of the company's DNS servers.

![](a/966a5a86a1ac92e5b24f540c85ab8810.png)

## Directory Enumeration


## Insecure Camera Subdomain

I took a look at the insecure remote camera domain to see what the attacker saw. Based on what I saw there, the attacker has discovered some critical information. "BSH" now knows that his victims are on to him, and aware of his presence. In response to this discovery, the attacker will likely take steps to enhance his persistence on the network. Alternatively, the attacker might cover tracks, cut losses, and trigger ransomware. This is why it's important to avoid letting on to the adversary that you've initiated an IR protocol.

## SQLi

![](a/08489ccd9e6fa43daf7c6fe743a1795b.png)

I found an SQLi bug in their playlist generation demo on the main site. I'll try some manual SQLi, and then an automated tool. 

![](a/25dbe0c1adc4673ea5acc316ef1d99b7.png)

![](a/cac118901b77cbd1c1aaca2b2e1288eb.png)

For the kick of it, I tried the automated tool's -a flag, which retrieves everything, instead of enumerating the tables one-by-one. It dumped a password hash table with usernames, a flag, and a bunch of other useful information:

![](a/e6bd20245ce831fb0ac3b07a0fb6cdc0.png)

![](a/e2c40e348fb6173d6ab6646b4b9ffcb9.png)

After I finished exploring the very long output from the -a flag, I decided to investigate the tables one-by-one for more concise results.

![](a/7d10543b62a7b81dbc7ee976761385c1.png)

I saved the password table I found for use with my favorite tool ever: hashcat!

![](a/e5776d4209586682b9ddb75e42d45673.png)

![](a/47ea4cbdc8cfea697aebb1357f500a01.png)

I'll crack the rest of these in a future entry.
## Command Injection

![](a/faeabdd625a408163659981a654db8bb.png)

On a subdomain for song uploads, I found an input field that errors when you don't enter the ID of a valid submission. After looking around I found the robots.txt file, inside of which was a disallow entry which I used to test in the input field

![](a/5eee0f1a17d0e842a12eda0f9dfd5374.png)

This resulted in a successful output and another flag, though I still needed to find a way to leverage this. I tried following the valid ID up with several attempts at linux command injection such as `&&`, `||`, and `;` but none of them were working. I took a hint from the sec504 curriculum and I also decided to check out https://github.com/payloadbox/command-injection-payload-list to figure out how I could exploit this. It turns out that I wasn't interacting with a linux webserver:

![](a/9b79875b60d3348dfd64c40e7903581b.png)

![](a/bdb4d80b1b3d2221c114a8ad3d0b5ad7.png)

Injection successful!

I saw an interesting file that probably contained some hashes, and might even be related to our adversary, *BSH,* and I'd rather explore that a little bit more comfortably, so I'm going to make that happen with my admin privileges.

# msfvenom Reverse Shell

![](a/1517851b4b3591d79d21d547c2967efb.png)

I served it up with an expedient python http.server.

![](a/07c86940f74ebbdce11af439ccda25b3.png)

![](a/a2717b396c9aeedb60f66690bb19d52c.png)

## LOL *Living off the land*

I used the certutil.exe Microsoft tool listed by the LOLBAS project to send my msfvenom reverse shell via a simple python server. https://lolbas-project.github.io/lolbas/Binaries/Certutil/

![](a/d0be665742597e4c0f067e47204b5739.png)

![](a/fc795518a6c045e42a427935955e23fb.png)

![](a/39a0505a3b379dc044b1c458c1655f57.png)

That feels awesome!

![](a/fd9d0e307f26e056a1a99c1e316af3ca.png)

![](a/b05e6cb05d3263b9f704fcf027912d27.png)

And that feels even better!

## Privilege Escalation

![](a/24d526f83e2371be0e8769f91da6e784.png)

![](a/f61e4f4742b18edd644dc83de6b8c557.png)

![](a/6697cae907db4c7eb43214ddf8f27173.png)

![](a/63d931b0aee78ea3ec5910318122749e.gif)

## That's enough for an entry. Thanks for reading! See you next time.