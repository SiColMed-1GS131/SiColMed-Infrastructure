
# SiColMed – Entorno de desarrollo

Sistema de manejo de certificaciones y recertificaciones de médicos en Panamá (Desarrollo de Software III).

Este repositorio (`SiColMed-Infrastructure`) contiene lo necesario para levantar **todo el sistema con un solo comando**: base de datos, API y web.

## 1. Repositorios

| Repositorio | Contenido |
|---|---|
| `SiColMed-BackEnd` | API en Python 3.12 + FastAPI, con su Dockerfile |
| `SiColMed-FrontEnd` | Web en PHP, HTML, CSS y JavaScript, con su Dockerfile |
| `SiColMed-Infrastructure` | `docker-compose.yml`, `.env.example` y esta documentación |

## 2. Herramientas

| Subgrupo | Lenguajes | Herramientas | Detalles |
|---|---|---|---|
| Backend | Python 3.12 | FastAPI, Visual Studio Code | Uso de Dockerfile. Patrón Modelo-Vista-Controlador |
| Frontend | PHP, CSS, JavaScript, HTML | Figma, Visual Studio Code | Sesiones, formularios y consumo de la API |
| Login | Python 3.12, CSS, HTML, JavaScript | Visual Studio Code | Autenticación con JWT |
| Pruebas | - | Visual Studio Code | `pytest` para la API y Swagger para pruebas manuales |
| Base de datos | SQL | PostgreSQL, draw.io / PlantUML | Diagramas en la carpeta `docs/` |

## 3. Requisitos previos

- **Docker Desktop** con el motor WSL 2 activado (Settings → General → *Use the WSL 2 based engine*).
- **GitHub Desktop** e inicio de sesión con tu cuenta.
- **Visual Studio Code** (recomendado: extensiones Python, PHP Intelephense y Docker).
- **Python 3** instalado en tu máquina (fuera de Docker), solo para generar la clave JWT del paso 4.3. Si no tienes Python a mano, puedes usar `openssl rand -base64 32` en su lugar.

No necesitas instalar PHP, PostgreSQL ni XAMPP: todo corre dentro de Docker.

## 4. Instalación

### 4.1 Clonar los tres repositorios

Crea una carpeta (por ejemplo `SiColMed`) y clona dentro los tres repos con GitHub Desktop (**File → Clone repository**).

Las carpetas deben quedar **como hermanas y con estos nombres exactos**, porque el compose las referencia por ruta:

```
SiColMed/
├── SiColMed-BackEnd/
├── SiColMed-FrontEnd/
└── SiColMed-Infrastructure/
```

### 4.2 Crear el archivo `.env`

Dentro de `SiColMed-Infrastructure`:

```bash
cp .env.example .env          # Linux, Mac o WSL
copy .env.example .env        # CMD de Windows
```

El `.env` es **personal y nunca se sube a GitHub**. Los valores del `.env.example` sirven para desarrollo local sin cambios, **excepto `JWT_SECRET_KEY`**, que debes generar tú mismo en el siguiente paso.

### 4.3 Generar la clave JWT

El backend no arranca el login sin una `JWT_SECRET_KEY` de al menos 32 caracteres. Genera una:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

o, si no tienes Python a mano:

```bash
openssl rand -base64 32
```

Copia el resultado y agrégalo a tu `.env` (dentro de `SiColMed-Infrastructure`):

```dotenv
JWT_SECRET_KEY=<el valor que generaste>
```

Sin este paso, cualquier intento de login responde `500 Internal Server Error` con `ConfiguracionJWTError`.

### 4.4 Levantar el sistema

```bash
docker compose up --build
```

La primera vez tarda porque descarga e instala las imágenes.

### 4.5 Aplicar migraciones

La base de datos arranca vacía (0 tablas). Con los contenedores ya corriendo:

```bash
docker compose exec api alembic upgrade head
```

Si te saltas este paso, cualquier endpoint falla con errores de tipo `function ... does not exist` o `relation ... does not exist`.

### 4.6 Crear datos de prueba (seed)

Para tener un usuario administrador y dos médicos con los que iniciar sesión:

```bash
docker compose exec api python -m app.scripts.seed
```

