# Emdee five for life

Navigating to the site, we see the following

[!efff1](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff1.png)

Try to md5 hash the string displayed and submit it.

[!efff2](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff2.png)

It says that we were too slow to submit the hash. The application probably has some sort of time limit to submit the hash of the string it generates. We can attempt to write a simple script to take in the displayed string, md5 hash it and post it back to the application.

## Examining the request

Examining the initial request using the browser, we see the following:

[!efff3](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff3.png)

A cookie is set in ```PHPSESSID``` which is the default identifier that PHP uses for cookies. While building our script we have to take this into account.
