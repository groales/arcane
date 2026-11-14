# Arcane con Docker Compose

Arcane es una interfaz web para administrar contenedores, imágenes, volúmenes, redes y proyectos Docker Compose del equipo. Este repositorio contiene una instalación de un solo host con datos persistentes locales.

## Enlaces oficiales

- Sitio y documentación: https://getarcane.app/docs
- Instalación oficial: https://getarcane.app/docs/get-started/installation.html
- Variables de entorno: https://getarcane.app/docs/reference/environment-variables.html
- Seguridad del socket Docker: https://getarcane.app/docs/security/socket-proxy.html
- Repositorio del mantenedor: https://github.com/getarcaneapp/arcane
- Imagen recomendada oficialmente: `ghcr.io/getarcaneapp/manager:latest`

La instalación oficial recomienda `ghcr.io/getarcaneapp/manager:latest`; es la imagen configurada en `compose.yaml`.

## Requisitos y puertos

- Docker Engine y el complemento Docker Compose.
- Socket Docker disponible en `/var/run/docker.sock`.
- Puerto TCP `3552` en el contenedor, publicado como `3552:3552` en todas las interfaces IPv4 e IPv6 del host.
- `ENCRYPTION_KEY` con una clave aleatoria de 32 bytes. Genera una hexadecimal con `openssl rand -hex 32`.
- `PROJECTS_DIRECTORY` debe ser una ruta absoluta existente del host. El compose la monta en la misma ruta dentro del contenedor para que Arcane pueda operar sobre los proyectos y sus volúmenes relativos. El directorio del host debe existir antes de iniciar el servicio.

## Despliegue clonando con Git

Después de alojar este repositorio en un remoto propio, clónalo y configura las variables privadas/locales:

```sh
git clone <URL_DEL_REPOSITORIO> arcane
cd arcane
cp .env.example .env
openssl rand -hex 32
```

Copia la clave generada a `ENCRYPTION_KEY` y cambia `PROJECTS_DIRECTORY` por la ruta absoluta existente del host que contiene tus proyectos Compose. El mismo valor se usa como ruta dentro del contenedor. Ajusta `APP_URL`, `PUID` y `PGID` a este despliegue. Luego crea `data/` y verifica que exista el directorio de proyectos indicado antes de ejecutar:

```sh
mkdir -p data
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

Este repositorio local no tiene remoto configurado ni se publica automáticamente.

## Despliegue manual sin Git

Crea una carpeta para Arcane y guarda en ella los dos archivos de referencia siguientes como `compose.yaml` y `.env.example`. Copia la plantilla a `.env`, genera una clave nueva y edita la ruta de proyectos para que sea absoluta y exista en el host. El mismo `PROJECTS_DIRECTORY` se monta en esa misma ruta dentro del contenedor.

### `compose.yaml`

```yaml
services:
  arcane:
    image: ghcr.io/getarcaneapp/manager:latest
    container_name: arcane
    ports:
      - "3552:3552"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
      - ${PROJECTS_DIRECTORY}:${PROJECTS_DIRECTORY} # Ruta del host donde están los proyectos
    environment:
      - APP_URL=${APP_URL}
      - PROJECTS_DIRECTORY=${PROJECTS_DIRECTORY}
      - PUID=${PUID:-1000}
      - PGID=${PGID:-1000}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
    cgroup: host
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "./arcane", "health", "--timeout", "2s"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 15s
```

### `.env.example`

```dotenv
# Copia a .env y genera ENCRYPTION_KEY con: openssl rand -hex 32
APP_URL=http://localhost:3552
ENCRYPTION_KEY=

# Ruta absoluta del host que contiene los proyectos Docker Compose que Arcane administrará.
# Se monta en el mismo path dentro del contenedor para conservar las rutas de los proyectos.
PROJECTS_DIRECTORY=/ruta/absoluta/a/proyectos

# UID/GID del propietario deseado para archivos creados por Arcane.
PUID=1000
PGID=1000
```

Copia la plantilla y edita sus valores:

```sh
cp .env.example .env
openssl rand -hex 32
```

Pega la clave generada en `ENCRYPTION_KEY`, reemplaza `PROJECTS_DIRECTORY` por la ruta absoluta real y ajusta `APP_URL`, `PUID` y `PGID`. Crea el directorio local persistente `data/` y asegúrate de que la carpeta de proyectos indicada exista. Después valida y levanta Arcane:

```sh
mkdir -p data
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

