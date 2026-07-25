# Nivel 1 — Fundamentos

## Objetivo

Establecer una base segura de administración del servidor antes de instalar
cualquier servicio. Sin esto, cualquier siguiente capa (Docker, servicios
expuestos, etc.) hereda las mismas vulnerabilidades básicas: acceso por
contraseña, ningún firewall, sistema desactualizado.

## Contexto del servidor

- Hardware: Dell OptiPlex 3010 (ver [hardware y red](../00-hardware-y-red.md))
- SO: Ubuntu Server 24.04.4 LTS
- Acceso: SSH sobre red local (CGNAT en IPv4, IPv6 pública real)

## Decisiones tomadas

- **Autenticación SSH por llave (ed25519) en vez de contraseña**
  Alternativas consideradas: mantener contraseña fuerte, usar RSA.
  Por qué ed25519 + llave: contraseñas son vulnerables a fuerza bruta
  automatizada; ed25519 es el estándar moderno recomendado (más rápido y
  seguro que RSA en tamaños de llave equivalentes).

- **PermitRootLogin no**
  Nadie se autentica como root directo por SSH. Reduce superficie de
  ataque, ya que root es el usuario más probado por bots automatizados.

- **UFW con política default deny (incoming)**
  Alternativa considerada: configurar iptables directo.
  Por qué UFW: mismo motor (netfilter) con sintaxis mucho más simple de
  mantener y auditar a futuro.

- **unattended-upgrades solo para parches de seguridad, sin reinicio automático**
  Alternativa considerada: automatizar todas las actualizaciones.
  Por qué no: actualizaciones no relacionadas a seguridad podrían romper
  algo sin que yo lo decida. Reinicio automático desactivado para evitar
  downtime sorpresa; se revisará manualmente cuando el sistema lo requiera.

  ## Backups (plan)

- Estrategia: disco secundario dedicado a backups (no RAID), liberando el
  puerto SATA del lector DVD original del OptiPlex 3010.
- Justificación: en un homelab de aprendizaje, los errores humanos
  (borrar algo por accidente, comandos destructivos mientras se aprende)
  son más probables que fallas de hardware. RAID protege de lo segundo,
  no de lo primero.
- Off-site: pendiente, evaluado y descartado por ahora (Nivel 1). Se
  reconsiderará cuando haya datos de valor real (ej. Vaultwarden, Gitea).
- Herramienta planeada: restic o borgbackup (backups incrementales,
  cifrados). Pendiente de instalación hasta contar con el disco físico.
- Estado: 🟡 Plan definido, implementación pendiente de hardware.

## Qué se implementó

1. Generación de par de llaves SSH (ed25519) en el cliente, con passphrase.
2. Copia de llave pública al servidor (`ssh-copy-id`).
3. Verificación de login por llave antes de desactivar contraseña.
4. Desactivación de `PasswordAuthentication` y `PermitRootLogin` en `sshd_config`.
5. Validación de sintaxis (`sshd -t`) antes de reiniciar el servicio SSH.
6. Instalación y activación de UFW, permitiendo únicamente OpenSSH (IPv4 e IPv6).
7. Instalación y configuración de `unattended-upgrades` para parches de seguridad.

## Problemas encontrados

- **Ninguno crítico.** El único punto de atención fue seguir la disciplina de
  probar cada cambio (SSH por llave, luego firewall) en una sesión nueva
  antes de cerrar la sesión activa, para evitar quedar bloqueado fuera del
  servidor.

## Aprendizajes clave

- La autenticación por llave pública/privada es un modelo "algo que tienes"
  vs. contraseña que es "algo que sabes" — más resistente a ataques remotos.
- UFW es un wrapper sobre netfilter/iptables; entender esto ayuda a
  anticipar conflictos futuros (ej. Docker modificando iptables directo,
  fuera del conocimiento de UFW — pendiente para Nivel 2).
- Separar actualizaciones de seguridad (automáticas) de actualizaciones
  generales (manuales) da control sin sacrificar parches críticos.

## Comandos utilizados

Ver [comandos.md](./comandos.md)

## Estado

✅ Completado — 2026-07-24