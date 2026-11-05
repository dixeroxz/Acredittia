# Acredittia

Plataforma web para armar, en un solo lugar, la carpeta de acreditación que una faena minera le pide a una empresa contratista. Proyecto de título CAPSTONE 2026, Ingeniería en Informática, Duoc UC, sede Alameda.

El código está en un repositorio privado. El clon exige acceso a esa cuenta.

| Recurso | Dirección |
|---|---|
| Repositorio (privado) | https://github.com/Njolul/ACREDITTIA-Desarrollo_FrontEND |
| Aplicación publicada | https://frontend-nine-khaki-95.vercel.app |

## Descripción

Una empresa contratista junta en Acredittia los documentos de la empresa, del personal y de los equipos, con sus vigencias y sus alertas. El trabajo queda ordenado antes de cargar esos documentos en la plataforma del mandante. El destino de la carga sigue siendo SIGA, INTELICONTROL u otra plataforma de la faena.

El circuito que este corte cierra es el de la contratista: cuentas, contratos, personal, equipos, documentos y alertas.

## Stack

| Capa | Tecnología |
|---|---|
| Lenguajes | TypeScript y Python 3.12 |
| Interfaz | Next.js 14, React y Tailwind |
| API | FastAPI y SQLAlchemy 2 |
| Trabajos en segundo plano | Celery |
| Base de datos | PostgreSQL 16 |
| Cola | Redis 7 |
| Ambiente local | Docker Compose |
| Frontend publicado | Vercel |
| Landing | Firebase Hosting |
| Nube prevista | Azure, con PostgreSQL y Blob Storage. En local los archivos quedan en disco y la base corre en Docker. |

La API publica su contrato OpenAPI en Swagger. La versión de la aplicación documentada en el diseño es 1.1.0.

## Cómo ejecutarlo

Hace falta Docker Desktop encendido y acceso de lectura al repositorio privado.

```bash
git clone https://github.com/Njolul/ACREDITTIA-Desarrollo_FrontEND.git
cd ACREDITTIA-Desarrollo_FrontEND/app
docker compose up --build
```

| Servicio | Dirección |
|---|---|
| Aplicación | http://localhost:3000 |
| API y Swagger | http://localhost:8000/docs |

La primera vez, Docker crea la base con los scripts de `modelo_datos/` y carga las faenas y las plantillas. La semilla trae 8 faenas y 42 plantillas. Entre las faenas de demostración están Los Pelambres, Minera Centinela, Antucoya y El Teniente.

Compose levanta PostgreSQL 16, Redis 7, un proceso de migración que aplica el SQL y termina, la API, el worker de Celery, el planificador y la aplicación web. La API queda en el puerto 8000 y la web en el 3000.

Para detener, dentro de `app/`:

```bash
docker compose down
```

## Cuentas de prueba

| Uso | Correo | Contraseña |
|---|---|---|
| Administrador | admin@acredittia.cl | Admin2026! |
| Empresa demo | demo@acredittia.cl | Demo2026! |
| Empresa de prueba | prueba@dfd.cl | Prue123456 |

La semilla local crea el administrador y la empresa demo, y los deja aprobados. Una empresa que se registra después queda pendiente: el administrador de la plataforma la aprueba antes de que el ingreso entregue sesión. La tercera cuenta es la de prueba que documenta el portafolio de título.

## Arquitectura

El navegador habla con Next.js. Esa aplicación llama a FastAPI bajo el prefijo `/api/v1`. La API lee y escribe PostgreSQL. La revisión de documentos y el recálculo de vencimientos pasan por Redis y Celery. En local los archivos quedan en disco. El diseño de nube los deja en Azure Blob Storage: el navegador sube el binario con una dirección de vigencia corta, y la API solo registra la ruta.

