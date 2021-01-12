# Running Pihole in Docker

### Start Pihole in container

Sources: https://hub.docker.com/r/pihole/pihole/ and https://github.com/pi-hole/docker-pi-hole

The docker-compose.yaml sets pihole's password to whatever the environment variable  `$PIHOLE_PASSWORD` is. Set it on the pi by running: `export PIHOLE_PASSWORD=’<password>’`.

The docker-compose.yaml also sets pihole's timezone to whatever the environment variable  `$PIHOLE_TIMEZONE` is. Set it on the pi by running: `export PIHOLE_TIMEZONE=’Europe/London’`.

Run the docker-compose.yml file using command `docker-compose up -d`.

### Whitelist common false-positives

Source: https://github.com/anudeepND/whitelist

The pihole docker image does not include python, which is how the whitelist gets installed. So we have to run the following on the host raspberry pi as it does have python v3 installed:

Run `git clone https://github.com/anudeepND/whitelist.git`, then `sudo python3 whitelist/scripts/whitelist.py --dir </home/docker/etc-pihole/> --docker`
