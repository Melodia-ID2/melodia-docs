# Auth Service

## Descripción General
El **Auth Service** es el componente encargado de gestionar el ciclo de vida de las cuentas en la plataforma: registro, inicio de sesión, renovación de tokens, verificación por correo, restablecimiento de contraseñas, administración de países y auditoría. Expone una API REST construida con **FastAPI** y SQLModel y sirve tanto a clientes móviles como a otros microservicios (por ejemplo `melodia-users-service`) que requieren seguridad basada en JWT.

Además de las operaciones de usuarios finales, ofrece un namespace de administración (roles, estado de cuentas y logs de auditoría) y endpoints especializados para enlaces con **Google OAuth**, envío de correos y generación de JWTs de administrador para pruebas locales.

## Tecnologías utilizadas
- **Lenguaje:** Python 3
- **Framework web:** FastAPI + SQLModel (sobre PostgreSQL)
- **Autenticación:** JOSE (JWT), passlib para bcrypt, tokens de acceso/refresh, cabeceras Bearer
- **Correo electrónico:** SMTP + Jinja2 templates (`app/resources/templates`) habilitados por `app.services.email`
- **OAuth:** integración con Google mediante `app.services.google_service`
- **Infraestructura:** Docker Compose (`compose.yml`, `compose.test.yml`), `pydantic-settings`, `sqlmodel`, `alembic`-like `init_db`

## Arquitectura Interna
El servicio sigue una arquitectura en capas clara:

- **Routers (`app/api/v1/routers/*.py`)**: definen los endpoints públicos para autenticación (`auth_router`), correo (`email_router`), Google OAuth (`google_router`), administración (`admin_router`), países (`region_router`) y salud (`system_router`).
- **Controllers (`app/controllers/*.py`)**: orquestan cada caso de uso invocando los servicios correspondientes y centralizando la firma de entrada/salida.
- **Services (`app/services/*.py`)**: contienen la lógica de negocio (registro/login, manejo de tokens, auditoría, Google OAuth, envío de correos, validaciones de contraseña, generación de logs). El controlador de Google delega en `auth_service` para reutilizar flujos.
- **Repositories (`app/repositories/*.py`)**: encapsulan la persistencia con PostgreSQL a través de SQLModel (`user_repository`, `credentials_repository`, `google_repository`, `admin_repository`).
- **Core (`app/core/*.py`)**: configura la base de datos (`database.py`), carga las variables del entorno (`config.py`) y define el middleware de seguridad (`security.py`) responsable de validar JWTs, roles y generación de accesos/refresh tokens.
- **Models (`app/models/*.py`)**: reflejan las tablas y esquemas, desde usuarios (`UserAccount`, `UserCredential`) hasta tokens (`RefreshToken`, `PasswordResetToken`, `GoogleTokens`), verificación de email (`EmailVerification`), auditoría (`AuditLog`, `AuditEventType`, `AuditScope`), países (`regions.Country`) y administradores (`AdminAccount`).
- **Schemas (`app/schemas/*.py`)**: definen los DTOs para requests/responses (usuarios, tokens, auditoría, emails, errores). `app.errors` y `app.schemas.error` uniformizan las respuestas HTTP en casos de error.
- **Middleware de errores (`app/errors/middleware.py`)**: intercepta excepciones personalizadas y devuelve `ErrorResponse` con código y detalle.

### Endpoints clave
- **`/register`, `/login`, `/refresh`, `/logout`** (`auth_router`): registro local, inicio de sesión, refresh y cierre de sesión (revocación de refresh tokens). También se manejan cambios de contraseña y verificación de email.
- **`/request-password-reset`, `/reset-password`, `/resend-password-reset`**: generan y consumen tokens de restablecimiento con envío de correos (`app.services.email`).
- **`/me`, `/add-password`**: consumen el JWT actual para devolver datos de perfil y permitir agregar contraseña local a cuentas Google.
- **`/google/*`**: endpoints para registrar/login/link/unlink con Google (almacenan tokens en `GoogleTokens` y sincronizan con `UserCredential`).
- **`/admin/*`**: registra/loguea administradores, actualiza el estado de cuentas de usuarios y crea logs de auditoría (`AuditLog`).
- **`/countries/*`**: expone la lista de países disponibles (`app.models.regions`) y permite a administradores ajustar el país de una cuenta específica.
- **`/health`**: confirma que el servicio está arriba.

## Seguridad y flujo de tokens
`app.core.security` provee utilidades para:
1. Hashear y verificar contraseñas (bcrypt).
2. Crear tokens JWT de acceso y refresh con campos `exp`, `iat`, `iss`, `jti`.
3. Validar el rol de administrador (`require_admin`) y obtener el `user_id` del payload (`get_current_user_id`).
4. Proteger todos los endpoints que consumen información sensible mediante dependencias (`Depends`).

El renovado de tokens crea entradas en `RefreshToken`, mientras que `auth_service.logout` las marca como revocadas para evitar reutilización.

## Infraestructura y despliegue
Para desarrollo y testeo local se emplean los archivos Docker Compose (`compose.yml`, `compose.test.yml`), permitiendo levantar el servicio y sus dependencias en contenedores.

- Primera vez: `docker compose up auth --build`
- Subsecuentes: `docker compose up auth`

Para pruebas unitarias e integrales se usa `compose.test.yml` con comandos similares (`auth-tests`, `auth-test-db`, `auth-integral-tests`).
El servicio expone Swagger en `http://localhost:8001/docs` y requiere configurar variables como `AUTH_SECRET`, `DATABASE_*`, `SMTP_*`, `GOOGLE_*` según `.env.example`.

**Despliegue en producción:**
La infraestructura productiva se basa en **Google Cloud Platform (GCP)**, utilizando **Cloud Run** para el despliegue automatizado. El pipeline de CI/CD está gestionado por **GitHub Actions**, que construye la imagen, la sube a Artifact Registry y la despliega en Cloud Run con las variables de entorno necesarias (ver `.github/workflows/deploy-cloudrun.yml`).

Las variables sensibles y de configuración se gestionan mediante `secrets` y `vars` en GitHub y GCP. El servicio se expone públicamente y se integra con otros microservicios mediante endpoints HTTP y JWT.