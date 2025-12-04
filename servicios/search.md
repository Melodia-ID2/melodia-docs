# Search Service

## Descripción General
El **Search Service** es el microservicio encargado de la búsqueda unificada en Melodia, permitiendo a los usuarios encontrar canciones, álbumes, playlists, artistas y perfiles de oyentes mediante una única API. Centraliza la indexación y consulta sobre el motor [Typesense](https://typesense.org/), integrando los datos provenientes de los microservicios de catálogo, usuarios, playlists y artistas.

El servicio expone endpoints REST para búsquedas rápidas y para la indexación/eliminación de documentos, permitiendo mantener el motor de búsqueda sincronizado con los cambios en el resto de la plataforma.

## Tecnologías Utilizadas
- **Lenguaje:** Python 3
- **Framework:** FastAPI
- **Motor de búsqueda:** Typesense (corriendo como servicio aparte, ver `melodia-typesense`)
- **Integración:** Los microservicios de catálogo, usuarios, playlists y artistas envían datos a este servicio para indexación y actualización de resultados de búsqueda.
- **Infraestructura:** Docker Compose para pruebas/desarrollo local; despliegue productivo en **Google Cloud Platform (GCP)** usando **Cloud Run** y pipelines automatizados con **GitHub Actions**.

## Arquitectura Interna
- **app/routers/index_router.py:** expone endpoints para indexar y eliminar documentos de canciones, usuarios, playlists y álbumes en Typesense.
- **app/routers/search_router.py:** expone el endpoint principal de búsqueda, permitiendo filtrar por tipo de entidad y paginar resultados.
- **app/core/typesense_client.py:** gestiona la conexión y operaciones con el motor Typesense.
- **app/services/search_service.py:** encapsula la lógica de consulta y ranking de resultados.
- **app/schemas:** define los modelos de datos para indexación y búsqueda.

## Endpoints Clave
- **`/search`**: búsqueda unificada por texto, con filtros por tipo (`song`, `album`, `artist`, `playlist`, `listener`, `genre_mood`). Permite paginación y personalización según el usuario autenticado.
- **`/index/*`**: endpoints para indexar (`POST`) y eliminar (`DELETE`) documentos de canciones, usuarios, playlists y álbumes. Usados por los microservicios de catálogo, usuarios y playlists para mantener la búsqueda actualizada.

## Integración con otros microservicios
- **Catálogo:** envía datos de canciones, álbumes y playlists para indexación y actualización.
- **Usuarios:** actualiza perfiles y artistas en el motor de búsqueda.
- **Playlists:** sincroniza cambios y nuevos playlists.
- **Orchestrator:** puede programar tareas de indexación masiva o eliminación en cascada.

La búsqueda personalizada puede considerar el contexto del usuario (por ejemplo, historial, preferencias) para mejorar la relevancia de los resultados.

## Infraestructura y Despliegue
- **Desarrollo y pruebas local:** se utiliza Docker Compose (`compose.yml`) para levantar Typesense y el servicio de búsqueda. Estos archivos no se emplean en producción.
- **Producción:** el despliegue se realiza en **Google Cloud Platform (GCP)** usando **Cloud Run**, con pipelines automatizados mediante **GitHub Actions**. Las variables sensibles y de configuración se gestionan con `secrets` y `vars` en GitHub y GCP. El servicio se expone públicamente y se integra con otros microservicios mediante endpoints HTTP.

---
Para más detalles sobre endpoints y payloads, consulta la documentación Swagger en `/docs` una vez desplegado el servicio.
