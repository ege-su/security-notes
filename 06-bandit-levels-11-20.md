<p align="right"><sub>October 16, 2026</sub></p>

# OverTheWire Bandit Commands — Levels 11-20

Here is the list of commands I used to pass the challenges 11 to 20. Level goals are taken directly from

	https://overthewire.org/wargames/bandit/bandit[X].html

with _X_ denoting the level number.

## Level 11 --> 12

*The password for the next level is stored in the file **data.txt**, where all lowercase (a-z) and uppercase (A-Z) letters have been rotated by 13 positions.*

The **Rot13** article on Wikipedia is added under *Helpful Reading Material*. In the article, the command `tr 'A-Za-z' 'N-ZA-Mn-za-m'` is supplied for the purpose of "mapping upper case A-Z to N-ZA-M and lower case a-z to n-za-m".

Let me pipe `cat data.txt` into `tr`.

```bash
cat data.txt | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

The password is revealed.

## Level 12 --> 13

*The password for the next level is stored in the file **data.txt**, which is a hexdump of a file that has been repeatedly compressed. For this level it may be useful to create a directory under /tmp in which you can work. Use mkdir with a hard to guess directory name. Or better, use the command “mktemp -d”. Then copy the datafile using cp, and rename it using mv (read the manpages!).*

This is my least favorite Bandit level so far. I am just copy-pasting my Terminal commands and outputs here. Determine the filetype, decompress the file, and repeat for eight times.

```text
bandit12@bandit:~$ mkdir /tmp/b12
bandit12@bandit:~$ xxd -r data.txt > /tmp/b12/bdata.txt
bandit12@bandit:~$ cd /tmp/b12
bandit12@bandit:/tmp/b12$ file bdata.txt
bdata.txt: gzip compressed data, was "data2.bin", last modified: Sat Sep 26 21:54:21 2026, max compression, from Unix, original size modulo 2^32 584
bandit12@bandit:/tmp/b12$ mv bdata.txt bdata.gz
bandit12@bandit:/tmp/b12$ ls
bdata.gz
bandit12@bandit:/tmp/b12$ gzip -d bdata.gz
bandit12@bandit:/tmp/b12$ ls
bdata
bandit12@bandit:/tmp/b12$ file bdata
bdata: bzip2 compressed data, block size = 900k
bandit12@bandit:/tmp/b12$ mv bdata bdata.bz2
bandit12@bandit:/tmp/b12$ bzip2 -d bdata.bz2
bandit12@bandit:/tmp/b12$ ls
bdata
bandit12@bandit:/tmp/b12$ file bdata
bdata: gzip compressed data, was "data4.bin", last modified: Sat Sep 26 21:54:21 2026, max compression, from Unix, original size modulo 2^32 20480
bandit12@bandit:/tmp/b12$ mv bdata bdata.gz
bandit12@bandit:/tmp/b12$ gzip -d bdata.gz
bandit12@bandit:/tmp/b12$ file bdata
bdata: POSIX tar archive (GNU)
bandit12@bandit:/tmp/b12$ mv bdata bdata.tar
bandit12@bandit:/tmp/b12$ tar -xf bdata.tar
bandit12@bandit:/tmp/b12$ ls
bdata.tar  data5.bin
bandit12@bandit:/tmp/b12$ file data5.bin
data5.bin: POSIX tar archive (GNU)
bandit12@bandit:/tmp/b12$ mv data5.bin data5.bin.tar
bandit12@bandit:/tmp/b12$ tar -xf data5.bin.tar
bandit12@bandit:/tmp/b12$ ls
bdata.tar  data5.bin.tar  data6.bin
bandit12@bandit:/tmp/b12$ file data6.bin
data6.bin: bzip2 compressed data, block size = 900k
bandit12@bandit:/tmp/b12$ mv data6.bin data6.bin.bz2
bandit12@bandit:/tmp/b12$ bzip2 -d data6.bin.bz2
bandit12@bandit:/tmp/b12$ ls
bdata.tar  data5.bin.tar  data6.bin
bandit12@bandit:/tmp/b12$ file data6.bin
data6.bin: POSIX tar archive (GNU)
bandit12@bandit:/tmp/b12$ mv data6.bin data6.bin.tar
bandit12@bandit:/tmp/b12$ tar -xf data6.bin.tar
bandit12@bandit:/tmp/b12$ ls
bdata.tar  data5.bin.tar  data6.bin.tar  data8.bin
bandit12@bandit:/tmp/b12$ file data8.bin
data8.bin: gzip compressed data, was "data9.bin", last modified: Sat Sep 26 21:54:21 2026, max compression, from Unix, original size modulo 2^32 49
bandit12@bandit:/tmp/b12$ mv data8.bin data8.bin.gz
bandit12@bandit:/tmp/b12$ ls
bdata.tar  data5.bin.tar  data6.bin.tar  data8.bin.gz
bandit12@bandit:/tmp/b12$ gzip -d data8.bin.gz
bandit12@bandit:/tmp/b12$ ls
bdata.tar  data5.bin.tar  data6.bin.tar  data8.bin
bandit12@bandit:/tmp/b12$ file data8.bin
data8.bin: ASCII text
bandit12@bandit:/tmp/b12$ cat data8.bin
```

## Level 13 --> 14

*The password for the next level is stored in **/etc/bandit_pass/bandit14 and can only be read by user bandit14**. For this level, you don’t get the next password, but you get a private SSH key that can be used to log into the next level. Look at the commands that logged you into previous bandit levels, and find out how to use the key for this level.*

*If you need help with this level: a hint file can be found in the home directory.*

*Make sure to read the error messages as they are informative.*

`ls` in the home directory reveals `HINT` and `sshkey.private`, and under *Helpful Reading Material*, there is a page from Ubuntu's Wiki, called **Transferring Files via SCP**, which appears to have been abandoned during the migration of Ubuntu's old Community Help Wiki to a new platform in September 2026. The page provides no help, but I get the idea. I am to download the key to my own computer using SCP.

Let me terminate the SSH connection with `exit` first. Then:

```bash
scp -P 2220 bandit13@bandit.labs.overthewire.org:sshkey.private .
```

to download the key directly to my home folder. I enter the current password when prompted. Once the key stops downloading, I try logging in to bandit14:

```bash
ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220
```

...and I receive this error:

```text
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0640 for 'sshkey.private' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "sshkey.private": bad permissions
```

Disappointing.

```
ls -l sshkey.private
```

shows me that the key's permissions are:

```text
-rw-r-----
```

So let me restrict the permissions for everyone except me (the file owner):

```
chmod 700 sshkey.private
```

Now `ssh -i sshkey.private bandit14@bandit.labs.overthewire.org -p 2220` works.

## Level 14 --> 15

*The password for the next level can be retrieved by submitting the password of the current level to **port 30000 on localhost**.*

*Commands you may need to solve this level* are provided as `ssh`, `telnet`, `nc`, `openssl`, `s_client`, `nmap`.

`nc` (`netcat`) is a computer networking utility used to read from and write to network connections using TCP or UDP protocol. It will be perfect for the job.

Flashback to the previous level's description:

*The password for the next level is stored in **/etc/bandit_pass/bandit14**...*

```bash
cat /etc/bandit_pass/bandit14
```

I copied the password and pasted it directly after entering the following command to submit it to port 30000 on localhost:

```
nc localhost 30000
```

I am returned the new password.

## Level 15 --> 16

*The password for the next level can be retrieved by submitting the password of the current level to **port 30001 on localhost** using SSL/TLS encryption.*

*Helpful note: Getting “DONE”, “RENEGOTIATING” or “KEYUPDATE”? Read the “CONNECTED COMMANDS” section in the manpage.*

I can use `OpenSSL` for opening an SSL connection to localhost.

I can create a client that will connect to localhost over SSL with the following command:

```
openssl s_client -connect localhost:30001
```

After I submit the password of the current level (directly paste it into the console), the server replies with the new password.

## Level 15 --> 16

_The password for the next level can be retrieved by submitting the password of the current level to **port 30001 on localhost** using SSL/TLS encryption.
**Helpful note: Getting “DONE”, “RENEGOTIATING” or “KEYUPDATE”? Read the “CONNECTED COMMANDS” section in the manpage.**_

I can use `OpenSSL` for opening an SSL connection to localhost.

I can create a client that will connect to localhost over SSL with the following command:

	openssl s_client -connect localhost:30001

After I submit the password of the current level (directly paste it into the console), the server replies with the new password.

## Level 16 --> 17

_The credentials for the next level can be retrieved by submitting the password of the current level to **a port on localhost in the range 31000 to 32000**. First find out which of these ports have a server listening on them. Then find out which of those speak SSL/TLS and which don’t. There is only 1 server that will give the next credentials, the others will simply send back to you whatever you send to it.
**Helpful note: Getting “DONE”, “RENEGOTIATING” or “KEYUPDATE”? Read the “CONNECTED COMMANDS” section in the manpage.**

I need to find the open ports between 31000 and 32000 on localhost first, then find out whether they are running SSL or not. I can use `nmap` to scan this range of ports. 

	nmap localhost -p 31000-32000 -sV

`-p` allows me to choose a range of ports to scan, and `-sV` is there to enable service and version detection so that I will be receiving all the information I need in a single command.

The output of the command is:

```Shell
PORT      STATE SERVICE     VERSION

