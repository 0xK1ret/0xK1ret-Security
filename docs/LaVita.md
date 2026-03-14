# LaVita Writeup  
**Name:** LaVita  
**OS:** Linux  
**Difficulty:**  Intermediate  
**Community Rating:**  Hard


## Phase 1: Reconnaissance  
  
To begin, we spawn in the machine and wait for it to assign an IP address. In this case, it appears we will be attacking `192.168.142.38`  
  
![Starting the VM and getting an IP address](assets/LaVita/enumeration-0.png)  
  
## Phase 2: Scanning  
Once the machine has had a chance to start up, we begin enumeration with a custom nmap script I created. Initial enumeration was performed using an automated script I created to streamline the reconnaissance process. This included a fast port discovery scan followed by a detailed service/script scan and a sweep of top 100 common UDP ports.  
  
![Running NMAP](assets/LaVita/enumeration-1.png)  

Upon reviewing the output, we are able to see that there is a webserver listening on port 80. Seeing this, it only makes sense to start a gobuster scan against it.
  
![Starting Gobuster](assets/LaVita/enumeration-2.5.png)  
  
Opening up our browser and navigating to the webpage, we can see that we land on a page titled `W3.CSS Template`. We can see the team name listed including `Johnny Skunk`, `Rebecca Flex`, `Jan Ringo`, and `Kai Ringo`. At the top of the page, we can see navigation items, `Team`, `Work`, `Price`, `Contact`, and `Dropdown` - which has a sub item labled as `Demo`.  Scrolling to the bottom of the page, we don't see any obvious version or build of application we are looking at.  
  
![The web applications landing page](assets/LaVita/enumeration-2.png)  
  
![The web applications landing page](assets/LaVita/enumeration-5.png)  
  
![The web applications landing page](assets/LaVita/enumeration-3.png)  
  
![The web applications landing page](assets/LaVita/enumeration-4.png)  
  
Navigating to the `Demo` sub-item, we are met with a log in page. Having no positively identified credentials, we turn our attention towards the registration page and create our own login.  

![The login page for LaVita](assets/LaVita/enumeration-6.png)  
  
![The registration page](assets/LaVita/enumeration-8.png)  
  
After registering, we are met with a page labeled as `Dashboard Testing Area`. I created a test file to see if it would allow me to upload a text file.  
  
![The page we encounter directly after registering](assets/LaVita/enumeration-9.png)  
  
![creating cat.txt](assets/LaVita/enumeration-10.png)  
  
![uploading cat.txt](assets/LaVita/enumeration-11.png)  
  
![The result of uploading cat.txt](assets/LaVita/enumeration-12.png)  
  
I went ahead and tried the same after enabling the `APP_DEBUG` setting.  
  
![Enabling APP_DEBUG](assets/LaVita/enumeration-13.png)  
  
![Trying with an actual image](assets/LaVita/enumeration-14.png)  

I had no luck with this. Moving on, I review the page source to see if any version or build number is hiding in the code. I also check the request that is being sent to `/upload-image` to see if it might provide me with any detail to proceed further.
  
![Source code](assets/LaVita/enumeration-15.png)  
  
![Request](assets/LaVita/enumeration-16.png)  
  
![Request](assets/LaVita/enumeration-17.png)  
  

No luck here either! Checking on my GoBuster scan now that it is completed, there doesn't appear to be any routes forward here.  
  
![Final output of GoBuster](assets/LaVita/enumeration-18.png)  
  
I checked burpsuite but there wasn't any information to gather here either - asides from us being redirected back to `/home` when we submit a file for upload.  
  
![Checking burpsuite](assets/LaVita/enumeration-21.png)  
  
Out of curiousity, I wondered if there was an `images` directory within `/home`. Entering this directory, into the browser - we get a 404 error that gives us a breadcrumb moving forward. We can see that the site is using `Laravel 8.4.0`. After a quick Google, it seems that Laravel is a PHP web application framework. Interestingly enough, we can see that there actually is an exploit `CVE-2021-3129`.  
  
![404 page revealing use of Laravel 8.4.0](assets/LaVita/enumeration-22.png)  
  
![Researching Laravel 8.4.0 vulnerabilities](assets/LaVita/enumeration-23.png)  
  

## Phase 3: Vulnerability verification & Exploitation  
  
I go ahead and download this exploit and investigate the code. It looks like it does call out to an external source `https://github.com/ambionics/phpggc.git` - but this does not appear to be a threat and for our purposes can be trusted.   
  
![Code](assets/LaVita/enumeration-24.png)  
  
![Code](assets/LaVita/enumeration-24.1.png)  
  
![Code](assets/LaVita/enumeration-24.2.png)  
  
![Code](assets/LaVita/enumeration-24.3.png)  
  
Moving forward, we attempt to run the python script. It seems that it worked alright, minus the invalid escape sequence - though this doesnt appear to impact the execution of the script itself.  
  
![Starting the VM and getting an IP address](assets/LaVita/enumeration-25.png)  
  
