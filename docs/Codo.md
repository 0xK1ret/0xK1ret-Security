# Codo Writeup
**Name:** Codo  
**OS:** Linux    
**Difficulty:**  Easy  
**Complexity:**  Low


## Phase 1: Reconnaissance  
  
To begin, we spawn in the machine and wait for it to assign an IP address. In this case, it appears we will be attacking `192.168.132.23`  

## Phase 2: Scanning
Initial enumeration was performed using an automated script I created to streamline the reconnaissance process. This included a fast port discovery scan followed by a detailed service/script scan and a sweep of top 100 common UDP ports. Results were then placed into the nmapscans directory.  
  
![Image of our nmap scan](assets/codo/enumeration-1.png)  
  
While the more detailed scan is running, I opted to take some time to check out the web server on port 80. Visiting `http://192.168.132.23` - we are met with a webpage titled "CODOLOGIC". We can take note that this is powered by "Codoforum". There appears to be a link, register, and login page in the upper right hand corner. We can additionally see that the user 'admin' made a post called "Welcome to Codoforum". In addition to the NMAP scan, I had started a simple GoBuster scan to enumerate sub directories of the web server.  
  
![Image of the webpage](assets/codo/enumeration-2.png)  
  
We can see that there a few interesting directories that we can explore. I begin by checking out `sys`. After combing through these files, there was no juicy information. Moving onto `cache`, and `sites` the same holds true.  
  
![Gobuster Scan](assets/codo/enumeration-3.png)  
![Image of the sys](assets/codo/enumeration-4.png)  
![Image of the cache](assets/codo/enumeration-5.png)  
![Image of the sites](assets/codo/enumeration-6.png)  
  
Finally, we get to `admin`. I checked the source to see if I could gather a version or build number, with no luck.  
  
![Image of the login page for Codoforums](assets/codo/enumeration-7.png)  
  
![Image of the page source for the login page](assets/codo/enumeration-8.png)  
  
To our luck, after trying a few basic common passwords manually - `admin:admin` appears to get us into the login! From here we are able to determine the version `V.5.1.105`.  
  
![Image of the dashboard and version number](assets/codo/enumeration-9.png)  

## Phase 3: Vulnerability verification & Exploitation  
  
In the terminal, I check to see if searchsploit has any known exploits - which it does! `CodoForum v5.1 - Remote Code Execution (RCE)` exists - so I opt to try it out and make a copy.  
![Searchsploit](assets/codo/enumeration-10.png)  
  
Reviewing the code, it seems pretty straight forward - it takes arguments in the form of a `target-url`, `username`, `password`, `listener-ip` and `port`.  
  
![The code of our copied searchsploit script](assets/codo/enumeration-11.png)  
  
In the terminal, I make sure that python3 will work with this script.  
  
![Check check](assets/codo/enumeration-12.png)  
  
Finally, I attempt to run it:    

![Failure - but thats ok](assets/codo/enumeration-13.png)  
  
Failure. No worries - it looks like we are able to do this manually! I navigate to https://www.revshells.com and pull the `PHP Ivan Sincek` reverse shell, using `/bin/bash`. I opt to use `443` as the listening port because it is less likely to be blocked and noticed. 
![revshells site](assets/codo/enumeration-14.png)  
  
Uploading this via the directions in the terminal, we land a shell as the user `www-data` - Nice!  
  
![location of our malicious shell.php](assets/codo/exploitation-1.png)  
![our newly landed shell as www-data](assets/codo/exploitation-2.png)  
  
I take a moment and upgrade my shell after determining python3 is located at `/usr/bin/python3`.  
  
![Checking where python3 binary is and using it to upgrade our mininmal shell](assets/codo/exploitation-3.png)    
  
Checking out the `/etc/passwd` file, we can see that the user `offsec` exists.  
![Gobuster Scan](assets/codo/post-exploitation-1.png)  
  
Moving to `/` we can don't see anything out of the ordinary.  
  
![Gobuster Scan](assets/codo/post-exploitation-2.png)  
  
Checking out `/var/www` we can see the directories we previously were exploring via browser.  
  
![Gobuster Scan](assets/codo/post-exploitation-3.png)  
  
After poking around here - I move to the temp directory, make a folder named `k1ret` to keep it tidy and transfer over `linpeas.sh`, `PwnKit` & `lse.sh`  
  
![Gobuster Scan](assets/codo/post-exploitation-4.png)  
  
![Gobuster Scan](assets/codo/post-exploitation-5.png)  
  
Running through linpeas, we notice that one of the files `/var/www/html/sites/default/config.php` has a password notated.  
  
![Gobuster Scan](assets/codo/post-exploitation-6.png)  
  
## Phase 4: Maintaining access & privilege escalation  
  
I open up this config file and see that this is actually the password to for mariadb.  
  
![Gobuster Scan](assets/codo/post-exploitation-7.png)  
  
Connecting to mariadb - I list the databases and choose to review codoforums in case anything stands out.  
  
![Gobuster Scan](assets/codo/post-exploitation-8.png)  
![Gobuster Scan](assets/codo/post-exploitation-9.png)  

After chasing this - I realized that this was not the intended path for priv esc. Out of curiousity, I tried the password against the root user and discovered I could switch the user to root with this password!  
  
![Gobuster Scan](assets/codo/post-exploitation-10.png)  
  
At this stage, several persistence mechanisms are available to maintain access to the system. One high-reliability method involves generating an SSH public/private key pair, as the current configuration permits root login (a setting that we can modify if necessary). Alternatively, a cron job could be established to ensure a persistent callback to a Command and Control (C2) server. The choice of implementation depends on the specific operational requirements and the desired level of stealth.  
  
## Phase 5: Reporting & Documentation  
  
The initial compromise was facilitated by the use of default administrative credentials to access the Codoforum dashboard. While various publicly available PoC's existed (CVE-2022-31854) and were identified, access was ultimately gained through manually uploading a malicious PHP payload via the forum’s logo upload functionality. This resulted in a Reverse Shell and a foothold on the underlying system as the `www-data` user.

During post-exploitation reconnaissance, the `config.php` file was discovered to contain cleartext MariaDB credentials. Due to credential reuse, these same credentials provided us with access to the root user account, allowing for full system compromise.
