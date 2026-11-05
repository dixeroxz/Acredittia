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

El esquema no lo crea la API al arrancar. Lo aplica el proceso de migración, para que varias réplicas no compitan por el mismo DDL. Cada petición de negocio fija la empresa en la sesión. PostgreSQL, con Row Level Security, devuelve solo las filas de esa empresa. Hay tres roles de cuenta: administrador de la plataforma, administrador de la empresa y administrador de un contrato. El personal y los equipos son sujetos de la carpeta, no usuarios del portal.

El planificador, en hora de Santiago, recalcula vencimientos, purga archivos temporales y dispara los reportes programados. La revisión de un documento queda encolada al confirmar el archivo. El motor por defecto es simulado.

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