31046/tcp open  echo
31518/tcp open  ssl/echo
31691/tcp open  echo
31790/tcp open  ssl/unknown
31960/tcp open  echo
```

Port `31790`, which runs an unknown SSL service, is particularly interesting. Let me try to connect to it.

	openssl s_client -connect localhost:31790 -quiet

The `-quiet` flag was necessary to suppress `KEYUPDATE`.

After I send the current level's password to the server listening on `31790`, I am sent an SSH private key by the server.

I copy the key, create a file called `sshkey17.private` in my home directory on my personal machine, and paste the key into the file.

I also need to give the new key the necessary permissions as I did on Level 13 --> 14 so I can use it to log into `bandit17`.

	chmod 700 sshkey17.private
	ssh -i sshkey17.private bandit17@bandit.labs.overthewire.org -p 2220

## Level 17 --> 18

*There are 2 files in the homedirectory: **passwords.old and passwords.new**. The password for the next level is in **passwords.new** and is the only line that has been changed between **passwords.old and passwords.new***

***NOTE: if you have solved this level and see ‘Byebye!’ when trying to log into bandit18, this is related to the next level, bandit19***

I need to compare the two files and find the line that has been changed.

	diff passwords.old passwords.new

The output identifies the changed line and shows the old and new contents, marked with `<` and `>` respectively, separated by `---`.

The line that follows  `>`  is the new password .

## Level 18 --> 19

*The password for the next level is stored in a file **readme** in the homedirectory. Unfortunately, someone has modified **.bashrc** to log you out when you log in with SSH.*

It is possible to run commands over SSH *while* connecting to a server. Since I've been explicitly told that `readme` is in the home directory, I chose to try reading the contents of `readme` without checking other files within `~`. 

	ssh bandit18@bandit.labs.overthewire.org -p 2220 cat readme

It works! I have the password to the next level.

## Level 19 --> 20

*To gain access to the next level, you should use the setuid binary in the homedirectory. Execute it without arguments to find out how to use it. The password for this level can be found in the usual place (/etc/bandit_pass), after you have used the setuid binary.*

*Helpful Reading Material*
- *[setuid on Wikipedia](https://en.wikipedia.org/wiki/Setuid)*

First, I navigate to `/etc/bandit_pass` and run `ls -la`.

The output is...

```Shell
total 152
drwxr-xr-x   2 root     root      4096 Sep 26 21:54 **.**
drwxr-xr-x 128 root     root     12288 Oct 16 14:15 **..**
-r--------   1 bandit0  bandit0      8 Sep 26 21:53 bandit0
-r--------   1 bandit1  bandit1     33 Sep 26 21:53 bandit1
-r--------   1 bandit10 bandit10    33 Sep 26 21:53 bandit10
-r--------   1 bandit11 bandit11    33 Sep 26 21:54 bandit11
-r--------   1 bandit12 bandit12    33 Sep 26 21:54 bandit12
-r--------   1 bandit13 bandit13    33 Sep 26 21:54 bandit13
-r--------   1 bandit14 bandit14    33 Sep 26 21:54 bandit14
-r--------   1 bandit15 bandit15    33 Sep 26 21:54 bandit15
-r--------   1 bandit16 bandit16    33 Sep 26 21:54 bandit16
-r--------   1 bandit17 bandit17    33 Sep 26 21:54 bandit17
-r--------   1 bandit18 bandit18    33 Sep 26 21:54 bandit18
-r--------   1 bandit19 bandit19    33 Sep 26 21:54 bandit19
-r--------   1 bandit2  bandit2     33 Sep 26 21:54 bandit2
-r--------   1 bandit20 bandit20    33 Sep 26 21:54 bandit20
-r--------   1 bandit21 bandit21    33 Sep 26 21:54 bandit21
-r--------   1 bandit22 bandit22    33 Sep 26 21:54 bandit22
-r--------   1 bandit23 bandit23    33 Sep 26 21:54 bandit23
-r--------   1 bandit24 bandit24    33 Sep 26 21:54 bandit24
-r--------   1 bandit25 bandit25    33 Sep 26 21:54 bandit25
-r--------   1 bandit26 bandit26    33 Sep 26 21:54 bandit26
-r--------   1 bandit27 bandit27    33 Sep 26 21:54 bandit27
-r--------   1 bandit28 bandit28    33 Sep 26 21:54 bandit28
-r--------   1 bandit29 bandit29    33 Sep 26 21:54 bandit29
-r--------   1 bandit3  bandit3     33 Sep 26 21:54 bandit3
-r--------   1 bandit30 bandit30    33 Sep 26 21:54 bandit30
-r--------   1 bandit31 bandit31    33 Sep 26 21:54 bandit31
-r--------   1 bandit32 bandit32    33 Sep 26 21:54 bandit32
-r--------   1 bandit33 bandit33    33 Sep 26 21:54 bandit33
-r--------   1 bandit4  bandit4     33 Sep 26 21:54 bandit4
-r--------   1 bandit5  bandit5     33 Sep 26 21:54 bandit5
-r--------   1 bandit6  bandit6     33 Sep 26 21:54 bandit6
-r--------   1 bandit7  bandit7     33 Sep 26 21:54 bandit7
-r--------   1 bandit8  bandit8     33 Sep 26 21:54 bandit8
-r--------   1 bandit9  bandit9     33 Sep 26 21:54 bandit9
```

Passwords for every level have been there the whole time? I'm nosy, so I try chain reading all of them with `cat bandit*`.

Sadly, I am allowed to view only `bandit19`, which matches the password I'd already discovered the previous level. Back to the task at hand.

I shouldn't have left `~`. I simply enter `cd` to return to home directory, and `ls -la` there.

There is a file called `bandit20-do`, and its permissions are `-rwsr-x---`. The lowercase `s` in the owner's execute position means that both the execute and setuid bits are enabled.

When I run `bandit20-do` without any arguments, as specified in the challenge instructions, this is the output I get:

```Shell
Run a command as another user.
  Example: ./bandit20-do whoami
