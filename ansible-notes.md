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

````


8:10