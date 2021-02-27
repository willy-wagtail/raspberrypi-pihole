# Linux notes

1. [ sudo ](#1-sudo)
2. [ Installing apps ](#2-installing-apps)
3. [ File permissions and file ownership ](#3-file-permissions-and-file-ownership)

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

 
Continue at https://youtu.be/wBp0Rb-ZJak?t=7822