```
Navegador
    │
    ▼
Next.js :3000  ── HTTP /api/v1 ──►  FastAPI :8000
                                        │
                      ┌─────────────────┼─────────────────┐
                      ▼                 ▼                 ▼
                PostgreSQL 16       Redis 7         Disco local
                :5432               :6379           /data/uploads
                                        │
                                        ▼
                                 Celery worker
                                 Celery beat (America/Santiago)
```

El esquema no lo crea la API al arrancar. Lo aplica el proceso de migración, para que varias réplicas no compitan por el mismo DDL. Cada petición de negocio fija la empresa en la sesión. PostgreSQL, con Row Level Security, devuelve solo las filas de esa empresa. Hay tres roles de cuenta: administrador de la plataforma, administrador de la empresa y administrador de un contrato. El personal y los equipos son sujetos de la carpeta, no usuarios del portal.

El planificador, en hora de Santiago, recalcula vencimientos a las 00:30, purga archivos temporales a las 04:00 y dispara los reportes programados cada hora, al minuto 5. La revisión de un documento queda encolada al confirmar el archivo. El motor por defecto es simulado.

En local esas piezas son contenedores. Hay dos imágenes, definidas en `app/backend/Dockerfile` y `app/frontend/Dockerfile`, y un solo orquestador, `app/docker-compose.yml`. La configuración entra por variables de entorno. Compose las declara en cada servicio. Quien corre la API fuera de Compose puede usar un archivo `.env`.

## Dockerfile

Hay dos Dockerfiles. Compose construye ambos con `docker compose up --build`.

### API, worker, beat y migración

`app/backend/Dockerfile` parte de `python:3.12-slim` y produce una sola imagen para cuatro procesos. El código es el mismo. Cambia el comando.

| Paso | Qué hace |
|---|---|
| Base | Python 3.12 en imagen slim. Instala `curl` para el chequeo de salud. |
| Dependencias | Copia `requirements.txt` e instala con pip, sin caché. |
| Código | Copia `app/`, `worker/`, `migrate/` y `tests/` a `/srv`. |
| Usuario | Crea el usuario `acredittia` (uid 10001), sin privilegios, y los directorios `/data/uploads` y `/data/keys`. |
| Salud | `GET http://localhost:8000/health` cada 30 segundos. |
| Comando por defecto | `uvicorn app.main:app --host 0.0.0.0 --port 8000`. Expone el puerto 8000. |

Compose reemplaza ese comando en tres servicios: `python -m migrate.run` en la migración, `celery -A worker.celery_app worker` en el worker y `celery -A worker.celery_app beat` en el planificador.

### Interfaz web

`app/frontend/Dockerfile` parte de `node:20-alpine` y usa tres etapas.

| Etapa | Qué hace |
|---|---|
| `deps` | Copia `package.json` e instala con npm. |
| `build` | Recibe `NEXT_PUBLIC_API_URL` (por defecto `http://localhost:8000`) y ejecuta `npm run build`. |
| Final | Copia el resultado `standalone`, los estáticos y `public/`. Arranca con `node server.js` en el puerto 3000. |

El build de Docker usa la salida `standalone` de Next.js. En Vercel esa salida se omite, porque `next.config.mjs` la desactiva cuando existe la variable `VERCEL`.

## Docker Compose

El archivo es `app/docker-compose.yml`. Un `docker compose up --build` dentro de `app/` levanta el circuito completo. El orden lo marcan las dependencias: PostgreSQL y Redis tienen que estar sanos, la migración aplica el SQL y termina, y solo entonces suben la API, el worker y el planificador. La web espera a la API.

| Servicio | Imagen o build | Proceso | Puerto |
|---|---|---|---|
| `db` | `postgres:16` | Base `acredittia`. Volumen `pgdata`. | 5432 |
| `redis` | `redis:7-alpine` | Cola, sin persistencia (`--save ""`). | 6379 |
| `migrate` | `./backend` | `python -m migrate.run`. Monta `modelo_datos/` en `/schema` y termina. | — |
| `api` | `./backend` | Uvicorn. Volúmenes `uploads` y `keys`. | 8000 |
| `worker` | `./backend` | Celery, concurrencia 2. Mismos volúmenes de archivos y claves. | — |
| `beat` | `./backend` | Celery beat. La agenda se escribe en `/tmp`. | — |
| `web` | `./frontend` | Next.js. El build recibe `NEXT_PUBLIC_API_URL=http://localhost:8000`. | 3000 |

