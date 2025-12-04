# Orchestrator Service

## Descripción General
El **Orchestrator Service** es el microservicio responsable de la orquestación y ejecución de tareas asíncronas en Melodia. Permite programar y encolar tareas HTTP hacia otros microservicios (por ejemplo, registrar reproducciones, enviar notificaciones, actualizar métricas) de forma confiable y desacoplada, gestionando reintentos, timeouts y ejecución diferida.

Desarrollado en **Python** con **FastAPI** y **SQLModel**, el servicio expone endpoints REST para programar (`/schedule`) y encolar (`/enqueue`) tareas, así como un endpoint interno para procesar payloads (`/worker-endpoint`).

## Tecnologías Utilizadas
- **Lenguaje:** Python 3
- **Framework:** FastAPI + SQLModel
- **Base de datos:** PostgreSQL (persistencia de tareas)
- **Infraestructura:** Docker Compose para pruebas/desarrollo local; despliegue productivo en **Google Cloud Platform (GCP)** usando **Cloud Run** y pipelines automatizados con **GitHub Actions**.

## Arquitectura Interna
- **app/main.py:** inicializa la aplicación, routers y middleware de errores.
- **app/api/v1/routers/worker_router.py:** expone los endpoints para programar y encolar tareas, y para procesar payloads.
- **app/services/worker_service.py:** contiene la lógica de negocio para la gestión y ejecución de tareas.
- **app/core/database.py:** inicializa la conexión a PostgreSQL.
- **app/errors:** maneja errores y respuestas uniformes.

## Endpoints Clave
- **`/schedule`**: programa una tarea para ejecución futura (con fecha/hora específica).
- **`/enqueue`**: encola una tarea para ejecución inmediata.
- **`/worker-endpoint`**: endpoint interno para procesar el payload de una tarea.
- **`/health`**: endpoint de salud para monitoreo y readiness.

## Infraestructura y Despliegue
- **Desarrollo y pruebas local:** se utiliza Docker Compose (`compose.yml`, `compose.test.yml`) para levantar el servicio y sus dependencias. Estos archivos no se emplean en producción.
- **Producción:** el despliegue se realiza en **Google Cloud Platform (GCP)** usando **Cloud Run**, con pipelines automatizados mediante **GitHub Actions**. Las variables sensibles y de configuración se gestionan con `secrets` y `vars` en GitHub y GCP. El servicio se expone públicamente y se integra con otros microservicios mediante endpoints HTTP.

---
Para más detalles sobre endpoints y payloads, consulta la documentación Swagger en `/docs` una vez desplegado el servicio.