# Comandos — Nivel 4

## Uptime Kuma

    mkdir -p ~/docker/uptime-kuma
    cd ~/docker/uptime-kuma
    nano docker-compose.yml

Contenido de `docker-compose.yml`:

    services:
      uptime-kuma:
        image: louislam/uptime-kuma:latest
        container_name: uptime-kuma
        restart: unless-stopped
        volumes:
          - uptime_kuma_data:/app/data
        networks:
          - proxy-net

    networks:
      proxy-net:
        external: true

    volumes:
      uptime_kuma_data:

Levantar:

    docker compose up -d

## Vaultwarden

    mkdir -p ~/docker/vaultwarden
    cd ~/docker/vaultwarden
    nano docker-compose.yml

Contenido de `docker-compose.yml`:

    services:
      vaultwarden:
        image: vaultwarden/server:latest
        container_name: vaultwarden
        restart: unless-stopped
        environment:
          - SIGNUPS_ALLOWED=false
          - DOMAIN=https://vault-lab.paselink.com
        volumes:
          - vaultwarden_data:/data
        networks:
          - proxy-net

    networks:
      proxy-net:
        external: true

    volumes:
      vaultwarden_data:

Levantar (con SIGNUPS_ALLOWED=true temporalmente para crear la cuenta,
luego revertir a false y reaplicar):

    docker compose up -d

## Homepage

    mkdir -p ~/docker/homepage/config
    cd ~/docker/homepage
    nano docker-compose.yml

Contenido de `docker-compose.yml`:

    services:
      homepage:
        image: ghcr.io/gethomepage/homepage:latest
        container_name: homepage
        restart: unless-stopped
        environment:
          - HOMEPAGE_ALLOWED_HOSTS=home-lab.paselink.com,192.168.100.150
        volumes:
          - ./config:/app/config
          - /var/run/docker.sock:/var/run/docker.sock:ro
        networks:
          - proxy-net

    networks:
      proxy-net:
        external: true

Configuración de servicios (`config/services.yaml`): ver contenido
completo en el repo.

Levantar:

    docker compose up -d
    docker compose restart homepage

## UFW — reglas agregadas en este nivel

    sudo ufw allow from 172.19.0.0/16 to any port 81 proto tcp

## Backup manual de Vaultwarden (mitigación temporal)

    docker run --rm -v vaultwarden_vaultwarden_data:/data -v ~/homelab-backups/vaultwarden:/backup alpine tar czf /backup/vaultwarden-volume-$(date +%Y%m%d).tar.gz -C /data .

Copiar a laptop:

    scp eric@192.168.100.150:~/homelab-backups/vaultwarden/vaultwarden-volume-*.tar.gz ~/homelab-backups/vaultwarden/

    ## Unbound — instalación y configuración

    sudo apt update
    sudo apt install unbound -y
    sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf

Contenido base (server block):

    server:
        verbosity: 0
        interface: 127.0.0.1
        port: 5335
        do-ip4: yes
        do-udp: yes
        do-tcp: yes
        do-ip6: no
        harden-glue: yes
        harden-dnssec-stripped: yes
        prefetch: yes
        num-threads: 2
        private-address: 192.168.0.0/16
        private-address: 172.16.0.0/12
        private-address: 10.0.0.0/8
        tls-cert-bundle: /etc/ssl/certs/ca-certificates.crt

    forward-zone:
        name: "."
        forward-tls-upstream: yes
        forward-addr: 1.1.1.1@853#cloudflare-dns.com
        forward-addr: 1.0.0.1@853#cloudflare-dns.com

Aplicar y verificar:

    sudo unbound-checkconf /etc/unbound/unbound.conf.d/pi-hole.conf
    sudo systemctl restart unbound
    dig @127.0.0.1 -p 5335 google.com

Diagnóstico de bloqueo de ISP (referencia):

    dig +time=5 +tries=1 @198.41.0.4 . NS

## Gravity — actualización automática

    sudo pihole -g
    sudo crontab -e

Línea agregada al crontab:

    0 4 * * 0 /usr/local/bin/pihole -g >> /var/log/pihole_gravity_update.log 2>&1

Verificar:

    sudo crontab -l
    cat /var/log/pihole_gravity_update.log

## Listas de bloqueo adicionales (Firebog)

Agregadas vía panel: Group Management → Adlists

    https://v.firebog.net/hosts/Easyprivacy.txt
    https://v.firebog.net/hosts/Prigent-Malware.txt
    https://v.firebog.net/hosts/AdguardDNS.txt

Aplicar:

    sudo pihole -g

## Tailscale

    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up
    tailscale ip -4

DNS de Tailscale configurado en https://login.tailscale.com/admin/dns
(Nameservers → Custom → IP de Tailscale del servidor → Override local DNS)

## UFW — reglas agregadas en esta extensión

    sudo ufw allow from 192.168.100.0/24 to any port 53 proto udp
    sudo ufw allow from 192.168.100.0/24 to any port 53 proto tcp
    sudo ufw allow from 100.64.0.0/10 to any port 53 proto udp
    sudo ufw allow from 100.64.0.0/10 to any port 53 proto tcp

## Pi-hole — listeningMode para aceptar tráfico de Tailscale

    sudo pihole-FTL --config dns.listeningMode all
    sudo systemctl restart pihole-FTL
    sudo pihole-FTL --config dns.listeningMode

## DNS por dispositivo (mitigación ante router sin acceso a config DNS)

Laptop (Ubuntu/NetworkManager):

    sudo nmcli connection modify "NOMBRE_RED" ipv4.dns "192.168.100.150 1.1.1.1"
    sudo nmcli connection modify "NOMBRE_RED" ipv4.ignore-auto-dns yes
    # reconectar manualmente vía interfaz gráfica para aplicar

Verificar:

    resolvectl status
    cat /etc/resolv.conf

iPhone: Ajustes → Wi-Fi → (i) en la red → Configurar DNS → Manual →
agregar 192.168.100.150 y 1.1.1.1