`db` y `redis` tienen healthcheck (`pg_isready` y `redis-cli ping`). La API, dentro de la imagen, también comprueba `GET /health`.

En este Compose los procesos de base usan el usuario de desarrollo `postgres`. La separación en tres roles (`acredittia_owner`, `acredittia_app`, `acredittia_worker`) está descrita para Azure y no está aplicada en el archivo local. Los valores de `JWT_SECRET`, de la base y de Redis que trae el archivo son de desarrollo.

Para detener y conservar los volúmenes: `docker compose down`. Para borrar también la base, los archivos y las claves locales: `docker compose down -v`.

## Variables de entorno

No hay un `.env.example` versionado. `.gitignore` ignora `.env` y `.env.*`, y deja pasar solo un `.env.example` que el repositorio no incluye. Compose no lee un archivo `.env` para arrancar: escribe las variables en cada servicio. La API sí acepta un `.env` junto al proceso, porque `app/backend/app/config.py` lo carga con Pydantic Settings. Sirve para correr Uvicorn fuera de Docker, con PostgreSQL ya migrado.

El bloque siguiente es el ejemplo local. Los secretos de terceros quedan vacíos. `JWT_SECRET` y la clave de PostgreSQL son los valores de desarrollo que ya declara `docker-compose.yml`. En un despliegue real se reemplazan. No se commitea este archivo.

```bash
# app/backend/.env  — ejemplo local, no versionar

# Base
DATABASE_URL=postgresql+psycopg://postgres:postgres@localhost:5432/acredittia
SCHEMA_DIR=/schema
SCHEMA_VERSION_ESPERADA=6
DB_APPLY_SCHEMA_ON_START=false
DB_RLS_ENABLED=true

# Sesión
JWT_SECRET=dev-secret-cambiar-en-produccion
ACCESS_TTL_MIN=30
REFRESH_TTL_DAYS=30
RESET_TTL_MIN=60
CORS_ORIGINS=http://localhost:3000

# Archivos: local en este corte. azure solo con cuenta y contenedor reales.
STORAGE_BACKEND=local
STORAGE_DIR=/data/uploads
AZURE_STORAGE_ACCOUNT=
AZURE_BLOB_CONN=
AZURE_BLOB_CONTAINER=docs
SAS_UPLOAD_TTL_MIN=15
SAS_DOWNLOAD_TTL_MIN=15
PUBLIC_BASE_URL=http://localhost:8000

# Cola. Compose usa celery. Sin Redis, inproc agenda dentro del mismo proceso.
QUEUE_BACKEND=celery
REDIS_URL=redis://localhost:6379/0

# Cifrado de credenciales de plataforma. keyvault es el diseño de producción.
JWE_BACKEND=local
AZURE_KEYVAULT_URL=
KEYVAULT_KEY_NAME=plataforma-cred
KEYS_DIR=/data/keys
CREDENCIAL_TTL_MESES=12

# Revisión. simulada no llama a un proveedor. claude exige la clave.
IA_BACKEND=simulada
ANTHROPIC_API_KEY=

# Correo y mensajería. Vacío: el reset de contraseña no envía correo.
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
EMAIL_FROM=no-reply@acredittia.cl
WHATSAPP_TOKEN=
WHATSAPP_PHONE_ID=
WHATSAPP_VERIFY_TOKEN=

# Pagos, fuera de este corte.
PAGOS_WEBHOOK_SECRET=

# Límites
MAX_UPLOAD_MB=20
PAGE_SIZE_DEFAULT=25
PAGE_SIZE_MAX=100
MATRIZ_FILAS_MAX=200
EXPORT_FILAS_MAX=5000
```

