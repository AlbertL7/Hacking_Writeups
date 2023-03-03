#hackthebox 

# Initial Nmap Scan

![Pasted image 20230228212834](https://user-images.githubusercontent.com/71300144/222603407-a04c5fa2-b72a-45b5-80e0-250a14f9d1e7.png)

SMB is open with an account called guest, so lets have a look at that, its worth noting that ssh is open as well

# SMB Enumeration

First thing I am going to do is check to see if I can list any available shares.

`smbclient -L ////10.10.10.134//`

![Pasted image 20230228212938](https://user-images.githubusercontent.com/71300144/222603467-4ff75270-9e67-4253-b9ee-4a1641687461.png)

Awesome looks like there are some shares available, Now I am going to check if any have read access, so maybe we can look at one as the guest user.

`smbmap -H 10.10.10.134 -U guest`

the Backups share has READ, WRITE access! 

![Pasted image 20230228215730](https://user-images.githubusercontent.com/71300144/222603522-74c338c5-be0f-42f4-b07a-560ecebd0486.png)

Connecting to the smb share Backups

`smbclient //10.10.10.134/Backups`

![Pasted image 20230228220647](https://user-images.githubusercontent.com/71300144/222603581-18ffc6cc-5d13-4450-b896-bb1ff8473dec.png)

From here I started to look at the available directories. I downloaded noted.txt, and nmap-test-file. digging a little bit deeper 

looking at note.txt

probably a file on the machine that is too big: a program / backup of some sort

![Pasted image 20230302165423](https://user-images.githubusercontent.com/71300144/222603656-ebd83266-d44d-4ced-a5ff-dfc1e7427b1f.png)

The next file I downloaded off of the smbshare and took a look at was nmap-test-file. At first it looked to be base64 but when I tried to decode it there was not change in output. So, I am just going to move on from here.

![Pasted image 20230302180109](https://user-images.githubusercontent.com/71300144/222603697-392e673c-5ff8-431d-b53f-d4a2186b9c2f.png)

Instead of looking at this smb share through smbclient I decided to mount the share to a directory I created called backups with the following command

`sudo mount -t cifs //10.10.10.134/Backkups backups -o user=,password=`

![Pasted image 20230228225915](https://user-images.githubusercontent.com/71300144/222603967-b4588e5d-6e66-40ff-b26e-bbd6d4b6802d.png)

Brownsing the backups share I found a list of what appears to be possible windows file system.

![Pasted image 20230302182952](https://user-images.githubusercontent.com/71300144/222603734-de43e669-afaa-4759-b190-17f4dc837a48.png)

You can see all of the files are xml except for 2. I asked chat GPT what a vhd file was.

"A VHD (Virtual Hard Disk) file is a file format used to represent a virtual hard disk drive. A virtual hard disk is a disk image file that contains the complete contents of a physical hard disk, including the operating system, programs, files, and data.

VHD files are commonly used in virtualization software such as Hyper-V, VirtualBox, and VMware. They can be created, configured, and managed using these virtualization platforms, and they are used to store the virtual machine's hard drive contents.

VHD files are similar to ISO files, which are used to store disk images of CDs or DVDs. However, while ISO files are read-only and can only be used to store data, VHD files are read/write, meaning that changes can be made to the virtual hard disk and the changes will be saved to the VHD file.

VHD files can be useful for a variety of purposes, such as testing software, creating virtual machines, and backup and disaster recovery. They can also be mounted as a virtual disk drive in the host operating system, allowing you to access the files and data stored in the virtual machine's hard drive."

# Mounting the VHD File

![Pasted image 20230301193052](https://user-images.githubusercontent.com/71300144/222604043-34f02788-8b26-4b3a-b16b-91a104da3756.png)

I asked chatgpt if you can mount a vhd on linux and you absolutely can. It gave me some tools to try like 'losetup', and 'qemu-nbd'. Those tools did not work and finally guestmount was mentioned and that worked with the following command.

`guestmount -a '/home/albert/pnpt/wpe/hackthebox/Bastion/backups/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd' -m /dev/sda1 --ro ~/pnpt/wpe/hackthebox/Bastion/vhd`

# Locate the SAM File

The SAM (Security Accounts Manager) file on Windows is located in the ``%SystemRoot%\System32\Config\`` folder. The file is named "SAM" and is one of several files that make up the Windows Registry.

The SAM file is important because it contains the user accounts and their corresponding password hashes for the local system. When a user logs in to a Windows system, the system uses the SAM file to verify the user's credentials and grant access to the system.

The SAM file is a critical component of Windows security, and it is closely guarded by the operating system. Unauthorized access to the SAM file can lead to serious security breaches, including the ability to log in as a privileged user, access sensitive data, and install malware or other malicious software on the system.

However, in some cases, it may be necessary to access the SAM file, such as when recovering a lost password or performing forensics analysis. It is important to note that accessing the SAM file without proper authorization is illegal and can result in criminal charges. 

- Copy the the SAM, SYSTEM, and SECURITY files.

The file explorer was not loading the system32 folder so I copied it via command line and it worked.

![Pasted image 20230301195203](https://user-images.githubusercontent.com/71300144/222604145-872f23ae-777e-4114-8bf6-b8713052d361.png)

I made a separate directory for the files on my machine called sam and moved the files in there.

![Pasted image 20230301195353](https://user-images.githubusercontent.com/71300144/222604229-b922327c-6191-47a7-9f5a-67ccde959d7a.png)

# Secretsdump.py Impacket

![Pasted image 20230301195924](https://user-images.githubusercontent.com/71300144/222604278-6b01c66a-e36d-4aab-a788-b709634f8c9a.png)

To use secretsdump.py you need all three files SAM SYSTEM and SECURITY

`python2 /usr/local/bin/secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL`

# Crack the Hashes

Since this is a CTF and not a real pentest I decided to try to crack the hashes online.

![Pasted image 20230301200756](https://user-images.githubusercontent.com/71300144/222604321-088b2a56-06d1-424f-bf96-59015c834408.png)

The admin and guest hash turned out to be nothing, but the L4mpje user turned out to be legitement 'bureaulampje' which is the default password found by secretsdump.py

# SSH 

![Pasted image 20230301201148](https://user-images.githubusercontent.com/71300144/222604362-43eb1ed9-b1c3-499c-919a-6d68e46e6766.png)

- The password worked! We did not need an rsa key

# Post Enumeration

- Note: look at the dates of the files maybe something will stand out
- I was having zero luck with downloading anything onto this machine I kept getting access denied and weird output errors when using powershell.

![Pasted image 20230301203437](https://user-images.githubusercontent.com/71300144/222604559-68b84c0a-3737-4017-8fd4-d2f5b5cfc6aa.png)

After long manual enumeration I will admit nothing stuck out to me so I looked at a writeup to give me a push in the right direction and it mention mRemoteNG.

The Program 'mRemoteNG' is not a default system program unlike the others listed below

ChatGPT very quickly explained what 'mRemoteNG' is but would not tell me if an exploit exists so I went out to google.

- mRemoteNG is a free and open-source remote connection management tool that  allows users to manage remote desktop sessions, remote shell (SSH) connections, and other types of remote connections all in one place. It is an enhanced version of mRemote, which was discontinued in 2009.

![Pasted image 20230301202349](https://user-images.githubusercontent.com/71300144/222604404-3dbc76b7-128a-4ea9-bf7a-a2f01229895a.png)

I searched for 'mRemoteNG exploit github' and found this link https://github.com/haseebT/mRemoteNG-Decrypt

I did a `git clone https://github.com/haseebT/mRemoteNG-Decrypt.git ` to pull down the tool to my opt directory

![Pasted image 20230301204406](https://user-images.githubusercontent.com/71300144/222604843-8d88a082-5068-46bb-811e-7697961214d5.png)

I did not know how to use this exploit because I did not have a password file or string to use.

![Pasted image 20230301204538](https://user-images.githubusercontent.com/71300144/222604865-5f732023-4c36-48bc-a6fa-2859e454edbb.png)

After some googling I came across this site https://www.errno.fr/mRemoteNG.html
which told me mRemoteNG stores its passwords in configuration files. So I then googled where does mRemeoteNG store its passwords configuration files, and came across this site http://forum.mremoteng.org/viewtopic.php?f=4&t=1550 which is a forum and it gave me the location. at `%userprofile%\AppData\Rooamin\gmRemoteNG\confCons.xml*` I do not have access to the administrators folder so I went back to L4mpje's profile

![Pasted image 20230301210123](https://user-images.githubusercontent.com/71300144/222604890-b6af9de6-4c6b-4edb-b1ab-3ae7884be9f6.png)

if you `type confCons.xml` you get the following output with username, and possible password strings!!!!

![Pasted image 20230301210346](https://user-images.githubusercontent.com/71300144/222604937-296dbd95-623f-461c-bfde-aa511356e9e5.png)

password string: 

`'aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw=='`

# Using mRemoteNG_decrypt.py

I then took the sting and used the following command to decrypt the password.

`python3 mremoteng_decrypt.py -s 'aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw=='`

you get 'thXLHM96BeKL0ER2' as the password

![Pasted image 20230301210556](https://user-images.githubusercontent.com/71300144/222605006-e4b5fda8-484d-47dc-a833-70e44e3ce448.png)

That should be the administrators password. All thats left is to try to ssh into the   machine now and hope the admin does not have an rsa_key

![Pasted image 20230301211319](https://user-images.githubusercontent.com/71300144/222605137-a1983ee7-b99d-4da8-b809-5552649f052e.png)

I am in as domain administrator, Box Owned!!

