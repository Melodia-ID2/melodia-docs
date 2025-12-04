# Playback Service

## Descripción General
El **Playback Service** es el microservicio encargado de gestionar la cola de reproducción, el historial y la generación de URLs presignadas para el streaming de canciones y videos en Melodia. Su principal objetivo es asegurar que los usuarios accedan a los contenidos musicales de forma segura, eficiente y conforme a las reglas de acceso y licenciamiento.

El servicio está desarrollado en **Go** usando el framework **Gin** y se integra con otros microservicios (auth, catálogo, users) para validar tokens, obtener metadatos y aplicar reglas de negocio. Expone una API RESTful que permite manipular la cola de reproducción (agregar, quitar, reordenar, avanzar, retroceder), consultar el historial y obtener URLs temporales para el streaming.

## Tecnologías Utilizadas
- **Lenguaje:** Go
- **Framework:** Gin
- **Base de datos:** MongoDB (cola, historial, canciones) y PostgreSQL (usuarios, si está configurado)
- **Almacenamiento de objetos:** Cloudflare R2 (archivos multimedia, URLs presignadas)
- **Autenticación:** JWT (validación vía middleware, integración con Auth Service)
- **Observabilidad:** New Relic (tracing y monitoreo)
- **Infraestructura:** Docker Compose para pruebas/desarrollo local; despliegue productivo en **Google Cloud Platform (GCP)** usando **Cloud Run** y pipelines automatizados con **GitHub Actions**.

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

## Endpoints Clave
- **`/queue/{user_id}`**: obtener, cargar, agregar, quitar, reordenar y manipular la cola de reproducción de un usuario.
- **`/queue/{user_id}/next` / `/prev` / `/current`**: avanzar, retroceder y consultar el estado actual de la cola, incluyendo URLs presignadas para streaming.
- **`/history`**: registrar y consultar el historial de reproducción.
- **`/songs/{id}/video`**: obtener la URL presignada para el video de una canción.
- **`/health`**: endpoint de salud para monitoreo y readiness.

## Seguridad y Acceso
Todos los endpoints relevantes están protegidos por middleware de autenticación JWT, que valida tokens emitidos por el Auth Service y verifica el rol y la propiedad del recurso. Las URLs presignadas generadas por Cloudflare R2 tienen expiración corta y sólo permiten acceso temporal al contenido.

## Infraestructura y Despliegue
- **Desarrollo y pruebas local:** se utiliza Docker Compose (`docker-compose.yml`, `docker-compose.test.yml`) para levantar el servicio y sus dependencias (MongoDB, PostgreSQL, R2 simulado). Estos archivos no se emplean en producción.
- **Producción:** el despliegue se realiza en **Google Cloud Platform (GCP)** usando **Cloud Run**, con pipelines automatizados mediante **GitHub Actions**. Las variables sensibles y de configuración se gestionan con `secrets` y `vars` en GitHub y GCP. El servicio se expone públicamente y se integra con otros microservicios mediante endpoints HTTP y JWT.

## Observabilidad y monitoreo
El servicio integra **New Relic** para trazabilidad y monitoreo de rendimiento, permitiendo detectar cuellos de botella y errores en tiempo real.

---
Para más detalles sobre endpoints y payloads, consulta la documentación Swagger en `/swagger/index.html` una vez desplegado el servicio.