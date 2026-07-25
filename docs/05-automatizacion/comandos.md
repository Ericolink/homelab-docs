# Comandos — Nivel 5

## Self-hosted runner — despliegue vía Docker

    mkdir -p ~/docker/github-runner-paselink
    cd ~/docker/github-runner-paselink
    nano docker-compose.yml

Contenido de `docker-compose.yml` (token gestionado fuera del repo, vía
variable de entorno o `.env` local):

    services:
      github-runner-paselink:
        image: myoung34/github-runner:latest
        container_name: github-runner-paselink
        restart: unless-stopped
        environment:
          - REPO_URL=https://github.com/Ericolink/PaseLink
          - RUNNER_NAME=optiplex-homelab
          - RUNNER_TOKEN=${GITHUB_RUNNER_TOKEN}
          - RUNNER_WORKDIR=/tmp/runner/work
          - LABELS=self-hosted,homelab,optiplex
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - runner_work:/tmp/runner/work

    volumes:
      runner_work:

Levantar y verificar:

    docker compose up -d
    docker logs github-runner-paselink -f

## Workflow de tests (`.github/workflows/tests.yml` en el repo PaseLink)

    name: Tests

    on:
      push:
        branches: [main]
      pull_request:
        branches: [main]

    jobs:
      test:
        runs-on: [self-hosted, homelab]
        if: github.event.pull_request.head.repo.full_name == github.repository || github.event_name == 'push'

        steps:
          - name: Checkout código
            uses: actions/checkout@v4

          - name: Instalar Node.js
            uses: actions/setup-node@v4
            with:
              node-version: '22'

          - name: Instalar dependencias
            run: npm ci

          - name: Correr tests
            run: npm test
            env:
              VITE_FIREBASE_API_KEY: ${{ secrets.VITE_FIREBASE_API_KEY }}
              VITE_FIREBASE_AUTH_DOMAIN: ${{ secrets.VITE_FIREBASE_AUTH_DOMAIN }}
              VITE_FIREBASE_PROJECT_ID: ${{ secrets.VITE_FIREBASE_PROJECT_ID }}
              VITE_FIREBASE_STORAGE_BUCKET: ${{ secrets.VITE_FIREBASE_STORAGE_BUCKET }}
              VITE_FIREBASE_MESSAGING_SENDER_ID: ${{ secrets.VITE_FIREBASE_MESSAGING_SENDER_ID }}
              VITE_FIREBASE_APP_ID: ${{ secrets.VITE_FIREBASE_APP_ID }}

## Diagnóstico de workflows existentes

    ls -la .github/workflows/
    grep -h "runs-on" .github/workflows/*.yml

## Configuración de seguridad del repositorio (vía interfaz web de GitHub)

Settings → Actions → General:
- Approval for running fork pull request workflows from contributors:
  "Require approval for all external contributors"
- Workflow permissions: "Read repository contents and packages permissions"