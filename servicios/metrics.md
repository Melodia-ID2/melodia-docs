# Metrics Service


## Descripción General
El **Metrics Service** centraliza la recolección, cálculo y exposición de métricas clave para la plataforma Melodia: reproducciones, likes, compartidos, actividad de usuarios, rankings de artistas y colecciones, y exportación de reportes. Permite a administradores y artistas consultar estadísticas agregadas y series temporales, así como exportar datos en CSV/Excel para análisis externo.

Además, este microservicio realiza análisis sobre los eventos recopilados, permitiendo endpoints de personalización y recomendación basados en la actividad de cada usuario. Estos endpoints han sido fundamentales para features como la creación de playlists asistidas y para historias de usuario que requieren personalización según las métricas y hábitos del usuario.

Desarrollado en **Python** con **FastAPI** y **SQLModel**, el servicio expone una API RESTful con endpoints para métricas de canciones, álbumes/playlists, artistas y usuarios, además de endpoints para exportar reportes y consultar actividad reciente y análisis personalizado.

## Tecnologías Utilizadas
- **Lenguaje:** Python 3
- **Framework:** FastAPI + SQLModel
- **Base de datos:** PostgreSQL (persistencia de métricas y eventos)
- **Infraestructura:** Docker Compose para pruebas/desarrollo local; despliegue productivo en **Google Cloud Platform (GCP)** usando **Cloud Run** y pipelines automatizados con **GitHub Actions**.

## Arquitectura Interna
- **app/main.py:** inicializa la aplicación, routers y middleware de errores.
- **app/api/v1/metrics_router.py:** expone endpoints para métricas agregadas, series temporales, rankings y exportación de reportes.
- **app/api/v1/events_router.py:** gestiona eventos POST para registrar actividad y métricas en tiempo real.
- **app/controllers, app/services:** encapsulan la lógica de negocio y acceso a datos.
- **app/core/database.py:** inicializa la conexión a PostgreSQL.
- **app/errors:** maneja errores y respuestas uniformes.

- **Métricas de canciones:** `/song-plays/{song_id}`, `/likes-song/{song_id}`, `/shares-song/{song_id}` y sus variantes de series temporales.
- **Métricas de álbumes/playlists:** `/plays-album/{playlist_id}`, `/likes-album/{playlist_id}`, `/shares-album/{playlist_id}` y series temporales.
- **Métricas de artistas:** `/monthly-listeners/{artist_id}`, `/artist-ranking/{artist_id}`, `/top-markets-by-artist/{artist_id}` y KPIs de seguidores, saves, shares y plays.
- **Métricas generales:** `/active-users`, `/registers`, `/retention` y endpoints de exportación CSV/Excel.
- **Actividad y análisis de usuarios:** `/user-activity/{user_id}` y otros endpoints de análisis personalizado sobre los eventos de cada usuario, utilizados para recomendaciones, creación de playlists asistidas y personalización de la experiencia.

## Infraestructura y Despliegue
- **Desarrollo y pruebas local:** se utiliza Docker Compose (`compose.yml`) y scripts de testeo (`run_tests.sh`). Estos archivos no se emplean en producción.
- **Producción:** el despliegue se realiza en **Google Cloud Platform (GCP)** usando **Cloud Run**, con pipelines automatizados mediante **GitHub Actions**. Las variables sensibles y de configuración se gestionan con `secrets` y `vars` en GitHub y GCP. El servicio se expone públicamente y se integra con otros microservicios mediante endpoints HTTP.

---
Para más detalles sobre endpoints y payloads, consulta la documentación Swagger en `/docs` una vez desplegado el servicio.