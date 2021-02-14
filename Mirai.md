### Mirai Writeup

## Overview
This machine shows the discovery and exploitation of IoT devices. There have
been so many user and root owns for this device, so this abridged writeup will
serve as a minimal walkthrough while the full writeup will serve as
an illustrative guide for the relative beginner with real-world tips and tricks
thrown in to prepare the reader for real-world pentests or for incident
response encounters. The full writeup with illustrations, diagrams, and links
can be found on the project's github page as well as the Spanish and Japanese
versions of the full writeup.

## Walkthrough
### Scanning
The very first scan is an nmap scan with augmented flags that fork from the
usual options. The scan we'll be using is:
`sudo nmap -p- -T4 -A -v 10.10.10.48 > mirai-scan`

The results show open ports:
* 22
* 53
* 80
* 1453
* 32400
* 32469

Right away we see that the default ssh and http ports are open which is a
normal starting place for challenges like this. Don't shy away from doing a
full port scan or changing the options if not enough information is gleaned
from the initial scan; it's better to be safe with full knowledge rather than
continuing blind. In this specific scenario, we had to include the verbosity
flag and change the timing to get a workable amount of information.

Another type of initial scan we'll conduct is a web fuzzing scan for
directories and interesting files (html, php, and txt primarily) using ffuf and
dir-buster with wfuzz's megabeast list and one of dir-buster's lowercase lists.
What results is a small list of directories with a notable `/admin` result.
### Gaining Access
Going to the admin portal shows a good deal of information: the device is a
Raspberry Pi running Pi Hole, maybe we can try the default credentials to log
in (always look for default credentials for *ANY* service you come across!).
Using the credentials `pi:raspberry`, we are able to get access to the system,
easy as pi (sorry). Let's grab the user.txt file in the `Desktop` directory and
let's move on to root.

### Maintaining Access
For this challenge, there aren't any circumstances that require us to set up
special tools for maintaining a connection like cron jobs resetting network
connections every five minutes, so we can just note the login credentials and
move on.

### Privilege Escalation
There are three major things you want to do when escalating privileges (all
have to do with information gathering):
* Checking running network services
* Check running processes (pspy is a great utility)
* See what you can get away with (permissions)

There weren't any interesting processes or network services, but on running the
command `sudo -i` (which you should always do on login) elevated us directly to
a root shell; that was pretty neat.

There's something wrong here though... root.txt isn't in the home directory,
but there's a file that says a copy might be on their usb stick. There are two
ways we can get the location quickly:
* Understand that usb devices mount to the `/media` directory and go there
* Understand that computers are stupid and check with `lsblk` just in case

Upon `cd`'ing into the `/media/usbstick` directory, we see that there's another
file, though not the one we want. It looks like we'll have to recover the data
ourselves with `dcfldd` or another utility. We can create a `/tmp` folder to
hold the output of the restoration. Neato, now open it up with vim and grab the
hash. Congratulations!

### Covering Tracks
The job's not over yet, we still have tracks on the system, like a
`.bash_history` file and the `/tmp` folder we used to restore the root hash.
Just get rid of these and we should be set to remove our access entries from
`/var/log` and head out.

## Conclusion
### Mitigation of Vulnerabilities
With more information in the full writeup, it's safe to say that you should
never keep default credentials on anything you set up. Secondly, consider each
user's permissions and what you'd like them to do with sudo, and often it's
best to limit any users with elevated privileges.
