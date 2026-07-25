# Nivel 3 — Networking

## Objetivo

Establecer un punto de entrada único, seguro y con HTTPS válido para todos
los servicios del homelab, resolviendo la restricción de CGNAT del ISP
mediante un túnel saliente en vez de exposición directa de puertos.

## Contexto

- Restricción de red confirmada en `docs/00-hardware-y-red.md`: sin IP
  pública IPv4 (CGNAT), IPv6 pública real disponible pero no utilizada
  como vía principal de exposición.
- Dominio propio disponible: `paselink.com` (dominio de producto,
  gestionado previamente en GoDaddy).

## Decisiones tomadas

- **Migración de DNS de `paselink.com` de GoDaddy a Cloudflare**
  Requisito técnico para poder usar Cloudflare Tunnel. Ejecutada con
  respaldo previo de todos los registros DNS existentes antes del cambio
  de nameservers, para no afectar la disponibilidad del producto en
  producción durante la migración.

- **Cloudflare Tunnel en vez de port forwarding tradicional**
  Alternativa descartada: port forwarding directo en el router.
  Por qué: inviable por CGNAT, el router no tiene IP pública real que
  exponer. Cloudflare Tunnel usa una conexión saliente desde el servidor,
  evitando la necesidad de puertos entrantes por completo.

- **Esquema de subdominios de un solo nivel** (`servicio-lab.paselink.com`)
  en vez de anidado (`servicio.lab.paselink.com`)
  Motivo: el certificado gratuito Universal SSL de Cloudflare solo cubre
  el dominio raíz y un nivel de subdominio (`*.paselink.com`). Subdominios
  de segundo nivel no están cubiertos sin Advanced Certificate Manager
  (add-on de pago). Se optó por aplanar el esquema de nombres en vez de
  pagar por cobertura adicional.

- **Nginx Proxy Manager (NPM) como reverse proxy interno, detrás del túnel**
  El túnel de Cloudflare apunta a un único destino interno (`npm:80`);
  NPM es quien decide, según el dominio solicitado, a qué contenedor
  interno reenviar cada petición. Permite agregar servicios nuevos sin
  crear una ruta de túnel distinta para cada uno.

- **Terminación TLS en el borde de Cloudflare, no en NPM**
  Los Proxy Hosts en NPM para servicios detrás del túnel no gestionan
  certificado Let's Encrypt propio (innecesario y en conflicto, ya que
  Cloudflare provee HTTPS válido de cara al público). El tramo NPM→servicio
  interno se maneja por separado, según lo que cada servicio exponga.

- **Regla de UFW restringida por subred de origen, no apertura general**
  Al conectar NPM (contenedor) con Pi-hole (servicio nativo del host) fue
  necesario abrir el puerto 8053 en UFW. Alternativa descartada: abrir el
  puerto a cualquier origen (`ufw allow 8053`). Se optó por restringir el
  origen permitido a la subred de la red Docker `proxy-net`
  (`172.19.0.0/16`), siguiendo el principio de mínimo privilegio ya
  aplicado en niveles anteriores — solo el tráfico que realmente lo
  necesita puede alcanzar ese puerto, no toda la LAN ni internet.

## Qué se implementó

1. Migración de DNS de `paselink.com` a Cloudflare (nameservers
   actualizados en GoDaddy, verificación de registros existentes antes
   del cambio).
2. Creación de túnel Cloudflare (`homelab-optiplex`), desplegado como
   contenedor Docker (`cloudflared`), conectado a la red `proxy-net`.
3. Ruta pública configurada: `portainer-lab.paselink.com` → `http://npm:80`.
4. Nginx Proxy Manager desplegado vía Compose, con puertos 80/443/81
   restringidos a la IP de LAN del servidor.
5. Proxy Host en NPM: `portainer-lab.paselink.com` → `https://portainer:9443`.
6. Todos los contenedores relevantes (`npm`, `cloudflared`, `portainer`)
   conectados a la red compartida `proxy-net` para descubrimiento por
   nombre.
7. Pi-hole (servicio nativo preexistente, no containerizado) integrado al
   mismo esquema de reverse proxy: `pihole-lab.paselink.com` → NPM →
   `192.168.100.150:8053` (panel web, reconfigurado desde el puerto 80
   original para liberar espacio a NPM).
8. Regla de firewall específica (UFW) para permitir que contenedores en
   `proxy-net` alcancen servicios nativos del host en puertos no estándar.

## Problemas encontrados

