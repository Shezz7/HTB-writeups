# Delivery writeup

This is a short writeup of ```Delivery```, an easy Linux box.

We start by scanning open ports on the box with nmap to see what we're dealing with. Running a TCP scan for all ports along with service/version detection, we get the following results:

## Port scanning

We start by scanning open ports on the box with nmap to see what we're dealing with. Running a TCP scan for all ports along with service/version detection, we get the following results:

```console
kali@kali:~/Desktop/htb/delivery$ nmap -sT -sV 10.10.10.222 -p 0-65535
Starting Nmap 7.91 ( https://nmap.org ) at 2021-05-21 14:10 EDT
Nmap scan report for 10.10.10.222
Host is up (0.037s latency).
Not shown: 65533 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http    nginx 1.14.2
8065/tcp open  unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 104.22 seconds
```

Note: Usually I like to run a UDP scan in the background just to cover all bases. Since it is very time consuming, it makes sense to initiate the scan and proceed with further enumeration in the meantime. In this case, the UDP scan yielded no results.

From the nmap scan above, we can draw the following conclusions:

- Ports 22, 80 and 8065 are open
- OpenSSH version is 7.9p1 Debian 10+deb10u2
- Nginx 1.14.2 on port 80
- OS: Linux

## Service enumeration

There are only 3 ports to deal with here and since SSH is usually pretty reliable, we can start with port 80. Visiting the webpage on a browser, we see the following:

![homepage-1](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/homepage-1.png)

The website uses a template from HTML5 UP from https://html5up.net/. The "contact us" page looks like the following:

![contact-us](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/contact-us.png)

The helpdesk link directs to http://helpdesk.delivery.htb. Adding the subdomain to the hosts file we get:

![homepage-2](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/homepage-2.png)

It looks like the website uses osTicket, an open source ticketing system.

Running a directory scan we get the following results:

```console
kali@kali:~/Desktop/htb/delivery$ gobuster dir -u http://helpdesk.delivery.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://helpdesk.delivery.htb
[+] Threads:        200
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/06/13 12:08:52 Starting gobuster
===============================================================
/images (Status: 301)
/pages (Status: 301)
/apps (Status: 301)
/assets (Status: 301)
/css (Status: 301)
/includes (Status: 403)
/js (Status: 301)
/kb (Status: 301)
/api (Status: 301)
/include (Status: 403)
/scp (Status: 301)
/included (Status: 403)
/includemanager (Status: 403)
/includedcontent (Status: 403)
===============================================================
2021/06/13 12:10:10 Finished
===============================================================
```
On port 8065 we have the following webpage:

![mattermost-1](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/mattermost-1.png)

Googling what Mattermost is we get the following:

> Mattermost is an open source collaboration tool for developers. As an alternative to proprietary SaaS messaging, Mattermost brings all your team communication into one place, making it searchable and accessible anywhere. It’s written in Golang and React and runs as a production-ready Linux binary under an MIT license with either MySQL or Postgres.

**Results:** 

10.10.10.222:80 

- Contact Us page which says that a user should register on a helpdesk to get access 
- Link to Mattermost server 
- HTML5up template used

helpdesk.delivery.htb 

- Support ticket system powered by osTicket (customer support software) 
- Sign in page: http://helpdesk.delivery.htb/login.php  
- Sign up page: http://helpdesk.delivery.htb/account.php?do=create 
- Sign in page for agents: http://helpdesk.delivery.htb/scp/login.php 
- Open new ticket: http://helpdesk.delivery.htb/open.php 
- Check ticket status: http://helpdesk.delivery.htb/view.php 

10.10.10.2222:8065 

- Mattermost login 
- Create account: http://10.10.10.222:8065/signup_email 
- Forgot password: http://10.10.10.222:8065/reset_password

## Exploitation

### User

We can try to create a ticket on the helpdesk and see what happens. Creating a ticket with random info:

![create-ticket-1](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/create-ticket-1.png)

The success message shows us a support email and a ticket ID.

![create-ticket-2](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/create-ticket-2.png)

Viewing our newly created ticket looks as follows:

![create-ticket-3](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/create-ticket-3.png)

We can now go to Mattermost and sign up with the support email and see what happens:

![mattermost-2](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/mattermost-2.png)

Once the account is successfully created, we get a prompt to verify the email:

![mattermost-3](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/mattermost-3.png)

Heading back to our ticketing system we can see that we have recieved a verification link in a comment:

![create-ticket-4](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/create-ticket-4.png)

Following that link we get:

![mattermost-4](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/mattermost-4.png)

Now we can sign in and we can get into what looks like the company's private channels:

![mattermost-5](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/mattermost-5.png)

Here we see the following message:

> *@developers Please update theme to the OSTicket before we go live. Credentials to the server are maildeliverer:Youve_G0t_Mail!*
> *Also please create a program to help us stop re-using the same passwords everywhere.... Especially those that are a variant of "PleaseSubscribe!"*
> *PleaseSubscribe! may not be in RockYou but if any hacker manages to get our hashes, they can use hashcat rules to easily crack all variations of common words or phrases.*

Trying the credentials ```maildeliverer:Youve_G0t_Mail!``` with SSH we get a user shell:

```console
kali@kali:~/Desktop/htb/delivery$ ssh maildeliverer@10.10.10.222
maildeliverer@10.10.10.222's password: 
Linux Delivery 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan  5 06:09:50 2021 from 10.10.14.5
maildeliverer@Delivery:~$ whoami
maildeliverer

```
