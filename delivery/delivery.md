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


