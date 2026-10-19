<p align="right"><sub>October 16, 2026</sub></p>

# OverTheWire Bandit Commands — Levels 11-15

Here is the list of commands I used to pass the challenges 11 to 15. Level goals are taken directly from

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
