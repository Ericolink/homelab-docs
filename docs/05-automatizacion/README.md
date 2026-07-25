# Nivel 5 — Automatización

## Objetivo

Incorporar infraestructura propia (el homelab) al flujo de trabajo real de
desarrollo de software, mediante un self-hosted runner de GitHub Actions
que ejecuta CI para un proyecto en producción (PaseLink), sin depender
exclusivamente de runners hosteados por GitHub.

## Contexto

- Requisito previo: Niveles 1-4 completos.
- Proyecto elegido: PaseLink (React 19 + TypeScript + Firebase), con
  suite de tests existente en Vitest y pipelines de CI/CD previos
  (`firebase-hosting-merge.yml`, `firebase-hosting-pull-request.yml`,
  `firestore-backup.yml`, `gitleaks.yml`, `uptime-check.yml`), todos
  corriendo en `ubuntu-latest` (infraestructura de GitHub).
- Repositorio de PaseLink es **público**, lo cual implica riesgos de
  seguridad específicos para runners self-hosted (ver sección de
  decisiones).

## Decisiones tomadas

- **Gitea Actions descartado a favor de GitHub Actions self-hosted**
  Consecuencia directa de la decisión de Nivel 4 de no instalar Gitea.
  GitHub Actions es, además, más representativo de lo que se usa en la
  industria, haciendo la habilidad más transferible.

- **Runner desplegado en Docker (imagen `myoung34/github-runner`), no
  instalado nativo en el sistema**
  Consistente con el patrón de todo el homelab: aislado, reproducible,
  documentado como código.

- **Aprobación obligatoria para ejecución de workflows de contribuidores
  externos ("Require approval for all external contributors")**
  Motivo: el repositorio de PaseLink es público. Sin esta protección,
  cualquier persona podría abrir un Pull Request diseñado para ejecutar
  código arbitrario en el runner self-hosted — el cual tiene acceso al
  socket de Docker del servidor, equivalente a compromiso total del host
  si se explotara. Se optó por la opción más estricta disponible (aplica
  a todo colaborador externo, no solo a cuentas nuevas o de primera
  contribución).

- **Workflow de tests no reemplaza los pipelines existentes en
  `ubuntu-latest`**
  Se evaluó migrar `firestore-backup.yml` y `uptime-check.yml` al runner
  propio, pero se decidió dejarlos en la infraestructura de GitHub por
  ahora: dependen de que corran de forma confiable incluso si el
  servidor homelab está apagado, y el servidor todavía no está
  configurado para operar 24/7 de forma definitiva.

- **`npm ci` en vez de `npm install` en el workflow**
  Garantiza instalación exacta según `package-lock.json`, apropiado para
  entornos de integración continua donde la reproducibilidad importa más
  que la conveniencia.

## Qué se implementó

1. Self-hosted runner registrado para el repositorio PaseLink, corriendo
   en contenedor Docker en el servidor homelab.
2. Workflow `tests.yml`: ejecuta la suite de Vitest (`npm test`) en cada
   push y pull request dirigidos a `main`, usando el runner propio
   (`runs-on: [self-hosted, homelab]`).
3. Variables de entorno de Firebase inyectadas vía GitHub Secrets
   (previamente ya configurados en el repo para otros workflows),
   resolviendo el fallo de inicialización de Firebase en tiempo de test.
4. Actualizada la versión de Node.js del workflow de 20 (deprecado) a 22.
5. Configuración de seguridad a nivel de repositorio: aprobación
   obligatoria para ejecutar workflows originados en Pull Requests de
   colaboradores externos.

## Problemas encontrados

- **4 archivos de test fallando con `auth/invalid-api-key` al ejecutar
  en el runner, mientras 43 tests individuales sí pasaban.**
  Causa: la inicialización de Firebase en `src/firebase/config.ts` corre
  al importar el módulo, antes de que el test mismo se ejecute; el
  runner (a diferencia del entorno local) no tenía acceso a las
  variables `VITE_FIREBASE_*` del `.env`, inexistente en una máquina
  limpia de CI.
  Solución: variables inyectadas mediante GitHub Secrets (ya existentes
  en el repo desde configuraciones previas) en el bloque `env:` del paso
  de test.

- **Advertencia de deprecación de Node.js 20 en runners de Actions.**
  Solución: actualizado `actions/setup-node@v4` a `node-version: '22'`.

## Aprendizajes clave

- Un self-hosted runner en un repositorio público es una superficie de
  ataque real, no una preocupación teórica — GitHub documenta esto
  explícitamente. La configuración de aprobación para colaboradores
  externos es una capa de defensa obligatoria, no opcional, en este
  escenario.
- Los tests que dependen de servicios externos inicializados a nivel de
  módulo (como Firebase) requieren que el entorno de CI reproduzca las
  variables de entorno necesarias — un test "puro" en apariencia puede
  fallar por completo si el entorno donde corre no tiene el contexto que
  el entorno de desarrollo da por sentado.
- No todo pipeline necesita migrarse al runner propio solo porque existe
  la opción — la disponibilidad del servidor (aún no 24/7 definitivo) es
  un factor real al decidir qué debe seguir corriendo en infraestructura
  de terceros.

## Comandos utilizados

Ver [comandos.md](./comandos.md)

## Estado

✅ Completado — 2026-07-25