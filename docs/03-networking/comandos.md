# Comandos — Nivel 3

## Diagnóstico de conflicto de puerto 80

    sudo ss -tlnp | grep :80
    docker ps --format "table {{.Names}}\t{{.Ports}}"

## Pi-hole — reconfigurar puerto del panel web (v6+)

    sudo pihole-FTL --config webserver.port
    sudo pihole-FTL --config webserver.port "8053o,[::]:8053o"
    sudo systemctl restart pihole-FTL
    sudo ss -tlnp | grep 8053

## Nginx Proxy Manager — despliegue vía Compose

    mkdir -p ~/docker/npm
    cd ~/docker/npm
    nano docker-compose.yml

Contenido de `docker-compose.yml`:

    services:
      npm:
        image: jc21/nginx-proxy-manager:latest
        container_name: npm
        restart: unless-stopped
        ports:
          - "192.168.100.150:80:80"
          - "192.168.100.150:443:443"
          - "192.168.100.150:81:81"
        volumes:
          - npm_data:/data
          - npm_letsencrypt:/etc/letsencrypt
        networks:
          - proxy-net

    networks:
      proxy-net:
        external: true

    volumes:
      npm_data:
      npm_letsencrypt:

Levantar:

    docker compose up -d
    docker compose ps

## Cloudflare Tunnel — despliegue vía Compose

    mkdir -p ~/docker/cloudflared
    cd ~/docker/cloudflared
    nano docker-compose.yml

Contenido de `docker-compose.yml` (token gestionado fuera del repo):

    services:
      cloudflared:
        image: cloudflare/cloudflared:latest
        container_name: cloudflared
        restart: unless-stopped
        command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
        networks:
          - proxy-net

    networks:
      proxy-net:
        external: true

Levantar:

    docker compose up -d
    docker logs cloudflared

## Portainer — agregar a la red compartida

    cd ~/docker/portainer
    nano docker-compose.yml
    # agregar bajo el servicio:
    #   networks:
    #     - proxy-net
    # agregar al final del archivo:
    # networks:
    #   proxy-net:
    #     external: true

    docker compose up -d

## Verificación de red compartida

    docker network inspect proxy-net

## Diagnóstico de conectividad interna entre contenedores

    docker exec -it npm sh -c "curl -I http://localhost:80"
    docker exec -it npm sh -c "curl -kI https://portainer:9443"

## Verificación externa end-to-end

    curl -v https://portainer-lab.paselink.com

## Pi-hole — integración con reverse proxy

Proxy Host en NPM:
- Domain Names: pihole-lab.paselink.com
- Scheme: http
- Forward Hostname/IP: 192.168.100.150
- Forward Port: 8053

## UFW — permitir tráfico de contenedores hacia servicios del host

    sudo ufw allow from 172.19.0.0/16 to any port 8053 proto tcp
    sudo ufw status verbose

## Diagnóstico escalonado NPM → servicio nativo del host

    curl -v --max-time 5 http://127.0.0.1:8053/admin/
    docker exec -it npm sh -c "curl -v --max-time 5 http://192.168.100.150:8053/admin/"
    curl -v -H "Host: pihole-lab.paselink.com" http://192.168.100.150:80/admin/

## Pi-hole — cambiar contraseña del panel

    sudo pihole setpassword