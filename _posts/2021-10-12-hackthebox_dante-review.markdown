---
layout: post
title:  "Hackthebox Dante Review"
date:   2021-12-15 19:00:00 +1100
author: Stephen Bradshaw
tags:
- dante
- hackthebox
- tips
- review
---
<p align="center">
  <img width="460" height="300" src="https://www.hackthebox.eu/images/press/dante/2.jpg" alt="Dante">
</p>


A while ago at my work we got an Enterprise Professional lab subscription to HackTheBox. With this subscription, I had a chance to complete the Dante Pro lab a few months ago, so I thought I'd do a review of it here.

The Enterprise Pro lab subscription gives you dedicated access to one lab at a time, and seeing that Dante is the "Beginner" lowest difficulty level lab in the Pro labs series, this was the first environment we had provisioned.

The description of Dante from HackTheBox is as follows:

> Dante Pro Lab is a captivating environment that features both Linux and Windows Operating Systems. You will level up your skills in information gathering and situational awareness, be able to exploit Windows and Linux buffer overflows, gain familiarity with the Metasploit Framework, and much more! Completion of this lab will demonstrate your skills in network penetration testing.
> 
> This Penetration Tester Level I lab will expose players to:
> * Enumeration
> * Exploit Development
> * Lateral Movement
> * Privilege Escalation
> * Web Application Attacks
> 
> 14 Machines and 26 Flags! Take up the challenge and go get them all!


This is a pretty accurate desciption of whats involved, although there is certainly more stuff in some categories than there is in others. I had to do only one each of a custom Windows and Linux buffer overflow exploit, but there was a whole heap of enumeration, privesc and web application exploits required, plus lateral movement to a sometimes ridiculous degree.

To a large extent Dante can be described as a collection of a whole lot of individual HackTheBox machines. If you have done some of the HackTheBox system challeges, you'll be familiar with the pattern of exploiting a service or application to gain access as a regular user, grabbing a flag, privescing to root/admin, and then grabbing another flag. There is a lot of that in Dante.

There were definitely a lot fewer dependencies between machines in the Dante network than I expected. There are a few cases where you will need to gather some intel from another box to gain an initial foothold on certain systems you can access quite early on, and using owned boxes as pivots to reach restricted subnets is necessary. However, that was about it in terms of interconnectivity. Extensive dependencies between machines is a feature of the more difficult Pro labs in the series.

I also found that Dante had a number of challenges that were quite contrived and unrealistic. In fact, this almost turned me off the lab entirely in the beginning, because the worst aspects of this are seen early on. If you have spent a certain amount of time pentesting and red teaming, you start to get a feel for the types of decisions users, sysadmins and developers make when they use/manage/build systems. You can learn to anticpiate the mistakes they are likely to make, and you select the techniques you will use accordingly, prioritising some and sometimes ignoring others entirely. This can wrong-foot you badly in Dante.

If you find youself hitting a wall early on, and you have dismissed the use of certain techniqes commonly taught to pentesting beginners because they dont make sense in context, I'd suggest you try them anyway. If something is going wrong with some bit of loot you have found, dont reject certain possible ideas for why that might be happening because "a real user wouldn't do that". Think of Dante more as a test of your ability to reproduce various pentesting techniques rather than a realistic network, and be prepared for system configurations and artefacts that would only exist as a result of a delierate attempt to troll someone trying to exploit a system.

This is my one main gripe with Dante, but luckily it is mostly an issue early on in the lab, and once you're past it (and are accounting for it in your approach) things are a lot more enjoyable.

From the opposite perspective, one thing I really **_liked_** about Dante is that it provides excellent experience for making you comfortable with operating through pivots. You start Dante by gaining access to a network environment where you can access one machine (that you need to first identify through scanning). You need to compromise this machine in order to proceed, and from there on, everything you do will be through _at least_ one pivot.

<figure>
<p align="center">
  <img width="460" height="300" src="https://i.kym-cdn.com/photos/images/newsfeed/000/531/557/a88.jpg" alt="Pivots galore">
  <figcaption style="text-align: center">We need more pivots</figcaption>
</p>
</figure>


