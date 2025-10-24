# Bitácora 2

## Alcance

Para el *Checkpoint 2*, el grupo definió inicialmente los siguientes objetivos:

### Historias (prioridad alta)

* [x] [Usuarios] Login con proveedor federado
* [x] [Biblioteca] Creación y gestión de playlists
* [x] [Artistas] Publicación de lanzamientos
* [x] [Artistas] Gestión de perfil del artista
* [x] [Perfil] Visualización de perfil de otros usuarios
* [x] [Reproducción] Gestión de cola
* [x] [Explorar] Búsqueda unificada por tipo de contenido
* [x] [Artistas] Discografía
* [x] [Social] Seguimiento de perfiles de usuario
* [x] [Métricas] Métricas de usuario
* [x] [Biblioteca] Historial de reproducción
* [x] [Administración de contenido] Transiciones y estado efectivo del catálogo
* [x] [Reproducción] Reproducción y controles básicos
* [x] [Biblioteca] Liked Songs

### Historias (prioridad baja)

* [x] [Reproducción] Marcado de Liked Song desde Player
* [x] [Administración de contenido] Bloqueo y desbloqueo con alcance
* [ ] [Administración de contenido] Disponibilidad por región y ventana  <-- **pasa a Checkpoint 3**
* [x] [Administración de contenido] Catálogo - Detalle y trazabilidad
* [ ] [Artistas] Disponibilidad por ventana  <-- **pasa a Checkpoint 3**
* [x] [Artistas] Popular (Top del artista)
* [x] [Artistas] Colaboraciones (Aparece en)
* [x] [Artistas] Perfil del artista
* [ ] [Explorar] Home  <-- **pasa a Checkpoint 3**

### Historias adicionales agregadas durante el checkpoint

* [x] [Reproducción] Reproducción On Demand  (agregada en sprint)

Durante el desarrollo se decidió trasladar a Checkpoint 3 las historias que implican programacion de endpoints, ventanas de disponibilidad y la implementación completa del Home/Explorar, ya que requieren un servicio dedicado para orquestar eventos y ejecuciones a una determinada fecha.
---

## Artefactos

La plataforma está compuesta por los siguientes artefactos principales:

* **Authentication Service**
* **Users Service**
* **Catalog Service**
* **Servidor web de backoffice (frontend)**
* **Aplicación móvil para usuarios**
* **Playlists Service(nuevo)**
* **Playback Service(nuevo)**
* **Metrics Service(nuevo)**
* **Router Service(nuevo)**
* **Search Service(nuevo)**



### Playlists Service (`playlists-service`)

* **Tecnología**: FastAPI (Python) + SQLModel + PostgreSQL

El Playlists Service es responsable de gestionar las relaciones entre los usuarios y los items del catalogo para la creacion y persistencia de playlists

### Playback Service (`playback-service`)

* **Tecnología**: Go + MongoDB

El Playback Service se encarga de controlar la reproduccion de cada usuario, guardando colas e historial

### Metrics Service (`metrics-service`)

* **Tecnología**: FastAPI (Python) + SQLModel + PostgreSQL

Componente encargado de exponer eventos para registrar datos de la aplicacion para ser analizados o mostrados en otros servicios

### Router Service (`router-service` / API Gateway)

* **Tecnología**: Traefik 

Enrutamiento hacia microservicios


### Search Service (`search-service`)

* **Tecnología**: Typesense



---

## Arquitectura y diagramas (resumen)



## Decisiones técnicas tomadas en este checkpoint

1. Introducción de un API Gateway (`router-service`) para centralizar TLS, CORS y routing.
2. Separación del `playback-service` en Go por rendimiento/concurrencia.
3. `search-service` con Typesense para búsqueda rápida y escalable.
4. `metrics-service` dedicado para centralizar eventos y facilitar observabilidad.
5. Se decidió postergar la implementación del worker (ventanas/regiones) en favor de diseñar un `orquestrator-service` en Checkpoint 3.

---

