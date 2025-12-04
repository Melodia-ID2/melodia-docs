# Users Service

## Descripción General
El **Users Service** administra los perfiles de oyentes y artistas: creación/actualización de perfiles, manejo de fotos y enlaces sociales, seguimiento entre usuarios, favoritos de artistas, notificaciones push y preferencias del usuario. Se complementa con el **Auth Service** para la identidad JWT y trabaja junto al servicio de búsqueda y al centro de notificaciones.

Diseñado como un microservicio implementado con FASTAPI orientado a usuarios, expone endpoints para administradores (listado, rol, borrado), para usuarios autenticados (perfil, fotos, seguidores, preferencias, tokens de dispositivo) y para eventos externos (notificaciones creadas desde otros servicios protegidas con `SERVICE_TOKEN`).

## Tecnologías utilizadas
- **Lenguaje:** Python 3
- **Framework:** FastAPI + SQLModel sobre PostgreSQL
- **Notificaciones:** Firebase Cloud Messaging (vía `firebase_admin`), servicio propio `fcm_service`
- **Almacenamiento de archivos:** Cloudinary para subir fotos de perfil y álbumes de artistas
- **Comunicación interna:** HTTPX hacia el orquestador y servicio de búsqueda (`search_service`)
- **Infraestructura:** Para desarrollo y testeo local se emplean archivos Docker Compose (`compose.yml`, `compose.test.yml`). En producción, la infraestructura está basada en **Google Cloud Platform (GCP)**, desplegando el servicio en **Cloud Run** mediante pipelines automatizados con **GitHub Actions**. Las variables sensibles y de configuración se gestionan con `secrets` y `vars` en GitHub y GCP.

## Arquitectura Interna

- **Routers (`app/api/v1/routers`)**: agrupan rutas por dominio. `users_router` maneja `/me`, creación y edición de perfiles, follow/unfollow y fotos; `artist_router` agrupa fotos, links y listado de artistas; `notifications_center_router` sirve la cola de notificaciones y acciones sobre las mismas; `device_token_router` gestiona tokens FCM; `users_preferences_router` cubre preferencias de historial, autoplay, notificaciones y artistas silenciados; `admin_router` expone herramientas para administradores; `system_router` ofrece `/health`.
- **Servicios (`app/services/*.py`)**: cada router delega en un servicio especializado: `users_service` (perfil, seguimientos y preferencias), `artist_service` (fotos/enlaces de artista), `notification_service` y `event_service` (gestión de notificaciones y eventos externos), `device_token_service` (registro/desactivación de tokens), `admin_service` (información detallada y cambios de rol) y `search_service` (indexación asíncrona vía orquestador).
- **Repositorios (`app/repositories/*.py`)**: encapsulan consultas a PostgreSQL (`users_repository`, `artist_repository`, `notification_repository`, `device_token_repository`, `muted_artists_repository`, `credentials_repository`, `admin_repository`).
- **Core (`app/core/*.py`)**: maneja configuración (`config.py`), inicialización de la base de datos (`database.py`), seguridad JWT (`security.py`, que reutiliza los tokens de `Auth Service`) y la inicialización de Firebase (`firebase.py`) para FCM.
- **Middlewares y errores:** `app.errors.middleware` captura excepciones propias y devuelve `ErrorResponse` consistentes `app.schemas.error`.

## Módulos de funcionalidad

- **Perfiles de usuarios:** `app.models.userprofile.UserProfile` guarda datos personales, foto, bio y contadores de seguidores. `users_service` permite crear, actualizar, leer perfiles propios y públicos, así como subir fotos almacenadas en Cloudinary (`cloudinary.uploader`).
- **Seguimientos:** `users_service` actualiza `UserFollows`, dispara notificaciones y mantiene los contadores de seguidores/seguidos. También expone endpoints para listar seguidores/seguidos por rol.
- **Artistas:** `artist_service` gestiona `ArtistPhoto` y `SocialLink`, permite reordenar fotos, eliminar y obtener listados paginados de artistas.
- **Notificaciones push:** `notification_service` decide si enviar notificaciones y registrarlas en `Notification` (bitmask en `UserAccount.preferences`). Usa `device_token_repository` y `muted_artists_repository`, y envía mensajes FCM a través de `fcm_service`. `event_service` recibe eventos externos (protegidos con `SERVICE_TOKEN`) para crear notificaciones masivas.
- **Preferencias:** `notification_flags.NotificationPreferences` representa las preferencias mediante una máscara de bits. Los endpoints `/preferences/*` permiten alternar historial, autoplay, notificaciones específicas y artistas silenciados.
- **Tokens de dispositivo:** `device_token_service` guarda (`DeviceToken`), lista, desactiva tokens y ofrece logout global de dispositivos.
- **Búsqueda:** `search_service` programa tareas en el orquestador para indexar o eliminar usuarios en el servicio de búsqueda. `admin_service` interactúa con esta capa al cambiar roles.

## Modelado y persistencia
- `UserAccount`: metadata de autenticación (rol, estado, país, preferencias). El bitmask para notificaciones/autoplay se define en `app.constants.notification_flags`.
- `UserProfile`: perfil extendido (nombre, bio, foto, contadores). Relacionado por FK con `useraccount` y usado por los endpoints `/me`, `/users/{id}`.
- `UserFollows`, `ArtistPhoto`, `SocialLink`: modelan relaciones sociales y datos multimedia.
- `Notification`, `DeviceToken`, `MutedArtist`: permiten históricas de notificaciones, tokens FCM y lista de artistas silenciados.
- Repositorios (`users_repository`, etc.) mantienen las operaciones CRUD y consultas específicas.

## Infraestructura y despliegue

Para desarrollo y testeo local se utiliza Docker Compose:

- Primera ejecución: `docker compose up users --build`
- Ejecuciones posteriores: `docker compose up users`

Los tests se ejecutan con `compose.test.yml` (`users-test-db`, `users-integral-tests`). Copiar `.env.example` a `.env` para configurar variables sensibles (bases de datos, `SERVICE_TOKEN`, claves Cloudinary/Firebase). Swagger corre en `http://localhost:8002/docs`.

**Despliegue en producción:**
El servicio se despliega en **Google Cloud Platform (GCP)** usando **Cloud Run**. El pipeline de CI/CD está automatizado con **GitHub Actions**, que construye la imagen, la sube a Artifact Registry y la despliega en Cloud Run con las variables de entorno necesarias (ver `.github/workflows/deploy-cloudrun.yml`).

Los endpoints protegidos por notificaciones requieren el encabezado `x-service-token` con el valor `SERVICE_TOKEN` para evitar accesos externos no autorizados.