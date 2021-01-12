# Setting up Raspberry Pi

### Sources

https://www.raspberrypi.org/documentation/configuration/security.md

https://www.youtube.com/watch?v=ukHcTCdOKrc

### Change password for pi user
Run `sudo raspi-config`, go to “System Options”, and then “Password”

### Set up ssh

Either create empty file named “ssh” at root level on your SD card before the first boot, or run `sudo raspi-config`, then go to “Interface Options” and then “SSH”.

### Create a new administrative superuser account

Run `sudo adduser <account_name>`, then `sudo gpasswd -a <account_name> adm`, and then `sudo gpasswd -a <account_name> sudo`.

Finally, check the new user is capable of logging in to the raspberrypi using a new terminal and is able to use sudo by running `sudo whoami`.

### Lock the pi account

We could delete the pi accound instead of locking it, but some software relies on the pi account which is why we aren't deleting it. 

Log onto the administrative superuser account set up in the previous step, then run `sudo passwd -l pi`.

### Updating and upgrading rasp pi OS

Run `sudo apt update`, then `sudo apt full-upgrade`, and finally `sudo apt clean`.

### Kill unnecessary system services

List running services and disable services you don't need - e.g. wifi or bluetooth.

To see all active services, run `sudo systemctl --type=service --state=active`.

To disable the wifi service now, run `sudo systemctl disable --now wpa_supplicant.service`.

To disable the bluetooth service now, run`sudo systemctl disable --now bluetooth.service`

If you wanted to enable a service again by running `sudo systemctl enable --now bluetooth.service`.

### Restrict ssh accounts

Run `sudoedit /etc/ssh/sshd_config`.

Under the line “# Authentication”, add `AllowUsers <account_name>`.

### Automatic rasp pi OS update and upgrading

You can use unattended-upgrades with rasp pi specific config change. Could also use a cron job. Not for production because of potential problems. See link to YouTube video above.

### Firewall

Use ufw (uncomplicated firewall). Need to be careful not to lock yourself out. See links to rasp pi doc and YouTube video above.

### Brute-force detection

Use fail2ban which watchs system logs for repeated login attempts and add a firewall rule to prevent further access for a specified time. See links to rasp pi doc and YouTube video above.