Dos variables no entran en ese archivo y solo aplican si el proceso se separa: `MIGRATE_DATABASE_URL` la lee el job de migración y `WORKER_DATABASE_URL` la lee el worker. En Compose ambos usan `DATABASE_URL`.

La web, en build, necesita:

```bash
# app/frontend  — argumento de build, no un secreto
NEXT_PUBLIC_API_URL=http://localhost:8000
# Distinto de 0: las pantallas de usuarios, sistemas e indicadores leen el simulador.
NEXT_PUBLIC_OPS_MOCK=1
```

PostgreSQL del Compose, solo en el servicio `db`: `POSTGRES_DB=acredittia`, `POSTGRES_USER=postgres`, `POSTGRES_PASSWORD=postgres`.

## Mapa de carpetas

```
ACREDITTIA-Desarrollo_FrontEND/
├── app/            producto: API, web y docker-compose.yml
├── modelo_datos/   scripts SQL, del esquema a la migración 1.1
├── landing/        sitio de presentación
├── admin/          consola interna
└── docs/           informes y evidencias de la asignatura
```

`app/` es lo que levanta Docker. `modelo_datos/` recorre el esquema hasta la migración 1.1. `landing/` es el sitio estático de presentación, publicado en Firebase Hosting, y no llama a la API. `admin/` es la consola interna, presente en el árbol y todavía sin montar en el arranque. `docs/` guarda informes y evidencias de la asignatura.

## Equipo

| Integrante | Responsabilidad |
|---|---|
| Diego Olvea | Modelo de datos y API (FastAPI, PostgreSQL) |
| Emmanuel Molina | Interfaz web (Next.js) |

## Metodología

El trabajo siguió un Scrum corto: seis sprints de dos semanas. El alcance se cortó con MoSCoW. El primer corte es el circuito de acreditación: cuentas, contratos, personal, equipos, documentos y alertas. El código está en GitHub. Cada entrega se probó contra la base local con Docker. La API publica su contrato en Swagger (`/docs`).

Pagos, conectores reales con las plataformas del mandante y el despliegue completo en Azure quedaron fuera de este corte.

## Estado del corte

### Qué está

Una empresa aprobada entra al portal, arma el contrato de una faena, carga personal y equipos, sube los archivos de la carpeta y consulta alertas, panel y calendario sobre los datos de su propia empresa. La revisión del documento queda en la cola. El ambiente que se puede repetir es Docker Compose. El frontend está publicado en Vercel y la landing en Firebase Hosting.

### Qué quedó fuera

- **Pagos.** Quedaron fuera del corte de título.
- **Conectores del mandante.** Hay clases para SIGA, Workmate, Metacontratas, Webcontrol, WhatsApp y Google Drive. La sincronización no está implementada. Se puede guardar la integración y la credencial cifrada. No se publica el estado de acreditación en la plataforma del mandante.
- **Azure.** El diseño contempla PostgreSQL, Blob Storage y el resto de la guía de producción (Container Apps, Redis administrado y Key Vault). Esa suscripción no forma parte de lo que este corte deja ejecutando. En local la base corre en Docker y los archivos quedan en disco.
- **Consola interna.** `admin/` describe veinte rutas bajo `/api/v1/admin/ops`, con SQL y trabajos propios. Ese router no está montado en `main.py`, y sus scripts no entran en la migración del producto. Las pantallas de usuarios, sistemas e indicadores del portal leen un simulador mientras esa consola no está cableada.
- **Reportes.** La salida de este corte es CSV. El enumerado admite PDF y Excel. No hay generador de PDF. Si falta la librería de planillas, el Excel también se escribe como CSV y el nombre del archivo lo indica.
- **Restablecimiento de contraseña.** El sistema guarda el token y responde igual exista o no el correo. El envío del enlace por correo no está implementado.
