## Netwerk
Er is een netwerk aangemaakt als Zerotier netwerk. Op dit netwerk is 10.242.0.0/16 bekend.

De Home Assistant server heeft Zerotier als root draaien buiten docker.
Deze server is 10.242.0.254.
De server waarop Traefik draait is: 10.242.187.115

Eigenlijk maakt het niet heel erg uit, maar mijn netwerk zit in elkaar dat iedere locatie een ander subnet kent.
* 10.242.0.0/24 = Thuis
* 10.242.255.0/24 = Draadloze apparaten (laptop en telefoon)
* 10.242.2.0/24 = Kantoor vaste workstation
* 10.242.1.0/24 = Ongedocumenteerde houtje touwtje setup
* 10.242.187.0/24 = Gedocumenteerde houtje touwtje setup met Traefik

De naam waar het een en ander op komt draaien is ha.example.org.

Binnen het netwerk draait Home Assistant op 192.168.0.253:8123. Via Zerotier is dit bereikbaar via 10.242.0.254:8123. Dit zal eerst moeten werken voordat al het gepruts met Traefik en routing nodig is. Dus test dit eerst met een laptop of iets dergelijks.
Als dit niet werkt, vergeet dan ip forwarding niet aan te zetten op de host waar de docker container draait waar HA staat.
### Externe bereikbaarheid
De server waarop Traefik draait heeft een extern IP. Traefik draait als eigen stack binnen Portainer.

## Totale setup
Er is een Portainer installatie die alles beheerd, maar in principe komt het vanuit Traefik om zaken naar buiten toe open te zetten.

## Voorbereiding
In de sysctl van de Traefik server staat ip_forward aan voor IPv4 (https://linuxconfig.org/how-to-turn-on-off-ip-forwarding-in-linux).

In de configuratie van Home Assistant staat de trusted_proxy geconfigureerd op beide het externe ip van de server en het interne ip van de Zerotier verbinding op de kant van de Traefik host. Daarnaast is duidelijk de externe host geconfigureerd in HA.
```configuration.yml
http:
        use_x_forwarded_for: true
        trusted_proxies:
                - mijn.externe.ip.vandezerotierserver # Zerotier external
                - 10.242.187.115 # Zerotier nieuwe relay
homeassistant:
        external_url: https://ha.example.org
        internal_url: http://192.168.0.253:8123 # Dit is hoe HA intern bereikbaar is
        country: NL

```

## Traefik stack
```docker-compose.yml
version: "3.3"

services:

  traefik:
    image: "traefik:v2.10"
    container_name: "traefik"
    network_mode: "host"
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--accesslog=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--serverstransport.insecureskipverify=true"
      #- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.myresolver.acme.email=geldig@email.adres.example.org"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
      - "--providers.docker.network=proxy"
      - "--providers.file.directory=/etc/traefik"
      - "--providers.file.watch=true"
    ports:
      - "443:443"
      - "8080:8080"
    volumes:
      - "/opt/traefik/config:/etc/traefik"
      - "/opt/traefik/letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

```
### Portainer
Er is een Portainer enterprise licentie aangevraagd (voor privé gebruik), deze draait als stack en is beheersbaar via Portainer. Opmerking: hieraan komen sloopt alles, dus hiervoor is er een nood container beschikbaar.

#### Docker-compose
```yml
version: '3.3'
services:
    portainer-ee:
        container_name: port
        restart: always
        environment:
            - PORTAINER_LICENSE_KEY=Licentie key hier
        labels:
            - traefik.enable=true
            - traefik.http.routers.port.rule=Host(`portainter.example.org`)
            - traefik.http.routers.port.tls=true
            - traefik.http.services.port.loadbalancer.server.port=9000
            - traefik.http.routers.port.tls.certresolver=myresolver
        volumes:
            - '/var/run/docker.sock:/var/run/docker.sock'
            - '/opt/portainer:/data'
        image: 'portainer/portainer-ee:2.19.0'

```

Deze stack dient altijd als eerste opgestart te zijn.

### Zerotier
Op de host waar Traefik staat, draait er nog een Zerotier stack met router mogelijkheden.
#### Docker-compos
```yml
version: '3'
services:
  zerotier:
    image: "zyclonite/zerotier:router"
    container_name: zerotier-one
    devices:
      - /dev/net/tun
    network_mode: host
    volumes:
      - '/opt/zerotier-one:/var/lib/zerotier-one'
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
      - NET_RAW
    restart: unless-stopped
    environment:
      - TZ=Europe/Amsterdam
      - PUID=999
      - PGID=994
      - ZEROTIER_ONE_LOCAL_PHYS=eno1
      - ZEROTIER_ONE_USE_IPTABLES_NFT=false
      - ZEROTIER_ONE_GATEWAY_MODE=both
      - ZEROTIER_ONE_NETWORK_IDS=Zerotier netwerk ID
```
In de configuratie van Zerotier staat onder de advanced features de Allow Ethernet Bridging aan.
Allow na de eerste opstart in de web console de nieuwe server.

Het is nu mogelijk om vanaf de host (buiten docker) de andere kant te pingen.
Als bovenstaande bevestigd is:
```bash
docker exec -ti traefik ping 10.242.0.254
PING 10.242.0.254 (10.242.0.254): 56 data bytes
64 bytes from 10.242.0.254: seq=0 ttl=64 time=53.450 ms

```

## HA configuratie
### Doorsturen van verzoeken HA vanaf public IP
Maak de file /opt/traefik/config/homeassistant.yml aan:
```yml
http:
  routers:
    nondocker:
      entryPoints:
      - web
      - websecure
      rule: "Host(`ha.example.org`)"
      service: homeassistant
      tls:
        certResolver: myresolver
  services:
    homeassistant:
      loadBalancer:
        servers:
        - url: http://10.242.0.254:8123

```

Vervolgens moet Home Assistant nog wel geconfigureerd worden conform:
https://www.home-assistant.io/integrations/google_assistant/

