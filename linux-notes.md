# Linux notes

1. [ sudo ](#1-sudo)
2. [ Installing apps ](#2-installing-apps)
3. [ File permissions and file ownership ](#3-file-permissions-and-file-ownership)
4. [ File and directory commands ](#4-file-and-directory-commands)
5. [ Find command ](#5-find-command)
6. [ Grep command ](#6-grep-command)
7. [ Processes ](#7-processes)
8. [ Services ](#8-services)
9. [ Crontabs ](#9-crontabs)
10. [ Users ](#10-users)
11. [ Networking ](#11-networking)
12. [ Linus hosts file ](#12-linux-hosts-file)
13. [ SSH ](#13-ssh)
14. [ SFTP ](#14-sftp)
15. [ Man command ](#15-man)

## 1 sudo

Writing to files other than your own home dir, you'll not have permissions

Two options
- ``sudo nano`` - use sudo to edit files (superuser do).
  - ``!!`` runs previous command
  - ``sudo !!`` runs previous command with sudo

- ``su`` switch user
  - ``sudo su`` - switch to root account
  - ``sudo willy`` - to go back to willy account

## 2 Installing apps

- Debian based Linux distros uses ``apt`` package manager
  - Ubuntu has repositories set up with indexes and package files, allowing us to run below commands to install from those repositories. there are programs not in the repositories.
  - ``apt install <app>`` 
  - ``apt remove`` - removes package
  - ``apt purge`` - removes package with config
  - ``apt update`` - refreshes repository index
  - ``apt upgrade`` - upgrades all upgradable packages
  - ``apt full-upgrade`` - upgrades packages with auto-handling of packages
  - ``apt search`` - searches for an application if we're not sure the name of
    - ``apt search bluefish*`` - for example, with asterisk
  - ``apt show`` - show package details
  - ``apt-cache policy gimp`` - shows whether we have gimp installed and what in the repository
- If program isn't in the repositories, then you can manually install a ``.deb`` file using ``dpkg`` command.
  - so we download from the web the chrome ``.deb`` file from [google chrome website](https://www.google.com/chrome/).
    - NOT the ``.rpm`` files - those are for redhat based distros.
    - Debian based, ``apt`` package manager uses ``.deb`` files.
  - ``sudo dpkg -i /home/willy/Downloads/google-chrome-stable_current_amd64.deb``

## 3 File permissions and file ownership

Create a text file ``sudo nano file.txt``
- ``ls -l`` - list files in long form
- ``-rw-r--r-- 1 root root 5 Dec 3 16:18 file.txt``
  - owned by the user root and that user's group root
  - ``-`` the first character as a ``-`` means it is a file, not a directory
    - a directory will have a ``d``
  - ``rw-`` the next three means the _user (u)_ can read and write
  - ``r--`` the next three means the _group (g)_ can read, not write
  - ``r--`` the last three means the all _other (o)_ users of the system can read

If we wanted to give ``file.txt`` write access to ``willy``,
- ``sudo chown root:willy file.txt`` - change owner command passing in ``<user>:<group>``.
  - here, keep the owner as ``root``, and share it with group ``willy``
- ``sudo chmod 664 file.txt`` change permissions of the file privilege to ``-rw-rw-r--``
  - this gives the users in the ``willy`` group read and write
    - the first digit is for _user
    - second for _group_
    - last for _other_
    - ``6`` means readable and writable
    - ``4`` means readable
    - ``7`` means directory - not used for single files

If we wanted to transfer ownership of the file to ``willy``
- ``sudo chown willy:willy file.txt``
- ``sudo chmod 644 file.txt`` - revert back to ``-rw--r--r--`` as that would give the user willy write permissions already without giving the group it

Delete the file ``rm file.txt``

Create a directory ``sudo mkdir mydir``
- ``ls -l`` gives ``drwxr-xr-x 2 root root 4096 Dec 3 16:23 mydir``
  - user can read, write, and execute
    - execute premission includes doing, say, running command ``cd mydir``
  - group and other can read and execute

Make 2 files inside the directory ``sudo nano ./file.txt``, ``sudo nano ./file2.txt``
  
Both files are owned by ``root:root``. We want to change it to ``willy:willy``
  - ``cd ..`` to go to the parent directory of ``mydir``
  - ``sudo chown -R nick:nick ./mydir`` 
    - changes ownership _recursively_ (i.e. ``mydir`` as well as the files/directories in it)
    - without ``-R``, it'll only change the directory ownership

## 4 File and directory commands

Create new files without going into an editor 
- ``touch file1.txt file2.txt file3.cpp file4.cpp``

Remove
- ``rm ./*.cpp`` removes only the .cpp files
- ``rm ./*`` remove everything in the dir leaving the dir intact
- ``cd ../`` go to the parent dir
- ``rm mydir/*`` remove everything in mydir leaving the dir intact
- ``rm -rf mydir`` remove directory and everything in it using the ``-rf`` options

Copy files ``cp file ./mydir/file2``

Move files ``mv filename filename2`` (same directory with diff name)

Create directory ``mkdir ./newDirectory``
- ``mv a.out newDirectory/a.out`` moves file a.out into the new directory
- ``mv mydir mySecondDirectory`` renames mydir to mySecondDirectory

## 5 Find command

Create a bunch of files ``touch file.php file2.php file3.php another.php somethingelse.PHP anotherone.PHP one.PhP two.txt three.txt four.TXT an.TXT``

Look for all php files ``find . -type f -name "*.php"``
- find in the current directory of type ``f`` for file and file name ``*.php``.
- this returns files with extensions .php, and is case sensitive
- omitting the ``-type``, it'll find both files and directories
- ``-type d`` will only find directories

Case insensitive version ``find . -type f -iname "*.php"``
- ``i`` in ``iname`` means it ignores case sensitivity

Look for all files with name starting with file like this ``find  . -type f -iname "file*"``

More practically, you can find all config files like this ``find /etc -type f -iname "*.conf"``

Look for all files with specific permissions using ``find . -type f -perm 0664``
- here, we will find files with these permissions ``-rw-rw-r--``

Look for all files by file sizes 
- ``file . -size +100k`` anything in current directory over 100kb
- ``file . -size 100k`` anything in current directory exactly 100kb
- ``file . -size 1M`` anything in current directory exactly 1Mb
- ``file . -size +1M`` anything in current directory over 1Mb
- ``file . -size -1M`` anything in current directory under 1Mb

Look for files which are NOT .php ``find . -type f -not -iname "*.php"``

``cd /etc`` etc directory holds configurations for applications which usually have .conf extension
- ``find . -type f -iname "*.conf"`` lots of files, and it will recursively search, as in it digs deep into each directory
- ``find . -maxdepth 1 -type f -iname "*.conf"`` here we set how deep to look, here only 1 level deep

## 6 Grep command

``grep`` is used to find things inside files.

Suppose file2.php has some functions defined in it. 
- ``grep "function" file2.php file3.php`` find "function" in each of the specified files
- ``grep -i "function" ./*`` ignore case sensitivity, look in all files for "function"
- ``grep -n -i "function" ./*`` also outputs the line number

 ``find . -type f -iname "*.php" -exec grep -i -n "function" {} +``
 - find will run first then exec is run
 - ``{} +`` will terminate the exec

``find . -type f -size -10k -iname "*.php" -exec grep -i -n "function" {} +``
- chaining more find command options

Redirecting the output of a command
- ``ls > outfile.txt`` rather than output ``ls`` in command line, it'll put it in a txt file called ``outfile.txt``
- ``find . -type f -size -10k -iname "*.php" -exec grep -i -n "sandwich" {} + > f.txt``
  - outputs it into ``f.txt``
  - does NOT output it to terminal
- ``find . -type f -size -10k -iname "*.php" -exec grep -i -n "function" {} + | tee of.txt``
  - use find to find all files less than 10Kb in size with .php extension
  - then find functions in these files, ignoring case sensitivity and numbering lines
  - then pipe over to ``tee`` command and the output file
    - ``tee`` will output to terminal as well as copying the output into a file
    - the pipe, ``|``, command letes you pass the output of one command as the input to the next

## 7 Processes

Use ``top`` shows the top running applications in real time.
- ``pid`` is the process id
- ``user`` is who the app is running as
- ``time+`` is how long the application has been open 
- ``command`` associated with the process

``ctrl + c`` to exit ``top``

Use ``ps aux`` to see the entire list, not only the top. The list is super long.
-  ``ps aux | grep liri-browser`` searches all processes for liri-browser anywhere in it
    - downside is, the output is messy
- ``pgrep liri-browser`` returns all the process ids where the "command" to run it is "liri-browser"
  - if more than 1 are running, it is listed in the order they were started
  - the command is the same as the command column when you run ``top``

To stop a process using process ids, use ``kill -9 6300`` where 6300 is the ``pid``
- can chain the ``pid``s, e.g. ``kill -9 6300 6284``

Kill all processes with a certain command using ``killall liri-browser``

## 8 Services

``apt-cache policy elasticsearch`` - v1.7.3 used as an example of a service.

- ``sudo service elasticsearch start`` start a service using the service name
- ``sudo service elasticsearch stop`` stops a running service

Change port of elasticsearch
- ``find /etc -type f -iname "elasticsearch*"``
- ``sudo nano /etc/elasticsearch/elasticsearch.yml``
- find the http.port and change it from 9200 to 1150

A running service won't update if the config has changed
- ``sudo service elasticsearch restart`` to restart the service
- it'll now be on port 1150

Nowadays, we can use ``systemctl`` instead - this is more standard
- ``sudo systemctl start elasticsearch``
- ``sudo systemctl stop elasticsearch``

## 9 Crontabs

Use command ``crontab -e`` to open the interface where you define build crontabs jobs.
- structure of a crontab is: _minutes hours day-of-month month day-of-week command-to-run_
  - so the values are [0-59] [0-23] [1-31] [1-12] [0-6] *command*
- e.g. ``15 14  * * * ls > home/willy/lt/cronres.txt``
  - runs on 15th minute, 14th hour, _regardless_ of day-of-month, month, and day-of-week
- e.g. ``00 05  * * 0 ls > home/willy/lt/cronres.txt``
  - runs at 5am every Sunday

E.g. backup all of your user accounts 
- ``0 5 * * 1 tar -zcf /var/backups/home.tgz /home/``
  - runs at 5am every Monday
  - creates a tar archive at ``/var/backups/home.tgz`` with all the contents of ``/home/``

Note that these crontabs are assocated with the user ``willy``, the ``ls`` command will output the ``willy`` user's home directory.

To open the crontab for the root user, run ``sudo crontab -e``.
- ``0 7 * * 1 apt upgrade -y``
  - every monday at 7am, upgrade packages. 
  - the ``-y`` means say yes to all prompts that may arise.

## 10 Users

``sudo adduser willy2`` adds user, adds a group for the user and adds user to the group
- ``su willy2`` switch user to willy2
- ``sudo whoami`` permission issue as it isnt in the sudo group
- ``su willy``
- ``sudo adduser willy2 sudo`` adds user to group
- ``sudo willy2``
- ``sudo whoami`` should work

``sudo passwd willy2`` change password for user

``sudo deluser willy2`` delete user

``touch text`` create file called text
- ``mkdir groupstuff``
- ``mv text groupstuff``
- ``cd groupstuff``
- ``ls -l`` gives ``-rw-rw-r-- 1 willy willy 0 Feb 9 17:52 text``

``sudo groupadd tt`` create a new user group called tt
- ``sudo chown willy:tt text`` change group ownership of file
- ``ls -l`` gives ``-rw-rw-r-- 1 willy tt 0 Feb 9 17:52 text``

``sudo adduser willy3 tt`` adds user to group
- ``su willy3`` switch user
- ``nano text`` should be able to edit file without using sudo

## 11 Networking

``ping google.com``

``ifconfig`` outputs details about network interfaces
- RX is received, TX is transmitted

``tcpdump`` analyse packets going in and out of your computer
- ``sudo apt install tcpdump``
- ``sudo tcpdump`` outputs all packets, its alot
- ``sudo tcpdump -c 10`` capture 10 packets
- ``sudo tcpdump -c 5 -A`` print out the actual packet contents in ASCII format
- ``sudo tcpdump -c 5 -i wlo1`` listen to a specific network interface which you can find using ``ifconfig``
- ``sudo tcpdump -c 5 -XX -i wlo1`` prints out packets in hex and ASCII format
- ``sudo tcpdump -i wlo1 port 22`` can listen to a specific port. port 22 is the ssh port.

``netstat`` network statistics command
- ``netstat -nr`` 
  - the n option gives IP address rather than domain names
  - the r option gives the kernel IP table
- ``netstat -i`` prints the kernel interface table and the bytes transferred
- ``netstat -ta`` prints active internet connections
  - ``netstat -tan`` same but with IP addr

``sudo apt install traceroute``
- ``traceroute google.com`` traces every server the request goes through before reaching google.com

``nmap`` network mapper tool
- ``sudo apt install nmap``
- ``nmap 192.168.0.100`` to see what ports are available, their state and what service it provides
- ``nmap -v 192.168.0.100`` verbose mode, gives more info on its progress as it is scanning
- ``nmap 192.168.0.100,1`` scan multiple IP addresses using comma
- ``nmap 192.168.0.1-100`` scan a range of IP addresses between 1 and 100
- ``nmap 192.168.0.*`` wildcard to scan from 0 to 255
- ``nmap <external domain name>`` can also scan external servers
- ``nmap -iL ~/nextworks.txt`` create a text file with a list of hosts for nmap to scan
- ``nmap -A 192.168.0.0-100`` turns on operating system and version detection and more details
- ``nmap -sP <>`` find out what server and devices are up and running
- ``nmap --reason <>`` reason its in a particular state
- ``nmap --iflist <>`` network interfaces available

## 12 Linux hosts file

Whenever you type in a domain name, Linux tries to resolve it from the hosts file before going out to the internet. The file is like an internal DNS. So we can intercept requested domains like below:

- ``sudo nano /etc/hosts``
  - local IP address 127.0.0.1
  - then type, for fun, ``google.com``
  - this means whenever you go to google.com, it'll reach 127.0.0.1 which is localhost

The file can take 3 columns:
- IP address we want to route to
- the domain name we want to route to the IP addr
- alias for the domain name

Can change hostname of your machine
- ``sudo hostnamectl set-hostname Megazord``
- ``sudo nano /etc/hosts`` - change localhost hostname to new name
- ``sudo service hostname restart``

## 13 SSH

Secure shell remotely connects to a host
- ``ssh willy@domainname.com`` ssh command using domain name
- or ``ssh willy@192.168.0.2`` ssh using IP address

When done, type ``exit`` to close connection

Setting up ssh server:
- ``sudo apt install openssh-server`` installs openssh-server
- ``sudo nano /etc/ssh/sshd_config`` open the ssh config file
  - can change the default port from 22 to, say, 2212
  - under authentication, can set ``PermitRootLogin no``to stop anyone logging in as root
  - add a line ``AllowUsers willy``. 
    - this takes a list of users separated by a space. 
    - this specifies which users are allowed ssh into this server
  - ``sudo systemctl restart ssh`` to restart the ssh process

## 14 SFTP

FTP = file transfer protocol.
- uses port 21
- transfer is via plain text so anyone can see packet contents
- recommend never using this

SFTP = secure file transfer protocol
- ``mkdir sftp-demo``
- ``cd sftp-demo``
- ``touch name.txt``
- ``sftp willy@192.168.0.2`` sftp connection using username and server IP addr or domain name
- ``ls`` will list files and directories in the directory you are in on the _remote_
- ``lls`` will list _local_ files and directories of the directory you are in
- ``put names.txt`` will upload the local file ``names.txt`` to the remote host
- ``get remotefile.txt`` will download a file on the remote host to your local machine

## 15 Man command

To find out more info about a command in linux, use ``man``.
- ``man ssh`` find out about ssh command
- ``q`` to quit
- e.g. ``man chromium-browser``, ``man nmap``, ... etc
