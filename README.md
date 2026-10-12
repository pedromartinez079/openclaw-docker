# OpenClaw con Docker: Linux y Windows

Guía para instalar y ejecutar [OpenClaw](https://openclaw.ai/) directamente con Docker Compose. **Portainer no es necesario** para el procedimiento principal; al final se incluye una sección opcional para administrarlo con Portainer.

Compatible con:

- Linux con Docker Engine y Docker Compose v2.
- Windows 10/11 con Docker Desktop y contenedores Linux.
- Servidores o equipos de escritorio donde Docker sea el entorno de ejecución.

> Consulta siempre la documentación oficial para comprobar cambios de la versión instalada: [OpenClaw Docker](https://docs.openclaw.ai/install/docker).

## Índice

- [Arquitectura](#arquitectura)
- [Requisitos](#requisitos)
- [Instalar Docker](#instalar-docker)
- [Crear el proyecto](#crear-el-proyecto)
- [Configurar las variables](#configurar-las-variables)
- [Inicializar OpenClaw](#inicializar-openclaw)
- [Iniciar OpenClaw con Docker Compose](#iniciar-openclaw-con-docker-compose)
- [Acceder al Control UI](#acceder-al-control-ui)
- [Configurar canales](#configurar-canales)
- [Actualizar OpenClaw](#actualizar-openclaw)
- [Copias de seguridad](#copias-de-seguridad)
- [Solución de problemas](#solución-de-problemas)
- [Usar Portainer opcionalmente](#usar-portainer-opcionalmente)
- [Seguridad](#seguridad)

## Arquitectura

OpenClaw ejecuta un Gateway que proporciona el Control UI, las sesiones, las conexiones con canales de mensajería y la integración con proveedores de modelos de IA.

Este procedimiento utiliza:

- Imagen oficial: `ghcr.io/openclaw/openclaw:latest`.
- Docker Compose v2.
- Un contenedor: `openclaw-gateway`.
- Tres volúmenes Docker persistentes:
  - `openclaw_state`: configuración y sesiones.
  - `openclaw_workspace`: espacio de trabajo.
  - `openclaw_auth`: secretos de perfiles de autenticación.

Los volúmenes nombrados funcionan igual en Linux y Windows y evitan problemas con rutas del anfitrión como `/srv/...` o `C:\...`.

## Requisitos

### Hardware recomendado

- 2 GB de RAM disponibles para Docker durante la descarga y ejecución.
- Espacio libre para la imagen, los logs y los datos persistentes.
- Una API key de un proveedor compatible, salvo que vayas a usar un proveedor local.

### Linux

Comprueba que Docker y Compose están instalados:

```bash
sudo docker --version
sudo docker compose version
sudo docker run --rm hello-world
```

### Windows

Instala [Docker Desktop](https://www.docker.com/products/docker-desktop/) con el backend WSL 2 recomendado y verifica desde PowerShell:

```powershell
docker --version
docker compose version
docker run --rm hello-world
```

Asegúrate de que Docker Desktop está configurado para **Linux containers** y muestra **Engine running**.

No es necesario instalar Node.js en el sistema anfitrión: la imagen de OpenClaw ya incluye Node.js.

## Instalar Docker

### Linux basado en Ubuntu

Para una instalación básica:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Cierra la sesión y vuelve a entrar para aplicar el grupo `docker`. Comprueba después:

```bash
sudo docker run --rm hello-world
```

Para entornos de producción, consulta la [instalación oficial de Docker Engine](https://docs.docker.com/engine/install/).

### Windows

1. Descarga e instala Docker Desktop.
2. Activa WSL 2 si el instalador lo solicita.
3. Reinicia Windows si es necesario.
4. Abre Docker Desktop y espera a que el motor esté iniciado.
5. Usa PowerShell para ejecutar los comandos de esta guía.

## Crear el proyecto

Crea una carpeta para OpenClaw y entra en ella.

### Linux

```bash
mkdir -p ~/openclaw-docker
cd ~/openclaw-docker
```

### Windows PowerShell

```powershell
New-Item -ItemType Directory -Force "$HOME\openclaw-docker"
Set-Location "$HOME\openclaw-docker"
```

## Crear el archivo Compose

Crea un archivo llamado `compose.yaml` con este contenido:

```yaml
services:
  openclaw-gateway:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway

    env_file:
      - .env

    environment:
      HOME: /home/node
      OPENCLAW_HOME: /home/node
      TERM: xterm-256color
      TZ: UTC

      # Las rutas siguientes son internas al contenedor.
      OPENCLAW_STATE_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_PATH: /home/node/.openclaw/openclaw.json
      OPENCLAW_WORKSPACE_DIR: /home/node/.openclaw/workspace

    volumes:
      - openclaw_state:/home/node/.openclaw
      - openclaw_workspace:/home/node/.openclaw/workspace
      - openclaw_auth:/home/node/.config/openclaw

    ports:
      - "18789:18789"
      - "18790:18790"
      - "3978:3978"

    command:
      - node
      - dist/index.js
      - gateway
      - --bind
      - lan
      - --port
      - "18789"

    restart: unless-stopped
    init: true

    cap_drop:
      - NET_RAW
      - NET_ADMIN

    security_opt:
      - no-new-privileges:true

    extra_hosts:
      - "host.docker.internal:host-gateway"

    healthcheck:
      test:
        - CMD
        - node
        - dist/docker-healthcheck.js
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 20s

volumes:
  openclaw_state:
    name: openclaw_state
  openclaw_workspace:
    name: openclaw_workspace
  openclaw_auth:
    name: openclaw_auth
```

## Configurar las variables

Crea un archivo `.env` en la misma carpeta que `compose.yaml`.

### Linux

```bash
cat > .env <<'EOF'
OPENCLAW_GATEWAY_TOKEN=CAMBIA_ESTE_VALOR
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GEMINI_API_KEY=
OPENROUTER_API_KEY=
TELEGRAM_BOT_TOKEN=
EOF
```

Genera un token seguro para el Gateway:

```bash
openssl rand -hex 32
```

Copia el resultado en `OPENCLAW_GATEWAY_TOKEN`.

Si no tienes OpenSSL:

```bash
python3 -c 'import secrets; print(secrets.token_hex(32))'
```

### Windows PowerShell

Crea el archivo con el Bloc de notas:

```powershell
notepad .env
```

Pega este contenido:

```dotenv
OPENCLAW_GATEWAY_TOKEN=CAMBIA_ESTE_VALOR
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GEMINI_API_KEY=
OPENROUTER_API_KEY=
TELEGRAM_BOT_TOKEN=
```

Genera un token seguro en PowerShell:

```powershell
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$token = ($bytes | ForEach-Object { $_.ToString('x2') }) -join ''
$token
```

Copia el resultado en `OPENCLAW_GATEWAY_TOKEN`.

### Añadir la API key

Rellena al menos una variable, según el proveedor que vayas a utilizar:

```dotenv
OPENAI_API_KEY=tu-clave-real
```

O, por ejemplo:

```dotenv
ANTHROPIC_API_KEY=tu-clave-real
```

No incluyas `.env` en GitHub. Crea un `.gitignore`:

```gitignore
.env
openclaw-backup/
```

Puedes publicar un `.env.example` sin secretos:

```dotenv
OPENCLAW_GATEWAY_TOKEN=replace-with-a-random-token
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GEMINI_API_KEY=
OPENROUTER_API_KEY=
TELEGRAM_BOT_TOKEN=
```

## Inicializar OpenClaw

Descarga la imagen:

```bash
sudo docker compose pull
```

El siguiente comando es equivalente en Linux y PowerShell, pero la continuación de línea cambia.

### Linux

```bash
sudo docker run --rm -it \
  --env-file .env \
  -v openclaw_state:/home/node/.openclaw \
  -v openclaw_workspace:/home/node/.openclaw/workspace \
  -v openclaw_auth:/home/node/.config/openclaw \
  -e HOME=/home/node \
  -e OPENCLAW_HOME=/home/node \
  -e OPENCLAW_STATE_DIR=/home/node/.openclaw \
  -e OPENCLAW_CONFIG_DIR=/home/node/.openclaw \
  -e OPENCLAW_CONFIG_PATH=/home/node/.openclaw/openclaw.json \
  -e OPENCLAW_WORKSPACE_DIR=/home/node/.openclaw/workspace \
  --entrypoint node \
  ghcr.io/openclaw/openclaw:latest \
  dist/index.js onboard \
  --mode local \
  --no-install-daemon
```

### Windows PowerShell

```powershell
docker run --rm -it `
  --env-file .env `
  -v openclaw_state:/home/node/.openclaw `
  -v openclaw_workspace:/home/node/.openclaw/workspace `
  -v openclaw_auth:/home/node/.config/openclaw `
  -e HOME=/home/node `
  -e OPENCLAW_HOME=/home/node `
  -e OPENCLAW_STATE_DIR=/home/node/.openclaw `
  -e OPENCLAW_CONFIG_DIR=/home/node/.openclaw `
  -e OPENCLAW_CONFIG_PATH=/home/node/.openclaw/openclaw.json `
  -e OPENCLAW_WORKSPACE_DIR=/home/node/.openclaw/workspace `
  --entrypoint node `
  ghcr.io/openclaw/openclaw:latest `
  dist/index.js onboard `
  --mode local `
  --no-install-daemon
```

El parámetro `--no-install-daemon` evita instalar `systemd` o un servicio del sistema dentro del contenedor.

Para una configuración no interactiva con OpenAI, la documentación oficial también contempla `--non-interactive`, `--auth-choice openai-api-key` y `--gateway-auth token`. El asistente interactivo es preferible para la primera instalación porque permite comprobar cada decisión.

## Iniciar OpenClaw con Docker Compose

Desde la carpeta del proyecto:

```bash
sudo docker compose up -d
```

En PowerShell se usa exactamente el mismo comando:

```powershell
docker compose up -d
```

Comprueba el estado:

```bash
sudo docker compose ps
sudo docker compose logs -f openclaw-gateway
```

Para detenerlo sin borrar datos:

```bash
sudo docker compose down
```

No uses `sudo docker compose down -v` salvo que quieras borrar también los volúmenes y perder la configuración persistente.

## Acceder al Control UI

Desde el mismo equipo:

```text
http://127.0.0.1:18789/
```

Desde otro equipo de la red local:

```text
http://IP_DEL_SERVIDOR:18789/
```

Introduce el valor de `OPENCLAW_GATEWAY_TOKEN` cuando la interfaz lo solicite.

No expongas el puerto directamente a Internet. Para acceso remoto utiliza una VPN, Tailscale o un proxy inverso HTTPS correctamente protegido.

## Configurar canales

### Telegram

Añade el token del bot al `.env`:

```dotenv
TELEGRAM_BOT_TOKEN=123456789:token-del-bot
```

Recrea el contenedor para que reciba la variable:

```bash
sudo docker compose up -d --force-recreate
```

Después, ejecuta el comando de configuración dentro del contenedor:

```bash
sudo docker compose exec openclaw-gateway \
  node dist/index.js channels add --channel telegram --use-env
```

Consulta la guía oficial: [OpenClaw Telegram](https://docs.openclaw.ai/channels/telegram).

### Otros canales

- [Channels](https://docs.openclaw.ai/channels)
- [Discord](https://docs.openclaw.ai/channels/discord)
- [WhatsApp](https://docs.openclaw.ai/channels/whatsapp)

## Actualizar OpenClaw

Primero realiza una copia de seguridad. Después:

```bash
sudo docker compose pull
sudo docker compose up -d
```

Para producción es preferible fijar una versión concreta en `compose.yaml` en vez de utilizar siempre `latest`:

```yaml
image: ghcr.io/openclaw/openclaw:VERSION
```

Sustituye `VERSION` por una etiqueta real publicada por OpenClaw.

## Copias de seguridad

Lista los volúmenes:

```bash
sudo docker volume ls --filter name=openclaw
```

Crea una carpeta de backup:

```bash
mkdir -p openclaw-backup
```

Haz copia de los datos en Linux o en una terminal compatible con Docker:

```bash
sudo docker run --rm \
  -v openclaw_state:/data:ro \
  -v "$PWD/openclaw-backup:/backup" \
  alpine:latest \
  tar czf /backup/openclaw_state.tar.gz -C /data .

sudo docker run --rm \
  -v openclaw_workspace:/data:ro \
  -v "$PWD/openclaw-backup:/backup" \
  alpine:latest \
  tar czf /backup/openclaw_workspace.tar.gz -C /data .

sudo docker run --rm \
  -v openclaw_auth:/data:ro \
  -v "$PWD/openclaw-backup:/backup" \
  alpine:latest \
  tar czf /backup/openclaw_auth.tar.gz -C /data .
```

En PowerShell, crea la carpeta con:

```powershell
New-Item -ItemType Directory -Force .\openclaw-backup
```

Guarda los backups fuera del repositorio si contienen datos privados o credenciales.

## Solución de problemas

### Ver logs

```bash
sudo docker compose logs --tail 200 openclaw-gateway
```

### El contenedor se reinicia

Comprueba:

- Que `OPENCLAW_GATEWAY_TOKEN` no esté vacío.
- Que la API key sea válida.
- Que Docker Desktop esté iniciado en Windows.
- Que la imagen se haya descargado correctamente.
- Que los volúmenes persistentes no se hayan eliminado.

### El panel no abre

```bash
sudo docker compose ps
sudo docker port openclaw-gateway
```

Comprueba también el firewall del sistema y que el puerto `18789` no esté ocupado.

### Comprobar variables en Linux

```bash
sudo docker inspect openclaw-gateway \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E 'OPENAI|ANTHROPIC|GEMINI|OPENROUTER|TELEGRAM'
```

En PowerShell:

```powershell
docker inspect openclaw-gateway --format '{{range .Config.Env}}{{println .}}{{end}}'
```

No publiques estas salidas: pueden contener secretos.

## Usar Portainer opcionalmente

Portainer no cambia la arquitectura de OpenClaw: únicamente proporciona una interfaz web para gestionar el mismo Compose, los contenedores y los volúmenes.

### Instalar Portainer en Linux

```bash
sudo docker volume create portainer_data

sudo docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

Abre:

```text
https://localhost:9443
```

Para Docker Desktop en Windows, consulta la [guía oficial de Portainer para Docker Desktop](https://docs.portainer.io/start/install-ce/server/docker/wsl), ya que el montaje del socket y el endpoint dependen de la configuración WSL 2.

### Crear el Stack en Portainer

1. Abre Portainer y selecciona el entorno Docker local.
2. Ve a **Stacks** y pulsa **Add stack**.
3. Usa el nombre `openclaw`.
4. Selecciona **Web editor**.
5. Pega el contenido de tu `compose.yaml`.
6. En **Environment variables**, añade las variables que normalmente están en `.env`:

| Variable | Uso |
|---|---|
| `OPENCLAW_GATEWAY_TOKEN` | Token del Gateway |
| `OPENAI_API_KEY` | Proveedor OpenAI |
| `ANTHROPIC_API_KEY` | Proveedor Anthropic |
| `GEMINI_API_KEY` | Proveedor Gemini |
| `OPENROUTER_API_KEY` | Proveedor OpenRouter |
| `TELEGRAM_BOT_TOKEN` | Bot de Telegram |

7. Pulsa **Deploy the stack**.

Para usar el Stack desde Portainer, elimina o comenta `env_file: - .env` del Compose pegado en el editor, porque Portainer proporciona las variables desde su propia sección de entorno. El resto del Compose permanece igual.

### Gestionar el Stack

Desde Portainer podrás:

- Ver logs.
- Reiniciar el contenedor.
- Detener o iniciar el Stack.
- Actualizar la imagen con **Re-pull image and redeploy**.
- Administrar los volúmenes.
- Abrir una consola dentro del contenedor.

No borres los volúmenes `openclaw_state`, `openclaw_workspace` y `openclaw_auth` durante una actualización.

## Seguridad

OpenClaw puede ejecutar herramientas y procesar mensajes externos. Antes de exponerlo a otros usuarios o a Internet:

- Mantén configurado el token del Gateway.
- No publiques `18789` directamente en Internet.
- Usa VPN, Tailscale o HTTPS con autenticación.
- No uses `--privileged`.
- No montes `/var/run/docker.sock` salvo que necesites explícitamente sandboxing basado en Docker y entiendas sus implicaciones.
- No publiques `.env`, API keys, tokens ni backups privados en GitHub.
- Configura listas de emparejamiento y permisos de los canales.
- Haz copias de seguridad de los tres volúmenes.

Documentación oficial:

- [Docker](https://docs.openclaw.ai/install/docker)
- [Security](https://docs.openclaw.ai/gateway/security)
- [Sandboxing](https://docs.openclaw.ai/gateway/sandboxing)
- [Channels](https://docs.openclaw.ai/channels)

## Licencia y atribución

Este README es una guía comunitaria de despliegue. El proyecto OpenClaw mantiene su propio código, licencia y documentación en:

- [OpenClaw en GitHub](https://github.com/openclaw/openclaw)
- [Documentación de OpenClaw](https://docs.openclaw.ai)
