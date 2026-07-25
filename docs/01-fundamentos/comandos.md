# Comandos — Nivel 1

## SSH — generación y copia de llave (desde el cliente/laptop)

    ssh-keygen -t ed25519 -C "eric-optiplex-homelab"
    ssh-copy-id -i ~/.ssh/id_ed25519.pub eric@192.168.100.150

## SSH — hardening (en el servidor)

    sudo nano /etc/ssh/sshd_config
    # PasswordAuthentication no
    # PubkeyAuthentication yes
    # PermitRootLogin no

    sudo sshd -t
    sudo systemctl restart ssh

## Firewall (UFW)

    sudo ufw status
    sudo ufw allow OpenSSH
    sudo ufw enable
    sudo ufw status verbose

## Actualizaciones automáticas

    sudo apt update
    sudo apt install unattended-upgrades apt-listchanges -y
    sudo dpkg-reconfigure --priority=low unattended-upgrades
    sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
    sudo systemctl status unattended-upgrades
    sudo unattended-upgrades --dry-run --debug