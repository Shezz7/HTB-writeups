# Jerry Writeup

This is a short writeup of HTB Jerry, an easy Windows box

## Port scanning

We start by scanning open ports on the box with nmap to see what we're dealing with. Running a TCP scan for all ports alongwith service/version detection, we get the following results:

```console
kali@kali:~/Desktop/htb/jerry$ nmap -sT -sV 10.10.10.95 -A -p0-65535
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-04-02 04:19 EDT
Nmap scan report for 10.10.10.95
Host is up (0.039s latency).
Not shown: 65535 filtered ports
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache Tomcat/Coyote JSP engine 1.1
|_http-favicon: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Apache-Coyote/1.1
|_http-title: Apache Tomcat/7.0.88
```

Note: Usually I like to run a UDP scan in the background just to cover all bases. Since it is very time consuming, it makes sense to initiate the scan and proceed with further enumeration in the meantime. In this case, the UDP scan yielded no results.

From the nmap scan above, we can draw the following conclusions:

- Port 8080 open
- Runs Apache Tomcat/Coyote JSP engine 1.1
- Version: Apache Tomcat 7.0.88
- No UDP ports open
- OS: Windows (HTB gives us this information)

## Service enumeration

We're dealing with only 1 port i.e. 8080 which is supposedly running a web server. Visiting the webpage on a browser, we see the following:

![webpage-1](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-1.png)

On the homepage we can see "manager app" which could possibly lead us to the admin console. Clicking on that button we see that it requires us to enter a username and password:

![webpage-2](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-2.png)

Trying out random creds such as ```admin:admin``` lands us a 403 and we see the following:

![webpage-3](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-3.png)

On the 403 page, we see that for this version of Tomcat (7.0.88), the default admin credentials are ```admin:s3cret```. Trying this out on the admin login, we get access to the admin console:

![webpage-5](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-5.png)

The credentials worked and this is our most lucrative option. We can see that there is an option to deploy war files and execute them. WAR (Web Application Archive) files are basically files used to distribute a collection of JAR-files, JavaServer Pages, Java Servlets, Java classes, XML files, tag libraries, static web pages and other resources that together constitute a web application.

Let's take a step back here and assume that we hadn't seen the default credentials on the 403. Having a quick look at how the login actually works. We fire up BurpSuite and turn intercept on to examine the traffic. Let's use the credentials ```blah:blah``` to see how it works:

![burp-1](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/burp-1.png)

Looking at the requests we see that it uses Basic Auth which just base64 encodes the username and password and forwards it.

![burp-2](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/burp-2.png)

We can decode the base64 string in BurpSuite as follows:

![burp-3](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/burp-3.png)

We can use [Hydra](https://github.com/vanhauser-thc/thc-hydra) to bruteforce the login. There are various wordlists that we can employ. A popular repository of wordlists is [SecLists](https://github.com/danielmiessler/SecLists) which contains various wordlists, one of which is default Tomcat passwords as shown below:

![hydra-1](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/hydra-1.png)

We can now initiate the Hydra bruteforce as follows:

```console
kali@kali:~/Desktop/htb/jerry$ hydra -C /usr/share/seclists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt -s 8080 10.10.10.95 http-get /manager/html

Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-04-03 04:34:21
[DATA] max 16 tasks per 1 server, overall 16 tasks, 79 login tries, ~5 tries per task
[DATA] attacking http-get://10.10.10.95:8080/manager/html
[8080][http-get] host: 10.10.10.95   login: tomcat   password: s3cret
[8080][http-get] host: 10.10.10.95   login: tomcat   password: s3cret
1 of 1 target successfully completed, 4 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-04-03 04:34:22
```

Concluding the service enumeration, its safe to assume that we have a very lucrative exploitation path with the admin console at our disposal. We can prioritize our exploitation paths as follows:

1. Admin console exploitation
2. Server software vulnerability assessment
3. SSH bruteforce

## Exploitation

Before we get into actual exploitation, we can do some more enumeration on the admin console now that we have access to it. This is done to understand the nature of the machine so we gain more information on how to proceed with our exploit. Exploring the admin console, we discover the following:

![webpage-6](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-6.png)

We now have the following:

- JVM Version: 1.8.0_171-b11
- JVM Vendor: Oracle Corporation
- OS Name: Windows Server 2012 R2
- OS Version: 6.3
- OS Architecture: amd64
