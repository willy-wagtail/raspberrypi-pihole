# Setting Up Raspberry Pi


### Sources
https://www.raspberrypi.org/documentation/installation/installing-images/README.md

https://www.raspberrypi.org/documentation/remote-access/ssh/README.md

https://www.raspberrypi.org/documentation/configuration/wireless/headless.md

### Preparing SD Card

Use Raspberry Pi Imager to flash OS onto SD card.

To enable ssh, create empty file named “ssh” at root level on your SD card before the first boot.

To automatically connect to a wifi network, add a wpa_supplicant.conf file at root level before first boot with the following:

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=<Insert 2 letter ISO 3166-1 country code here>

network={
 ssid="<Name of your wireless LAN>"
 psk="<Password for your wireless LAN>"
}
```

### First boot

Ssh into the raspberry pi using the username "pi" and password "raspberry".

# Securing Raspberry Pi

### Sources

https://www.raspberrypi.org/documentation/raspbian/updating.md

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

We could delete the pi accound instead of locking it, but some software still relies on the pi account to work.

Log onto the administrative superuser account set up in the previous step, then run `sudo passwd --lock pi`.

### Updating and upgrading rasp pi OS

Run `sudo apt update`, then `sudo apt full-upgrade -y`, and finally `sudo apt clean` to clean up the downloaded package files.

### Kill unnecessary system services

List running services and disable services you don't need - e.g. wifi or bluetooth.

To see all active services, run `sudo systemctl --type=service --state=active`.

To disable the wifi service now, run `sudo systemctl disable --now wpa_supplicant.service`.

To disable the bluetooth service now, run`sudo systemctl disable --now bluetooth.service`

If you wanted to enable a service again by running `sudo systemctl enable --now bluetooth.service`.

### Restrict ssh accounts

Run `sudoedit /etc/ssh/sshd_config`.

Under the line “# Authentication”, add `AllowUsers <account_name1> <account_name2>`.

After the change, you will need to restart the sshd service using `sudo systemctl restart ssh` or rebooting.

### Firewall

Use ufw (uncomplicated firewall). Need to be careful not to lock yourself out. See links to rasp pi doc and YouTube video above.

### Brute-force detection

Use fail2ban which watchs system logs for repeated login attempts and add a firewall rule to prevent further access for a specified time. See links to rasp pi doc and YouTube video above.

### Automatic package update and upgrade

Not for production because of potential compatibility problems that may arise.

Use unattended-upgrades with raspberry pi specific config. Another option is to set up a cron job to run the update/upgrade commands. 

See link to YouTube video above for more details.

# Installing Docker

### Sources

https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/

https://blog.alexellis.io/getting-started-with-docker-on-raspberry-pi/

https://sanderh.dev/setup-Docker-and-Docker-Compose-on-Raspberry-Pi/

https://dev.to/rohansawant/installing-docker-and-docker-compose-on-the-raspberry-pi-in-5-simple-steps-3mgl

https://www.zuidwijk.com/blog/installing-docker-and-docker-compose-on-a-raspberry-pi-4/

### Install Docker

Run `curl -sSL https://get.docker.com | sh`, then run `sudo gpasswd -a pi docker`, replacing pi with whatever account is the main account. Logout using `logout` command and log back in.

Test it by showing version and running the hello-world container: `docker version` and `docker run hello-world`. Afterwards, clean up by removing the container and the hello-world image: `docker rm <container id>` then `docker image rm hello-world`.

### Install Docker Compose

Run `sudo apt install libffi-dev libssl-dev python3 python3-pip`, then `sudo apt remove python-configparser`, and finally run `sudo pip3 -v install docker-compose`. Reboot the pi.

# Pihole with Docker Compose

### Sources

https://hub.docker.com/r/pihole/pihole/ 

https://github.com/pi-hole/docker-pi-hole

### Start Pihole in container

Use the docker-compose.yaml in the above links. 

Edit the file to uncomment the environment property `WEBPASSWORD` and point it to the host's environment variable - i.e. `WEBPASSWORD: $PIHOLE_PASSWORD`. If we don't give it a password, a random one will be generated.

The docker-compose.yml sets pihole's password to whatever the environment variable `$PIHOLE_PASSWORD` is. Set it on the pi by running: `export PIHOLE_PASSWORD=’<password>’`.

Run the `docker-compose.yml` in `/docker-pihole` directory using command `docker-compose up -d`.

### Start Pihole and Unbound in a single container

Sources: https://docs.pi-hole.net/guides/unbound/ and https://github.com/chriscrowe/docker-pihole-unbound

The docker-compose.yaml sets pihole's password to whatever the environment variable  `$PIHOLE_PASSWORD` is. Set it on the pi by running: `export PIHOLE_PASSWORD=’<password>’`.

Run the `docker-compose.yml` in `/docker-unbound-pihole` directory using command `docker-compose up -d`.

### Whitelist common false-positives

Source: https://github.com/anudeepND/whitelist

The pihole docker image does not include python, which is how the whitelist gets installed. So we have to run the following on the host raspberry pi as it does have python v3 installed:

Run `git clone https://github.com/anudeepND/whitelist.git`, then `sudo python3 whitelist/scripts/whitelist.py --dir </home/docker/etc-pihole/> --docker`
