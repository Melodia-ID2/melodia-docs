# Recommendations Service

## Descripción General
El **Recommendations Service** tiene como función principal generar **recomendaciones musicales personalizadas** para cada usuario basándose en su historial de reproducción, artistas seguidos, canciones favoritas y géneros musicales.

El servicio implementa tres tipos principales de recomendaciones:
- **Made For You**: Colección que incluye Discover Weekly y Daily Mixes personalizados
- **Discover Weekly**: Playlist semanal de 50 canciones no escuchadas con alta diversidad
- **Daily Mix**: 2-6 playlists diarias agrupadas por géneros musicales del usuario
- **Song Radio**: Radio continua generada a partir de una canción semilla

Todas las recomendaciones se generan considerando la región del usuario y se almacenan en caché para optimizar el rendimiento.


## Tecnologías Utilizadas
- **Lenguaje:** Python 3
- **Framework:** FastAPI + SQLModel
- **Base de datos:** PostgreSQL (caché de recomendaciones)
- **Autenticación:** JWT (validación de tokens desde Auth Service)
- **Infraestructura:** Docker Compose (``compose.yml``, ``compose.test.yml``), ``pydantic-settings``, ``sqlmodel``, ``alembic-like``, ``init_db``


## Arquitectura Interna
El servicio sigue una **arquitectura por capas** con separación clara de responsabilidades:

### Estructura general

```
routers → services → clients → external services
             ↓
        repositories → database (PostgreSQL)
```

### Descripción de capas

- **Routers (`app/api/v1/routers/*.py`)**  
  Define los endpoints HTTP del servicio:
  - `made_for_you_router.py`: Endpoints para Made For You completo, Discover Weekly y Daily Mixes
  - `song_radio_router.py`: Endpoints para crear y extender radios por canción

- **Services (`app/services/*.py`)**  
  Contiene la lógica de negocio principal:
  - `made_for_you_service.py`: Servicio agregador que combina Discover Weekly y Daily Mixes
  - `discover_weekly_service.py`: Genera playlists semanales con máxima diversidad
  - `daily_mix/service.py`: Genera 2-6 mixes diarios agrupados por géneros
  - `song_radio_service.py`: Genera radios continuas desde canciones semilla
  - `recommendation_engine.py`: Motor de recomendaciones con algoritmos de scoring
  - `diversity_service.py`: Maximiza diversidad de géneros y artistas
  - `user_context.py`: Centraliza información del usuario (región, historial, afinidad)

- **Clients (`app/clients/*.py`)**  
  Encapsulan la comunicación con otros microservicios:
  - `catalog_client.py`: Obtiene metadatos de canciones y colecciones
  - `playback_client.py`: Historial de reproducción y canciones recientes
  - `users_client.py`: Artistas seguidos, información de usuario
  - `metrics_client.py`: Métricas de afinidad con artistas (reproducciones + likes)
  - `playlists_client.py`: Creación y gestión de playlists

- **Repositories (`app/repositories/*.py`)**  
  Abstrae el acceso a la base de datos:
  - `made_for_you_repository.py`: Operaciones CRUD sobre caché de recomendaciones

- **Models (`app/models/*.py`)**  
  Define las entidades de la base de datos:
  - `cache.py`: Modelos de caché (DiscoverWeeklyCache, DailyMixCache)

- **Schemas (`app/schemas/*.py`)**  
  DTOs para requests/responses:
  - `made_for_you.py`: Respuestas de Made For You, Discover Weekly y Daily Mix
  - `radio.py`: Respuestas de Song Radio
  - `activity_event.py`: Eventos de actividad del usuario
  - `error.py`: Estructura de errores

- **Core (`app/core/*.py`)**  
  Configuración y utilidades:
  - `config.py`: Variables de entorno y configuración
  - `database.py`: Conexión a PostgreSQL
  - `security.py`: Validación de JWTs y extracción de user_id
  - `dependencies.py`: Dependencias inyectables (UserContext)

