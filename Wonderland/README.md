# Enumeration
We beginning by scanning for open ports. Autorecon tells us that onlyports 22 and 80 are open.

We use an nmap scan to confirm, then follow up with a detailed scan.

![1402bc9fc0a7aa8de486270f52fc47aa.png](./_resources/1402bc9fc0a7aa8de486270f52fc47aa.png)

![dd191c17809e05702459689f56922653.png](./_resources/dd191c17809e05702459689f56922653.png)

![0bafb4ef1e14d2cdcb1cf2a2c7a42ad1.png](./_resources/0bafb4ef1e14d2cdcb1cf2a2c7a42ad1.png)

Let's take a look at the web server.

![fbc72a09302f0f96f7836442a73368ee.png](./_resources/fbc72a09302f0f96f7836442a73368ee.png)

Enumerating the directories with gobuster gives us a few options.

```bash
gobuster dir -u http://10.10.253.179 -w /usr/share/dirb/wordlists/common.txt -x html,php,txt,jpg,jpeg,png
```

![70a6d8f1790fe8ed47652da1a1a526ff.png](./_resources/70a6d8f1790fe8ed47652da1a1a526ff.png)

(**Note**: If we take a look at the autorecon scan results, we see that feroxbuster caught another directory: the `/poem` directory. Navigating here displays (as you might expect) a poem, but nothing else. Gobuster didn't find any subdirectories.)

The  `/img` directory contains three images.

![5aaabf272f41dbe304444525046cce31.png](./_resources/5aaabf272f41dbe304444525046cce31.png)

We'll download them and analyze them using some [steganography tools](https://0xrick.github.io/lists/stego/). Using stegseek, we find a hint within one of the image files. We can also use steghide with an empty password.

![9516f6f0794d7e870fc44e31dcb2e808.png](./_resources/9516f6f0794d7e870fc44e31dcb2e808.png)

![44bef80c61d6ea9d4c715ee00c07a20d.png](./_resources/44bef80c61d6ea9d4c715ee00c07a20d.png)

We also found a directory titled `/r`. 

![67dc54632ef7310134c58c98ba998226.png](./_resources/67dc54632ef7310134c58c98ba998226.png)

Let's follow the page's advice and look for further subdirectories. Eventually we find `/a`.

![b1eaf558c936338e8f4f57af1fc15925.png](./_resources/b1eaf558c936338e8f4f57af1fc15925.png)

![a19624f55dd83be5bf3fa1582588b7ed.png](./_resources/a19624f55dd83be5bf3fa1582588b7ed.png)

You can probably see where this is going. Repeating the gobuster search using our newly-found subdirectory, we eventually find:

![ab54cd471f6a9bc3b3ded91f0540bc96.png](./_resources/ab54cd471f6a9bc3b3ded91f0540bc96.png)

![e1147705e1a179ed5f92b0c058035e63.png](./_resources/e1147705e1a179ed5f92b0c058035e63.png)

![26f8db86f3f6fa1d35392a266a2a5a48.png](./_resources/26f8db86f3f6fa1d35392a266a2a5a48.png)

![e832aa8397dd299747d5f6c511d3e4ad.png](./_resources/e832aa8397dd299747d5f6c511d3e4ad.png)

The last page also includes the `alice_door.png` we found earlier. Another gobuster scan relative to the `/r/a/b/b/i/t` subdirectory reveals that there are no more subdirectories to find here. Taking a look at the source code reveals what appear to be credentials. The only other service we have to work with is ssh, so let's try that there.

![549531c09a6bd445b86f096292d451db.png](./_resources/549531c09a6bd445b86f096292d451db.png)

![ee37e297354afa6e91748ee52716b474.png](./_resources/ee37e297354afa6e91748ee52716b474.png)

Success!
<br>

# Exploitation
Now that we have initial access to the machine now, let's take a look around. We see a `root.txt` file immediately, but unfortunately we don't have access.

![75ba39ae5687e5b678037d20b581386e.png](./_resources/75ba39ae5687e5b678037d20b581386e.png)

Let's use `sudo -l` to see if Alice can run anything as sudo.

![f70324630690d03d93feb712433594dc.png](./_resources/f70324630690d03d93feb712433594dc.png)

We see that Alice can execute the python script in her home directory using the rabbit's privileges. This will be relevant later when we try to escalate privileges. For now, we still need the user flag.

At this point, we realize that it is somewhat strange that the root flag is contained in the user directory (since, typically, the root flag is in the root directory). Following the hint and making a guess, we use `cat /root/user.txt`, and it works!

![d3d7387b4e0179a6457367719630a62d.png](./_resources/d3d7387b4e0179a6457367719630a62d.png)

<br>

# Post-Exploitation
Running through standard Linux playbook, we eventually find that our current user has some capabilities set. We can check this with `getcap -r / 2>/dev/null`.

![e5f476b7ee1fa3645104059f0b26ca17.png](./_resources/e5f476b7ee1fa3645104059f0b26ca17.png)

We can also use [LinPEAS](https://github.com/carlospolop/PEASS-ng/releases/tag/20220717) to highlight potential privesc methods. Among many other things, LinPEAS gives us a list of files with capabilities.

![bf841acca9d1f33483c3c798495e9d27.png](./_resources/bf841acca9d1f33483c3c798495e9d27.png)

Checking [GTFOBins](https://gtfobins.github.io/gtfobins/perl/#capabilities), we see that there is a way to exploit this to escalate privileges. However, Alice doesn't seem to have permission to run perl. We'll keep this in mind for later.

Going back to Alice's home directory, note that there is also a python script. Running it produces a random list of lines from a poem which can be found in the python script.

![4d7b7a4905a5f496e3d8f28acfb70121.png](./_resources/4d7b7a4905a5f496e3d8f28acfb70121.png)

Using `ls -l` , we see that we can't write to the python script. But, if we look at the script we see it begins by importing the `random` library.

![67db64f5f0dc930d69e90d9cca4b75f9.png](./_resources/67db64f5f0dc930d69e90d9cca4b75f9.png)

So let's try creating our own `random.py` script; by placing it in our home directory, we can force the python script to spawn a shell with root privileges. Our `random.py` will look like:

```py
import os

os.system("/bin/bash")
```

Using `sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py`, we can run python3.6 (applied to the python script in Alice's home directory) as the rabbit. This spawns a shell with the rabbit's privileges.

![059aebbce465d1c27d35ddc900fcafa9.png](./_resources/059aebbce465d1c27d35ddc900fcafa9.png)

Checking the rabbit's home directory, we find a file called `teaParty`. Running it gives us a message.

![6b5fcd3b091900dbc0c3a8f13a062a58.png](./_resources/6b5fcd3b091900dbc0c3a8f13a062a58.png)

The date printed is precisely one hour after the current time. Running the file again verifies this. So, it seems like the `teaParty` binary is making use of the `date` command. Printing out its content verifies this.
 
![4c32184cfd8696942b2b52228559343a.png](./_resources/4c32184cfd8696942b2b52228559343a.png)

We can make use of this by altering the `$PATH` variable and creating our own script called `date`. In this way, when `teaParty` runs it will run our `date` script first. To do this, we add a path to the rabbit's home directory as follows.

![ae73568b6620f48b231e3eb0e3e7686f.png](./_resources/ae73568b6620f48b231e3eb0e3e7686f.png)

Our `date` script will look like:

```txt
#!/bin/bash
/bin/bash
```

Using `chmod +x date`, we make the file executable. Now running `teaParty` gives us a shell as the hatter.

![f0998a3e81b84333f548c28121c95965.png](./_resources/f0998a3e81b84333f548c28121c95965.png)

Within the hatter's home directory, we find a `password.txt` file.

![d0512ddddb095a20d95ccbd15c17b3ff.png](./_resources/d0512ddddb095a20d95ccbd15c17b3ff.png)

We can verify that this is the hatter's ssh password. Using this, we can run `sudo -l` to see if the hatter can run anything with sudo privileges.

![bae8cd11426156efb4d18d08da8876ed.png](./_resources/bae8cd11426156efb4d18d08da8876ed.png)

Nothing. However, recall that we discovered earlier that perl has capabilities set. Before we couldn't make use of this since we didn't have permission to run perl. Going back to the `/usr/bin` directory, we see that the hatter can execute perl.

![b9644318f2d71e158b394695dde73cbd.png](./_resources/b9644318f2d71e158b394695dde73cbd.png)

So, now we can actually make use of the exploit we found on GTFOBins. For this to work, we simply run the following command from within the `/usr/bin` directory. Furthermore, we need to spawn a full shell as the hatter, otherwise we can't run perl. This is why we needed the hatter's ssh password.

After ssh'ing into hatter's account and navigating to `/usr/bin`, we simply run the following command.

`./perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'`

![8fbda7ac5ede7dcbb23ce7057356159d.png](./_resources/8fbda7ac5ede7dcbb23ce7057356159d.png)

Success! We have a root shell. Since we already know where the `root.txt` flag is, we just `cat` it and we're done!