Es idempotente: puedes correrlo de nuevo sin duplicar usuarios. Credenciales que crea por defecto (pueden sobreescribirse con las variables `DEMO_*` del `.env`):

| Usuario | Correo | Contraseña |
|---|---|---|
| Admin | `admin@colmed.dev` | `Admin123!` |
| Médico 1 | `medico1@colmed.dev` | `Medico123!` |
| Médico 2 | `medico2@colmed.dev` | `Medico123!` |

Este seed es solo para desarrollo local; cada persona lo corre sobre su propia base, nunca se comparte entre máquinas.

## 5. Servicios y direcciones

| Servicio | Dirección | Qué es |
|---|---|---|
| Web (PHP) | http://localhost:3000 | Interfaz de usuario |
| API (FastAPI) | http://localhost:8000 | Backend |
| Swagger | http://localhost:8000/docs | Documentación y pruebas manuales de la API |
| Salud de la API | http://localhost:8000/health | Debe responder `{"status":"ok"}` |
| Salud de la BD | http://localhost:8000/health/db | Debe responder `{"database":"ok"}` |
| PostgreSQL | `localhost:5432` | Base de datos |

## 6. Comandos de uso diario

```bash
docker compose up --build     # levantar (reconstruye si hubo cambios)
docker compose up             # levantar sin reconstruir
docker compose down           # detener (conserva los datos)
docker compose down -v        # detener y BORRAR la base de datos local
docker compose restart api    # reiniciar solo la API tras cambiar código
docker compose logs api       # ver los mensajes de un servicio
docker compose ps             # ver el estado de los servicios
```

> ⚠️ **`docker compose down -v` borra todas las tablas, funciones y datos de tu base local**, incluido el seed. Si lo ejecutas por error, repite los pasos 4.5 y 4.6 (`alembic upgrade head` y el seed) antes de poder volver a loguearte — `restart` o `up` sin `--build` **no** recrean la base por sí solos.

Para entrar a la consola de PostgreSQL:

```bash
docker compose exec db psql -U sicolmed -d sicolmed
```

Comandos útiles dentro de `psql`: `\l` (bases de datos), `\dt` (tablas), `\q` (salir).

## 7. Base de datos

- Cada persona tiene **su propia base de datos local**, dentro de un volumen de Docker (`pgdata`). No se comparte ni se sube a GitHub.
- Lo que sí se comparte es la **estructura** (migraciones) y los **datos de prueba** (scripts), nunca datos reales de médicos.
- Para conectar una herramienta gráfica (DBeaver, pgAdmin): host `localhost`, puerto `5432`, y la base, el usuario y la contraseña de tu `.env`.
- Las variables `POSTGRES_*` solo se aplican **la primera vez** que se crea el volumen. Si las cambias después, ejecuta `docker compose down -v` para recrear la base.

## 8. Arquitectura

```
Navegador → Web PHP (:3000) → API FastAPI (:8000) → PostgreSQL (:5432)
```

Reparto de responsabilidades:

| Capa | Responsabilidad |
|---|---|
| **PHP (frontend)** | Presentación, sesiones (el token JWT se guarda en la sesión del servidor), validación de formularios para dar mensajes claros, páginas según el rol |
| **FastAPI (backend)** | **Toda la lógica de negocio, la validación real, la autenticación y los permisos.** La API debe validar siempre, aunque el frontend ya lo haya hecho |
| **PostgreSQL** | Almacenamiento de los datos |

Dentro de la red de Docker, PHP llama a la API con `http://api:8000`. Desde el navegador, la API se ve en `http://localhost:8000`.

## 9. Estructura del backend (MVC)

```
app/
├── main.py        # Punto de entrada de FastAPI
├── config.py      # Configuración desde variables de entorno
├── database.py    # Conexión y sesión de SQLAlchemy
├── models/        # Modelo: tablas de la base de datos
├── schemas/       # Esquemas Pydantic: forma de los datos de entrada y salida
├── routers/       # Controlador: rutas y lógica de cada endpoint
├── services/      # Lógica de negocio (opcional, si crece)
└── scripts/       # Scripts de mantenimiento (ej. seed.py)
```

La **Vista** es el frontend, que consume el JSON de la API.

## 10. Flujo de trabajo con Git