### Módulos especializados de Daily Mix

El servicio de Daily Mix está modularizado en `app/services/daily_mix/`:
- `service.py`: Lógica principal de generación y caché
- `clustering.py`: Agrupa artistas por géneros usando K-Means
- `cover_generator.py`: Genera portadas compuestas usando la API de Cloudinary para manipular imágenes dinámicamente, y así lograr portadas compuestas por la imagen del artista principal del mix y el overlay correspondiente.
    <img src="../assets/imgs/covers.jpeg" alt="Made For You Covers" width="70%">
- `mix_generator.py`: Construye mixes con balanceo de canciones familiares y nuevas


### Endpoints Principales


* **`GET /made-for-you`**: Devuelve la colección completa de recomendaciones personalizadas del usuario. Si las recomendaciones están en caché y no ha pasado mucho tiempo desde su última generación, se devuelven directamente. De lo contrario, se regeneran automáticamente.

* **`POST /made-for-you/regenerate`**: Fuerza la regeneración completa de todas las recomendaciones personalizadas, ignorando el caché existente.

* **`POST /made-for-you/refresh-from-activity`**: Actualiza los Daily Mixes de forma incremental basándose en eventos de actividad del usuario, como "me gusta", reproducciones o nuevos artistas seguidos.

* **`POST /radio/song/{song_id}`**: Genera una radio continua a partir de una canción semilla, incluyendo canciones del mismo artista y de artistas similares.

## Lógica de Recomendaciones

### Discover Weekly

**Objetivo:** Playlist semanal de 50 canciones no escuchadas con máxima diversidad.

**Algoritmo detallado:**

1. **Construcción del pool de artistas (con priorización)**:
   - Se obtienen artistas de 2 fuentes, en orden de prioridad:
     - **Artistas seguidos**: Lista completa de artistas que el usuario sigue
     - **Artistas recientes**: Top 30 artistas más reproducidos en los últimos 60 días
   - Se eliminan duplicados manteniendo el orden de prioridad

2. **Obtención de canciones por artista**:
   - Se procesan hasta 15 artistas del pool (mezclados aleatoriamente)
   - Por cada artista se solicitan hasta 20 canciones al Catalog Service
   - Se seleccionan aleatoriamente entre 2-4 canciones por artista (evita saturación)
   - Se excluyen canciones ya reproducidas (últimas del historial)

3. **Maximización de diversidad de géneros**:
   - Se agrupan canciones por género primario (primer elemento del array `genres`)
   - Algoritmo round-robin: toma 1 canción de cada género en rotación
   - Repite el proceso hasta alcanzar 50 canciones
   - Si un género se agota, continúa con los demás

4. **Finalización**:
   - Se mezclan las 50 canciones resultantes aleatoriamente
   - Se crea/actualiza la playlist en Playlists Service
   - Se guarda en caché (válido 1 semana)

**Actualización:** Semanal (lunes)

### Daily Mix

**Objetivo:** 2-6 playlists diarias agrupadas por géneros con alta familiaridad.

**Algoritmo detallado:**

1. **Obtención de canciones por artista**:
   - Por cada artista afín se obtienen hasta 50 canciones desde Catalog Service
   - Se filtran canciones bloqueadas por región del usuario

2. **Cálculo de género dominante por artista**:
   - Por cada artista se analizan todas sus canciones
   - Se cuenta la frecuencia de cada género en el array `genres` de las canciones
   - El género más frecuente se define como **género dominante** del artista

3. **Clustering por géneros**:
   - Se agrupan artistas por su género dominante
   - **Géneros con 5+ artistas**: Se dividen en sub-clusters de 4 artistas máximo
   - **Géneros con 2-4 artistas**: Forman 1 cluster
   - **Géneros con 1 artista**: Se fusionan con clusters del mismo género o se descartan
   - Se limitan a máximo 6 clusters (Daily Mix 01 a 06)
   - **Mínimo 2 artistas por cluster** para garantizar variedad

