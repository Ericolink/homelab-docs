# Comandos — Nivel 2

## Instalación de Docker Engine (repo oficial)

    sudo apt remove docker docker-engine docker.io containerd runc -y
    sudo apt update
    sudo apt install ca-certificates curl -y

    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc

    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

    sudo apt update
    sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

## Verificación

    sudo systemctl status docker
    sudo docker run hello-world
    docker compose version

## Grupo docker (acceso sin sudo)

    sudo usermod -aG docker eric
    # cerrar sesión y volver a entrar
    docker run hello-world

## Portainer — despliegue vía Compose

    mkdir -p ~/docker/portainer
    cd ~/docker/portainer
    nano docker-compose.yml

Contenido de `docker-compose.yml`:

    services:
      portainer:
        image: portainer/portainer-ce:latest
        container_name: portainer
        restart: unless-stopped
        ports:
          - "192.168.100.150:9443:9443"
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - portainer_data:/data

    volumes:
      portainer_data:

Levantar el stack:

    docker compose up -d
    docker compose ps
    docker ps

Obtener setup token del primer arranque:

    docker logs portainer

## Redes de Docker

    docker network ls
    docker network create proxy-net