Attempting to run it against our host, we see it gets stuck.  
  
![Attempting to exploit](assets/LaVita/enumeration-26.png)  
  
Taking the CVE - I go back to Google and see if I can hunt down a seperate working PoC of `CVE-2021-3129`. This is accomplished by the first entry here by `joshuavanderpoll` (https://github.com/joshuavanderpoll/CVE-2021-3129). We clone this down into our exploit folder, review the code, and get it ready to exploit.  
  
![Searching for other PoCs](assets/LaVita/enumeration-27.png)  
  
![Downloading PoC](assets/LaVita/enumeration-28.png)  
  
![Checking the contents of our directory](assets/LaVita/enumeration-29.png)  
  
It looks like we will need to install some requirements, so I create a virtual environment for python and activate it. Then install the requirements for the exploit via pip. Finally, testing whether the requirements are satisfied - and it looks like they are!  
  
![Creating a virtual python environment](assets/LaVita/enumeration-30.png)  
  
![Installing exploit requirements within our virtual environment](assets/LaVita/enumeration-31.png)  
  
![Testing the PoC](assets/LaVita/enumeration-32.png)  
  
This exploit expects us to enter the command that we wish to execute preceeded with `execute`. So to test, we try out `execute whoami`. Reviewing the output, we can see that laravel/rce1 fails to execute our payload. I tried a different command after this seetting up a listening on my local machine in the event the code was still executing - just blindly. But we can see that this is unsuccessful as well in our terminal with nc listening on port 9999.  
  
![Testing the PoC and attempting to exploit](assets/LaVita/enumeration-34.png)  
  
![Testing to see if it is being done in a blind way](assets/LaVita/enumeration-35.png)  
  
![Our listener](assets/LaVita/enumeration-36.png)  
  
Choosing to try the next chain by entering `Y`, we actually do see our victim machin reach out to the listener we have running - meaning that remote code execution was achieved.  
  
![Using the next chain](assets/LaVita/enumeration-37.png)  
  
![RCE proof on the victim machine within our listener](assets/LaVita/enumeration-38.png)  
  
Now that we know we can execute code on the sysytem, it is a matter of time before we land a reverse shell. Locating the nc binary on the victim system, we are able to utilize this in a fifo reverse shell. I utilize port 443 as it is less likely to get noticed and unlikely to get blocked egress.  
  
![locating the nc binary on the victim system](assets/LaVita/enumeration-39.png)  
  
![Setting up our listener on port 443](assets/LaVita/enumeration-40.png)  
  
![creating the fifo reverse shell](assets/LaVita/enumeration-41.png)  
  
![Running the exploit](assets/LaVita/enumeration-42.png)  
  
![Landing a shell](assets/LaVita/enumeration-43.png)  
  
From here we can grab out local flag - however, I will be excluding that from this write up for obvious reasons.  
  
## Phase 4: Maintaining access & privilege escalation  
  
Time for our post-exploitation enumeration. Looking in our home directory - we can see another user does have a directory `skunk`. This must be `Johnny Skunk` who we saw on the landing page - neat! I further check to see whether we can look at items within his directory and attempt to make a .ssh directory with no success due to our lack of permissions to do so.  
  
![Checking home directory](assets/LaVita/post-exploitation-1.png)  
  
![Checking home directory](assets/LaVita/post-exploitation-2.png)  
  
![Attempting to make ssh folder](assets/LaVita/post-exploitation-3.png)  
  
Moving on I also check the contents of `/var/www/html`.  
  
![Checking common web directory](assets/LaVita/post-exploitation-4.png)  
  
![Listing contents of the web directory](assets/LaVita/post-exploitation-5.png)  
  
It is here that we examine the `.env` file and note that there is a hardcoded credential for the locally hosted sql database. We make sure to save this so we have it moving forward.  
  
![Listing contents of the env file](assets/LaVita/post-exploitation-6.png)  
  
![Saving the credential](assets/LaVita/post-exploitation-7.png)  
  
After attempting to connect to the database - I accidently kill my own shell and have to run through the exploitation process again. Whoops!  
  
![whoops](assets/LaVita/post-exploitation-8.png)  
  
This time I make sure to start a new shell via python3 so we can interact witht he mysql prompt.  
  
![Accessing mysql](assets/LaVita/post-exploitation-9.png)  
  
Listing out the databases we have access to, I opt to check out `lavita` and see if there is anything interesting in there.  
  
![databases listed](assets/LaVita/post-exploitation-10.png)  
  
![tables listed](assets/LaVita/post-exploitation-11.png)  
  
The table `users` looks promising. Selecting all content from this table, we can see that it is in fact not useful to us - only invluding the credentials for the user that we created. It was worth a shot checking out!  
  
![All content from users table](assets/LaVita/post-exploitation-12.png)  
  
I check out `/var/mail` but do not see any items of interest (it is empty). I do the same thing with `/opt` and encounter the same result.  
  
![Contents of mail directory](assets/LaVita/post-exploitation-13.png)  
  
![Contents of opt](assets/LaVita/post-exploitation-14.png)  
  
The root directory yields no items of interest either.  
  
![Root directory](assets/LaVita/post-exploitation-15.png)  
  
Next, I move into the `tools` folder I created and copy over my enumeration toolset with a script I made to simply copy them into the current working directory. I then transfer them onto the victim machine via wget in the temporary directory (then creating a folder named `k1ret` to keep things more tidy.  
  
![Copying enum tools](assets/LaVita/post-exploitation-16.png)  
  
![Checking they copied OK](assets/LaVita/post-exploitation-17.png)  
  
![Copying to victim machine](assets/LaVita/post-exploitation-18.png)  
  
![Copying to victim machine](assets/LaVita/post-exploitation-19.png)  
  
![Organizing](assets/LaVita/post-exploitation-20.png)  
  
I attempt PwnKit as a low hangin fruit - but it is unsuccessful.  
  
![Not pwned](assets/LaVita/post-exploitation-21.png)  
  
Next, I check out SUID's with `suid3num.py`. Sadly, we discover nothing interesting.  
  
![No interesting SUIDs](assets/LaVita/post-exploitation-22.png)  
  
Out of curiousity, I tired swithcing user to skunk with the database password that we had discovered. No dice. I also took a second to check `/etc/shadow`, but it appears we don't have permission. Sudo -l also yields no useful routes forward with our `www-data` user.  
  
![Possible PW reuse - nope](assets/LaVita/post-exploitation-23.png)  
  
![Testing access to shadow file](assets/LaVita/post-exploitation-24.png)  
  
![Testing sudo with l switch](assets/LaVita/post-exploitation-25.png)  
  
After scratching my head for about 30 minutes, I ran through the output of both `linpeas.sh` and `lse.sh` (using the -l 2 switch), I finally found that there is a process running in the context of skunk located at `/var/www/html/lavita/artisan`. We have the ability to write to this location path as well with the `www-user`.  
  
![Output of lse](assets/LaVita/post-exploitation-26.png)  
  
![Checking perms](assets/LaVita/post-exploitation-27.png)  
  
Let's exploit this! I navigated to https://www.revshells.com and grabbed `Ivan Sinceks` php reverse shell and placed it into a file named `artisan`. I then made a backup of the original file, uploaded it to the host, removed the original, put our own in place and landed a reverse shell as `skunk`!  
  
![Reverse shell creation](assets/LaVita/post-exploitation-28.png)  
  
![Transfering](assets/LaVita/post-exploitation-29.png)  
  
![Backing up and exploiting](assets/LaVita/post-exploitation-30.png)  
  
![Landed shell](assets/LaVita/post-exploitation-31.png)  
  
At this point we have successfully moved laterally from `www-data` to `skunk`. Natually, I upgrade our shell via python3. I generate a public/private key pair. I attempted to added my public key to the `authorized_keys` file under the `.ssh` directory I created within skunks home directory, but was ultimately unsuccessful signing in with that generated private key.  
  
![Upgrading our shell](assets/LaVita/post-exploitation-32.png)  
  
![Generating keys](assets/LaVita/post-exploitation-33.png)  
  
![Putting keys in place](assets/LaVita/post-exploitation-34.png)  
  
![No dice](assets/LaVita/post-exploitation-35.png)  
  
No problem - we will just have to be careful with our shell. (spoiler alert - I accidently killed the shell once!) I check the output of `sudo -l` with our user, `skunk`. It appears we can run `sudo /usr/bin/composer --working-dir\=/var/www/html/lavita *` without entering a password - nice!  
  
![Checking output of sudo with l switch](assets/LaVita/post-exploitation-36.png)  
  
Checking out the ever useful https://gtfobins.org, we find that they have an entry for composer!  
  
![GTFOBins ftw](assets/LaVita/post-exploitation-37.png)  
  
I attempted to add this `composer.json` file with our user skunk, but was unable to - so I simply created it with out `www-data` user.  
  
![Preping the priv esc](assets/LaVita/post-exploitation-38.png)  
  
Finally we execute the binary with sudo referencing the `x` script we created and we are in as root! 
  
![who is root? we are!](assets/LaVita/post-exploitation-39.png)  
  
## Phase 5: Reporting & Documentation  

After gaining root access, we've identified some key areas that we can focus on to harden the host. We first gathered information from an overly-verbose 404 error page that revealed the Laravel version. This could be fixed by using generic error pages and patching the software (which was ultimately how we facilitated remote code execution). We then used the artisan file that was running as the `skunk` account to move laterally and gain a foothold within that account which ultimately lead to full system compromise. This could have been prevented by running such files under a more restricted, isolated account that had no access to such elevated commands if they are required. In a perfect system, a complex password should be required to run in the context of root in any circumstance. Though, it did not lead to immediate privilege escalation, it would be suggested to investigate hardening permissions on sensitive files like `.env`. 
