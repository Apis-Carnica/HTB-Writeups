## Blocky
##### Ursa
### Overview
This machine, based on a Minecraft server discovers the dangers of
public-facing credentials and poor password policies. There's a bit of poor
privilege management as well. At the time of writing, there are plenty of user
and root owns, so this writeup will focus on theoretical knowledge and covers
some tips and tricks for managing information.

### Walkthrough
#### Scanning
The nmap scan performed shows the normal ports (`22` & `80`) and going to the
website shows that this is a minecraft server. There is a user found: `notch`
that we'll use later for ssh. Upon further port scanning, we
can see the default minecraft server port `25565` is open as well. Using
`ffuf` and dirbuster's medium wordlist, we can see several interesting
directories. In the `/plugins` directory, there's a java file which contains
SQL credentials (which work for ssh and phpmyadmin.

#### Gaining Access
There are a few ways to gain access to this machine, but this page covers ssh.
If you'd like to know more, go check out my full writeup [here](https://github.com/nateac1/HTB-Writeups/blob/master/Blocky-en.pdf).
Using the username `notch` with the password found in the java file, we have
access to the machine, and can find the `user.txt` file in the user's home
directory.

#### Maintaining Access
Not much is to be done about maintaining access; just take down the login
credentials and take notes of all actions you do on the system. Screenshots and
diagrams are also very helpful when trying to figure out what you've done in
the past.

#### Escalating Privileges
Escalating privileges on this machine was rather simple; by issuing the
`sudo -i` command and supplying the user password, we were able to gain root
access on the machine.

#### Covering Tracks
Starting from initial access, if you decided to log into the phpmyadmin or
Wordpress pages, then you can delete all changes you made to the users or
databases. After that, any system account you accessed, from `www-data` to
`notch`, should have the access logs deleted as well (found in `/var/log`).
Finally, if any temporary directories or files were created, they should be
taken out as well, closely followed by the access logs for the root user. We
can now log off the box.
### Conclusion
This was overall a nice simple box with a few things to learn about keeping
user and password files for login attempts (since we can try to mix and match).
The biggest challenge was adding PHP code to the Wordpress site to get a remote
shell, but you didn't have to do that to gain access to the machine. Thanks for
your time, I look forward to learning with you again soon!
