# Pihole with Unbound DNS

## 1 Setup

Create environment variables referenced in the `docker-compose.yml` file:

- `sudo nano .env` in same directory as `docker-compose.yml`
- add the referenced env variables to the file, e.g.:
  ```
  PIHOLE_PASSWORD=<password Pihole for web UI>
  PIHOLE_TIMEZONE=Europe/London
  PIHOLE_ServerIP=<local IP Address of host>
  ```

Build and start pihole-unbound docker container:

- `docker-compose up -d`
- `docker ps -a` to get container id
- `docker logs <container_id>` to check the startup logs

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
