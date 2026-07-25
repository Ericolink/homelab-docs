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