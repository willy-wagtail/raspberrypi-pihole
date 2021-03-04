# Ansible notes

1. [Inventory](#1-inventory)

## 1 Inventory

- An inventory defines a collection of hosts managed by Ansible
- Hosts can be assigned to groups
- Groups can be managed collectively
- Groups can contain child gruops
- hosts can be members of multiple groups
- vars can be set that apply to hosts and gruops

Define using text file
- can be written in INI-style or in .yml
- this is called _static inventory_ - needs to be manually updated
- a _dynamic inventory_ is auto generated and updated using scripts

Inventory location is controlled by your current Ansible configuration file
- ``ansible --version`` will show you which configuration file is in use
- location of inventory is defined in that config file like this:
 ```
 [defaults]
 inventory = ./inventory
 ```
- if this isnt set, ``/etc/ansible/hosts`` is used by default

INI-formatted inventory file - simplest form
- list of host names or IP addresses, e.g.
  ```
  web1.example.com
  192.0.2.42
  ```
- host groups allow you to collectively automate a set of systems using square brackets, e.g
  - group names should not include dashes, but underscore is fine
  - avoid confusion - do not give a gruop the same name as a host
  ```
  [webservers]
  web1.example.com
  web2.example.com
  192.0.2.42

  [db_servers]
  db1.example.com
  db2.example.com
  ```
- a host can be a member of multiple groups
- this allows you to organise groups in different ways dependeing on how you want to manage them
  - web services or db servers
  - servers in prod or test
  - servers in different geolocations
  - e.g.
  ```
  [webservers]
  web1.example.com
  web2.example.com
  192.0.2.42

  [db_servers]
  db1.example.com
  db2.example.com

  [east_datacenter]
  web1.example.com
  db1.example.com

  [west_datacenter]
  web2.example.com
  db2.example.com

  [production]
  web1.example.com
  web2.example.com
  db1.example.com
  db2.example.com

  [development]
  192.0.2.42
  ```

Two host groups always exist
- ``all`` includes every host in the inventory
- ``ungrouped`` includes every host in all that is not a memeber of another group

Groups can include other groups called _nested groups_
- in INI-formatted inventory, you can add nested  groups with the ``:children`` suffix
- e.g. usa and canada are inside the north_america group
  ```
  [usa]
  washington01.example.com
  washington02.example.com

  [canada]
  ontario01.example.com
  ontario02.example.com

  [north_america:children]
  canada
  usa
  ```

Range shortcut
- ``192.168.[4:7].[0:255]``
  - matches all IPv4 adddresses in the range ``192.168.4.0`` through ``192.168.7.255`` (the 192.168.4.1/22 CIDR-notation network)
- ``server[01:20].example.com`` 
  - matches all hosts named ``server01.example.com`` through to ``server20.example.com``
- ``[a:c].dns.example.com`` 
  - matches hosts named ``a.dns.example.com``, ``b.dns.example.com`` and ``c.dns.example.com``
- if leading zeros are included in the range, they are used in the pattern.
  - so ontario01.example.com matches, but ontario1.example.com does not
  ```
  [usa]
  washington[1:2].example.com

  [canada]
  ontario[01:02].example.com
  ```

YAML format
```
all:
  children:
    north_america:
      children:
        canada:
          hosts:
            ontario01.example.com: {}
            ontario02.example.com: {}
        usa:
          hosts:
            washington1.example.com: {}
            washington2.example.com: {}
```

``ansible-inventory`` command used to verify inventory
- ``-i`` option can be used to check any file rather than the current inventory

``ansible-inventory -y --list`` will display the current inventory in YAML format
- easy way to convert INI files to YAML files

``ansible washington1.example.com --list-hosts`` - used to verify a machine's presence in the inventory
- outputs ``hosts (1): washington1.example.com``


In this course, ansible control host running RHEL8, managing 2 web servers and two db servers.
- ``vim /etc/ansible/ansible.cfg`` - default ansible config file
  - you can see that the default inventory is configured in ``/etc/ansible/hosts`` file
  - ``vim /etc/ansible/hosts``
- ``vim inventory`` create own inventory file with basic entries for the 4 hosts
  ```
  [webservers]
  web01
  web02

  [databases]
  db[01:02]
  ```
- ``ansible-inventory -i inventory --list`` to list all hosts listed in ``inventory`` file in json format

## 2 Connection settings and privilege escalation

Ansible does not require you to install an agent on managed hosts.

Protocols and software included with the OS are used: 
- ssh and python on linux systems
- other protocols for windows such as Windows remote management and powershell

Advantages of using common well tested and understool tools.
- simpler to prepare systems 
- reduces security risks

Ansible on the control node needs some information to successfully connect to managed hosts.
- the location of the inventory file
- the connection protocol to use (default ssh)
- whether a non-standard network port is needed to connect to the server
- what user it can login as
- if the user is not root, whether ansible should escalate privileges to root
- how ansible should become root (by default with ``sudo``)
- whether to prompt for an ssh password to log in or a ``sudo`` password to gain privileges

You can set default selections for this information in your Ansible configuration file, or passing flags on the command line during invocation.

Ansible chooses its config from one of several locations, in order the given order
- if ``ANSIBLE_CONFIG`` is set, its value is the path to the file
- if environment variable is not set, Ansible will look for config in the following places
  - ``./ansible.cfg`` in the current directory where you ran the ansible command
  - ``~/.ansible.cfg`` as a dot file in the user's home directory
  - ``/etc/ansible/ansible.cfg`` the default config

``ansible --version`` clearly identifies which config file is currently being used
- ``ansible-config --version`` can also get this info

``ansible.cfg`` file
- each section contains settings as key value pairs
- section titles are closed in square brackets
- basic operations use two sections:
  - ``[defaults]`` sets defaults for Ansible operation
  - ``[privilege_escalation]`` configures how Ansible performs privilege escalation on managed hosts

Connect settings in config file is in ``[defaults]`` section
- ``remote_user`` specified the user you want to use on the managed host
  - if none specified, it uses your current user name
- ``remote_port`` specifies which port sshd is using on the managed host
  - if none specified, the default is port 22
- ``ask_pass`` controls whether Ansible will prompt you for the ssh password
  - by default it will NOT prompt for password, assuming you are using SSH key based auth

In ``[privilege_escalation]`` section of config file
- ``become`` controls whether you will auto use privilege escalation
  - default is no, and can override this at the command line or in platbooks
- ``become_user`` controls what user on the managed host Ansible should become
  - default is root
- ``become_method`` controls how Ansible will become that user
  - default uses ``sudo``, there are other options like ``su``
- ``become_ask_pass`` controls whether to prompt you for a password for your become method
  - default is no

The following is a typical ``ansible.cfg`` file
```
[defaults]
inventory = /.inventory
remote_user = ansible
ask_pass = false

[privilege_escalation]
become = true
become_user = root
become_ask_pass = false
```

Host-based connection and privilege escalation variables variables

You can apply settings specific to a particular host by setting connection variables
- easiest is placing the settings in a file in the ``host_vars`` directory in the same directory as the inventory file
  - e.g. see below file directory representation. there are two host-specific files for server1.example.com and server2.example.com
  - the hosts should appear with those names in the inventory
    ```
    project
      ansible.cfg
      host_vars
        server1.example.com
        server2.example.com
      inventory
    ```
- these settings override the ones in ``ansible.cfg``
- they have slightly different syntax and naming

- ``ansible_host`` specifies a diff IP or hostname to use for connection to this host instead of one in inventory
- ``ansible_port`` specifies the port to use for the ssh connection on this host
- ``ansible_user`` specifies whether to use privilege escalation
- ``ansible_become`` specifies the user to become on this host
- ``ansible_become_user`` specfies the user to become on this host
- ``ansible_become_method`` specifies the privilege escalation method to use for this host

Example of ``host_vars/server1.example.com``
  ```
  # connection variables for server1.example.com
  ansible_host: 192.0.2.104
  ansible_port: 34102
  ansible_user: root
  ansible_become: false
  ```
- these settings only affect ``server1.example.com`` in the inventory

Preparation of the managed host
- set up ssh key-based authentication to an unprovileged account that can use ``sudo`` to become root without password
- advantage is that you can use an account that only ansible uses and tie that to a particular ssh private key, but still have passwordless authentication
- alternatively, ssh key-based authentication to the unprivileged account, then require the ``sudo`` password for authentication to ``root``
- ansible allows you to select the mix of settings that works best for your security policy and stance

demo
``vim /etc/ansible/ansible.cfg``
  - search for ``become`` to find the area that deals with this
  - you can see defaults in ``[privileged_escalation]`` section
    - become default is true, method is sudo, user is root, and pass is false

``vim ansible.cfg`` create our own custom ``ansible.cfg``
```
[defaults]
inventory = /home/demo/ansible/inventory

[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```

``ansible --version`` to check that ansible will use our custom config file

``mkdir host_vars`` create a ``host_vars`` directory to contain our host connection settings
- we will create one file per host to specify values that supplies values that override defaults
- ``cd host_vars/``
- ``vim db01``
  ```
  # host variables for the db01 system
  ---
  become_ask_pass: True
  ```

``ansible databases --limit db01 -m ping``
- try running ansible on the databases inventory group, limited to db01 system
- error, because i'm located in teh ``host_vars`` directory. need to go up one level
- ``cd ..``
- rerun ``ansible databases --limit db01 -m ping``

``cd host_vars``
- ``vim db01`` change file by adding line 
  ```
  # host variables for the db01 system
  ---
  become_ask_pass: True
  custom_db_port: 1234
  ```
- ``cd ..``
- ``ansible databases --limit db01 -m ping``

``ansible-config dump --only-changed`` to check what we've overridden with our custom ansible.cfg

After creating and populating our hosts with keys, we can actually just directly ssh into our hosts without supplying a password
- ``ssh web01``
- ``logout``

## 3 Adhoc commands


