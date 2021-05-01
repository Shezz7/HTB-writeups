# Emdee five for life

Navigating to the site, we see the following

[!efff1](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff1.png)

Try to md5 hash the string displayed and submit it.

[!efff2](https://raw.githubusercontent.com/Shezz7/HTB-writeups/master/challenges/resources/efff2.png)

It says that we were too slow to submit the hash. The application probably has some sort of time limit to submit the hash of the string it generates. We can attempt to write a simple script to take in the displayed string, md5 hash it and post it back to the application.
