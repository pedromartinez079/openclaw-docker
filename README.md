# OpenClaw con Docker y Portainer

Guía multiplataforma para desplegar [OpenClaw](https://openclaw.ai/) con Docker y administrar el contenedor desde [Portainer](https://www.portainer.io/).

Compatible con:

- Linux con Docker Engine y Docker Compose v2.
- Windows 10/11 con Docker Desktop usando contenedores Linux.
- Instalaciones administradas mediante Portainer.

> **Estado de la documentación:** basada en la documentación oficial de OpenClaw. Comprueba siempre las instrucciones de la versión que vayas a desplegar en [docs.openclaw.ai/install/docker](https://docs.openclaw.ai/install/docker).

## Índice

- [Arquitectura](#arquitectura)
- [Requisitos](#requisitos)
- [Instalar Docker y Portainer](#instalar-docker-y-portainer)
- [Crear los volúmenes persistentes](#crear-los-volúmenes-persistentes)
- [Inicializar OpenClaw](#inicializar-openclaw)
- [Desplegar el Stack en Portainer](#desplegar-el-stack-en-portainer)
- [Acceder al Control UI](#acceder-al-control-ui)
- [Configurar canales](#configurar-canales)
- [Actualizar OpenClaw](#actualizar-openclaw)
- [Copias de seguridad](#copias-de-seguridad)
- [Solución de problemas](#solución-de-problemas)
- [Seguridad](#seguridad)

## Arquitectura

OpenClaw ejecuta un Gateway que proporciona:

- Control UI web.
- Sesiones y configuración.
- Conexiones con Telegram, Discord, WhatsApp y otros canales.
- Integración con proveedores de modelos de IA.

Este procedimiento utiliza:

- Imagen oficial: `ghcr.io/openclaw/openclaw:latest`.
- Un contenedor principal: `openclaw-gateway`.
- Tres volúmenes Docker persistentes:
  - `openclaw_state`: configuración y sesiones.
  - `openclaw_workspace`: espacio de trabajo del agente.
  - `openclaw_auth`: secretos de perfiles de autenticación.

Los volúmenes Docker son preferibles a rutas específicas del sistema porque funcionan igual en Windows y Linux y evitan problemas con rutas como `C:\...` o `/srv/...`.

## Requisitos

### Hardware recomendado

- 2 GB de RAM disponibles para el contenedor y la imagen.
- Espacio libre para la imagen Docker, logs y datos persistentes.
- Conexión a Internet para descargar la imagen y acceder al proveedor de IA.

### Linux

- Docker Engine.
- Docker Compose v2, normalmente disponible como `docker compose`.
- Portainer CE opcional, pero recomendado para administrar el Stack.

Comprobar la instalación:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

### Windows

- Windows 10 o Windows 11.
- Docker Desktop actualizado.
- Docker Desktop configurado para **Linux containers**.
- Portainer accesible desde el navegador.

Comprobar la instalación en PowerShell:

```powershell
docker --version
docker compose version
docker run --rm hello-world
```

No es necesario instalar Node.js en el sistema anfitrión. Node.js se incluye en la imagen de OpenClaw.

## Instalar Docker y Portainer

### Linux: Docker Engine

En distribuciones basadas en Ubuntu, una instalación sencilla para comenzar es:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Cierra la sesión y vuelve a entrar para que el grupo `docker` se aplique. Después comprueba:

```bash
docker run --rm hello-world
```

Para entornos de producción, consulta también la [documentación oficial de Docker Engine](https://docs.docker.com/engine/install/).

### Windows: Docker Desktop

1. Descarga Docker Desktop desde [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/).
2. Instálalo con el backend WSL 2 recomendado.
3. Reinicia Windows si el instalador lo solicita.
4. Comprueba que Docker Desktop muestra **Engine running**.
5. Usa PowerShell para verificar Docker.

### Portainer CE

Si todavía no tienes Portainer, puedes instalarlo en Linux con:

```bash
docker volume create portainer_data

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

En Windows con Docker Desktop, ejecuta el equivalente desde PowerShell:

```powershell
docker volume create portainer_data

docker run -d `
  -p 8000:8000 `
  -p 9443:9443 `
  --name portainer `
  --restart=always `
  -v //var/run/docker.sock:/var/run/docker.sock `
  -v portainer_data:/data `
  portainer/portainer-ce:lts
```

Abre Portainer en:

```text
https://localhost:9443
```

> En Docker Desktop, si el montaje del socket no funciona en una instalación concreta, utiliza el método de instalación recomendado por la documentación de Portainer para Docker Desktop/WSL 2.

## Crear los volúmenes persistentes

Ejecuta una vez en Linux, PowerShell o CMD:

```text
docker volume create openclaw_state
docker volume create openclaw_workspace
docker volume create openclaw_auth
```

También se pueden crear automáticamente al desplegar el Stack desde Portainer.

## Inicializar OpenClaw

Antes del primer arranque, genera un token seguro para proteger el Gateway.

### Linux

```bash
export OPENCLAW_GATEWAY_TOKEN="$(openssl rand -hex 32)"
```

Si `openssl` no está instalado:

```bash
python3 -c 'import secrets; print(secrets.token_hex(32))'
```

### Windows PowerShell

```powershell
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Fill($bytes)
$OPENCLAW_GATEWAY_TOKEN = ($bytes | ForEach-Object { $_.ToString('x2') }) -join ''
$OPENCLAW_GATEWAY_TOKEN
```

Guarda el token en un gestor de contraseñas. No lo publiques en GitHub.

### Inicialización con proveedor OpenAI

Sustituye `TU_TOKEN_DEL_GATEWAY` y `TU_API_KEY_DE_OPENAI` por tus valores. No guardes este comando en un repositorio.

#### Linux

```bash
docker run --rm -it \
  -v openclaw_state:/home/node/.openclaw \
  -v openclaw_workspace:/home/node/.openclaw/workspace \
  -v openclaw_auth:/home/node/.config/openclaw \
  -e HOME=/home/node \
  -e OPENCLAW_HOME=/home/node \
  -e OPENCLAW_STATE_DIR=/home/node/.openclaw \
  -e OPENCLAW_CONFIG_DIR=/home/node/.openclaw \
  -e OPENCLAW_CONFIG_PATH=/home/node/.openclaw/openclaw.json \
  -e OPENCLAW_WORKSPACE_DIR=/home/node/.openclaw/workspace \
  -e OPENCLAW_GATEWAY_TOKEN="TU_TOKEN_DEL_GATEWAY" \
  -e OPENAI_API_KEY="TU_API_KEY_DE_OPENAI" \
  --entrypoint node \
  ghcr.io/openclaw/openclaw:latest \
  dist/index.js onboard \
  --mode local \
  --no-install-daemon
```

#### Windows PowerShell

```powershell
docker run --rm -it `
  -v openclaw_state:/home/node/.openclaw `
  -v openclaw_workspace:/home/node/.openclaw/workspace `
  -v openclaw_auth:/home/node/.config/openclaw `
  -e HOME=/home/node `
  -e OPENCLAW_HOME=/home/node `
  -e OPENCLAW_STATE_DIR=/home/node/.openclaw `
  -e OPENCLAW_CONFIG_DIR=/home/node/.openclaw `
  -e OPENCLAW_CONFIG_PATH=/home/node/.openclaw/openclaw.json `
  -e OPENCLAW_WORKSPACE_DIR=/home/node/.openclaw/workspace `
  -e OPENCLAW_GATEWAY_TOKEN="TU_TOKEN_DEL_GATEWAY" `
  -e OPENAI_API_KEY="TU_API_KEY_DE_OPENAI" `
  --entrypoint node `
  ghcr.io/openclaw/openclaw:latest `
  dist/index.js onboard `
  --mode local `
  --no-install-daemon
```

El parámetro `--no-install-daemon` evita intentar instalar `systemd` o un servicio del sistema dentro del contenedor.

Para otros proveedores, cambia `OPENAI_API_KEY` por la variable correspondiente, por ejemplo `ANTHROPIC_API_KEY` o `GEMINI_API_KEY`, y selecciona el proveedor durante el asistente.

## Desplegar el Stack en Portainer

1. Abre Portainer.
2. Ve a **Stacks**.
3. Pulsa **Add stack**.
4. Asigna el nombre `openclaw`.
5. Selecciona **Web editor**.
6. Pega este `compose.yaml`:

```yaml
services:
  openclaw-gateway:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw-gateway

    environment:
      HOME: /home/node
      OPENCLAW_HOME: /home/node
      TERM: xterm-256color
      TZ: UTC

      OPENCLAW_STATE_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_PATH: /home/node/.openclaw/openclaw.json
      OPENCLAW_WORKSPACE_DIR: /home/node/.openclaw/workspace

      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      GEMINI_API_KEY: ${GEMINI_API_KEY:-}
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY:-}
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN:-}

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

7. En **Environment variables**, añade al menos:

| Variable | Valor |
|---|---|
| `OPENCLAW_GATEWAY_TOKEN` | Token generado anteriormente |
| `OPENAI_API_KEY` | Clave del proveedor elegido |

Variables opcionales:

| Variable | Uso |
|---|---|
| `ANTHROPIC_API_KEY` | Proveedor Anthropic |
| `GEMINI_API_KEY` | Proveedor Gemini |
| `OPENROUTER_API_KEY` | Proveedor OpenRouter |
| `TELEGRAM_BOT_TOKEN` | Bot de Telegram |

8. Pulsa **Deploy the stack**.

> Portainer utiliza estas variables para interpolar el Compose. Evita escribir claves directamente en el YAML.

## Acceder al Control UI

En el navegador abre:

```text
http://IP_DEL_SERVIDOR:18789/
```

En el mismo equipo puede ser:

```text
http://127.0.0.1:18789/
```

Cuando lo solicite, introduce el valor de `OPENCLAW_GATEWAY_TOKEN`.

Para ver los logs desde el terminal:

```bash
docker logs -f openclaw-gateway
```

O desde Portainer:

1. **Containers**.
2. `openclaw-gateway`.
3. **Logs**.

## Configurar canales

### Telegram

Añade `TELEGRAM_BOT_TOKEN` en las variables del Stack y vuelve a desplegarlo. También puedes ejecutar el comando desde la consola del contenedor:

```bash
node dist/index.js channels add --channel telegram --use-env
```

El token debe permanecer disponible como variable de entorno para que el Gateway pueda utilizarlo después de reiniciar.

Consulta la guía oficial: [OpenClaw Telegram](https://docs.openclaw.ai/channels/telegram).

### Otros canales

Consulta la documentación oficial de canales:

- [Channels](https://docs.openclaw.ai/channels)
- [Discord](https://docs.openclaw.ai/channels/discord)
- [WhatsApp](https://docs.openclaw.ai/channels/whatsapp)

## Actualizar OpenClaw

Antes de actualizar, realiza una copia de seguridad de los volúmenes.

En Portainer:

1. Abre el Stack `openclaw`.
2. Pulsa **Edit**.
3. Activa **Re-pull image and redeploy**.
4. Pulsa **Update the stack**.

> Para producción es preferible fijar una etiqueta de versión en lugar de `latest`, por ejemplo:
>
> ```yaml
> image: ghcr.io/openclaw/openclaw:VERSION
> ```
>
> Sustituye `VERSION` por una versión real publicada por OpenClaw. No uses etiquetas inventadas.

## Copias de seguridad

Lista los volúmenes:

```bash
docker volume ls --filter name=openclaw
```

Ejemplo de copia de seguridad en Linux:

```bash
mkdir -p ./openclaw-backup

docker run --rm \
  -v openclaw_state:/data:ro \
  -v "$PWD/openclaw-backup:/backup" \
  alpine:latest \
  tar czf /backup/openclaw_state.tar.gz -C /data .

docker run --rm \
  -v openclaw_workspace:/data:ro \
  -v "$PWD/openclaw-backup:/backup" \
  alpine:latest \
  tar czf /backup/openclaw_workspace.tar.gz -C /data .

docker run --rm \
  -v openclaw_auth:/data:ro \
  -v "$PWD/openclaw-backup:/backup" \
  alpine:latest \
  tar czf /backup/openclaw_auth.tar.gz -C /data .
```

En Windows PowerShell, crea primero la carpeta:

```powershell
New-Item -ItemType Directory -Force .\openclaw-backup
```

El procedimiento de copia puede ejecutarse desde una terminal compatible con Docker. Guarda los archivos resultantes fuera del repositorio si contienen credenciales o datos privados.

## Solución de problemas

### El contenedor se reinicia continuamente

Comprueba los logs:

```bash
docker logs --tail 200 openclaw-gateway
```

Causas frecuentes:

- Token del Gateway vacío o incorrecto.
- Configuración creada con una versión incompatible.
- Imagen no descargada correctamente.
- Variables del proveedor no disponibles.

### El panel no abre

Comprueba que el contenedor está activo:

```bash
docker ps --filter name=openclaw-gateway
```

Comprueba el puerto:

```bash
docker port openclaw-gateway
```

En Windows, verifica que Docker Desktop está ejecutándose y que el firewall permite el puerto `18789`.

### Error de permisos en Linux

Los volúmenes nombrados normalmente evitan problemas de permisos. No cambies el contenedor a `privileged` como solución rápida. Revisa primero los logs y el estado de los volúmenes.

### No se encuentra la API key

Comprueba que la variable está definida en el Stack de Portainer y que el Stack se ha recreado después de modificarla:

En Linux, puedes comprobar que las variables están presentes con:

```bash
docker inspect openclaw-gateway --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -E 'OPENAI|ANTHROPIC|GEMINI|OPENROUTER'
```

En PowerShell:

```powershell
docker inspect openclaw-gateway --format '{{range .Config.Env}}{{println .}}{{end}}'
```

No publiques la salida de estos comandos: puede contener secretos.

## Seguridad

OpenClaw puede ejecutar herramientas y recibir mensajes externos. Antes de exponerlo a otros usuarios o a Internet:

- No publiques directamente el puerto `18789` en Internet.
- Usa el token del Gateway.
- Prefiere una VPN, Tailscale o un proxy inverso con HTTPS.
- No uses `--privileged`.
- No montes `/var/run/docker.sock` salvo que hayas entendido las implicaciones y necesites explícitamente sandboxing basado en Docker.
- Protege las API keys con el gestor de secretos de Portainer o con variables protegidas.
- Revisa las listas de emparejamiento y permisos de los canales.
- Haz copias de seguridad de los volúmenes `openclaw_state`, `openclaw_workspace` y `openclaw_auth`.

Consulta la documentación oficial:

- [Security](https://docs.openclaw.ai/gateway/security)
- [Sandboxing](https://docs.openclaw.ai/gateway/sandboxing)
- [Docker](https://docs.openclaw.ai/install/docker)

## Licencia y atribución

Este documento es una guía de despliegue comunitaria. OpenClaw mantiene su propio código, licencias y documentación en:

- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw Docs](https://docs.openclaw.ai)