- **Conflicto de puerto 80 con Pi-hole preinstalado.**
  Causa: Pi-hole (instalado previamente, fuera de este proceso
  documentado) ocupaba el puerto 80 con su panel web, vía `pihole-FTL`
  (Pi-hole v6+ ya no usa lighttpd como versiones anteriores).
  Solución: reconfigurado el puerto del panel de Pi-hole a 8053 mediante
  `pihole-FTL --config webserver.port`, liberando 80/443 para NPM.

- **Error de negociación TLS (`SSL_ERROR_NO_CYPHER_OVERLAP`) al conectar
  NPM → Portainer.**
  Causa: el Proxy Host se configuró inicialmente con esquema `http` hacia
  el puerto 9443 de Portainer, que expone únicamente HTTPS/TLS. Desajuste
  de protocolo en el tramo interno.
  Solución: cambiado el esquema del Proxy Host a `https` manteniendo el
  puerto 9443.

- **502 Bad Gateway con certificado ya válido.**
  Causa: el contenedor `portainer` no estaba conectado a la red
  `proxy-net` (el cambio se había editado en el archivo `docker-compose.yml`
  pero no se había aplicado con `docker compose up -d`). NPM no podía
  resolver el nombre `portainer` por DNS interno de Docker.
  Solución: reaplicado el compose de Portainer; verificado con
  `docker network inspect proxy-net` que los tres contenedores
  (`npm`, `cloudflared`, `portainer`) aparecieran listados antes de
  volver a probar.

- **Certificado inválido con subdominio de dos niveles.**
  Causa: `portainer.lab.paselink.com` (dos niveles de subdominio) no está
  cubierto por el certificado Universal SSL gratuito de Cloudflare, que
  solo cubre el dominio raíz y un nivel de subdominio.
  Solución: renombrado el esquema a un solo nivel
  (`portainer-lab.paselink.com`), cubierto automáticamente por el
  certificado gratuito.

- **502 Bad Gateway persistente al conectar NPM → Pi-hole, con Pi-hole
  respondiendo correctamente de forma local.**
  Causa: a diferencia de la comunicación contenedor-a-contenedor (que no
  pasa por UFW), el tráfico de un contenedor hacia la IP real del host sí
  es evaluado por UFW como tráfico entrante normal. La política
  default-deny bloqueaba silenciosamente las peticiones de NPM hacia
  `192.168.100.150:8053`, generando timeout (visible como 502 en el
  navegador, sin mensaje de error claro sobre la causa real).
  Diagnóstico: aislado mediante pruebas escalonadas — Pi-hole en
  `127.0.0.1` (funcionó), luego el mismo request desde dentro del
  contenedor NPM (timeout) — lo que apuntó directo a un problema de
  firewall entre el namespace de red de Docker y el host.
  Solución: regla UFW explícita permitiendo el origen `172.19.0.0/16`
  (subred de `proxy-net`) hacia el puerto 8053/tcp.

## Aprendizajes clave

- Un reverse proxy centraliza el enrutamiento por dominio y, típicamente,
  la terminación HTTPS — pero el protocolo del tramo *interno* (proxy →
  servicio) es independiente y debe verificarse servicio por servicio.
- Un túnel saliente (Cloudflare Tunnel) resuelve el problema de exposición
  de servicios bajo CGNAT sin requerir IP pública ni port forwarding,
  usando solo conexiones salientes desde el servidor.
- Los certificados wildcard gratuitos tienen límites de profundidad de
  subdominio bien documentados mas fáciles de ignorar hasta toparse con
  ellos en la práctica — vale la pena diseñar el esquema de nombres con
  esto en mente desde el inicio, no después.
- Verificar el estado real (`docker network inspect`, `docker ps`) es más
  confiable que asumir que un archivo editado ya se aplicó — un cambio en
  `docker-compose.yml` no tiene efecto hasta el siguiente `up`.
- UFW trata de forma distinta el tráfico *publicado por Docker hacia
  afuera* (lo ignora, como se documentó en Nivel 2) del tráfico de *un
  contenedor hacia la IP del host* (sí lo evalúa como tráfico entrante
  normal). Son dos caras del mismo par Docker/UFW, con comportamientos
  opuestos según la dirección del tráfico.
- Aislar un problema probando cada tramo de la cadena por separado
  (host local → contenedor→contenedor → externo) es mucho más rápido
  que adivinar la causa a partir del síntoma final (502 genérico).

## Comandos utilizados

Ver [comandos.md](./comandos.md)

## Estado
✅ Completado — 2026-07-25