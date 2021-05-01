# Emdee five for life

Navigating to the site, we see the following

![efff1](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff1.png)

Try to md5 hash the string displayed and submit it.

![efff2](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff2.png)

It says that we were too slow to submit the hash. The application probably has some sort of time limit to submit the hash of the string it generates. We can attempt to write a simple script to take in the displayed string, md5 hash it and post it back to the application.

## Examining the request

Examining the initial request using the browser, we see the following:

![efff3](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff3.png)

A cookie is set in ```PHPSESSID``` which is the default identifier that PHP uses for cookies. While building our script we have to take this into account.

## Getting the displayed string

To get the displayed string we first do a simple curl:

```console
kali@kali:~$ curl http://206.189.121.131:30299/
<html>
<head>
<title>emdee five for life</title>
</head>
<body style="background-color:powderblue;">
<h1 align='center'>MD5 encrypt this string</h1><h3 align='center'>mHDGmix66zipSFZUAUM0</h3><center><form action="" method="post">
<input type="text" name="hash" placeholder="MD5" align='center'></input>
</br>
<input type="submit" value="Submit"></input>
</form></center>
</body>
</html>
```
We can then filter the string out using ```grep``` and ```curl```:

```console
kali@kali:~$ curl -s http://206.189.121.131:30299/ | grep h3 | cut -d '>' -f 4 | cut -f 1 -d '<'
ObO2YLXyrOu0sw57HgC4
```

## Calculating the MD5 hash

Now that we have the string, we can pipe this in to ```md5sum``` to get the md5 hash.

```console
kali@kali:~$ curl -s http://206.189.121.131:30299/ | grep h3 | cut -d '>' -f 4 | cut -f 1 -d '<' | md5sum
021b1ae9acdfd446bdc204c3bc8f1df9  -
```

The trailing hypen needs to go because we will post the hash in the field. To do so we do the following:

```console
kali@kali:~$ curl -s http://206.189.121.131:30299/ | grep h3 | cut -d '>' -f 4 | cut -f 1 -d '<' | md5sum | cut -d ' ' -f 1
125428a5f9dc739151c926579c243e8d
```
