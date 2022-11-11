# Pihole with Unbound DNS

## 1 Setup

### 1.1 Environment Variables

Create environment variables referenced in the `docker-compose.yml` file:

- `sudo nano .env` in same directory as `docker-compose.yml`
- add environment variables referenced by `docker-compose.yml` to the file. See https://github.com/pi-hole/docker-pi-hole/#environment-variables for explanation of the variables. E.g.:
  ```
  FTLCONF_LOCAL_IPV4=<local IP Address of host>
  TZ=Europe/London
  WEBPASSWORD=<password Pihole for web UI>
  REV_SERVER=true
  REV_SERVER_DOMAIN=<Router's DCHP Domain name e.g local>
  REV_SERVER_TARGET=<Router's IP, e.g. 192.168.1.1>
  REV_SERVER_CIDR=192.168.0.0/24
  HOSTNAME=pihole
  DOMAIN_NAME=pihole.local
  PIHOLE_WEBPORT=80
  WEBTHEME=default-light
  ```

### 1.2 Build and start

Build and start pihole-unbound docker container:

- `docker-compose up -d`
- `docker ps -a` to get container id
- `docker logs -f <container_id>` to follow the startup logs

### 1.3 Validate

Test Unbound is working (See Pihole's Unbound guide for more info):
- `docker exec -it <container-id-of-pihole-unbound> bash` to access the bash terminal in the docker container
- `dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335` should give a status report of `SERVFAIL` and no IP address.
- `dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335` should give `NOERROR` plus an IP address. 

Confirm Pihole is up and pointing to Unbound dns server
- Go to the local IP address of the host raspberry pi, e.g 192.168.0.2. You should see the Pihole dashboard at `<192.168.0.2>/admin`. Log in using the password set in the environment variable `$WEBPASSWORD`. 
- Go to "Settings" and under the "DNS" tab, check that it is pointing to `127.0.0.1#5335` - which is the port we configured the unbound DNS server to listen to (see the `/docker-pihole-unbound/unbound-pihole.conf` file).

### 1.4 Whitelists (optional)

Add whitelist of common false-positives (optional)

- `git clone https://github.com/anudeepND/whitelist.git ~/Documents/pihole-whitelist`
- `cd ~/Documents/pihole-whitelist/`
- `sudo python3 ./scripts/whitelist.py --dir ~/.pihole-unbound/etc-pihole/ --docker`

On your home network's router, change the default DNS server to point to the IP address of the pihole-unbound host.

Verify that advert blocking works using this [ad-blocker test](https://ads-blocker.com/testing/).

## 2 Teardown

Stop pihole-unbound docker container

- `docker-compose down`

## 3 Update

Stop pihole-unbound docker container

- `docker-compose down`

Start pihole-unbound docker container again, forcing it to build again by supplying flag.

- `docker-compose up --build -d`
