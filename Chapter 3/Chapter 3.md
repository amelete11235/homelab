# Chapter 3:  
* Setting up another barebones server for Suricata
* More resource management
* Initial nmap scan
##### *October 4th, 2023*
---
# Intro
In this chapter, my goal is to set up a Suricata IDS on my VLAN, modify it's .yaml file and watch for my own malicious activity.

# Suricata Setup
### Adapter Config
![](a/57837351015ae15e9b66974b9223f98e.png)
![](a/73a7ed16ee379547cc62df496451c93f.png)
![](a/555142a9de2e947a50b68329252a643a.png)
I modified the `/etc/network/interfaces` file so that it connects enp0s8 to the vbox intnet at boot. I'm leaving the virtual network cable for enp0s3 unplugged and I set it to connect with the vbox NAT whenever I need to download some packages from the internet. I did the same with my evil Parrot box.
### Real-world considerations
I suppose that a corporate environment would have a proxy set up, and perhaps an update caching server for compartmentalizing any updates or software installs to this IDS, but I'm not a security architect so I've never built a corporate network from scratch, and so I'm going to enable a vbox NAT whenever I need to download anything for now. Later on, I'd love to set up a firewall, content filter, vpn gateway, and web proxy with deep packet inspection using another laptop so that I can compartmentalize the machines on my laptop into a "trusted" network segment. I doubt that I create enough traffic for it to be a big performance hit, but it will be interesting to see how my hardware holds up when I complete that project.
### suricata.yaml
![](a/5147adb1e859f47173f3b713a54dd667.png)
![](a/c8d0e799a051bbdd0d679f0f441a20e5.png)
![](a/088d74b5789f9d972f4af57324f0eb6e.png)
# More Resource Management
Some packages were hanging when I tried running all four VM's at once. I'm going to install a new DC running server 2022 core, transfer the system state from the server 2019, and then decommission the original DC.
![](a/d4dcdb9c15078ef2816e1a606065cb20.png)
![](a/7a8bafa8e9acc4b4de31feabeafa3f40.png)
![](a/f755f96774874c67c9d1b99448a448a6.png)
![](a/9c0fc12304f8cfe23ef7c2d7707c0b46.png)
![](a/a8daaddd19fdd7a528db8acc8d1e5582.png)
![](a/eb3fcd05ae8fdd2aa0e64a51ee1370ff.png)

After quite a lot of breaking things and rebuilding them, I managed to get them up and running, minus another windows host, but I have 2 gigs of memory left according to htop, so I'm going to do more optimizing later, or I'll set up my other laptop and connect to the VLAN.
# Making some noise 
I'll run some basic nmap scans against Suricata and the DC just to see if anything shows up in Suricata logs or the event viewer with default settings.
### Scanning the DC
![](a/94661829d96ce1b822c6b70f6221b3af.png)
![](a/7f50954223b7a009f1bd81ce93fd4ca6.png)
![](a/b6e591bc791e0ee7abbab3e549c152c0.png)
### Scanning Parrot "Architect Edition" w/ Suricata
![](a/aaf450f8b92d437f060c346ca4bd5471.png)
*I can see that SSH is open by default. It must have to do with Parrot OS or remote admin for Suricata*

![](a/957558ad3b30aeb79bb3e751beea8c32.png)
![](a/579768a535c3185777865ca183194b73.png)
# See you next time!
That's enough for today. There's a whole lot of breaking and fixing stuff behind the scenes so I've been doing a lot more than what you can see in these journal entries. I'll start again tomorrow to look at the logs, and then I plan to configure suricata in-depth.