I've heard of lots of different approaches that people use to deal with this. Some people swear by [sshuttle](https://github.com/sshuttle/sshuttle). My own approach was to use a combination of the following:
* Regular old ssh sessions, making use of a bunch of helpful ssh features
* A [http to socks proxy](https://github.com/oyyd/http-proxy-to-socks) to allow Burp to selectively route traffic both to the Dante network and to the Internet. Certain sites in Dante will load very poorly if you cant access Internet based resources loaded within some pages, and Burp's regular SOCKS proxy option is all or nothing
* [proxychains4](https://github.com/rofl0r/proxychains-ng) for enabling regular tools to work through the dynamic SOCKS proxies created by ssh
* Meterpreter payloads on pivot hosts to use Metasploit's routing capabilities

For ssh I made sure to use keys for authentication - if a key wasn't already in place for a user on a system, I would add my own key (one generated specifically for Dante). I would also setup an entry for the host in my ssh config file. This made it much more convenient to use ssh for various purposes, including enabling quick and easy transfers of files to and from the host. This also made it much easier to chain multiple ssh connections together via the _ProxyCommand_ option. 

An example entry might look something like the following:

    host dante-host2
        Hostname 10.10.10.10
        ProxyCommand ssh dante-host1 -W %h:%p
        User root
        IdentityFile ~/.ssh/id_rsa_dante
        DynamicForward 1080
        Port 22


Hopefully most of this is pretty straightforward, but two lines might benefit from some explanation. 

The _DynamicForward_ option would open a SOCKS proxy on port 1080 on my local host. Traffic sent through this proxy would then be routed through the remote host. This is great for forwarding traffic from my host to the network locally attached to the remote ssh host. I would add one of these (with a unique port) to each ssh config entry I wanted to use as a routing point for other traffic.

I would use these to route traffic from Burp using the afore mentioned http to socks proxy, but I would also have individual proxychains4 config files for each SOCKS proxy port. I would use these when running proxychains4 to route traffic to the appropriate networks, using like so:

    proxychains4 -q -f proxychains_1080.conf <command> <options>

One tip for using proxychains is to ensure that if you are running an interpreted program (like a Python script) its a good idea to explicitly reference the Python binary before that script, even if the script starts with a hash bang, e.g.:

    proxychains4 -q -f proxychains_1080.conf python python_script.py

Without this specific reference to the script interpreter, sometimes the traffic generated from the script will fail to be routed through the proxy as you intended, and the network connection will fail.

The _ProxyCommand_ option refers to another proxy config entry in the same file named "dante-host1". This causes your ssh client to first open a connection to dante-host1, and to then tunnel the connection to dante-host2 through that session. So basically, this auto pivots you through dante-host1 to reach dante-host2. You can chain these entries together as well, and have a similar entry for dante-host3 with a _ProxyCommand_ entry referring to dante-host2, which would then go through host1 and host2 to reach its final destination of host3. This is a massive convenience when you have to ssh into a host that requires multiple hops to be accessed.

I would occaisionally also use local forwarding on an ad hoc basis, and when this was required I would use the ssh escape sequence (which by default is ~C) to access the ssh command line and create them as needed. If you're not familiar with the ssh escape sequence, when the appropriate key combination is pressed at the regular ssh command prompt as the next keypress after _Enter_ it drops you to a special prompt like so:

    ssh>

At this prompt, you gain the ability to enable a number of ssh options without having to enable them from the command line when establishing a new session. You can, for example create a local port forward from port 8888 to the remote host and port 172.16.1.1:8000 in the current session like so:

    ssh> -L 8888:172.16.1.1:8000

Metasploit was another tool I used frequently throughout Dante, and using its routing options to pivot as needed was very helpful. With the appropriate routes in place you can use Metasploit from your host like it was directly connected to targets in other networks.

To use the routing you need to have a Meterpreter type payload on the host you want to use to pivot, and then modify the Metasploit routing table using the _route_ command in the Metasploit console. As an example, if you want to route traffic to network 172.16.2.0/24 through the Meterpter agent on session 2, you would run the route command as follows:

    route add 172.16.2.0 255.255.255.0 2

If you follow my suggested method of using ssh with keys to connect to compromised hosts on the network, another nice trick you can try with Metasploit is to use the "auxiliary/scanner/ssh/ssh_login_pubkey" module to get a shell session on those hosts in Metasploit. As an example, to connect to host 10.10.10.10 as user root with key ~/.ssh/id_rsa_dante, do the following: 

    use auxiliary/scanner/ssh/ssh_login_pubkey
    set USERNAME root
    set RHOSTS 10.10.10.10
    set KEY_PATH ~/.ssh/id_rsa_dante
    exploit

Then, when you get your session opened, you can upgrade it to Meterpreter so you can do routing with it using the upgrade option. To upgrade session 1, you would run a command like the following, which would open the Meterpreter session to that host with the next available session number:

    sessions -u 1

You can also use Metasploits SOCKS proxy module (auxiliary/server/socks_proxy) to forward traffic from external tools in a manner similar to the ssh proxying approach already mentioned. This also uses Metasploit's inbuilt routing for traffic forwarding, so it allows you to direct other attack traffic via proxychains to any system that Metasploit can talk to.  I didnt use this too much in DANTE, as there were ssh servers that could be used to route traffic where it was required. In cases where you have to chain through servers not running ssh however, this can be very useful. This approach does also have the advantage of allowing a single SOCKS port to direct traffic to multiple different network subnets - as long as Metasploit can talk to that segment via its routing through Meterpreter agents.

Another tip I would give is to keep comprehensive notes. Using a heirarchical note taking app, so you can keep categorised notes of your progress, is a great idea. Cherrytree is one option. I used Joplin during Dante, but it did start to grind a little once the pages got bigger, so Im still looking for alternatives. 

I had individual pages for each host, where I would keep information related to each host, and a few other pages to summarise various other bits of information about the network.

I took notes of:
* The flag values I collected, where I found them, and the name that the flag had in the HackTheBox Dante progress page 
* Credentials I foiund on any machine, to make it easier to try and reuse those credentials on other hosts
* Each machine I discovered on each network segment. I had a summary page listing all hosts I had found so far and whether I had obtained root/system on each, and a page for each machine to take notes of things I tried in exploiting it, and how I eventually got access
* A playbook for reestablishing access to systems, especially key tunneling hosts. If the lab gets reverted, or you need to get back into hosts you have already exploited to check if any critical information has been missed, this will be a big time saver. Using existing credentials or keys that will survive lab reversions is always the best option where available


There are also a few other random tools and resources that proved very helpful during Dante. These include:
* [Peas](https://github.com/carlospolop/PEASS-ng) - A privilege escalation checker for Linux and Windows
* [pspy](https://github.com/DominicBreuker/pspy) - A tool for snooping on process execution events on Linux with unprivileged users 
* [hacktricks.xyz](https://book.hacktricks.xyz/) - A wiki collecting a bunch of hacking techniques that I referred to a lot durung Dante

I hope this review gave you a good idea of what the Dante pro lab is like, and some useful tips in how to operate in it. I did enjoy the experience of doing the lab, and am planning to do a few more HackTheBox Pro labs when time permits.