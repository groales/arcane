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
- Puerto TCP `3552` en el contenedor, publicado como `3552:3552` en el host.
- `ENCRYPTION_KEY` con una clave aleatoria de 32 bytes. Genera una hexadecimal con `openssl rand -hex 32`.
- `PROJECTS_DIRECTORY` debe ser una ruta absoluta existente del host. El compose la monta en la misma ruta dentro del contenedor para que Arcane pueda operar sobre los proyectos y sus volúmenes relativos.

## Despliegue clonando con Git

Después de alojar este repositorio en un remoto propio, clónalo y configura las variables privadas/locales:

```sh
git clone <URL_DEL_REPOSITORIO> arcane
cd arcane
cp .env.example .env
openssl rand -hex 32
```

Copia la clave generada a `ENCRYPTION_KEY` y cambia `PROJECTS_DIRECTORY` por la ruta absoluta del host que contiene tus proyectos Compose. Ajusta `APP_URL`, `PUID` y `PGID` a este despliegue. Luego ejecuta:

```sh
mkdir -p data
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

Este repositorio local no tiene remoto configurado ni se publica automáticamente.

## Despliegue manual sin Git

Crea una carpeta, guarda `compose.yaml` en ella y crea un archivo `.env` en la misma carpeta. Usa estas variables, sustituyendo la ruta y generando una clave propia:

```dotenv
APP_URL=http://localhost:3552
ENCRYPTION_KEY=REEMPLAZAR_POR_CLAVE_GENERADA_CON_OPENSSL
PROJECTS_DIRECTORY=/ruta/absoluta/a/tus/proyectos
PUID=1000
PGID=1000
```

Crea el directorio local persistente y levanta el servicio:

```sh
mkdir -p data
docker compose config --quiet
docker compose up -d --pull always
docker compose ps
```

## Variables y persistencia

Compose lee automáticamente `.env` para interpolar la configuración:

| Variable | Función | Valor usado en este equipo |
|---|---|---|
| `APP_URL` | URL base de Arcane | `http://localhost:3552` |
| `PROJECTS_DIRECTORY` | Ruta absoluta de proyectos del host y contenedor | `/Users/groales/Developer/projects/docker` |
| `PUID` y `PGID` | UID/GID que Arcane usa para archivos administrados | `501` y `20` |
| `ENCRYPTION_KEY` | Clave de cifrado persistente | Generada localmente; no se versiona |

`./data` conserva la base de datos y la configuración de Arcane. `PROJECTS_DIRECTORY` conserva los proyectos y archivos administrados, en el directorio preexistente que se haya indicado en `.env`. El socket Docker se monta desde el host.

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

Abre http://localhost:3552 para acceder. La documentación de instalación describe el inicio de sesión inicial `arcane` / `arcane-admin`; cambia la contraseña inmediatamente después de entrar.

## Seguridad

- El montaje directo del socket da a Arcane control amplio del daemon Docker y, en la práctica, privilegios elevados en el host. Para restringir operaciones, usa un proxy de socket según la documentación oficial.
- El mapeo `3552:3552` publica el servicio en todas las interfaces de red del host. Limita el acceso mediante firewall o proxy inverso con TLS, autenticación y soporte WebSocket; no lo expongas directamente a Internet.
- `.env`, `data/`, `projects/` y `.DS_Store` están excluidos por `.gitignore`. No guardes secretos en Git ni los incorpores a imágenes.
- Conserva la misma clave `ENCRYPTION_KEY` y realiza copias de seguridad antes de actualizar la etiqueta `latest`.
