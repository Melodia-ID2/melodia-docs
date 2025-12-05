# Informe Final

## Resumen

Melodia cuenta con una app móvil (React Native), backoffice (Next.js) y 9 microservicios (Python/Go) detrás de Traefik. Se cubren todas las historias obligatorias y la mayoría de opcionales, incluyendo recomendaciones personalizadas, notificaciones push, reproducción con auto play, playlists temáticas y un servicio extra de covers con IA.

## Arquitectura y despliegue

- **Servicios**: Auth, Users, Catalog, Playlists, Playback, Metrics, Search, Recommendations, Orchestrator, Cover IA.
- **Bases**: PostgreSQL (auth/users/metrics/recs/playlists/orchestrator), MongoDB (catalog/playback), Typesense (search), Cloudflare R2 (audio/video), Cloudinary (imágenes).
- **Infra**: Docker multi-stage, Cloud Run + Artifact Registry (CI/CD en GitHub Actions), Traefik como gateway, Firebase FCM para push, New Relic para monitoreo.
- **Datos móviles/web**: Tokens JWT validados en gateway/servicios; preferencias y contexto de reproducción sincronizados vía Playback/Users.

## Funcionalidad entregada (por épica)

- **Usuarios/Perfil**: Registro/login (local + Google), recuperación de contraseña, edición de perfil y foto, roles, bloqueo, bitmask de preferencias (historial, notificaciones, autoplay, artistas silenciados), géneros y artistas favoritos.
- **Artistas/Catálogo**: Perfil de artista, popular, discografía, colaboraciones, trazabilidad; publicación programada con ventanas/regiones vía Orchestrator; bloqueo/desbloqueo; fast-complete de metadatos y portadas (Gemini/ACRCloud).
- **Administración/Backoffice**: Gestión de usuarios y catálogo; edición de disponibilidad; playlists temáticas administrables (activar temporada/ocultar).
- **Explorar/Recomendaciones**: Búsqueda unificada (Typesense); Made For You (Discover Weekly + Daily Mixes) cacheado; Radio por canción; Mood Mix (playlists contextuales IA).
- **Reproducción/Biblioteca**: Player con cola/controles avanzados, videos musicales, historial, liked songs, reordenamiento de playlists, reproducción on demand, auto play al finalizar contexto.
- **Notificaciones/Social**: Push FCM con deep links (new_release, new_follower, playlist events, content_shared); seguir/dejar de seguir usuarios; compartir canciones/playlists; actividad de amigos.
- **Onboarding**: Pasos para géneros, artistas favoritos y notificaciones; configuración persistida en Users/Recommendations.
- **Extra**: Servicio de Covers IA (RVC + Demucs) con despliegue híbrido CPU/GPU.

## Testing, calidad y CI/CD

- **Cobertura**: Todos los servicios cuentan con tests de integración con más de 75% de cobertura.
- **Pipelines**: GitHub Actions ejecuta lint/tests, build de imagen y deploy a Cloud Run; Codecov en Users/Metrics.
- **Observabilidad**: Logs estructurados (niveles), New Relic en Playback/Cover, métricas de API en servicios Python.

## Limitaciones y riesgos conocidos

- Algoritmos de auto play/radio son heurísticos.
- Pruebas de carga/end-to-end móviles/web acotadas.

## Documentación relacionada
- Arquitectura y diagramas: `diseño/arquitectura.md`
- Bitácoras de cada entrega: `bitacoras.md`
