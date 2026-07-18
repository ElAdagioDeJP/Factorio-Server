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
| `config/map-exchange-string.txt` | **El mundo** (exchange string exportado del juego, v2.0.73) | Solo al generar el save (paso 3) |
| `config/map-gen-settings.json` | Recursos, biters, terreno (fallback) | Solo si se genera sin exchange string |
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
3. **Generar el mundo desde el exchange string** (one-off, antes del primer deploy).
   Primero activar los mods de Space Age para la generación, luego crear el save:
   ```bash
   sudo mkdir -p /opt/factorio/mods /opt/factorio/saves
   sudo tee /opt/factorio/mods/mod-list.json >/dev/null <<'EOF'
   {"mods":[{"name":"base","enabled":true},{"name":"elevated-rails","enabled":true},{"name":"quality","enabled":true},{"name":"space-age","enabled":true}]}
   EOF
   sudo chown -R 845:845 /opt/factorio

   MES=$(curl -fsSL https://raw.githubusercontent.com/ElAdagioDeJP/Factorio-Server/main/config/map-exchange-string.txt)
   docker run --rm -u 845:845 -v /opt/factorio:/factorio \
     --entrypoint /opt/factorio/bin/x64/factorio \
     factoriotools/factorio:2.0.77 \
     --create /factorio/saves/mundo-amigos.zip \
     --map-exchange-string "$MES"
   ```
   Debe terminar con el save creado en `/opt/factorio/saves/mundo-amigos.zip`.
4. En Dokploy: nuevo servicio → **Compose** → source: este repo de GitHub, archivo
   `docker-compose.yml`. **No** configurar dominio/Traefik para este servicio.
5. Deploy. En los logs debe aparecer la carga de `mundo-amigos` con los mods
   `space-age`, `quality`, `elevated-rails` y luego `Hosting game at ...:34197`.
   (El compose usa `GENERATE_NEW_SAVE=false`: si olvidaste el paso 3, el contenedor
   falla con "no saves found" en vez de generar un mapa equivocado.)

## Conectarse

Factorio → Multijugador → **Conectar a dirección** → `IP_DEL_VPS:34197` → password
del juego (ver `game_password` en `config/server-settings.json`).

- El server NO aparece en el buscador público (deliberado).
- `require_user_verification` está en `false`: no exige cuenta de factorio.com.
- Con Space Age activo, el cliente necesita tener los mods del DLC. Si alguien no
  puede entrar por eso: poner `DLC_SPACE_AGE=false` en el compose, borrar
  `/opt/factorio/saves/` (o cambiar `SAVE_NAME`) y redeploy — genera mundo nuevo sin DLC.

## Cambiar el mundo

- **Nuevo exchange string**: reemplazar `config/map-exchange-string.txt`, commit,
  y en el VPS: parar el servicio en Dokploy, vaciar `/opt/factorio/saves/`
  (**se pierde el progreso**), repetir el paso 3 y redeploy.
- **Save ya creado** (alternativa): subir el `.zip` a `/opt/factorio/saves/` con
  `chown 845:845` — el server carga siempre el save más reciente.

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
