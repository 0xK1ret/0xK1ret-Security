# Machine info
**Name:** Blue  
**OS:** Windows  
**Difficulty:** :green_square: Easy  
**Complexity:** :green_square: Low


## Phase 1: Reconnaissance 
After spawning in our machine, we can see that our machines IP address is `10.129.15.253`. From the info provided in the HTB dashboard, we know that this is a Windows machine.
  
## Phase 2: Scanning
We start out our engagement by taking the information we collected in the reconnaissance phase (e.g. the IP address) and begin an nmap scan. 
`sudo nmap -Pn -n 10.129.15.253 -sC -sV -p- -oN nmap-scan-all`  
  
![Image of our nmap scan](/assets/blue-1.png)  
  
From this output, we can gather several pieces of key information. Multiple ports are observed as being open, notably SMB (445). We can additionally see that the host is running `Windows 7 Professional 7601 Service Pack 1`  
Taking this information over to Google, we can find several sources citing that this OS is vulnerable to MS17-010. From our research, we can determine that this is a a critical vulnerability in SMBv1 that can allow for arbitrary code exexution on a target system.  
  
![Google search output](/assets/blue-2.png)  

## Phase 2: Vulnerability verification & Exploitation
Clicking on the github link above, we find that there is an nmap script that allows us to determine whether our target system is vulnerable or not.  
  
![The code snippet from github](/assets/blue-3.png)  
  
Using this code snippet, we run `nmap -p445 --script smb-vuln-ms17-010 10.129.15.253`  
  
![Verifying that our target host is vulnerable](/assets/blue-4.png)  
  
We can see that our target machine is indeed vulnerable. Let's go ahead and start up Metasploit and explore what modules we have at our disposal. Typing `msfconsole` to start Metasploit, we are then able to search modules relevant to MS17-010.
  
![Starting Metasploit framework](/assets/blue-5.png)  
  
![Searching for modules relevant to MS17-010](/assets/blue-6.png)  
  
Looking through these modules, it appears our best bet is to use # 0. Looking through our options, there are several that we can set. In this instance, we will be leaving the payload as the default reverse TCP x64 Meterpreter and will only be setting our remote host and listening host.  
  
![Metasploit options for our selected module](/assets/blue-8.png)  
  
![setting Metasploit options](/assets/blue-9.png)  
  
Once set, all we need to do is type `exploit` and hit enter. . . 
  
![Running the exploit](/assets/blue-10.png)  
  
Success! We have landed a shell on the machine. 
  
## Maintaining access & privlege escalation

Now that we have a shell on the machine, we can check our access and collect our loot. Given the nature of EternalBlue, we do not need to perform any type of privlege escalation as we are already running as `nt authority\system`

![Checking out who we are](/assets/blue-11.png)  
  
![Collecting the user flag](/assets/blue-12.png)  
  
![Collecting the root flag](/assets/blue-13.png)  

If we were in an active engagement we can dump the hashes from the machine and attempt to crack them or utilize them in other attacks in an attempt to move laterally within the network. 

![Checking out who we are](/assets/blue-15.png)  
  
## Reporting & Documentation  

Now that we have successfully exploited our target system, we can consolidate all of our gathered observations into a report for our client. We may also make recommendations to our client as well in an effort to patch or mitigate the possibilities of this system being compromised. You could highly recommend patching this device (apply the security patch MS17-10), disabling SMBv1 if it is in the organizations ability to do so, or even segmenting the legacy system where it is signifigantly less likely to be abused.
  
![Checking out who we are](/assets/complete.png)  
