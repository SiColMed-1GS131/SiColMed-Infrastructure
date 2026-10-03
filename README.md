# SiColMed – Entorno de desarrollo

Sistema de manejo de certificaciones y recertificaciones de médicos en Panamá (Desarrollo de Software III).

Este repositorio (`SiColMed-Deploy`) contiene lo necesario para levantar **todo el sistema con un solo comando**: base de datos, API y web.

## 1. Repositorios

| Repositorio | Contenido |
|---|---|
| `SiColMed-BackEnd` | API en Python 3.12 + FastAPI, con su Dockerfile |
| `SiColMed-FrontEnd` | Web en PHP, HTML, CSS y JavaScript, con su Dockerfile |
| `SiColMed-Deploy` | `docker-compose.yml`, `.env.example` y esta documentación |

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

No necesitas instalar Python, PHP, PostgreSQL ni XAMPP: todo corre dentro de Docker.

## 4. Instalación

### 4.1 Clonar los tres repositorios

Crea una carpeta (por ejemplo `SiColMed`) y clona dentro los tres repos con GitHub Desktop (**File → Clone repository**).

Las carpetas deben quedar **como hermanas y con estos nombres exactos**, porque el compose las referencia por ruta:

```
SiColMed/
├── SiColMed-BackEnd/
├── SiColMed-FrontEnd/
└── SiColMed-Deploy/
```

### 4.2 Crear el archivo `.env`

Dentro de `SiColMed-Deploy`:

```bash
cp .env.example .env          # Linux, Mac o WSL
copy .env.example .env        # CMD de Windows
```

El `.env` es **personal y nunca se sube a GitHub**. Los valores del `.env.example` sirven para desarrollo local sin cambios.

### 4.3 Levantar el sistema

```bash
docker compose up --build
```

La primera vez tarda porque descarga e instala las imágenes.

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
└── services/      # Lógica de negocio (opcional, si crece)
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
- El `.env.example` solo lleva valores de ejemplo, nunca claves reales.
- Antes de cada commit, revisa en GitHub Desktop la lista de **Changes**.
- Usa solo datos de prueba ficticios.

## 12. Problemas comunes

| Síntoma | Causa y solución |
|---|---|
| `empty compose file` | El `docker-compose.yml` está vacío o sin guardar. Pega el contenido y guarda con Ctrl+S |
| `no configuration file provided` | Estás en la carpeta equivocada. Ejecuta los comandos dentro de `SiColMed-Deploy` |
| `invalid tag ... must be lowercase` | Los nombres de imagen de Docker van en minúsculas |
| `port is already allocated` | Otro programa usa ese puerto. Ciérralo o cambia el mapeo en el compose (por ejemplo `"5433:5432"`) |
| `password authentication failed` | La base se creó con otra contraseña. Ejecuta `docker compose down -v` y vuelve a levantar |
| `Field required` / `database_url` | La API no recibió la variable. Revisa que el `.env` exista junto al `docker-compose.yml` |
| `ImportError` o `Attribute "app" not found` | Un archivo de Python está vacío, sin guardar o con otro contenido. Revísalo y guarda |
| La API no recarga al guardar | Las carpetas de `/mnt/d/` detectan mal los cambios en WSL. Ejecuta `docker compose restart api` |
| VS Code marca imports como "module not found" | El editor no ve las librerías que están en Docker. Crea un `.venv` local, instala `requirements.txt` y elígelo con *Python: Select Interpreter* |
| La web no muestra datos de la API | Revisa `docker compose logs api` y que `/health/db` responda |

## 13. Documentación adicional

La carpeta `docs/` guardará los diagramas del proyecto (modelo entidad-relación, arquitectura y casos de uso), hechos con draw.io o PlantUML.
