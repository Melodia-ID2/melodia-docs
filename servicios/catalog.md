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

---

## Tecnologías Utilizadas
- **Lenguaje:** Go  
- **Framework:** Gin  
- **Base de datos:** MongoDB  
- **Almacenamiento de objetos:** Cloudflare R2  
- **Gestión de imágenes:** Cloudinary  

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
- DATABASE_URI, DATABASE_NAME
- R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET
- CLOUDINARY_CLOUD_NAME, CLOUDINARY_API_KEY, CLOUDINARY_API_SECRET, CLOUDINARY_URL
- AUTH_SECRET, AUTH_ISSUER, AUTH_ALGORITHM
- SEARCH_SERVICE_URL
- ENVIRONMENT

Ejemplo (.env.example):
```
DATABASE_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/
DATABASE_NAME=catalog
R2_ACCOUNT_ID=<account-id>
R2_ACCESS_KEY_ID=<access-key>
R2_SECRET_ACCESS_KEY=<secret-key>
R2_BUCKET=catalog-audio
CLOUDINARY_CLOUD_NAME=<cloud-name>
CLOUDINARY_API_KEY=<api-key>
CLOUDINARY_API_SECRET=<api-secret>
CLOUDINARY_URL=cloudinary://<api-key>:<api-secret>@<cloud-name>
AUTH_SECRET=<secret>
AUTH_ISSUER=catalog-service
AUTH_ALGORITHM=HS256
SEARCH_SERVICE_URL=https://search-service.example.com
ENVIRONMENT=production
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
| `state` | string | Estado (por ejemplo: `Publicado`, `Bloqueado`). |
| `cover` | string (URL) | Imagen de portada almacenada en Cloudinary. |
| `published_at` | string (RFC3339) | Fecha de publicación. |
| `created_at` | string (RFC3339) | Fecha de creación. |
| `updated_at` | string (RFC3339) | Última actualización. |

---

### SongUpdateRequest
Objeto utilizado para actualizar una canción parcialmente.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `regions` | array[string] | Lista actualizada de regiones. |
| `track_number` | integer | Nuevo número de pista. |

---

### Collection
Representa un conjunto de canciones  y sus metadatos.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `id` | string | Identificador único. |
| `title` | string | Nombre de la colección. |
| `description` | string | Descripción textual de la colección. |
| `cover` | string (URL) | Imagen de portada en Cloudinary. |
| `songs` | array[string] | IDs de canciones incluidas. |
| `artist_id` | string | ID del artista principal. |
| `state` | string | Estado de publicación. |
| `regions` | array[string] | Regiones donde está disponible. |
| `created_at` | string | Fecha de creación. |
| `updated_at` | string | Última modificación. |

---

### Genre
Representa un género musical.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `code` | string | Código identificador del género. |
| `name` | string | Nombre descriptivo del género. |

---

### Region
Representa una región o país donde una canción o colección está disponible.

| Campo | Tipo | Descripción |
|-------|------|--------------|
| `code` | string | Código ISO del país o región. |
| `name` | string | Nombre de la región. |

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
  "type": "https://example.com/problems/invalid-parameter",
  "title": "Invalid request parameter",
  "status": 400,
  "detail": "The field 'title' is required.",
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

---

### Canciones (`/songs`)
- **GET `/songs`**  
  Lista canciones con filtros de texto, estado, región, presencia de video, fechas de publicación, y ordenamiento.  
- **POST `/songs`**  
  Crea una nueva canción (metadatos + archivo).  
- **GET `/songs/{id}`**  
  Obtiene los metadatos de una canción por su ID.  
- **PUT `/songs/{id}`**  
  Actualiza los campos de una canción (actualización parcial).  
- **DELETE `/songs/{id}`**  
  Elimina una canción por su ID.  
- **PUT `/songs/{id}/block`**  
  Bloquea una canción (cambia su estado a “bloqueada”).  
- **PUT `/songs/{id}/unblock`**  
  Desbloquea una canción (cambia su estado a “activa”).

---

### Colecciones (`/collection`)
- **GET `/collection`**  
  Lista las colecciones disponibles con filtros de búsqueda, estado, región, fechas, y ordenamiento.  
- **GET `/collection/{id}`**  
  Devuelve los metadatos de una colección específica.  
- **PUT `/collection/{id}`**  
  Actualiza los campos de una colección.  
- **DELETE `/collection/{id}`**  
  Elimina una colección por ID.  
- **PUT `/collection/{id}/block`**  
  Bloquea una colección (estado: bloqueada).  
- **PUT `/collection/{id}/unblock`**  
  Desbloquea una colección (estado: activa).  
- **GET `/collection/artist/{artist_id}`**  
  Lista las colecciones que incluyen un artista dado, con filtros adicionales.

---

### Catálogo General
- **GET `/catalog`**  
  Devuelve el catálogo completo combinando canciones y colecciones con filtros y paginación.



## Seguridad

## Tests

## Dificultades