4. **Generación de mixes por cluster**:
   - Por cada cluster se genera una playlist de ~50 canciones:
     - **80% familiares**: Canciones ya escuchadas o de artistas seguidos (38-40 canciones)
     - **20% nuevas**: Canciones no escuchadas de artistas seguidos (10-12 canciones)
   - Se filtran duplicados y canciones ya reproducidas recientemente

**Actualización:** Diaria (24 horas)

**Actualización incremental:**
El servicio también soporta actualización incremental mediante eventos de actividad:
- `follow`: Actualiza cuando el usuario sigue un nuevo artista, y si el último cache es de hace mas de 6 hs.
- `like`: Actualiza cuando el usuario da "me gusta" a una canción, y si el último cache es de hace más de 5 hs.
- `play`: Actualiza basándose en reproducciones, y si el último caché es de hace más de 4 hs.

### Song Radio

**Objetivo:** Radio continua generada desde una canción semilla.

**Composición:**
- **60% Artista de la semilla**: Canciones del mismo artista que la canción semilla
- **40% Artistas similares**: Artistas relacionados (mismo género)

**Filtros aplicados:**
- Excluye canciones ya reproducidas recientemente
- Excluye la canción semilla
- Maximiza diversidad de géneros
- Mezcla aleatoria para variedad

### Actualización Incremental
El servicio admite actualizaciones incrementales basadas en eventos de actividad del usuario. Estas actualizaciones permiten mantener las recomendaciones actualizadas sin necesidad de regenerarlas completamente:

* ``follow``: Se actualizan las recomendaciones cuando el usuario sigue a un nuevo artista, siempre que el caché tenga más de 6 horas de antigüedad.
* ``like``: Se actualizan las recomendaciones cuando el usuario marca una canción como "me gusta", si el caché tiene más de 5 horas.
* ``play``: Se actualizan las recomendaciones basándose en nuevas reproducciones, siempre que el caché tenga más de 4 horas.


## Integración con Metrics Service

El servicio consume métricas del usuario para mejorar la personalización:

**Métricas utilizadas:**
- **Top artistas reproducidos**: Artistas con más reproducciones
- **Top artistas likeados**: Artistas con más canciones en "me gusta"
- **Afinidad de artistas**: Combinación ponderada de ambas métricas

**Uso en recomendaciones:**
- **Discover Weekly**: Prioriza artistas con alta afinidad al construir el pool
- **Daily Mix**: Ordena artistas por afinidad para clustering y generación

**Cliente:** `MetricsClient` (`app/clients/metrics_client.py`)

**Métodos principales:**
- `get_top_artists_reproduced(user_id, n=5)`
- `get_top_liked_artists(user_id, n=5)`
- `get_artist_affinity(user_id, top_n=5)`


## Sistema de Caché

El servicio implementa un sistema de caché para optimizar el rendimiento:

### Modelos de caché

**`DiscoverWeeklyCache`**
- Almacena Discover Weekly por usuario
- Validez: 1 semana (lunes a lunes)
- Campos: `user_id`, `week_number`, `year`, `song_ids`, `last_updated`

**`DailyMixCache`**
- Almacena Daily Mixes por usuario
- Validez: 24 horas
- Campos: `user_id`, `mix_number`, `cluster_name`, `song_ids`, `cover_url`, `last_updated`
- **Campos de tracking** (detección de cambios):
  - `last_playback_timestamp`: Último timestamp de reproducción
  - `following_artists_snapshot`: Snapshot de artistas seguidos
  - `liked_songs_count`: Cantidad de canciones likeadas

### Estrategia de invalidación

**Discover Weekly:**
- Se regenera automáticamente cada lunes
- Se puede forzar regeneración manualmente

**Daily Mix:**
- Se regenera después de 24 horas
- Se puede forzar regeneración manualmente
- Detección de cambios significativos:
  - Nuevos artistas seguidos
  - Nuevas canciones likeadas
  - Nuevo historial de reproducción