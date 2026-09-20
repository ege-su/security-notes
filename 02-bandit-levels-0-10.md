# OverTheWire Bandit Commands — Levels 0-10

Here is the list of commands I used to pass the challenges 0 to 10. Level goals are taken directly from

	https://overthewire.org/wargames/bandit/bandit[X].html

with _X_ denoting the level number.


## Level 0

_The goal of this level is for you to log into the game using SSH. The host to which you need to connect is **bandit.labs.overthewire.org**, on port 2220. The username is **bandit0** and the password is **bandit0**._

The command I used is:

	ssh bandit.labs.overthewire.org -p 2220 -l bandit0


## Level 0 –> 1

*The password for the next level is stored in a file called **readme** located in the home directory. Use this password to log into bandit1 using SSH. Whenever you find a password for a level, use SSH (on port 2220) to log into that level and continue the game.*

The command I used is:

	cat readme

(And a single `exit` to terminate the current SSH session to be able to start the next one.)


## Level 1 –> 2

_The password for the next level is stored in a file called **-** located in the home directory._

Dashed filenames are tricky. I had to pass the file's relative path in my command:

	cat ./-


## Level 2 –> 3

*The password for the next level is stored in a file called `--spaces in this filename--` located in the home directory.*

I chose to use the relative path again, with the filename wrapped in quotes this time (for the spaces).

	cat ./"--spaces in this filename--"


## Level 3 –> 4

_The password for the next level is stored in a hidden file in the **inhere** directory._

To list the contents—including the hidden files—of **inhere**:

	ls inhere -a

The flag `-a` is needed for the hidden files.
A file named **...Hiding-From-You** appears inside **inhere**. To read its contents:

	cat inhere/...Hiding-From-You

(I didn't need to wrap the filename in quotes since it doesn't contain any spaces.)


## Level 4 –> 5

_The password for the next level is stored in the only human-readable file in the **inhere** directory. Tip: if your terminal is messed up, try the “reset” command._

The command `ls inhere -a` showed me there are 10 files inside the **inhere** directory: **-file00** to **-file09**. I decided to print them all to the terminal to see what their contents are like.

	cat inhere/-*

The `-*` part means _every file that has a filename starting with "-"_.

A password-like string appeared near the end of the output, so I've decided that the "human-readable" file must be either **-file06** or **-file07**.

It was **-file07**:

	cat inhere/-file07


## Level 5 –> 6

_The password for the next level is stored in a file somewhere under the **inhere** directory and has all of the following properties:

- *human-readable*
- *1033 bytes in size*
- *not executable*

This level asks me to `ls` the contents of the directory with _details_. The flag `-l` is needed for the task. I chose to list the hidden files with `-a` too:

	ls inhere -al

There are 20 directories. I need to use recursive search with `-R` and sort by file size with `-S` to see all their contents.

	ls inhere -alRS

Those 20 directories each have multiple files inside, so the output of my command is too long. I could highlight the results for _1033_ using `grep`:

	ls inhere -alRS | grep 1033

The file I am looking for is **.file2**, whose name is printed after the directory **maybehere07**. (Though I was lucky that the output didn't need much scrolling...)

	cat inhere/maybehere07/.file2


## Level 6 –> 7

*The password for the next level is stored **somewhere on the server** and has all of the following properties:

- *owned by user bandit7
- *owned by group bandit6
- *33 bytes in size*

I move up two levels to the root directory.

	cd ../..

Now I need to use `find`. I am unfamiliar with its flags, so I type `find --help` first. `-user` and `-group` flags should suffice for now.

	find -user bandit7 -group bandit6

The output is long, but I spotted a file with _password_ in its filename with its full path being **./var/lib/dpkg/info/bandit7.password**:

	cat ./var/lib/dpkg/info/bandit7.password


## Level 7 –> 8

*The password for the next level is stored in the file **data.txt** next to the word **millionth**.*

`ls` in home directory shows me that **data.txt** is already in here. I need to print the line containing the word **millionth**:

	grep "millionth" data.txt


## Level 8 –> 9

_The password for the next level is stored in the file **data.txt** and is the only line of text that occurs only once._

**data.txt** is again inside the home directory.

I couldn't find a `grep` flag that would provide me the unique strings within a file, so I looked into the other commands given as hints, and ultimately decided to use `sort` and `uniq` together to print the line that appears exactly once in the file. 

	sort data.txt | uniq -u


## Level 9 –> 10

*The password for the next level is stored in the file **data.txt** in one of the few human-readable strings, preceded by several ‘=’ characters.*

*Preceded by several '=' characters* means to *find the lines that contain '='* to me. Since the **data.txt** of this level contains binary data, I decide to extract the printable chars using `strings` and pipe the result into `grep`.

	strings data.txt | grep "="

The output is only 13 lines. One of them contains the password, which is preceded by 10 "=" characters.


## Level 10 –> 11

_The password for the next level is stored in the file **data.txt**, which contains base64 encoded data._

I'm familiar with `base64`! I should decode the file using `base64 -d`, with `-d` standing for _decode_.

	base64 -d data.txt