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

## Extensión — Pi-hole avanzado

### Objetivo

Evolucionar Pi-hole de "bloqueador de anuncios básico" a una capa real de
privacidad y seguridad DNS: resolución recursiva propia, actualización
automática de listas, cobertura ampliada de bloqueo, y protección extendida
fuera de la red local vía VPN mesh.

### Decisiones tomadas

- **Unbound como resolver recursivo, con fallback a DNS-over-TLS**
  Alternativa original: resolución recursiva pura (Unbound hablando
  directo con los servidores raíz de internet).
  Por qué se ajustó: el ISP (Totalplay) bloquea tráfico DNS saliente
  (puerto 53) hacia servidores que no sean los suyos — confirmado
  mediante `dig` directo a un servidor raíz real, con timeout total.
  Solución adoptada: forward-zone con `forward-tls-upstream: yes` hacia
  Cloudflare (puerto 853, cifrado), evadiendo el bloqueo del puerto 53
  sin sacrificar la privacidad de que el ISP no pueda leer las consultas
  en texto plano.

- **DNS del router (Totalplay) no configurable — mitigado por dispositivo**
  Se intentó configurar Pi-hole como DNS primario a nivel de router
  (DHCP), para que toda la red lo usara automáticamente. Los campos de
  DNS en la interfaz del router aparecían permanentemente deshabilitados
  incluso tras desactivar "DHCP Relay", consistente con un patrón de
  routers de operador que restringen configuración avanzada a los
  usuarios finales. Se optó por configurar el DNS manualmente por
  dispositivo (laptop vía `nmcli`, iPhone vía ajustes de Wi-Fi) en su
  lugar.

- **`dns.listeningMode` cambiado de `LOCAL` a `all`**
  Necesario para que Pi-hole aceptara consultas desde la interfaz virtual
  de Tailscale (`100.64.0.0/10`), no reconocida por default como red
  local por dnsmasq. UFW sigue siendo la capa de control de acceso real
  (reglas explícitas por subred); este cambio solo retira un filtro
  redundante de dnsmasq que ya no correspondía a la topología de red
  actual (Tailscale sumó una interfaz nueva y legítima).

- **Grupos de Pi-hole no implementados (por ahora)**
  Evaluados y descartados por falta de necesidad concreta actual — sin
  dispositivos con requisitos de bloqueo distintos entre sí. Queda como
  mejora disponible si surge fricción real (una app/dispositivo específico
  rompiéndose por exceso de bloqueo).

### Qué se implementó

1. **Unbound** instalado nativo, configurado como resolver en
   `127.0.0.1:5335`, con forward-zone TLS hacia Cloudflare (1.1.1.1 /
   1.0.0.1, puerto 853) como mitigación al bloqueo de puerto 53 del ISP.
2. Pi-hole reconfigurado (**Settings → DNS**) para usar Unbound
   (`127.0.0.1#5335`) como único upstream, en vez de un DNS público directo.
3. Reglas UFW añadidas para permitir tráfico DNS desde la subred de LAN
   real (`192.168.100.0/24`), además de la ya existente para `proxy-net`.
4. **Actualización automática de gravity**: cronjob semanal (domingo
   4:00am) ejecutando `pihole -g`, con log de salida en
   `/var/log/pihole_gravity_update.log`.
5. **Listas de bloqueo adicionales** agregadas desde Firebog (tracking,
   malware/phishing, publicidad complementaria), ampliando la cobertura
   más allá de la lista default de Pi-hole.
6. **Tailscale** instalado nativo en el servidor, conectando el homelab
   a una red privada mesh accesible desde cualquier red externa.
7. **DNS de Tailscale a nivel de cuenta** (Admin → DNS → Nameservers)
   apuntando a la IP de Tailscale del servidor, con "Override local DNS"
   activo — todos los dispositivos con Tailscale conectado usan Pi-hole
   automáticamente, sin configuración por dispositivo adicional (salvo
   instalar la app de Tailscale misma).
8. Regla UFW adicional permitiendo el rango de Tailscale
   (`100.64.0.0/10`) hacia el puerto 53.
9. `dns.listeningMode` cambiado a `all` para que dnsmasq aceptara tráfico
   desde la interfaz de Tailscale.

### Problemas encontrados

- **Unbound en timeout total (`priming . IN NS` colgado indefinidamente).**
  Diagnóstico en capas: descartado un crash del servicio (estaba
  `active`), descartado error de sintaxis (`unbound-checkconf` limpio),
  descartado bloqueo de UFW en loopback. Aislado mediante `dig` directo
  a un servidor raíz real (`198.41.0.4`), confirmando bloqueo del ISP al
  puerto 53 saliente hacia terceros.
  Solución: forward-zone con DNS-over-TLS (puerto 853) en vez de
  resolución recursiva pura por puerto 53.

- **Monitor/tráfico bloqueado repetidamente por UFW al integrar nuevas
  fuentes de tráfico (Uptime Kuma, LAN real, Tailscale).**
  Patrón recurrente ya documentado en Nivel 3: cada nueva subred que
  necesita alcanzar un servicio nativo del host (Pi-hole, puerto 53)
  requiere una regla UFW explícita — no hay una regla "general" que lo
  cubra todo, es deliberadamente caso por caso.

- **Router (Totalplay) con campos de DNS deshabilitados, incluso tras
  desactivar DHCP Relay.**
  No resuelto a nivel de router — limitación de la interfaz de
  administración expuesta por el operador. Mitigado configurando DNS
  manualmente por dispositivo.

- **`dnsmasq warning: ignoring query from non-local network` con tráfico
  de Tailscale.**
  Causa: dnsmasq, independientemente de UFW, filtra por default las
  consultas a redes que no reconoce como "locales" — Tailscale crea una
  interfaz nueva no cubierta por ese reconocimiento automático.
  Solución: `pihole-FTL --config dns.listeningMode all`.

### Aprendizajes clave

- Un bloqueo de ISP a nivel de puerto puede ser indistinguible, a
  primera vista, de un error de configuración propio — el diagnóstico en
  capas (confirmar cada tramo por separado: servicio activo, sintaxis
  válida, firewall local, y solo al final la red externa) fue lo que
  permitió aislar la causa real sin adivinar.
- Existen al menos dos capas de filtrado independientes en un servidor
  DNS: el firewall del sistema operativo (UFW) y el filtrado propio del
  motor DNS (dnsmasq/`listeningMode`) — permitir tráfico en una no
  garantiza que la otra también lo acepte.
- No toda configuración de red puede resolverse de forma centralizada
  (router) cuando el hardware es administrado parcialmente por el
  operador — la solución por dispositivo, aunque menos elegante, es una
  mitigación válida y documentable.
- Tailscale extiende la utilidad de servicios internos (como Pi-hole)
  más allá de la red física, sin requerir IP pública ni exposición
  directa — mismo principio ya aplicado con Cloudflare Tunnel, pero para
  tráfico de VPN en vez de HTTP/HTTPS.

### Comandos utilizados

Ver [comandos.md](./comandos.md)

### Estado

✅ Completado — 2026-07-25 (pendiente: Pi-hole como DHCP, segundo Pi-hole
para redundancia)