## Uso detrás de un proxy

`compose.override.yaml` es una configuración opcional para integrar Arcane con un proxy inverso que ya esté conectado a la red Docker externa `proxy`. Docker Compose carga automáticamente `compose.yaml` y `compose.override.yaml` cuando ejecutas los comandos desde esta carpeta. El override:

- Conecta el servicio `arcane` a la red externa ya existente `proxy`; no crea esa red.
- Elimina la publicación del puerto del archivo principal mediante `ports: !reset []`. Arcane deja de estar publicado directamente en los puertos del host.
- Permite que el proxy alcance Arcane por el nombre de contenedor/servicio `arcane` y el puerto interno `3552`, por ejemplo como destino `http://arcane:3552`.

La red `proxy` debe existir previamente y el proxy inverso debe estar conectado a ella. El archivo versionable `compose.override.example.yaml` contiene esta configuración. Para activarla, copia la plantilla a `compose.override.yaml`, que está excluido de Git porque puede contener ajustes específicos del host:

```sh
cp compose.override.example.yaml compose.override.yaml
docker network inspect proxy
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

La plantilla `compose.override.example.yaml`:

```yaml
# Plantilla versionable para ejecutar Arcane detrás de un proxy inverso.
# Copia este archivo a compose.override.yaml para activarla.
services:
  arcane:
    # Retira el mapeo de puertos definido en compose.yaml.
    ports: !reset []
    networks:
      - proxy

networks:
  # La red debe existir previamente y el proxy inverso debe estar conectado a ella.
  proxy:
    external: true
    name: proxy
```

Para un despliegue manual sin Git detrás del proxy, copia ambos archivos de override junto a `compose.yaml`. Si no necesitas el proxy y quieres publicar el puerto del host directamente, no crees `compose.override.yaml`.

## Variables y persistencia

Compose lee automáticamente `.env` para interpolar la configuración:

| Variable | Función | Valor usado en este equipo |
|---|---|---|
| `APP_URL` | URL base de Arcane | `http://localhost:3552` |
| `PROJECTS_DIRECTORY` | Ruta absoluta de proyectos del host y contenedor | `~/Developer/projects/docker/arcane/` |
| `PUID` y `PGID` | UID/GID que Arcane usa para archivos administrados | `501` y `20` |
| `ENCRYPTION_KEY` | Clave de cifrado persistente | Generada localmente; no se versiona |

`./data` conserva la base de datos y la configuración de Arcane. `PROJECTS_DIRECTORY` conserva los proyectos y archivos administrados, en el directorio preexistente que se haya indicado en `.env`; se monta en el mismo path del contenedor. El socket Docker se monta desde el host. En este despliegue, la ruta se representa como `~/Developer/projects/docker/arcane/`.

No pierdas ni regeneres `ENCRYPTION_KEY` después de guardar datos cifrados. Protege y respalda `data/`, los proyectos y el archivo `.env`.

## Operación

Ejecuta estos comandos desde la carpeta del proyecto:

```sh
# Ver estado y registros
docker compose ps
docker compose logs -f arcane

# Descargar la imagen actual y actualizar
docker compose up -d --pull always

# Detener Arcane sin borrar los datos
docker compose down
```

En el equipo anfitrión, abre http://localhost:3552 para acceder. Como el mapeo `3552:3552` escucha en todas las interfaces, desde otro equipo usa la IP o el nombre de host del servidor, por ejemplo `http://<IP_DEL_HOST>:3552`, solo si el acceso de red está permitido. La documentación de instalación describe el inicio de sesión inicial `arcane` / `arcane-admin`; cambia la contraseña inmediatamente después de entrar.

## Seguridad

- El montaje directo del socket da a Arcane control amplio del daemon Docker y, en la práctica, privilegios elevados en el host. Para restringir operaciones, usa un proxy de socket según la documentación oficial.
- El mapeo `3552:3552` publica el servicio en todas las interfaces IPv4 e IPv6 del host. Limita el acceso mediante firewall o proxy inverso con TLS, autenticación y soporte WebSocket; no lo expongas directamente a Internet. Para acceso solo local, cambia el mapeo a `127.0.0.1:3552:3552`.
- `.env`, `data/`, `projects/` y `.DS_Store` están excluidos por `.gitignore`. No guardes secretos en Git ni los incorpores a imágenes.
- Conserva la misma clave `ENCRYPTION_KEY` y realiza copias de seguridad antes de actualizar la etiqueta `latest`.
