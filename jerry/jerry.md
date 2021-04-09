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
