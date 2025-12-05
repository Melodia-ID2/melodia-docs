# Playback Service

## Descripción General
El **Playback Service** es un microservicio desarrollado en **Go** utilizando el framework **Gin**.  
Su función principal es gestionar la **experiencia de reproducción** del usuario, lo que incluye:
1.  **Gestión de la Cola (Queue):** Mantener el estado de la lista de reproducción actual, posición, y orden de las canciones.
2.  **Historial de Reproducción:** Registrar qué canciones escuchó el usuario y desde dónde (contexto).
3.  **Entrega de Contenido:** Generar URLs firmadas (presigned URLs) para el streaming desde Cloudflare R2.

Este servicio actúa como el "cerebro" del reproductor, asegurando que el estado se mantenga sincronizado entre dispositivos y sesiones.

**Documentación API (Swagger):** [https://playback-809458893396.us-central1.run.app/swagger/index.html](https://playback-809458893396.us-central1.run.app/swagger/index.html)

---

## Tecnologías Utilizadas
- **Lenguaje:** Go
- **Framework:** Gin
- **Base de datos:** MongoDB (cola, historial, canciones) y PostgreSQL (usuarios)
- **Almacenamiento de objetos:** Cloudflare R2 (archivos multimedia, URLs presignadas)
- **Autenticación:** JWT (validación vía middleware, integración con Auth Service)
- **Observabilidad:** New Relic (tracing y monitoreo)
- **Infraestructura:** Docker Compose para pruebas/desarrollo local; despliegue productivo en **Google Cloud Platform (GCP)** usando **Cloud Run** y pipelines automatizados con **GitHub Actions**.

---

## Arquitectura Interna

El servicio sigue una arquitectura modular y desacoplada:

- **cmd/main.go:** punto de entrada, inicializa configuración, New Relic y el router Gin.
- **internal/server:** carga variables de entorno, configura conexiones a MongoDB/PostgreSQL y R2, y expone el router principal.
- **internal/routes:** define los endpoints REST para la cola (`/queue`), historial (`/history`) y streaming de canciones/videos (`/songs`).
- **internal/handlers:** implementa la lógica de cada endpoint (cola, historial, canciones), incluyendo la generación de URLs presignadas y validaciones de acceso.
- **internal/service:** contiene la lógica de negocio para manipular la cola, historial y streaming.
- **internal/storage:** abstrae la interacción con Cloudflare R2 para subir, eliminar y obtener URLs presignadas.
- **internal/security:** valida JWT y roles, integrando con el Auth Service.
- **internal/db:** inicializa y gestiona las conexiones a MongoDB y PostgreSQL.

---

## Observabilidad y monitoreo
El servicio integra **New Relic** para trazabilidad y monitoreo de rendimiento, permitiendo detectar cuellos de botella y errores en tiempo real.

---

## Infraestructura y Despliegue

El servicio se despliega en **Google Cloud Run** mediante **GitHub Actions**.

### Variables de entorno (referencia)
- **App**: `PORT`
- **Database**: `DATABASE_URI`, `DATABASE_NAME`
- **R2 Storage**: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`
- **Auth**: `AUTH_SECRET`, `AUTH_ISSUER`, `AUTH_ALGORITHM`
- **Monitoring**: `NEW_RELIC_LICENSE_KEY`, `NEW_RELIC_APP_NAME`

---

## Modelos y Esquemas de Datos

### Queue (Cola de Reproducción)
Representa el estado actual del reproductor de un usuario.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `id` | string | ID único de la cola. |
| `user_id` | string | ID del usuario propietario. |
| `items` | array[string] | Lista de IDs de canciones en la cola. |
| `sources` | array[string] | Origen de cada canción (ej. "playlist:123", "search"). |
| `current_index` | integer | Índice de la canción que se está reproduciendo actualmente. |
| `current_position_ms` | integer | Progreso de reproducción en milisegundos (para reanudar). |

### History (Historial)
Registro de una reproducción completada o significativa.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `user_id` | string | ID del usuario. |
| `song_id` | string | ID de la canción escuchada. |
| `listened_at` | datetime | Fecha y hora de la escucha. |
| `listened_from_id` | string | ID del contexto (playlist, álbum). |
| `listened_from_name` | string | Nombre del contexto. |
| `listened_for_ms` | integer | Tiempo total escuchado en milisegundos. |

---

## Endpoints

### Gestión de la Cola (`/queue`)
Todos los endpoints requieren autenticación y que el `user_id` coincida con el token (o ser admin).

- **GET `/queue/{user_id}`**  
  Obtiene el estado completo de la cola del usuario.

- **GET `/queue/{user_id}/current`**  
  Obtiene el ítem actual y una ventana de ítems próximos/anteriores (optimizado para el player).

- **POST `/queue/{user_id}/load`**  
  Reemplaza la cola actual con una nueva lista de canciones.
  *Body:* `{ "items": ["id1", "id2"], "index": 0, "source": "album:xyz" }`

- **POST `/queue/{user_id}/enqueue`**  
  Agrega canciones al final de la cola.
  *Body:* `{ "items": ["id3"], "source": "search" }`

- **POST `/queue/{user_id}/enqueue_front`**  
  Agrega canciones justo después de la actual (Play Next).

- **POST `/queue/{user_id}/next`**  
  Avanza al siguiente tema en la cola.

- **POST `/queue/{user_id}/prev`**  
  Retrocede al tema anterior.

- **POST `/queue/{user_id}/set_current`**  
  Salta a un índice específico o actualiza la posición de reproducción.
  *Body:* `{ "index": 5, "position_ms": 30000 }`

- **POST `/queue/{user_id}/remove`**  
  Elimina un ítem de la cola por su índice.

- **POST `/queue/{user_id}/clear`**  
  Limpia la cola completa.

- **POST `/queue/{user_id}/reorder`**  
  Reordena los ítems de la cola (Drag & Drop).

---

### Historial (`/history`)

- **GET `/history`**  
  Obtiene el historial de reproducción del usuario (paginado).

- **POST `/history/song`**  
  Registra una canción como escuchada.
  *Body:* `{ "song_id": "...", "listened_for_ms": 120000 }`

- **DELETE `/history`**  
  Borra el historial del usuario.

---

### Canciones y Playback (`/songs`)

- **GET `/songs/{id}/video`**  
  Obtiene la URL firmada para reproducir el video asociado a una canción (si existe).

---

## Seguridad
- **JWT**: Se valida el token en cada petición.
- **Ownership**: Los endpoints de `/queue/{user_id}` verifican que el usuario autenticado sea el dueño de la cola (`RequireOwnerOrAdmin`).