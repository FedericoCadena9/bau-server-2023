# Entorno local de Strapi

Esta guía sirve para crear y operar una instalación local después de reiniciar la computadora. No requiere una copia de base de datos compartida: cada desarrollador configura sus propias credenciales y datos locales. No copies secretos, credenciales productivas, dumps ni medios al repositorio.

El backend usa Strapi 4.11.0 y requiere Node.js 18.16.0. La configuración de la base se toma de variables de entorno (`PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER` y `PGPASSWORD`).

## 1. Configurar valores privados

En la raíz del repositorio, crea tu archivo ignorado `.env` a partir de la plantilla:

```sh
cp .env.example .env
```

Edita `.env` con valores propios para desarrollo. Genera valores distintos para `APP_KEYS`, `API_TOKEN_SALT`, `ADMIN_JWT_SECRET`, `JWT_SECRET` y `TRANSFER_TOKEN_SALT`; los valores de ejemplo no son secretos utilizables. Añade también la configuración de PostgreSQL. Los valores `POSTGRES_*` inicializan el contenedor y deben corresponder a los valores `PG*` que usa Strapi:

```dotenv
PGHOST=127.0.0.1
PGPORT=5433
PGDATABASE=bau_local
PGUSER=bau_local
PGPASSWORD=<tu contraseña local>

POSTGRES_DB=bau_local
POSTGRES_USER=bau_local
POSTGRES_PASSWORD=<tu contraseña local>
```

Usa la misma base, usuario y contraseña en ambos grupos. Mantén `.env` en tu máquina; Git ignora ese archivo. No agregues aquí valores productivos.

## 2. Iniciar Docker y PostgreSQL

Inicia Docker Desktop desde el menú de aplicaciones. En macOS también puedes abrirlo con `open -a Docker`; en Linux, inicia el servicio Docker con el mecanismo de tu distribución (por ejemplo, `sudo systemctl start docker`). Espera a que Docker esté listo:

```sh
docker info
```

Si todavía no existe el contenedor local, créalo una sola vez. El volumen nombrado conserva los datos aunque pares o recrees el contenedor:

```sh
docker run -d \
  --name bau-server-postgres \
  --restart unless-stopped \
  --env-file .env \
  --platform linux/amd64 \
  -p 127.0.0.1:5433:5432 \
  -v bau-server-postgres-data:/home/postgres/pgdata/data \
  timescale/timescaledb-ha:pg15.5-ts2.12.2
```

La opción `--platform linux/amd64` permite usar esta imagen x86_64 en Macs ARM64 mediante la emulación de Docker Desktop; también funciona en hosts amd64. La imagen indicada ofrece PostgreSQL 15.5, TimescaleDB 2.12.2 y Toolkit 1.17.0. Para otros requisitos, elige una imagen compatible y mantén las extensiones necesarias. La primera creación prepara una base vacía; carga tus propios datos locales solo si los necesitas.

En reinicios posteriores, Docker suele arrancarlo automáticamente por `--restart unless-stopped`. Si está detenido, inícialo sin recrearlo:

```sh
docker start bau-server-postgres
```

Comprueba que el contenedor está activo y PostgreSQL acepta conexiones:

```sh
docker ps --filter name=bau-server-postgres
docker exec bau-server-postgres pg_isready
```

`pg_isready` debe responder `accepting connections`. Si el contenedor aún se está inicializando, espera unos segundos y vuelve a ejecutar el comando.

## 3. Activar Node e iniciar Strapi

Activa Node.js 18.16.0 con el administrador de versiones que uses. Por ejemplo, con `nvm`:

```sh
nvm install 18.16.0
nvm use 18.16.0
node --version
```

La última línea debe mostrar `v18.16.0`. La aplicación carga automáticamente las variables de `.env` desde la raíz del proyecto. Instala dependencias si aún no tienes `node_modules` y ejecuta Strapi en modo desarrollo:

```sh
npm install
npm run develop
```

Deja ese proceso en ejecución mientras trabajas. Si el proyecto ya tiene dependencias instaladas, no hace falta repetir `npm install`.

El entorno `development` desactiva SSL para la conexión PostgreSQL local y guarda los medios con el proveedor local de Strapi en `public/uploads/`. Estos overrides aplican al ejecutar `npm run develop`; no cambian la configuración de producción, que mantiene su proveedor Cloudinary. Los archivos subidos localmente se ignoran en Git y cada desarrollador conserva sus propios medios.

## 4. URLs y comprobaciones

- API local: <http://127.0.0.1:1337>
- Panel de administración: <http://127.0.0.1:1337/admin>
- PostgreSQL: `127.0.0.1:5433`

Con Strapi iniciado, comprueba el endpoint de salud del API y que el panel responde por HTTP:

```sh
curl -fsS http://127.0.0.1:1337/_health
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' http://127.0.0.1:1337/admin
```

El endpoint de salud debe responder correctamente; el panel normalmente devuelve un código HTTP `200` o una redirección `3xx`.

## 5. Apagar sin borrar datos

Detén Strapi con `Ctrl+C` en la terminal donde corre `npm run develop`. Después detén el contenedor:

```sh
docker stop bau-server-postgres
```

Esto conserva el volumen `bau-server-postgres-data` y sus datos. Para volver a trabajar, inicia Docker, ejecuta `docker start bau-server-postgres`, espera a que `pg_isready` acepte conexiones y vuelve a activar Node e iniciar Strapi.

No uses `docker rm -v`, `docker volume rm` ni `docker volume prune` para apagar el entorno: esos comandos pueden borrar los datos persistentes.

## Solución breve de problemas

- **Docker no responde:** abre Docker Desktop o inicia el servicio y espera a que `docker info` termine correctamente.
- **El puerto 5433 está ocupado:** detén el proceso que lo usa o elige otro puerto libre; actualiza el mapeo `-p` y `PGPORT` para que coincidan.
- **PostgreSQL no está listo:** revisa `docker ps -a` y `docker logs bau-server-postgres`; confirma que `.env` define los tres `POSTGRES_*` con valores locales válidos.
- **Strapi no conecta a la base:** confirma que el contenedor acepta conexiones y que `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER` y `PGPASSWORD` en `.env` corresponden a esa instancia.
- **Versión de Node incompatible:** activa `18.16.0` y comprueba `node --version` antes de ejecutar `npm run develop`.
- **Faltan dependencias:** ejecuta `npm install` en la raíz del repo. No subas `.env`, `node_modules`, dumps ni medios generados localmente.
