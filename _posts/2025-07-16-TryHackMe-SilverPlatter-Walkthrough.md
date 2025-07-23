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
5. Since I was stuck during the privesc phase, I had to give a peek to a video in the course that essentially says "I am not gonna tell you yet how to privesc this machine, but I am going to tell you that it has something to do with the fact that tim belongs to the adm group." Sincerely, I don't know how many times I had already looked at the logs and didn't see the password, but once I knew that the solution HAD to be in the logs, I was finally able to see it; so what did I spend the last 7 days doing? In short, chasing rabbit holes, but it does not mean that it was all for nothing, in fact I believe that trying and failing is how I will eventually learn how to hack faster and better.

# Reconnaissance (information gathering)

#### Action: Go to the /tmp directory
Do not forget to move to the /tmp directory:

```
┌──(username㉿machinename)-[/~]
└─$ cd /tmp
```

This way when we download something or try to output a new file, we will not get "permission denied" errors because we have  no write access to whatever directory we are in; this is especially relevant in our target machine, because the user "tim" we will eventually impersonate has a home directory that is owned by root, so tim himself does not have write access to his own home!

## Port Scanning
We start with port scanning, that normally means using nmap; in fact, we are going to use nmap *indirectly* via rustscan because rustscan is fast, and we like not having to wait too long; this and other outputs can be quite long, so for the sake of this document, we are going to cut all outputs down to their essential parts.

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
4   149.33 ms silverplatter.thm (10.10.105.52)
```

From the TRACEROUTE section we see that the domain name is **silverplatter.thm** with IPv4 address **10.10.105.52**

#### Action: Add a line to the /etc/hosts file
The file **/etc/hosts** works as a sort of local DNS by mapping domain names to IP addresses.
We add both the domain and the IPv4 address to the /etc/hosts file, so we will be better able to reuse our CLI commands even if the IP address of the VM changes (for instance, because the VM's timeout expires), so inside the **/etc/hosts** file we add this line:

```
10.10.105.52    silverplatter.thm
```

Of course, every time the IP address changes, we will have to keep updating it inside this file as well.

#### Action: ping silverplatter.thm
Let's verify that **/etc/hosts** works by trying to ping silverplatter.thm

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

#### Note
Not all CLI commands support converting the domain names in **/etc/hosts** so with experience we will have to learn which ones do (like ping) and which ones don't.

## 22/tcp (ssh) OpenSSH 8.9p1 [part 1]
From the port scanning output we gather that OpenSSH is on version 8.9p1; in general, ssh is not the first thing we would consider hacking, because it is very secure, and this machine is no exception, so we move on.

## Our Strategy for ports 80 and 8080
Let me give you a litte bit of a foreshadow: while examining port 80, we will talk about using hydra hoping to find the password for a user named scr1ptkiddy.

But in order to make hydra work correctly, we will have to already know that we need to point hydra toward port 8080.

So, how do we know that 8080 is the port we are supposed to use with hydra? Simple, when we look at the CONTACT page on the web site on port 80, we will learn about the existence of a Silverpeas server; because of such information disclosure, of course we will try to point our web browser at **http://silverplatter.thm:8080/silverpeas** (notice that the port is 8080 and that silverpeas has been appended to the end of the URL) and discover that there is indeed a login page for Silverpeas hiding at **http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp** .

In short:
1. We pointed the web browser to the web site behind port 80 and from its CONTACT page we discovered that there is a user scr1ptkiddy on a Silverpeas server.
2. We got a hunch that the Silverpeas server may be located at port 8080 (where rustscan/nmap told us that an http-proxy lives), and so we pointed our web browser to http://silverplatter.thm:8080/silverpeas and discovered a login page located at **http://silverplatter.thm:8080/silverpeas/defaultLogin.jsp** .
3. We now know that we need to point hydra to port 8080 to try finding the password for scr1ptkiddy.

## 80/tcp (http) nginx 1.18.0 (Ubuntu)
From the port scanning output we gather that nginx (static web server) is on version 1.18.0 and that the operating system is Ubuntu Linux.

### Any nginx vulnerabilities?
We should look for vulnerabilities for version 1.18.0 of nginx, but in this case, after extensive googling, I couldn't find any exploitable vulnerability, so I moved on.

### silverplatter.thm:80
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

#### View Source (look at the source code)
It is also important, when looking at a web site, to use the View Source feature available in all modern web brosers and see whether there are any obvious pieces of interesting information left by the developers in comments (things like users, passwords, and generally speaking, any type of information disclosure that would help us gain an advantage in our attempt to get into the machine.) Unfortunately, in this particular web site's source code there is nothing noteworthy to point out.

#### cewl
Since we know about a user named scr1ptkiddy for a Silverpeas server running on the machine we are attacking, it is worth collecting all keywords from the pages of the web site, and later use them with a program like hydra to search the password for scr1ptkiddy. Just to clarify, we are talking about collecting all the words that make the text of the http://silverplatter.thm web site and store them in a single file, one word per line, with the intention of using it later with a program like hydra to find the password for scr1ptkiddy.

Here is the command to scan the web site and collect all its keywords in a file named **silverplatter_keywords.txt**:

```
┌──(kali㉿kali)-[/tmp]
└─$ cewl http:silverplatter.thm > silverplatter_keywords.txt
```

This command collects all keywords in silverplatter.thm and saves them into a file.

#### hydra
We are going to use hydra to *password spray* all the keywords in **silverplatter_keywords.txt** in an attemot to find the password for user scr1ptkiddy.

##### hydra requires several parameters.
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
└─$ curl -v -d "Login=scr1ptkiddy&Password=wrongpassword&DomainID=0" http://silverplatter.thm:8080/silverpeas/AuthenticationServlet
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

# Exploitation (use information gathered during reconnaissance)
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
└─$ ssh tim@silverplatter.thm #Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
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

Normally, this would be a concern, and on a production machine we would better ask the system administrator whether things are fine, but since this is a virtual machine we are connecting to, and it is pretty common for servers to be rebuilt/reinstalled in VM environments, we can safely proceed issuing the command the warning above suggests:

```
┌──(kali㉿kali)-[/tmp]
└─$ ssh-keygen -f '/home/kali/.ssh/known_hosts' -R 'silverplatter.thm'
# Host silverplatter.thm found: line 1
# Host silverplatter.thm found: line 2
# Host silverplatter.thm found: line 3
/home/kali/.ssh/known_hosts updated.
Original contents retained as /home/kali/.ssh/known_hosts.old
```

And we can try a second time to ssh into our target machine:

```
┌──(kali㉿kali)-[/tmp]
└─$ ssh tim@silverplatter.thm #Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
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

Notice how tim's home is owned by root, so tim cannot write in his own home!
This is one good reason to move to a directory where most users have write access:

```
tim@ip-10-10-30-149:~$ cd /tmp
tim@ip-10-10-30-149:/tmp$ ls -lad
drwxrwxrwt 12 root root 4096 Jul 23 01:24 .
```
Notice that everybody can do pretty much anything in this directory.


# Lessons Learned
Key takeaways from this challenge...
