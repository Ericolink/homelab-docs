# Nivel 2 — Docker

## Objetivo

Instalar y entender la plataforma de contenedores que va a alojar todos
los servicios futuros del homelab, de forma aislada, reproducible y
documentada como código (Compose), en vez de instalaciones manuales
directas sobre el sistema operativo base.

## Contexto

- Requisito previo: Nivel 1 completo (SSH endurecido, firewall activo,
  usuario administrador con sudo).
- Hardware relevante: 7.7 GB RAM, CPU de 2 núcleos/4 hilos — suficiente
  para varios contenedores ligeros simultáneos, no para máquinas
  virtuales completas.

## Decisiones tomadas

- **Docker Engine desde el repositorio oficial, no `docker.io` de Ubuntu**
  Alternativa considerada: paquete `docker.io` de los repos estándar de Ubuntu.
  Por qué el oficial: el paquete de Ubuntu suele ir desactualizado respecto
  a las versiones que mantiene Docker directamente; el repo oficial
  garantiza actualizaciones más oportunas y el plugin de Compose integrado.

- **Usuario `eric` agregado al grupo `docker`**
  Alternativa considerada: usar siempre `sudo docker ...`.
  Por qué sí, con reservas: el grupo `docker` es funcionalmente equivalente
  a acceso root (el daemon corre como root y cualquier miembro del grupo
  puede montar el filesystem completo del host dentro de un contenedor).
  Se acepta el riesgo porque `eric` ya tiene sudo completo — no se otorga
  privilegio adicional real. Decisión explícita: **no** agregar futuros
  usuarios de menor privilegio a este grupo sin evaluarlo caso por caso.

- **Docker Compose como método estándar, no `docker run` suelto**
  Cada servicio se define en su propio `docker-compose.yml`, versionado
  en este repo, sirviendo como documentación viva de la configuración.

- **Bind mount vs Named volume — criterio aplicado**
  Bind mount para acceso directo a recursos específicos del host (ej.
  socket de Docker). Named volume para datos gestionados internamente
  por el contenedor (ej. base de datos interna de Portainer).

- **Puertos publicados restringidos a la IP de LAN, no a `0.0.0.0`**
  Motivo: Docker modifica `iptables` directamente y **no respeta las
  reglas de UFW**, por lo que un puerto publicado sin restricción queda
  expuesto a toda interfaz de red disponible, sin que UFW lo bloquee.
  Se adoptó como práctica estándar especificar la IP de bind explícita
  en cada `ports:` de cada servicio.

## Qué se implementó

1. Instalación de Docker Engine + Compose plugin desde el repo oficial,
   con verificación de llave GPG.
2. Verificación de funcionamiento (`docker run hello-world`).
3. Usuario `eric` agregado al grupo `docker`, con entendimiento explícito
   de la implicación de seguridad.
4. Primer servicio real desplegado vía Compose: **Portainer** (panel de
   administración visual de Docker).
5. Identificación y mitigación del conflicto Docker/UFW (bind restringido
   a IP de LAN en vez de todas las interfaces).
6. Creación de red compartida `proxy-net`, preparada para Nivel 3
   (Nginx Proxy Manager y servicios detrás de él).

## Problemas encontrados

- **Setup token de Portainer no visible en la interfaz web.**
  Solución: obtenido vía `docker logs portainer`, donde Portainer lo
  imprime al arrancar por primera vez, como medida de seguridad para
  evitar que cualquiera con la URL pueda crear el usuario administrador.

- **Falsa sensación de seguridad con UFW.**
  `ufw status` mostraba solo SSH permitido, pero Portainer era accesible
  igual desde la LAN en el puerto 9443 sin que UFW lo hubiera autorizado
  explícitamente. Causa: Docker inserta sus propias reglas en `iptables`
  (cadena `DOCKER-USER`), con prioridad sobre las reglas de UFW.
  Solución aplicada: bind explícito del puerto a la IP de LAN del
  servidor en vez de `0.0.0.0`.

## Aprendizajes clave

- Un contenedor comparte el kernel del host (a diferencia de una VM
  completa), lo cual lo hace mucho más liviano pero también implica que
  el aislamiento no es total — de ahí la importancia del grupo `docker`
  y de revisar qué se monta dentro de cada contenedor.
- UFW y Docker operan en capas distintas del sistema de firewall; UFW no
  es una fuente de verdad completa sobre qué puertos están realmente
  expuestos cuando hay contenedores publicando puertos.
- Docker Compose crea redes bridge aisladas por proyecto de forma
  automática; servicios en distintos `docker-compose.yml` no se
  descubren entre sí por nombre a menos que se conecten explícitamente
  a una red externa compartida.

## Comandos utilizados

Ver [comandos.md](./comandos.md)

## Estado

✅ Completado — 2026-07-25