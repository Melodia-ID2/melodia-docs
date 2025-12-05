# Catalog Service

## Descripción General
El **Catalog Service** es un microservicio desarrollado en **Go** utilizando el framework **Gin**.  
Su función principal es gestionar un **catálogo de canciones y colecciones musicales**, incluyendo todos sus metadatos asociados: título, artista, géneros, regiones, duración, fecha de publicación y estado.  

El servicio centraliza la información musical y expone una API RESTful para crear, listar, modificar, bloquear o eliminar canciones y colecciones.  

**Infraestructura de almacenamiento:**
- **MongoDB:** guarda los metadatos principales de canciones y colecciones.  
- **Cloudflare R2:** almacena los archivos multimedia (`.mp3`, videos).  
- **Cloudinary:** maneja las imágenes de portadas (covers) de álbumes y colecciones.  

Además, incluye endpoints auxiliares para obtener géneros y regiones disponibles, y permite búsquedas y filtrados avanzados por texto, estado, región, rango de fechas, etc.

**Documentación API (Swagger):** [https://catalog-809458893396.us-central1.run.app/swagger/index.html](https://catalog-809458893396.us-central1.run.app/swagger/index.html)

---

## Tecnologías Utilizadas
- **Lenguaje:** Go  
- **Framework:** Gin  
- **Base de datos:** MongoDB (Metadatos), PostgreSQL (Cuentas de usuario)
- **Almacenamiento de objetos:** Cloudflare R2  
- **Gestión de imágenes:** Cloudinary
- **Inteligencia Artificial:** Google Gemini (Generación de metadatos y portadas)
- **Reconocimiento de Audio:** ACRCloud

---

## Arquitectura Interna
El servicio sigue una **arquitectura por capas**, que promueve la separación de responsabilidades y facilita la mantenibilidad y escalabilidad del código.  
Cada capa tiene un propósito definido y se comunica únicamente con la capa inmediatamente inferior:

### Estructura general

```
routes → handlers → services → repositories → database (MongoDB)
```


### Descripción de capas

- **Routes (`routes.go`)**  
  Define las rutas HTTP del servicio y las asocia con los handlers correspondientes.  

- **Handlers (capa de presentación)**  
  Reciben las solicitudes HTTP y desglosan los datos de la request utilizando los **schemas** definidos.  
  Su función principal es validar la estructura de entrada, invocar el servicio correspondiente y construir la respuesta HTTP.  

- **Services (capa de lógica de negocio)**  
  Contiene la lógica principal del dominio.  
  Se encarga de combinar información proveniente de distintas fuentes, aplicar validaciones adicionales, y coordinar la interacción entre los handlers y los repositories.

- **Repositories (capa de acceso a datos)**  
  Encapsula toda la interacción con la base de datos MongoDB.  
  Utiliza los **models** definidos para mapear los documentos almacenados y realizar operaciones CRUD (Create, Read, Update, Delete).  
  Esta capa abstrae los detalles de persistencia, permitiendo reemplazar MongoDB por otro motor sin modificar la lógica de negocio.

---


## Infraestructura y Despliegue

El servicio corre en **Google Cloud Platform (GCP)** y se despliega automáticamente con **GitHub Actions** hacia **Cloud Run** (contenedores administrados). El workflow de CI/CD (deploy-cloudrun.yml) se ejecuta al hacer push a la rama main, construye la imagen Docker, la publica en **Artifact Registry** y despliega a Cloud Run. Las variables de entorno se inyectan desde **GitHub Secrets** a través de parámetros de despliegue.

### Variables de entorno (referencia)
- **App**: `HOST`, `PORT`, `ENVIRONMENT`
- **Database (MongoDB)**: `DATABASE_URI`, `DATABASE_NAME`
- **Database (PostgreSQL)**: `DATABASE_SQL_HOST`, `DATABASE_SQL_NAME`, `DATABASE_SQL_USER`, `DATABASE_SQL_PASSWORD`, `DATABASE_SQL_PORT`
- **R2 Storage**: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`
- **Cloudinary**: `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`, `CLOUDINARY_URL`
- **Auth**: `AUTH_SECRET`, `AUTH_ISSUER`, `AUTH_ALGORITHM`
- **AI & Recognition**: `GEMINI_API_KEY`, `ACRCLOUD_HOST`, `ACRCLOUD_ACCESS_KEY`, `ACRCLOUD_SECRET_KEY`

Ejemplo (.env.example):
```
# App
HOST=0.0.0.0
PORT=8080
ENVIRONMENT=development

# Database (MongoDB)
DATABASE_URI=mongodb+srv://user:pass@cluster.mongodb.net/
DATABASE_NAME=melodia_catalog

# R2 Storage
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key
R2_SECRET_ACCESS_KEY=your_secret_key
R2_BUCKET=melodia-tracks

# Cloudinary
CLOUDINARY_CLOUD_NAME=cloud_name
CLOUDINARY_API_KEY=api_key
CLOUDINARY_API_SECRET=api_secret
CLOUDINARY_URL=cloudinary_url

# JWT Authentication
AUTH_SECRET=dev-secret
AUTH_ISSUER=auth-service
AUTH_ALGORITHM=HS256

# PostgreSQL (for user accounts)
DATABASE_SQL_HOST=localhost
DATABASE_SQL_NAME=melodia
DATABASE_SQL_USER=postgres
DATABASE_SQL_PASSWORD=postgres
DATABASE_SQL_PORT=5432

# AI & Recognition
GEMINI_API_KEY=your_gemini_key
ACRCLOUD_HOST=identify-eu-west-1.acrcloud.com
ACRCLOUD_ACCESS_KEY=your_acr_key
ACRCLOUD_SECRET_KEY=your_acr_secret
```

### Servicios en la nube
- **Cloud Run**: hospeda el contenedor (—allow-unauthenticated, control de auth a nivel aplicación).
- **MongoDB Atlas**: base de datos gestionada.
- **Cloudflare R2**: objetos multimedia (.mp3, videos).
- **Cloudinary**: portadas con optimización/transformación.


**Resumen del flujo de despliegue:**

```
GitHub (push main)
↓
CI/CD Pipeline
↓
Docker Build
↓
Catalog Service (Cloud Run )

```

---


## Modelos y Esquemas de Datos

### Song
Representa una canción con sus metadatos principales.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `id` | string | Identificador único (ObjectID de MongoDB). |
| `title` | string | Título de la canción. |
| `track_number` | integer | Número de pista dentro de un álbum o colección. |
| `genres` | array[string] | Lista de géneros asociados. |
| `regions` | array[string] | Regiones donde está disponible. |
| `duration_ms` | integer | Duración en milisegundos. |
| `has_video` | boolean | Indica si la canción tiene video asociado. |
| `anticipated` | boolean | Indica si es un lanzamiento anticipado (pre-save/pre-release). |
| `state` | string | Estado (por ejemplo: `Publicado`, `Bloqueado`). |
| `cover_url` | string (URL) | Imagen de portada almacenada en Cloudinary. |
| `published_at` | string (RFC3339) | Fecha de publicación. |
| `created_at` | string (RFC3339) | Fecha de creación. |
| `updated_at` | string (RFC3339) | Última actualización. |

---

### Collection
Representa un conjunto de canciones y sus metadatos (Álbum, Single, EP).

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `id` | string | Identificador único. |
| `title` | string | Nombre de la colección. |
| `type` | string | Tipo de colección (`album`, `single`, `compilation`). |
| `description` | string | Descripción textual de la colección. |
| `cover_url` | string (URL) | Imagen de portada en Cloudinary. |
| `songs` | array[string] | IDs de canciones incluidas. |
| `artists_ids` | array[string] | IDs de los artistas principales. |
| `state` | string | Estado de publicación. |
| `regions` | array[string] | Regiones donde está disponible. |
| `scheduled_date` | string (RFC3339) | Fecha programada para publicación futura. |
| `created_at` | string | Fecha de creación. |
| `updated_at` | string | Última modificación. |

---

### Genre
Representa un género musical.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `code` | string | Código identificador del género (ej. `POP`, `ROCK`). |
| `name` | string | Nombre descriptivo del género. |

---

### Region
Representa una región o país donde una canción o colección está disponible.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `code` | string | Código ISO del país o región (ej. `AR`, `US`). |
| `name` | string | Nombre de la región. |
| `level` | string | Nivel de la región (ej. `country`, `continent`). |

---

### ArtistPick
Representa la "selección del artista" para mostrar en su perfil.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `artist_id` | string | ID del artista. |
| `collection_id` | string | ID de la colección destacada. |
| `message` | string | Mensaje opcional del artista. |

---

### ErrorResponse (RFC 7807)
El servicio implementa el formato de error definido en el **RFC 7807 — Problem Details for HTTP APIs**, que estandariza la forma en que las APIs comunican errores.

Cada respuesta de error devuelve un objeto JSON con los siguientes campos:

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `type` | string (URI) | Identificador del tipo de error (puede ser una URL con más información). |
| `title` | string | Título breve que resume el problema. |
| `status` | integer | Código de estado HTTP correspondiente. |
| `detail` | string | Descripción detallada del error. |
| `instance` | string | URI de la instancia o recurso donde ocurrió el error. |

**Ejemplo:**
```json
{
  "type": "about:blank",
  "title": "Bad request error",
  "status": 400,
  "detail": "Invalid genre code: LATIN",
  "instance": "/songs"
}
```

---

## Endpoints

### Géneros y Regiones
- **GET `/available-genres`**  
  Devuelve la lista de géneros musicales disponibles.  

- **GET `/available-regions`**  
  Devuelve la lista de regiones disponibles (códigos y nombres).

- **GET `/regions`**
  Devuelve regiones organizadas por nivel jerárquico.

---

### Canciones (`/songs`)
- **GET `/songs`**  
  Lista canciones con filtros de texto, estado, región, presencia de video, fechas de publicación, y ordenamiento.  
- **POST `/songs`**  
  Crea una nueva canción (metadatos + archivo). (Requiere rol `artist` o `admin`)
- **GET `/songs/{id}`**  
  Obtiene los metadatos de una canción por su ID.  
- **PUT `/songs/{id}`**  
  Actualiza los campos de una canción (actualización parcial). (Requiere rol `artist` o `admin`)
- **DELETE `/songs/{id}`**  
  Elimina una canción por su ID.  
- **GET `/songs/{id}/playback`**
  Obtiene la URL firmada para reproducir la canción.
- **GET `/songs/similar-artists/{artist_id}`**
  Devuelve artistas similares basados en el catálogo.
- **POST `/songs/{id}/video`**
  Sube y asocia un video a una canción existente. (Requiere rol `artist` o `admin`)
- **POST `/songs/suggest-metadata`**
  Sugiere metadatos para una canción basada en el archivo de audio (AI). (Requiere rol `artist` o `admin`)

#### Administración de Canciones
- **PUT `/songs/{id}/block`**  
  Bloquea una canción (cambia su estado a “bloqueada”). (Admin only)
- **PUT `/songs/{id}/unblock`**  
  Desbloquea una canción (cambia su estado a “activa”). (Admin only)

---

### Colecciones (`/collection`)
- **GET `/collection`**  
  Lista las colecciones disponibles con filtros de búsqueda, estado, región, fechas, y ordenamiento.  
- **GET `/collection/{id}`**  
  Devuelve los metadatos de una colección específica.  
- **POST `/collection`**
  Crea una nueva colección. (Requiere rol `artist` o `admin`)
- **PUT `/collection/{id}`**  
  Actualiza los campos de una colección. (Requiere rol `artist` o `admin`)
- **DELETE `/collection/{id}`**  
  Elimina una colección por ID.  
- **GET `/collection/artist/{artist_id}`**  
  Lista las colecciones que incluyen un artista dado, con filtros adicionales.
- **GET `/collection/{id}/songs`**
  Devuelve las canciones que pertenecen a una colección.

#### Publicación y Programación
- **POST `/collection/{id}/publish`**
  Publica una colección inmediatamente.
- **POST `/collection/{id}/schedule`**
  Programa la publicación de una colección para una fecha futura.
- **POST `/collection/{id}/try-publish`**
  Intenta publicar una colección programada (usado por el orquestador).
- **POST `/collection/{id}/cover`**
  Sube la imagen de portada de la colección.

#### Feeds de Lanzamientos
- **POST `/collection/new-releases-from-artists`**
  Obtiene nuevos lanzamientos de una lista de artistas seguidos.
- **POST `/collection/upcoming-releases-from-artists`**
  Obtiene próximos lanzamientos (programados) de una lista de artistas.

#### Administración de Colecciones
- **PUT `/collection/{id}/block`**  
  Bloquea una colección (estado: bloqueada). (Admin only)
- **PUT `/collection/{id}/unblock`**  
  Desbloquea una colección (estado: activa). (Admin only)

---

### Selección del Artista (`/artist-picks`)
- **GET `/artist-picks/{artist_id}`**
  Obtiene la selección actual de un artista.
- **POST `/artist-picks/{artist_id}`**
  Establece o actualiza la selección del artista. (Artist only)
- **DELETE `/artist-picks/{artist_id}`**
  Elimina la selección del artista. (Artist only)

---

### Inteligencia Artificial y Reconocimiento
- **POST `/ai/generate-cover`**
  Genera una imagen de portada utilizando IA generativa.
- **POST `/recognize`**
  Identifica una canción a partir de un fragmento de audio (Audio Fingerprinting).

---

### Catálogo General
- **GET `/catalog`**  
  Devuelve el catálogo completo combinando canciones y colecciones con filtros y paginación.

---

## Seguridad
El servicio utiliza JWT (JSON Web Tokens) para la autenticación y autorización.
- **AuthMiddleware**: Verifica la validez del token y extrae la información del usuario (ID, roles).
- **Roles**:
    - `user`: Acceso de lectura básico.
    - `artist`: Permiso para crear y gestionar su propio contenido.
    - `admin`: Permiso total para moderación y gestión global.

## Tests
El proyecto incluye tests unitarios y de integración.
- **Unitarios**: Prueban la lógica de servicios y handlers de forma aislada.
- **Integración**: Prueban el flujo completo incluyendo base de datos y llamadas HTTP (usando `httptest`).
