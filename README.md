# Contents

1. [ Setting Up Raspberry Pi ](#setuppi)
2. [ Securing Raspberry Pi ](#securingpi)
3. [ Installing Git ](#installinggit)
4. [ Installing Docker and Docker Compose ](#installingdocker)
5. [ Pihole and Unbound with Docker Compose ](#piholeandunboundwithdockercompose)
6. [ Log2Ram ](#log2ram)
7. [ Flashing IoT Devices with Tasmota using Tuya-Convert ](#tuyaconvert)
8. [ Home Assistant ](#homeassistant)

<a name="setuppi"></a>
# Setting Up Raspberry Pi

### Sources

https://www.raspberrypi.org/documentation/installation/installing-images/README.md  
https://www.raspberrypi.org/documentation/remote-access/ssh/README.md  
https://www.raspberrypi.org/documentation/configuration/wireless/headless.md  

### Preparing SD Card

On another device, download and use the "Raspberry Pi Imager" to flash RaspberryPi OS onto a micro-SD card. The Lite version does not include the desktop GUI, so use that if you are setting the raspberry pi up headless.

To enable ssh, create empty file called `.ssh` at root level on your SD card before the first boot.

I personally use ethernet, but if you want to use wifi, you can automatically connect to a wifi network on the first boot by adding a ``wpa_supplicant.conf`` file at root level before first boot with the following:

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

Put the micro-SD card into the raspberrypi and power it on. 

Wait a little for it to start up, then you can ssh into the raspberry pi using the default username "pi" and password "raspberry". Use your router or another network analyser tool (like Fing app) to find the IP address of the pi and run this command `ssh pi@<IP address>`.


### Change hostname

You can optionally change the hostname of your device as follows. Run ``sudo raspi-config`` then go to "System Options", then "Hostname". You can then type in your new hostname and reboot.

Alternatively, run ``sudo nano /etc/hosts``, change the old raspberry pi hostname to your new one, save and exit by hitting Ctrl-X and the "Y" for yes. Then run ``sudo nano /etc/hostname``, change the hostname there to the new one, save and exit. Finally reboot by running ``sudo reboot``.

### Correctly shutting down the Raspberry Pi

To restart the pi, run ``sudo reboot``.

When you want to shut down the pi, pulling power cord without properly shutting down the system increases the risk of corrupting the micro SD card. Anything running will also not save and exit gracefully. To properly shutdown, run ``sudo shutdown -h now``. Give it a second to send SIGTERM, SIGKILL signals to all running processes, and unmount all file systems. Only when the system is halted can you pull the power to the device with minimised risk. 

To start the Raspberry pi back up, simply turn on the power.

There are ways to create a power button using the GPIO pins on the board. // TODO: explore this

### Power Consumption Tweaks

If you are running a headless Raspberry Pi, then according to [this blog by Jeff Geering](https://www.jeffgeerling.com/blogs/jeff-geerling/raspberry-pi-zero-conserve-energy), you can save a little bit of power by disabling the HDMI display circuitry.

Run the command ``/usr/bin/tvservice -o`` to disable HDMI. Also run ``sudo nano /etc/rc.local`` and add the command there too in order to disable HDMI on boot.

(To enable again, run ``/usr/bin/tvservice -p``, and remove from ``/etc/rc.local``).

<a name="securingpi"></a>
# Securing Raspberry Pi

### Sources

https://www.raspberrypi.org/documentation/raspbian/updating.md  
https://www.raspberrypi.org/documentation/configuration/security.md  
https://www.youtube.com/watch?v=ukHcTCdOKrc  

### Change password for pi user
Run `sudo raspi-config`, go to “System Options”, and then “Password”.

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

To reduce idling power usage, and also reduce the areas for compromise, we can list all running services and disable services you don't need - e.g. wifi, bluetooth or sound card drivers.

To see all active services, run `sudo systemctl --type=service --state=active`.

To disable the wifi service or the bluetooth service now, run `sudo systemctl disable --now wpa_supplicant.service` or `sudo systemctl disable --now bluetooth.service` respectively.

If you wanted to enable a service again, run `sudo systemctl enable --now bluetooth.service`.

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

Use unattended-upgrades with raspberry pi specific config. 
Alternatively, set up a cron job to run the update/upgrade commands. 

See link to YouTube video above for more details.

<a name="installinggit"></a>
# Installing Git

### Sources 

https://projects.raspberrypi.org/en/projects/getting-started-with-git/3

### Install git

Run `sudo apt install -y git`, then check the installation by running `git --version`.

### Configuring git

Set up your username and email by running:
`git config --global user.name "Harry Potter"`, then `git config --global user.email "h.potter@hogwarts.prof"`.

Check the config by running `git config --list`.

The configuration is stored in the `~/.gitconfig` file. Edit this directly, or via `git config` to make further changes if required.

You can also tell git what text editor you'd like to use, for example this sets it to nano: `git config --global core.editor nano`.

<a name="installingdocker"></a>
# Installing Docker and Docker Compose

### Sources

https://www.raspberrypi.org/blog/docker-comes-to-raspberry-pi/  
https://blog.alexellis.io/getting-started-with-docker-on-raspberry-pi/  
https://sanderh.dev/setup-Docker-and-Docker-Compose-on-Raspberry-Pi/  
https://dev.to/rohansawant/installing-docker-and-docker-compose-on-the-raspberry-pi-in-5-simple-steps-3mgl  
https://www.zuidwijk.com/blog/installing-docker-and-docker-compose-on-a-raspberry-pi-4/  

### Install Docker

Run `curl -sSL https://get.docker.com | sh`.

Then add your user to the docker usergroup by running `sudo gpasswd -a pi docker` - replacing pi with whatever account you use docker on your raspberry pi. Logout using `logout` command and log back in to make sure the group setting is applied. (You can check that your username is in the right groups using this command `grep '<username>' /etc/group`.)

Now test docker is installed with `docker version`.

You can also check that your user can run a docker container by running the hello-world container, `docker run hello-world`. Afterwards, clean up by removing the container and the hello-world image by firstly getting the container id by running `docker ps -a`. Then force remove the container by running `docker rm -f <container id>` which will stop and remove it. Finally, remove the downloaded image by running `docker image rm hello-world`.

### Install Docker Compose

Run `sudo apt install -y libffi-dev libssl-dev python3 python3-pip`, then `sudo apt remove python-configparser`, and finally run `sudo pip3 -v install docker-compose`. 

Reboot the pi by running `sudo reboot`.

### Dump of Common Docker Commands

``docker ps -a``  
``docker stop <container-id>``  
``docker image ls``  
``docker image rm <image-id>``  
``docker rm -f <container-id>``  
``docker exec -it <container-id> bash``  
``docker build -t <docker-username>/<image-name>``  
``docker push <docker-username>/<image-name>``

``docker-compose pull``  
``docker-compose down``  
``docker-compose up -d``

<a name="piholeandunboundwithdockercompose"></a>
# Pihole and Unbound with Docker Compose

### Sources

https://docs.pi-hole.net/   
https://hub.docker.com/r/pihole/pihole/  
https://github.com/pi-hole/docker-pi-hole  
https://docs.pi-hole.net/guides/unbound/  
https://github.com/chriscrowe/docker-pihole-unbound  
https://github.com/anudeepND/whitelist  
https://discourse.pi-hole.net/t/solved-dns-resolution-is-currently-unavailable/33725/3  

### Start Pihole in container

> *Go to the next section if you want to start Pihole with Unbound. This is for starting Pihole only in a docker container.*

Using the `docker-compose.yaml` file from https://hub.docker.com/r/pihole/pihole/ as reference, create the file by running `touch docker-compose.yml`, then `sudo nano docker-compose.yml` to open the file in an editor, and finally copy pasting the contents in. 

Edit the file to uncomment the environment property `WEBPASSWORD` and point it to an environment variable - i.e. `WEBPASSWORD: $PIHOLE_PASSWORD`. Note that if we don't give it a password, a random one will be generated which you can find in the pihole startup logs.

Now pihole's web UI password will be set to whatever the environment variable `$PIHOLE_PASSWORD` is. You can set it on the pi by running: `export PIHOLE_PASSWORD=’<password>’`. A more permanent alternative is to create a file in the same directory as the docker-compose.yml file called `.env` by running `touch .env`, then go into the file using an editor by running `sudo nano .etc` and add your environment variables - e.g. `PIHOLE_PASSWORD=password`.

In the directory where the `docker-compose.yml` is in, run `docker-compose up -d`. You can find new container's id by running `docker ps -a`. Using the container id, you can use it to tail the logs as it starts up using `docker logs -f <docker container id>`.

### Start Pihole and Unbound in a single container

Clone this git repository by running `git clone https://github.com/willypapa/raspberrypi.git`. It will clone the files into `/raspberrypi` directory, where you will find the docker-compose.yml file. We will now refer to the `pihole-unbound` service in the docker-compose.yml. 

> Note that this uses my own docker image on docker hub. You should create your own and not rely on me.
>
> Go to `/raspberrypi/docker-pihole-unbound` directory where the Dockerfile is found which builds the pihole and unbound image.
>
> Log into docker using command `docker login` and supply your dockerhub username and password. 
>
> Now build and push your own image to your own DockerHub using `docker build -t <your_docker_username>/pihole-unbound`, then `docker push <your_docker_username>/pihole-unbound`. 
>
> Change the `image` property in the `docker-compose.yml` file from to your own, `image: <your_docker_username>/pihole-unbound:latest`.

In the directory where the `docker-compose.yml` is (i.e. `/raspberrypi`), create a `.env` file by running `sudo touch .env`, then populate it with the following environment variables. These are referred to within the docker-componse.yml file.

```
PIHOLE_PASSWORD=password
PIHOLE_TIMEZONE=Europe/London
PIHOLE_ServerIP=<IP address of the host raspberry pi - e.g. 192.168.0.2>
```

Again, in the same directory as the docker-compose.yml, run `docker-compose up -d`. You can find new container's id by running `docker ps -a`. Using the container id, you can use it to tail the logs as it starts up using `docker logs -f <docker container id>`.

To test unbound is working, access the bash terminal in the docker container by running ``docker exec -it <container-id-of-pihole-unbound> bash``. Once in, run these two commands to test DNSSEC validation: ``dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335`` and ``dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335``. The first should give a status report of ``SERVFAIL`` and no IP address. The second should give ``NOERROR`` plus an IP address. See pihole's unbound docs for more info.

To confirm Pihole is up and pointing to unbound dns server, go to the local IP address of the host raspberry pi, e.g 192.168.0.2. You should see the Pihole dashboard at /admin. Login using the password set in the environment variable ``$PIHOLE_PASSWORD``. Go to "Settings", and under the "DNS" tab, check that it is pointing to ``127.0.0.1#5335`` - which is the port we configured the unbound DNS server to listen to (see /docker-pihole-unbound/unbound-pihole.conf file).

### Whitelist common false-positives

This is optional. The github repo we refer to here keeps a list of common false-positive domains for us to whitelist.

The whitelist is installed using a python script. However, the pihole/pihole docker image does not include a python installation. So we have to run the following on the host raspberry pi itself which should have python3 installed.

Run `git clone https://github.com/anudeepND/whitelist.git`, then `sudo python3 whitelist/scripts/whitelist.py --dir <path to /etc-pihole/ volume> --docker`

### Change Your Router's DNS Server

Log onto your home network's router (usually device 1 on your subnet - e.g. 192.168.0.1), and change the default DNS server to be the IP address of the raspberry pi running pihole. Instructions on how to do this will vary depending on your router.

### Set up VPN Server

Set up a OpenVPN or WireGuard VPN server so that your devices away from home can browse the internet through your locally hosted Pihole. See https://docs.pi-hole.net/guides/vpn/openvpn/overview/.

My router supports OpenVPN so I managed to set one up using that. Otherwise you could consider using PiVPN to setup a Wireguard or OpenVPN server on your raspberry pi. See https://www.pivpn.io/.

### Troubleshooting: DNS resolution is currently unavailable

I encountered [this issue](https://discourse.pi-hole.net/t/solved-dns-resolution-is-currently-unavailable/33725) a couple of times. When starting the pihole-unbound docker container, I see in the logs that the "DNS resolution is currently unavailable".

This is solved by ``sudo nano /etc/resolv.conf``, and removing ``nameserver <host-IP-address>``, leaving your home router's IP address. Or, replace it with ``search home`` there instead of having your host IP address.

Restart the pihole-unbound container after making the change.

<a name="log2ram"></a>
# Log2Ram

### Sources

https://github.com/azlux/log2ram  
https://levelup.gitconnected.com/extend-the-lifespan-of-your-raspberry-pis-sd-card-with-log2ram-5929bf3316f2  
https://www.geekbitzone.com/posts/log2ram/log2ram-raspberry-pi/  

### Purpose

Log2ram is software that redirects logs to memory instead of the micro-SD card, only writing to the micro-SD card at set intervals or during system shutdown. By default, the interval is once a day. This supposedly extends the lifespan of micro-SD cards by reducing the number of writes to disk. 

> If you use Docker on your Raspberry Pi, note that each container has its logs written inside their respective containers rather than ``/var/log``, so won't benefit from log2ram. 
>
> I've yet to explore a way to map them to /var/log to get benefit from log2ram (e.g. as suggested in [this github issue](https://github.com/gcgarner/IOTstack/issues/8)).



### Installing Log2Ram

Add Log2Ram repository to our apt sources list, ``echo "deb http://packages.azlux.fr/debian/ buster main" | sudo tee /etc/apt/sources.list.d/azlux.list``.

Download the public key to allow us to install Log2Ram, ``wget -qO - https://azlux.fr/repo.gpg.key | sudo apt-key add -``.

Update your apt packages, ``sudo apt update``, then install log2ram, ``sudo apt install log2ram``.

Reboot once installed, ``sudo reboot``.

### Verify that it works

After reboot, check that log2ram is mounted on ``/var/log`` by runninng ``df -h``.

Also verify that log2ram is mounted to ``/var/log`` by running ``mount``.

### Uninstall

To uninstall, run ``sudo apt remove log2ram --purge``. The purge option removes the config files as well. Using the verify steps above, check that log2ram has unmounted.

<a name="tuyaconvert"></a>
# Flashing IoT Devices with Tasmota Using Tuya-Convert

### Sources

https://github.com/ct-Open-Source/tuya-convert  
https://www.youtube.com/watch?v=dt5-iZc4_qU  
https://tasmota.github.io/docs/About/

https://tasmota.github.io/docs/Upgrading/  
http://ota.tasmota.com/tasmota/  

https://templates.blakadder.com/index.html  
https://templates.blakadder.com/howto.html  
https://templates.blakadder.com/gosund_UP111.html  

### Device Compatibility

This documents what I did to successfully flash the [Gosund (UP111) UK smart plugs](https://www.amazon.co.uk/gp/product/B0856T6TJC/) over-the-air (OTA) with Tasmota firmware.

These smart plugs were bought from Amazon on January 2021. Devices purchased later may have had their firmware updated which prevents this method from working, so do your own research online to check your device's compatibility.

### Installing Tuya-Convert

Carry out this process on a fresh Raspberry Pi OS install by swapping out your micro SD card, or re-flashing it for this purpose.

Perform a ``sudo apt update`` and install git, ``sudo apt install git -y``.

 Clone the Tuya-Convert git repo, ``git clone https://github.com/ct-Open-Source/tuya-convert``, then ``cd tuya-convert``, and run ``./install_prereq.sh``.

 ### Flashing the IoT device

Start the flashing process by running ``./start_flash.sh``. You'll be presented with a warning: when you are happy to proceed, type "yes" and enter. It'll ask you if it can terminate dnsmasq process to free up port 53, type "y". It'll ask you to terminate mosquitto to free up port 1883, type "y". The script should have turned the raspberry pi into a WiFi access point with SSID "vtrust-flash".

Connect a smartphone (or any other device) to the "vtrust-flash" WiFi access point.

Plug your IoT device in and put it in autoconfig/smartconfig/pairing mode - you should know it's in the right mode when the LED starts blinking. This is usually done by pressing and holding the primary button of the device. In my case, I pressed the power button of the plug for 5 seconds.

Once the device is in pairing mode, press enter to start flashing.

After some setup, it'll then prompt you to ask which image you'd like to load onto the device. For Tasmota, select the tasmota.bin option by typing "2", then confirm with "y". After it's done, we have Tasmota on our device! Now we can need to configuring it. 

> **Remember to keep your device plugged in for the next step.**

### Configure WiFi on Tasmota

The device should now broadcast a Wifi access point with SSIP ``tasmota-xxxx``. Connect to it with your phone or another device to configure Tasmota.

One connected to the Tasmota Wifi, you can configure your home WiFi credentials. Make sure to double check the credentials before pressing "Save".

After pressing "Save", the device should restart and automatically connect to your home network. Find the IP address of the IoT device on your home network via your router or a network analyser app like Fing and connect to it to see the device's Tasmota web page.

### Upgrade Tasmota Firmware if required

Now on my device is Tasmota v8.1.0.2. However the configuration template (see next section) for my device is for Tasmota v9.1+. So next we'll need to upgrade the firmware. Note that there is a [defined upgrade path](https://tasmota.github.io/docs/Upgrading/#upgrade-flow) which we must follow, so we'll upgrade in steps from v8.1.0 to v8.5.1 to v9.1. Download the files by clicking on those versions on the [Tasmota Upgrade page](https://tasmota.github.io/docs/Upgrading/#upgrade-flow).

On the device's Tasmota web page, click on "Firmware Upgrade", upload the downloaded v8.5.1 ``tasmota-lite.bin`` file in the "Upgrade by file upload" section, and "Start upgrade". After some time, you should see it say "Upload Successful". It will then reboot. 

Repeat the above for v9.1. Note that the downloaded v9.1 file ``tasmota-lite.bin.gz`` is gzip compressed (new feature).

Lastly, either copy the URL to the non-lite and latest ``tasmota.bin.gz`` file, which can be found [here](http://ota.tasmota.com/tasmota/), in the "Upgrade by web server" section, or download it and uploading the file as before, then clicking "Start upgrade".

### GPIO Configuration on Tasmota 

> See this link for screenshots of the below steps: https://templates.blakadder.com/howto.html

On the device's Tasmota web page, go to the IP address, go to "Configuration", then "Configure Other". 

Search for your specific device's GPIO configuration on [Tasmota templates](https://templates.blakadder.com/index.html). My smart plug's configuration template is found mine [here](https://templates.blakadder.com/gosund_UP111.html). 

Paste in the device's GPIO configuration template string in "Templates" input field, check "Activate", then press "Save". The device should now reboot with the module name specified in the template selected - in this case ``Gosund UP111 Module``.

<a name="homeassistant"></a>
# Home Assistant

### Sources

https://www.home-assistant.io/getting-started/  
https://www.home-assistant.io/docs/installation/docker/#docker-compose  

### Installing Home Assistant

Create docker-compose.yml file with [these contents](https://www.home-assistant.io/docs/installation/docker/#docker-compose). Change the volume map by adding your host's path to config. Run ``docker-compose up -d``.

### Onboarding Home Assistant

Go to ``http://<IP-address-of-pi>:8123``.
