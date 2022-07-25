# Enumeration

We begin with an autorecon scan, followed by a couple of nmap scans for confirmation and more details.

![0edb3bf6a67a39571177d06cfc96e91e.png](./_resources/0edb3bf6a67a39571177d06cfc96e91e.png)

![54e1d49e318d9c10c9492bd058bbb3b9.png](./_resources/54e1d49e318d9c10c9492bd058bbb3b9.png)

![4fa943868827a4625a15a429304569a3.png](./_resources/4fa943868827a4625a15a429304569a3.png)

We find ports 21, 22 and 80 are open. Taking a look at the web server, we find an image of a [well-known crew of bounty hunters](https://en.wikipedia.org/wiki/Cowboy_Bebop) together with a transcript of a conversation. A directory scan reveals nothing of note, so let's investigate ftp instead. Access the ftp server with `ftp 10.10.11.78`. There we can find two files: `task.txt` and `locks.txt`. Let's transfer them over to the local system with `get` and read them. The aptly-named tasks file gives a brief list of tasks.

![c27b1152fc67a891dd5da53ff8cf28df.png](./_resources/c27b1152fc67a891dd5da53ff8cf28df.png)

Meanwhile, the locks file contains a list of (presumably) passwords, which we might be able to use to ssh into the machine.

```txt
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
```

<br>

# Exploitation

Let's create file containing the seven possible usernames we've encountered: the user who wrote the note (which is probably the correct username, given that we found both files above in the same location) plus the six people mentioned on the web page. We'll call this `users.txt`. Now run hydra with `hydra -L users.txt -P locks.txt 10.10.11.78 -t 10 ssh` to brute force the ssh login. Eventually, we find the correct pair of credentials.

![8183750ac3d2afb4a62573723b527f9f.png](./_resources/8183750ac3d2afb4a62573723b527f9f.png)

Now that we have a pair of ssh credentials, we get initial access to the machine. From here, we easily find the user.txt flag.

![75d14f6ba7f2050d0de03d0c20dc1c04.png](./_resources/75d14f6ba7f2050d0de03d0c20dc1c04.png)

<br>

# Post-exploitation

Let's use `sudo -l` to see what our user is allowed to run with sudo privileges.

![c297e53e3cd057c10e2894748ef8fabf.png](./_resources/c297e53e3cd057c10e2894748ef8fabf.png)

Heading over to [GTFOBins](https://gtfobins.github.io/gtfobins/tar/#sudo), we find a quick exploit which we can use to spawn a root shell. Simply run `sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`, and we're done!

![d11f43e43b7255d8564c38b907d760b4.png](./_resources/d11f43e43b7255d8564c38b907d760b4.png)