Here you can find a few (semi-)detailed write-ups of rooms on TryHackMe, with an emphasis on rooms from the Offensive Pentesting learning path. These notes are mostly intended for personal use, but may be helpful for others who are working through these rooms. For obvious reasons, I've tried to avoid spoilers as much as possible.

Although these are write-ups for TryHackMe rooms, the write-ups don't make too many references to the individual tasks or exercises. My main goal here was to develop a methodology for pentesting and note-taking. Rather than dividing each write-up by task, I opted for a more uniform way of presenting the information. Thus, each write-up consists of three parts:

1. **Enumeration.** This step always begins with an autorecon/nmap scan. Once we've identified any ports of interest, we begin a deeper dive into each of the services running on our ports. Our goal is to identify at least one possible vulnerability which we can exploit in order to gain initial access to the target machine.
2. **Exploitation.** Once we've identified the vulnerabilities, we begin looking for an exploit. This step consists of the execution of the exploit used to gain initial access. This section usually ends once we have obtained the user flag, assuming it wasn't already found earlier.
3. **Post-Exploitation.** Once we have initial access, we try to escalate privileges. This section consists of methods and (some unsuccessful) attempts to gain root access, and ends once we have obtained the root flag.

Useful resources and tools:

- TJnull's [Joplin pentest template](https://github.com/tjnull/TJ-JPT) for note-taking.
- TJnull's [OSCP preparation guide](https://www.netsecfocus.com/oscp/2019/03/29/The_Journey_to_Try_Harder-_TJNulls_Preparation_Guide_for_PWK_OSCP.html).
- John J Hacking's [OSCP preparation guide](https://johnjhacking.com/blog/the-oscp-preperation-guide-2020).
- [HackTricks](https://book.hacktricks.xyz) and [Pentest book](https://pentestbook.six2dez.com) for general pentesting methodology.
- [AutoRecon](https://github.com/Tib3rius/AutoRecon), an incredibly powerful network reconnaissance tool.
- [GTFOBins](https://gtfobins.github.io/), a list of Unix binaries which can be used for privilege escalation.
- Cheat sheets from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Methodology%20and%20Resources), and a useful reverse shell cheat sheet from [pentestmonkey](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).
