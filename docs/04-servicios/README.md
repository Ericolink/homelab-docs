# Nivel 4 — Servicios

## Objetivo

Desplegar servicios de uso real y diario, aprovechando la base de
infraestructura (Docker, reverse proxy, túnel, firewall) construida en
niveles anteriores, priorizando utilidad práctica sobre completar una
lista genérica de aplicaciones.

## Contexto

- Requisito previo: Niveles 1-3 completos (fundamentos, Docker, networking).
- Todos los servicios de este nivel se integran al esquema existente:
  contenedor en `proxy-net` → Proxy Host en NPM → ruta en Cloudflare Tunnel
  → subdominio público bajo `paselink.com`.

## Decisiones tomadas

- **Gitea evaluado y descartado**
  Se consideró como parte del roadmap original, pero se decidió no
  instalarlo: GitHub ya cubre la necesidad de control de versiones de
  forma satisfactoria, y duplicar la función sin una necesidad concreta
  no era buen uso del tiempo ni de los recursos del servidor.
  Consecuencia: Nivel 5 (Automatización) se apoyará en GitHub Actions con
  un runner self-hosted, en vez de Gitea Actions — decisión que además
  resulta más transferible a un entorno laboral real, donde GitHub
  Actions es considerablemente más común.

- **Homepage con acceso de solo lectura al socket de Docker**
  A diferencia de Portainer (que requiere control completo de Docker
  para administrar contenedores), Homepage solo necesita leer el estado
  para mostrar widgets. Se montó el socket como `:ro` (read-only),
  aplicando el principio de mínimo privilegio.

- **SQLite como base de datos para servicios que lo permiten**
  Alternativa considerada: bases de datos separadas (PostgreSQL) para
  cada servicio.
  Por qué SQLite: para uso personal de un solo usuario administrador, el
  overhead de mantener bases de datos separadas no se justifica en este
  hardware. Se documenta como decisión consciente, no como limitación
  desconocida — la alternativa con Postgres es la que se usaría si el
  uso creciera a nivel de equipo.

- **Separación de responsabilidades en monitoreo de Pi-hole**
  Se configuraron monitores independientes para el panel web y para la
  función real de resolución DNS de Pi-hole en Uptime Kuma, en vez de un
  único monitor genérico — permite distinguir cuál de las dos
  responsabilidades falló específicamente.

- **Backup de Vaultwarden temporal y manual, sin disco secundario aún**
  Mitigación mientras no hay presupuesto para el disco de backups
  planeado desde Nivel 1: exports cifrados manuales del vault + `tar.gz`
  del volumen Docker, copiados a almacenamiento externo (laptop). Se
  documenta como solución intermedia, no definitiva.

## Qué se implementó

1. **Uptime Kuma**: monitoreo activo de Portainer, NPM, Pi-hole (panel web
   y función DNS por separado), desplegado sin puertos publicados
   directamente (acceso solo vía NPM/túnel).
2. **Vaultwarden**: gestor de contraseñas self-hosted compatible con
   clientes Bitwarden. Registro de cuentas cerrado tras crear el único
   usuario administrador (`SIGNUPS_ALLOWED=false`).
3. **Homepage**: dashboard central con enlaces a todos los servicios
   desplegados hasta el momento, con acceso de solo lectura a Docker.
4. Todos los servicios integrados al esquema de reverse proxy + túnel ya
   existente, con subdominios propios bajo `paselink.com`.

## Problemas encontrados

- **Monitor DNS de Pi-hole en Uptime Kuma marcado como caído
  incorrectamente.**
  Causa: el campo "Servidor de resolución" del monitor tenía Cloudflare
  como valor por default, en vez de apuntar a la IP real de Pi-hole
  (`192.168.100.150`) — el monitor estaba probando la resolución DNS de
  Cloudflare, no la de Pi-hole.
  Solución: cambiado explícitamente el servidor de resolución a la IP
  del servidor.

- **Monitor de NPM marcado como caído.**
  Causa: mismo patrón que el conflicto Docker/UFW documentado en Nivel 3
  — el contenedor de Uptime Kuma no podía alcanzar el puerto 81 del host
  por falta de regla de firewall específica.
  Solución: `sudo ufw allow from 172.19.0.0/16 to any port 81 proto tcp`.

- **Homepage: "Host validation failed" al primer acceso.**
  Causa: validación de seguridad interna de Homepage (protección contra
  DNS rebinding) rechaza por default cualquier host no reconocido
  explícitamente.
  Solución: variable de entorno `HOMEPAGE_ALLOWED_HOSTS` configurada con
  el dominio público y la IP de LAN del servidor.

## Aprendizajes clave

- No todo lo planeado en un roadmap inicial tiene que implementarse tal
  cual — descartar Gitea con una justificación clara es una decisión de
  arquitectura tan válida como instalar algo nuevo.
- El patrón de conflicto Docker/UFW (contenedor → host requiere regla
  explícita de firewall) se repite con cada servicio nuevo que necesite
  alcanzar algo nativo del sistema — ya es un chequeo de rutina al
  integrar cualquier monitor o integración nueva.
- Separar el nivel de privilegio de acceso al socket de Docker según la
  necesidad real de cada servicio (lectura vs. control completo) es una
  aplicación concreta y cotidiana del principio de mínimo privilegio.
- Un backup manual imperfecto pero ejecutado es mejor que uno automatizado
  perfecto pero pospuesto indefinidamente por falta de hardware — se
  optó por una mitigación intermedia documentada en vez de dejar
  Vaultwarden sin ninguna protección.

## Comandos utilizados

Ver [comandos.md](./comandos.md)

## Estado

✅ Completado — 2026-07-25 (Gitea descartado por decisión; backup de
Vaultwarden pendiente de disco secundario)