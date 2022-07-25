# Enumeration

`autorecon 10.10.63.128` discovers open TCP ports 22 and 80. We use `nmap -T4 -sC -sV --version-all --osscan-guess -A -p 22,80 10.10.63.128` for a more detailed scan.
 
![9916f7d7f6c49c76455d68cef78841c2.png](/Game%20Zone/_resources/9916f7d7f6c49c76455d68cef78841c2-1.png)
 
Following the task, we will investigate the http web server. We encounter a login form on the front page which we suspect is susceptible to SQL injection. Input `' or 1=1 -- -` for the username and leave the password blank. This gives us access to the `portal.php` webpage.
 
![5e2ce6cfa84fb3304ac174847809f7ea.png](/Game%20Zone/_resources/5e2ce6cfa84fb3304ac174847809f7ea-1.png)
 
Our next goal is to intercept a search request using BurpSuite. Turn on Foxy Proxy, initiate a search on the portal.php, and catch the request in BurpSuite.
 
![e2b4cafa8bdc0bb0d1f6462c886638d6.png](/Game%20Zone/_resources/e2b4cafa8bdc0bb0d1f6462c886638d6-1.png)
 
Now copy the entire contents of this request into a text file `request.txt`. We will use this file with SQLMap to dump our target's SQL database. Run `sqlmap -r request.txt --dbms=mysql --dump 10.10.63.128` choosing the default options along the way. We find a table named `post`.
 
![fbe24360509738e502c615ceed086581.png](/Game%20Zone/_resources/fbe24360509738e502c615ceed086581-1.png)
 
We also find a hash for user agent47; SQLMap also tells us the hash type.
 
![2bb7f59482fb40ffe0467fda0db6ce38.png](/Game%20Zone/_resources/2bb7f59482fb40ffe0467fda0db6ce38-1.png)

![e0568d22708b92a5afb0e6160102d556.png](/Game%20Zone/_resources/e0568d22708b92a5afb0e6160102d556-1.png)
 
(Note: It might be a good idea to see how to do the SQLi manually. [Here](https://5ysk3y.github.io/thm/guides/gamezone/) is an excellent writeup of this.)

Next, we will use John to crack agent47's hash. We run `john --wordlist=/usr/share/wordlists/rockyou.txt agent47hash.txt --format=Raw-SHA256
` and almost immediately we obtain the corresponding password.

Finally, we know from our enumeration phase that ssh is running on port 22. Using the credentials we found above, we ssh into the machine.
 
![0e454a1c15ac625327ebf80b18ce0298.png](/Game%20Zone/_resources/0e454a1c15ac625327ebf80b18ce0298-1.png)
 
Using `cat user.txt` gives us the flag.

<br>

# Exploitation

Now that we have initial access, we will examine the sockets running on the target machine. We use the command `ss -tulpn` to determine what socket connections are running.
 
![df9519e4b3c196b3e4f105e8425fe0f4.png](/Game%20Zone/_resources/df9519e4b3c196b3e4f105e8425fe0f4-1.png)

![437af3e5986988fec43aeaf29188933d.png](/Game%20Zone/_resources/437af3e5986988fec43aeaf29188933d-1.png)
 
From the above output, we see that there is a service running on port 10000 which is presumably blocked by a firewall rule (as it didn't show up in our nmap scans). We can try using `cat /etc/iptables/rules.v4`, but our current user doesn't have access.

To bypass the firewall, we use [reverse SSH port forwarding](https://blog.devolutions.net/2017/03/what-is-reverse-ssh-port-forwarding/). This technique allows us to "forward" a port on the target system to a port on the local system. In this case, we wish to forward the blocked port to a port on the local machine which we can then access. The command to use here is `ssh -L <local port>:localhost:<blocked port> <username>@<ip>`. The `-L` switch specifies local tunnel mode, in which we forward traffic from the client to our own server. (Using `-R` instead does the opposite, where we forward traffic to the client.)

Specifying all parameters, the command is `ssh -L 9999:localhost:10000 -f -N agent47@10.10.63.128`. We run this command on the attacking machine. Navigating to `localhost:10000` gives us access to the newly-exposed web server.

**Note:** Following the suggestion of another write-up, I had to add in the `-f` and `-N` switches, since the command given by the exercise didn't seem to work for me. An explanation of these switches is given by explainshell; I've included the relevant snippets below.

|  flag   | description    |
| --- | --- |
| -f  | Requests **ssh** to go to background just before command execution. This is useful if **ssh** is going to ask for passwords or passphrases, but the user wants it in the background. |
| -N  | Do not execute a remote command. This is useful for just forwarding ports. |

Navigating to `localhost:9999` shows us that we've successfully forwarded port 10000 on the target machine to our port 9999.
 
![7bde271f26f183e4a55c763fc2e6f1b1.png](/Game%20Zone/_resources/7bde271f26f183e4a55c763fc2e6f1b1-1.png)
 
Running nmap against this port with the standard loopback address of `127.0.0.1` gives us some more information about the service running on this port. This also reveals the version number of the CMS, which we will need later.
 
![46776e313aeb93b9a6af2f01d21235f0.png](/Game%20Zone/_resources/46776e313aeb93b9a6af2f01d21235f0-1.png)
 
Alternatively, we can log in to the CMS using the credentials we found earlier.
 
![c306f5c824a3328468197f69d6d26b0c.png](/Game%20Zone/_resources/c306f5c824a3328468197f69d6d26b0c-1.png)
 
<br>

# Post-exploitation

To finish, we need to escalate our privileges and find the root flag. To this end, we will use Metasploit search for the CMS together with the version number we found. We will use `exploit/unix/webapp/webmin_show_cgi_exec`, which is an exploit for remote command execution. We set the relevant options, then `run`.
 
![51f57f22248b3479f0b72aac6c2ba2ea.png](/Game%20Zone/_resources/51f57f22248b3479f0b72aac6c2ba2ea-1.png)
 
Note that since we have forwarded the target port 10000 to the localhost 9999, we set our options accordingly. Now switch to the created session, and note that we have root privileges.
 
![c31a92ca2cc205c6e80e6d8ede9d23f6.png](/Game%20Zone/_resources/c31a92ca2cc205c6e80e6d8ede9d23f6-1.png)
 
Navigating to `\root` and using `cat root.txt` gives us the flag.
 
![d3ad78ec957be9558eb58cf0c475f19c.png](/Game%20Zone/_resources/d3ad78ec957be9558eb58cf0c475f19c-1.png)