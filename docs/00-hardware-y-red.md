# Hardware y Red

Este documento describe el estado actual del hardware y la topología de red
del homelab. Se actualiza cada vez que cambia algo físico o de conectividad
(nuevo disco, cambio de ISP, etc.) — no está atado a un nivel específico del
roadmap.

## Hardware

| Componente | Detalle |
|---|---|
| Modelo | Dell OptiPlex 3010 (SFF) |
| CPU | Intel Core i3-3220 @ 3.30GHz — 2 núcleos / 4 hilos (Ivy Bridge) |
| RAM | 7.7 GB |
| Disco principal | 232.9 GB, particionado con LVM |
| SO | Ubuntu Server 24.04.4 LTS (kernel 6.8.0-107-generic) |
| Conexión de red | Ethernet cableada (`enp2s0`) |

### Almacenamiento — detalle

| Partición/Volumen | Tamaño | Punto de montaje |
|---|---|---|
| `sda1` (EFI) | 1 GB | `/boot/efi` |
| `sda2` (boot) | 2 GB | `/boot` |
| `sda3` → LVM | 229.8 GB | — |
| `ubuntu--vg-ubuntu--lv` | 100 GB (de 229.8 GB disponibles) | `/` |

**Estado:** ~130 GB del disco principal sin asignar al volumen lógico.
Pendiente de extender vía `lvextend` + `resize2fs` (Nivel 1, tarea abierta).

**Almacenamiento secundario (backups):**
- Estado: 🟡 Planeado, no instalado aún.
- Plan: liberar el puerto SATA usado actualmente por el lector de DVD
  (desconectarlo) e instalar un disco dedicado exclusivamente a backups.
- Ver decisión completa en [Nivel 1 — Fundamentos](./01-fundamentos/README.md#backups-plan).

## Red

| | Detalle |
|---|---|
| IP local (LAN) | `192.168.100.150/24` (DHCP) |
| IPv4 pública | No disponible — el ISP asigna IP privada al router (`10.49.22.143` en WAN), detrás de **CGNAT** |
| IPv6 pública | Sí, real y sin NAT (`2806:2f0:3301:f40e::/64`, asignada directo a la interfaz del servidor) |
| Router | Router/ONT del ISP, con acceso parcial confirmado (panel de estado WAN visible) |
| Port forwarding (IPv4) | No viable por CGNAT |

### Implicaciones de diseño

- **No se puede exponer servicios a internet vía port forwarding IPv4 tradicional** — el ISP no asigna IP pública real por ese protocolo.
- **IPv6 pública real disponible** — abre la posibilidad de exponer servicios directo por IPv6, sujeto a evaluación de seguridad (firewall estricto obligatorio si se usa esta vía).
- **Estrategia de acceso remoto elegida:** VPN mesh (Tailscale/WireGuard) para acceso personal, y Cloudflare Tunnel para lo que eventualmente se quiera hacer público — ambas evitan depender de IP pública IPv4 o de tocar configuración NAT del router.

## Consumo eléctrico (estimado)

- TDP de CPU: ~55W
- Consumo estimado del equipo completo en reposo/carga ligera: 25-45W
- Decisión de disponibilidad: pendiente de confirmar, pero viable dejar el equipo **24/7** dado el bajo consumo (~$50-100 MXN/mes estimado en México).

## Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-24 | Documento inicial: specs de hardware, diagnóstico de red (CGNAT en IPv4, IPv6 pública confirmada) |