```

I run the example command, `./bandit20-do whoami`. The output is simply `bandit20`.

<img width="285" height="31" alt="bandit20-do whoami" src="https://github.com/user-attachments/assets/b47f5caa-8d04-4748-9c3f-b74f490fc9ab" />

So this setuid binary allows me to run commands as `bandit20`. Why not try reading `bandit20`'s password with it? 

	./bandit20-do cat /etc/bandit_pass/bandit20

I have the password!

## Level 20 --> 21

*There is a setuid binary in the homedirectory that does the following: it makes a connection to localhost on the port you specify as a commandline argument. It then reads a line of text from the connection and compares it to the password in the previous level (bandit20). If the password is correct, it will transmit the password for the next level (bandit21).*

***NOTE:** Try connecting to your own network daemon to see if it works as you think*

First, I run `ls -la` to learn the binary's name, which turns out to be `suconnect`, and then run it without any arguments to see if it'll give me any specific instructions.

```Shell
**bandit20@bandit**:**~**$ ./suconnect 
Usage: ./suconnect <portnumber>
This program will connect to the given port on localhost using TCP. If it receives the correct password from the other side, the next password is transmitted back.
```

Should I set up a listener myself? Let me try setting up one at port `22590` (any available port will do):

	echo "[password for bandit20]" | nc -lp 22590 &

I choose not to type out the password here since OTW changes passwords regularly. `&` is also important so that the listener will run in the background and not block my terminal window.

Let me now try connecting to port `22590`:

	./suconnect 22590

The listener supplies the current password to `suconnect`, which validates it and sends the next password back through the same TCP connection:

```
**bandit20@bandit**:**~**$ ./suconnect 22590
Read: [password for bandit20]
Password matches, sending next password
[password for bandit21]
[1]+  Done                       echo "[password for bandit20]" | nc -lp 22590
```
