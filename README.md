# Factorio-Server

Servidor dedicado headless de Factorio **2.0.77 estable** con Space Age, desplegado
con Dokploy (Docker Compose) en Hetzner CX33. Puerto de juego UDP directo al host
(no pasa por Traefik).

## Estructura

| Archivo | Qué es | Cuándo aplica |
|---|---|---|
| `docker-compose.yml` | Servicio para Dokploy (tipo Compose) | Cada deploy |
| `config/server-settings.json` | Nombre, password, autosaves, verificación | Cada reinicio |
| `config/server-adminlist.json` | Admins (usuario de Factorio) | Cada reinicio |
| `config/map-exchange-string.txt` | Exchange string (alternativa manual, ver "Cambiar el mundo") | Solo si se genera el save a mano |
| `config/map-gen-settings.json` | Recursos, biters, terreno | Solo al generar el save (primer arranque) |
| `config/map-settings.json` | Evolución, expansión, pollution | Cada reinicio (mayoría de campos) |

Datos persistentes en el VPS: `/opt/factorio/` (saves, mods, logs). Nunca los toca un redeploy.

## Despliegue (una vez)

1. En el VPS, preparar el directorio de datos:
   ```bash
   sudo mkdir -p /opt/factorio/config
   sudo chown -R 845:845 /opt/factorio
   ```
2. Abrir el puerto UDP:
   ```bash
   sudo ufw allow 34197/udp   # y/o regla inbound UDP 34197 en Hetzner Cloud Firewall
   ```
3. En Dokploy: nuevo servicio → **Compose** → source: este repo de GitHub, archivo
   `docker-compose.yml`. **No** configurar dominio/Traefik para este servicio.
4. Deploy. En el primer arranque el contenedor genera `churros.zip` desde
   `config/map-gen-settings.json` y `config/map-settings.json`
   (`GENERATE_NEW_SAVE=true` + `SAVE_NAME=churros`); en los siguientes NO lo
   regenera (ya existe) y `LOAD_LATEST_SAVE=true` carga el save más reciente —
   redeploys idempotentes, sin pérdida de mundo. En los logs deben aparecer los
   mods `space-age`, `quality`, `elevated-rails` y luego `Hosting game at ...:34197`.

## Conectarse

Factorio → Multijugador → **Conectar a dirección** → `IP_DEL_VPS:34197` → password
del juego (ver `game_password` en `config/server-settings.json`).

- El server NO aparece en el buscador público (deliberado).
- `require_user_verification` está en `false`: no exige cuenta de factorio.com.
- Con Space Age activo, el cliente necesita tener los mods del DLC. Si alguien no
  puede entrar por eso: poner `DLC_SPACE_AGE=false` en el compose, borrar
  `/opt/factorio/saves/` (o cambiar `SAVE_NAME`) y redeploy — genera mundo nuevo sin DLC.

## Cambiar el mundo

- **Regenerar con otra config**: editar `config/map-gen-settings.json`, commit,
  parar el servicio en Dokploy, borrar `/opt/factorio/saves/churros.zip`
  (**se pierde el progreso**) y redeploy — se genera de nuevo.
- **Desde exchange string** (manual): parar el servicio, borrar el save y crear uno nuevo:
  ```bash
  MES=$(cat config/map-exchange-string.txt)
  docker run --rm -u 845:845 -v /opt/factorio:/factorio \
    --entrypoint /opt/factorio/bin/x64/factorio \
    factoriotools/factorio:2.0.77 \
    --create /factorio/saves/churros.zip \
    --map-exchange-string "$MES"
  ```
- **Save ya creado**: subirlo como `/opt/factorio/saves/churros.zip` con `chown 845:845`.

## Administración

- Promover admin in-game: `/promote nombre` (o editar `config/server-adminlist.json`).
- RCON: puerto 27015 solo en localhost del VPS. Password en `/opt/factorio/config/rconpw`.
  Desde tu PC: `ssh -L 27015:localhost:27015 usuario@vps` y apuntar el cliente RCON a
  `localhost:27015`.

## Actualizar versión

1. Backup del save: `sudo tar czf ~/factorio-saves-$(date +%F).tgz -C /opt/factorio saves`
2. Cambiar el tag en `docker-compose.yml` (ej. `2.0.78`), commit, redeploy en Dokploy.
3. El save migra hacia adelante automáticamente. **No se puede volver a una versión
   anterior** después de cargar el save en la nueva.

Nunca usar el tag `latest` (apunta a experimental 2.1.x).

## Backups

Los autosaves rotan en `/opt/factorio/saves/` (8 slots, cada 10 min) pero viven en el
mismo disco. Recomendado: cron diario (o flujo de n8n) que suba un tar de `saves/` a
almacenamiento externo (Storage Box / S3 / rclone).
