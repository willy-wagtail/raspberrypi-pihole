FROM pihole/pihole:latest

RUN apt update -y && apt install -y unbound

COPY unbound-pihole.conf /etc/unbound/unbound.conf.d/pi-hole.conf

COPY start_unbound_and_pihole.sh start_unbound_and_pihole.sh

RUN chmod +x start_unbound_and_pihole.sh

ENTRYPOINT ./start_unbound_and_pihole.sh