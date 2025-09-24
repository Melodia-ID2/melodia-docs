# Bitácora 1

## Alcance

Para el *Checkpoint 1*, el grupo definió inicialmente los siguientes objetivos:

* [x] Registro de usuarios
* [x] Inicio de sesión con email y contraseña
* [x] Catálogo – exploración y búsqueda de contenido
* [x] Visualización de perfil propio
* [x] Edición de perfil
* [x] Visualización de perfil de otros usuarios
* [x] Listado de usuarios del sistema
* [ ] Creación y gestión de playlists

Durante el desarrollo, y a medida que se avanzó con las funcionalidades planificadas, se consensuó un ajuste en el alcance original. En lugar de implementar la creación y gestión de playlists, se decidió priorizar funcionalidades que complementen mejor el conjunto actual. En particular, se incorporaron las siguientes:

* [x] Bloqueo de usuarios
* [x] Recuperación de contraseña

## Artefactos

La plataforma está compuesta por los siguientes artefactos principales:

* **Authentication Service**
* **Users Service**
* **Catalog Service**
* **Servidor web de backoffice (frontend)**
* **Aplicación móvil para usuarios**

Cada uno de estos desarrollado en su propio repositorio privado dentro de nuestra organización de GitHub ["Melodia-ID2"](https://github.com/Melodia-ID2). Cada uno de ellos se detalla a continuación:

### Authentication service
<!-- 
Componente encargada del proceso de autenticación de un usuario. Abarca responsabilidades como:

* Generación del token JWT de acceso
* Validación del token JWT de acceso
* Generación de un token JWT de recuperación
* Validación del token JWT de recuperación -->
### Users service

### Catalog service

### Servidor web frontend de backoffice

### Aplicación móvil para usuarios

## Decisiones técnicas

Durante este checkpoint, el equipo debió tomar decisiones que no resultaron triviales. Estas requirieron un análisis previo, discusión interna y el consejo de los tutores. A continuación detallamos dos decisiones que consideramos relevantes.

### Infraestructura empleada para el restablecimiento de contraseña

En relación con la historia de usuario de recuperación de contraseña, con el siguiente criterio de aceptación:

> **[Restablecimiento de contraseña con enlace válido](https://ingenieria-del-software-2.github.io/tps/2025/2/tpgrupal/#recuperaci%c3%b3n-de-contrase%c3%b1a:~:text=CA%202%3A%20Restablecimiento%20de%20contrase%C3%B1a%20con%20enlace%20v%C3%A1lido)**
> *Dado un usuario que hace clic en el enlace de restablecimiento de contraseña recibido por correo electrónico*
> *Entonces se le presenta una página segura donde puede ingresar una nueva contraseña.*

El equipo decidió implementar la página segura para el restablecimiento de contraseña dentro del mismo dominio de la página web ya desplegada (la utilizada para el backoffice).

Antes de llegar a esta conclusión, se evaluaron otras alternativas:

1. **Implementar la página dentro de la aplicación móvil.**

   * Descartada porque, si el usuario abre el enlace desde un dispositivo diferente (PC, tablet, etc.), no podría acceder a la aplicación móvil para completar el proceso.

2. **Crear un sitio web independiente exclusivamente para la recuperación de contraseña.**

   * Descartada porque implicaba un nuevo proyecto, con el costo de desarrollo y despliegue adicional que consideramos innecesario para este caso.

La decisión final se basó en la simplicidad de la integración con la opción elegida y en las desventajas de las alternativas. Además, al discutirlo con los tutores, ellos coincidieron en que la solución adoptada era la más conveniente, teniendo en cuenta tanto la complejidad de implementación como las restricciones de tiempo.

### Estrategia de testeo para funcionalidades externas

En relación con las siguientes historias de usuario y criterios de aceptación:

>* **Recuperación de contraseña: [Solicitud de restablecimiento exitosa](https://ingenieria-del-software-2.github.io/tps/2025/2/tpgrupal/#recuperaci%c3%b3n-de-contrase%c3%b1a:~:text=Solicitud%20de%20restablecimiento%20de%20contrase%C3%B1a%20exitosa)**
>
>   * *Dado un usuario registrado que ha olvidado su contraseña*
>   * *Entonces el usuario recibe un correo electrónico con un enlace para restablecer su contraseña.*

>* **Edición de perfil: [Cambio de foto de perfil](https://ingenieria-del-software-2.github.io/tps/2025/2/tpgrupal/#recuperaci%c3%b3n-de-contrase%c3%b1a:~:text=Cambio%20de%20foto%20de%20perfil)**
>
>   * *Dado un usuario autenticado que edita su perfil*
>   * *Entonces el usuario puede subir una nueva foto de perfil.*

El equipo se encontró con la dificultad de testear funcionalidades externas involucradas, como:

* **El envío real de correos electrónicos.**
* **La interacción con servicios de almacenamiento en la nube para la gestión de imágenes.**

Decidimos **no realizar pruebas integrales con los servicios reales**, principalmente por dos razones:

1. **Limitaciones de uso y costos asociados**: los proveedores imponen cuotas (cantidad de requests, espacio de almacenamiento, etc.), lo que dificulta realizar pruebas automatizadas a gran escala.

2. **Alcance de los tests**: se estaría validando el correcto funcionamiento de servicios externos, lo cual escapa a la responsabilidad de nuestro sistema. Nuestro foco es garantizar que se realicen correctamente las llamadas a dichos servicios, no verificar que ellos cumplan su parte.

En consecuencia, la estrategia adoptada fue **mockear estas dependencias externas** durante las pruebas, validando que nuestro sistema:

* Genere las solicitudes correctamente.
* Construya los datos esperados (ejemplo: contenido de los correos).
* Interactúe de forma adecuada con las APIs externas.

De esta manera se mantiene la robustez de los tests sin depender de factores fuera de nuestro control.
