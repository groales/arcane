# Arcane con Docker Compose

Arcane es una interfaz web para administrar contenedores, imágenes, volúmenes, redes y proyectos Docker Compose en este equipo. Este directorio mantiene una instalación de un solo host con datos locales persistentes.

## Enlaces oficiales

- Sitio y documentación: https://getarcane.app/docs
- Instalación oficial: https://getarcane.app/docs/get-started/installation.html
- Variables de entorno: https://getarcane.app/docs/reference/environment-variables.html
- Seguridad del socket Docker: https://getarcane.app/docs/security/socket-proxy.html
- Código e imagen publicada por el mantenedor: https://github.com/getarcaneapp/arcane
- Imagen usada: `ghcr.io/getarcaneapp/manager:latest`

La guía oficial actualmente recomienda la imagen `ghcr.io/getarcaneapp/manager:latest`. No se usa `ghcr.io/getarcaneapp/arcane` como imagen de instalación: esa cadena no coincide con el ejemplo de despliegue oficial.

## Requisitos y puertos

- Docker Engine y el complemento Docker Compose.
- Acceso local al socket Docker `/var/run/docker.sock`.
- Puerto TCP `3552` en el contenedor; por defecto se publica en `127.0.0.1:3552` para limitar el acceso a este equipo.
- Una clave de cifrado de 32 bytes (`ENCRYPTION_KEY`).

## Despliegue clonando con Git

Una vez publicado este proyecto en un remoto propio, clónalo y crea la configuración privada:

```sh
git clone <URL_DEL_REPOSITORIO> arcane
cd arcane
cp .env.example .env
openssl rand -hex 32
```

Copia el valor generado en `ENCRYPTION_KEY` dentro de `.env`, revisa el puerto, `APP_URL`, UID/GID y la ruta de proyectos, y después ejecuta:

```sh
mkdir -p data projects
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

Este repositorio local no tiene remoto configurado ni se publica automáticamente.

## Despliegue manual sin Git

Crea una carpeta, guarda `compose.yaml` en ella y crea a su lado un `.env` con este contenido (sustituye la clave por una generada con `openssl rand -hex 32`):

```dotenv
ARCANE_IMAGE=ghcr.io/getarcaneapp/manager:latest
ARCANE_BIND_IP=127.0.0.1
ARCANE_PORT=3552
APP_URL=http://localhost:3552
ENCRYPTION_KEY=REEMPLAZAR_POR_CLAVE_ALEATORIA_DE_64_CARACTERES_HEXADECIMALES
PUID=1000
PGID=1000
PROJECTS_HOST_PATH=./projects
```

Después, en la carpeta:

```sh
mkdir -p data projects
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

## Configuración, variables y persistencia

Compose carga automáticamente `.env` para interpolar los valores. Las variables principales son:

| Variable | Uso | Valor predeterminado |
|---|---|---|
| `ARCANE_IMAGE` | Imagen de Arcane | `ghcr.io/getarcaneapp/manager:latest` |
| `ARCANE_BIND_IP` | Dirección local donde publicar el puerto | `127.0.0.1` |
| `ARCANE_PORT` | Puerto publicado en el host | `3552` |
| `APP_URL` | URL usada por la aplicación | `http://localhost:3552` |
| `ENCRYPTION_KEY` | Clave de cifrado persistente, obligatoria | Sin valor; configúrala en `.env` |
| `PUID`, `PGID` | UID/GID para los archivos que crea Arcane | `1000` |
| `PROJECTS_HOST_PATH` | Carpeta del host para proyectos administrados | `./projects` |

`./data` conserva base de datos y configuración de Arcane; `./projects` conserva los proyectos que administra. El socket Docker se monta desde el host. Protege y respalda `data/` y `projects/`. No cambies ni pierdas `ENCRYPTION_KEY`: es necesaria para descifrar datos protegidos.

## Operación

Desde esta carpeta:

```sh
# Estado y registros
docker compose ps
docker compose logs -f arcane

# Actualizar a la imagen publicada más reciente
docker compose up -d --pull always

# Detener sin borrar los datos
docker compose down
```

Para acceder, abre http://localhost:3552. La documentación de instalación indica las credenciales iniciales `arcane` / `arcane-admin`; cámbialas inmediatamente tras iniciar sesión. Si modificas el puerto publicado, usa `http://localhost:<ARCANE_PORT>` y ajusta `APP_URL` según corresponda.

## Seguridad

- El socket Docker otorga a Arcane control amplio sobre el daemon y, en la práctica, privilegios elevados en el host. La opción de montaje directo sigue la instalación oficial más sencilla; para reducir permisos, configura un proxy de socket siguiendo la guía oficial.
- El puerto se vincula a `127.0.0.1` por defecto. No lo expongas directamente a Internet; para acceso remoto usa un proxy inverso con TLS, autenticación y WebSockets.
- `.env` está excluido de Git. Nunca incluyas claves o contraseñas en el repositorio, en la imagen ni en registros.
- La etiqueta `latest` recibe actualizaciones; revisa los cambios y conserva una copia de seguridad antes de actualizar en entornos importantes.