Ramas:

- `main`: versión estable, la que se entrega.
- `develop`: integración del trabajo diario.
- `feature/nombre-corto`: una rama por funcionalidad (por ejemplo `feature/login`).

Ciclo de trabajo en GitHub Desktop:

1. **Fetch / Pull origin** al empezar, estando en `develop`.
2. **Current Branch → New Branch**: crea tu rama `feature/...` a partir de `develop`.
3. Haz commits pequeños y frecuentes, con mensajes como `feat: agrega login`, `fix: corrige validación` o `docs: actualiza README`.
4. **Publish branch** y luego **Create Pull Request** hacia `develop`.
5. Otra persona del equipo revisa y aprueba. Después se hace el merge.
6. `main` se actualiza desde `develop`, también por pull request.

Quien abre el pull request no lo aprueba por sí mismo.

## 11. Seguridad

Los repositorios son **públicos**. Por eso:

- **Nunca subas el archivo `.env`**, contraseñas, claves ni datos reales de médicos. Si pasa, cambia esas claves de inmediato: borrar el archivo después no basta porque queda en el historial.
- El `.env.example` solo lleva valores de ejemplo, nunca claves reales. `JWT_SECRET_KEY` es la excepción a "sin cambios" del paso 4.2: cada persona debe generar la suya propia, nunca reutilizar una clave vista en otro `.env` o en un chat.
- Antes de cada commit, revisa en GitHub Desktop la lista de **Changes**.
- Usa solo datos de prueba ficticios. Los correos de prueba del seed usan el dominio `@colmed.dev`; evita dominios reservados como `.local`, `.test`, `.example` o `.invalid`, porque el validador de email del backend los rechaza.

## 12. Problemas comunes

| Síntoma | Causa y solución |
|---|---|
| `empty compose file` | El `docker-compose.yml` está vacío o sin guardar. Pega el contenido y guarda con Ctrl+S |
| `no configuration file provided` | Estás en la carpeta equivocada. Ejecuta los comandos dentro de `SiColMed-Infrastructure` |
| `invalid tag ... must be lowercase` | Los nombres de imagen de Docker van en minúsculas |
| `port is already allocated` | Otro programa usa ese puerto. Ciérralo o cambia el mapeo en el compose (por ejemplo `"5433:5432"`) |
| `password authentication failed` | La base se creó con otra contraseña. Ejecuta `docker compose down -v` y vuelve a levantar |
| `Field required` / `database_url` | La API no recibió la variable. Revisa que el `.env` exista junto al `docker-compose.yml` |
| `ConfiguracionJWTError` / `JWT_SECRET_KEY debe tener al menos 32 caracteres` | Falta `JWT_SECRET_KEY` en el `.env`, o se agregó pero el contenedor `api` no se recreó. Genera la clave (paso 4.3), agrégala al `.env`, y corre `docker compose up -d api` (no `restart`: no relee el `.env`) |
| `function ... does not exist` en cualquier endpoint | La base está vacía o desactualizada. Corre `docker compose exec api alembic upgrade head` |
| `422 ... special-use or reserved name` al hacer login | El correo usa un dominio reservado (`.local`, `.test`, etc.). Usa `@colmed.dev` o similar en los datos de prueba |
| No hay usuarios para loguearse | Falta correr el seed: `docker compose exec api python -m app.scripts.seed` |
| `ImportError` o `Attribute "app" not found` | Un archivo de Python está vacío, sin guardar o con otro contenido. Revísalo y guarda |
| La API no recarga al guardar | Las carpetas de `/mnt/d/` detectan mal los cambios en WSL. Ejecuta `docker compose restart api` |
| VS Code marca imports como "module not found" | El editor no ve las librerías que están en Docker. Crea un `.venv` local, instala `requirements.txt` y elígelo con *Python: Select Interpreter* |
| La web no muestra datos de la API | Revisa `docker compose logs api` y que `/health/db` responda |
```
- Mención de `.scripts/` en el árbol de la sección 9.
- Ampliación de la sección 11 (seguridad) sobre `JWT_SECRET_KEY` y dominios de correo.
- Cuatro filas nuevas en la tabla de problemas comunes (sección 12), con los errores reales que encontraron hoy.
