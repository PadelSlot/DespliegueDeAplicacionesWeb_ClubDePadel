# **Plan de Despliegue – PadelSlot**

## **Introducción**

PadelSlot es una aplicación web para reservar pistas de pádel, pensada primero para el móvil (Mobile-First). La desplegamos con **Angular** en **Vercel** (frontend), una **API REST propia en contenedores Docker** detrás de **Nginx** (backend) y **Supabase** como base de datos PostgreSQL gestionada y sistema de login. Usamos solo planes gratuitos: el coste en esta fase académica es **0 €**.

Somos un equipo de 4 desarrolladores. Hemos aplicado lo que hemos visto en el módulo (Docker y Nginx) en la parte que más lo necesita, la API. Donde no aporta valor administrarlo nosotros (la CDN del frontend y la base de datos), usamos servicios gestionados.

Este documento es nuestro plan: recoge las decisiones que hemos tomado y cómo las llevaremos a cabo cuando implementemos el despliegue. Cada fase explica **qué decidimos**, **cómo lo haremos** y **por qué**, y al final hay una hoja de ruta con el orden de trabajo y el reparto entre los 4\.

| Capa | Tecnología | Por qué la elegimos |
| :---- | :---- | :---- |
| Código y trabajo en equipo | Git \+ GitHub | Ya lo usamos en el proyecto y se integra con Vercel y GitHub Actions |
| Frontend | Angular | TypeScript, componentes reutilizables y diseño adaptable al móvil |
| Alojamiento del frontend | Vercel | Despliegue automático, HTTPS, CDN y una URL de prueba por cada Pull Request |
| Backend (API REST) | Node.js \+ Express en Docker | Mismo lenguaje que Angular (TypeScript); el contenedor funciona igual en los 4 portátiles y en el servidor |
| Servidor web / proxy inverso | Nginx (en Docker) | HTTPS con Let's Encrypt, balanceo entre contenedores, CORS, caché y límite de peticiones |
| Servidor | Máquina virtual Ubuntu en Oracle Cloud (Always Free) | Gratis y con recursos de sobra: hasta 4 OCPU y 24 GB de RAM ([Oracle](https://docs.oracle.com/en-us/iaas/Content/FreeTier/resourceref.htm)) |
| Base de datos y login | Supabase (PostgreSQL \+ Auth) | Base de datos relacional gestionada y login con JWT incluido |
| CI/CD | GitHub Actions | Build, pruebas, imágenes Docker y despliegue automáticos |

## **Fase 1: Control de versiones y trabajo en equipo**

Trabajaremos con Git en GitHub, en un único repositorio y con un GitFlow simplificado. Ningún cambio llegará a producción sin un Pull Request revisado por otro compañero y con todas las pruebas superadas.

| Decisión | Cómo lo haremos | Por qué |
| :---- | :---- | :---- |
| GitHub como plataforma | Usaremos la organización PadelSlot que ya tenemos, con sus repositorios e *issues* | Se integra directamente con Vercel, GitHub Actions y el registro de imágenes Docker. No necesitamos otra herramienta |
| Un solo repositorio (monorepo) | Frontend, API, configuración de Nginx y Docker, y cambios de base de datos, cada uno en su carpeta | Un cambio que afecta a la app y a la API va en un solo Pull Request, y nunca se desincronizan las versiones |
| GitFlow simplificado | Tres tipos de rama: main, develop y feature | Tenemos entorno de pruebas, así que necesitamos develop (GitHub Flow no la tiene). El GitFlow completo, con ramas release y hotfix, es demasiado para 4 personas |
| Pull Requests obligatorios | main y develop protegidas: sin push directo, con 1 aprobación y el pipeline en verde | Dos personas ven cada cambio y un error no llega a producción por despiste |
| Tareas como *issues* | Un tablero de GitHub Projects: cada *issue* tiene un responsable, una rama y un Pull Request | Se sabe quién hace qué y por qué se hizo cada cambio |
| Mensajes de commit con convención | Prefijos como feat (nueva función), fix (arreglo) o docs (documentación) | El historial se lee de un vistazo y es más fácil revisar |
| Base de datos versionada | Cada cambio de tablas se guarda como una migración SQL dentro del repositorio ([Supabase CLI](https://supabase.com/docs/guides/cli/getting-started)) | La base de datos evoluciona junto al código y se puede reconstruir desde cero |
| Secretos fuera del repositorio | Contraseñas y claves en un fichero de entorno ignorado por Git y en los secretos cifrados de GitHub | Si el repositorio es público, nadie puede ver nuestras claves |

**Ramas y entornos**

| Rama | Para qué sirve | Dónde se despliega |
| :---- | :---- | :---- |
| main | Código en producción, siempre estable | Vercel producción \+ servidor de producción \+ Supabase de producción |
| develop | Pre-producción: juntamos y probamos las funcionalidades | Vercel preview \+ servidor de pruebas \+ Supabase de pruebas |
| feature/nombre | La tarea de un miembro, por ejemplo feature/api-reservas o feature/calendario-movil | Docker en el portátil \+ URL de preview en el Pull Request |
| fix/nombre | Arreglo urgente de un fallo en producción | Sale de main, vuelve a main y después a develop |

Tener pruebas y producción separadas no cuesta dinero. La capa gratuita de Oracle permite repartir sus recursos en varias máquinas, por ejemplo dos de 2 OCPU ([Oracle](https://docs.oracle.com/en-us/iaas/Content/FreeTier/resourceref.htm)), y Supabase permite 2 proyectos gratuitos.

**Así trabajará cada miembro del equipo**

1. Coge una *issue* del tablero y se la asigna.  
2. Crea su rama feature a partir de develop.  
3. Trabaja en local: con Docker tiene en su portátil la API, Nginx y la base de datos, igual que en el servidor.  
4. Abre un Pull Request hacia develop. El pipeline compila, pasa las pruebas y publica una preview.  
5. Otro compañero revisa el código y prueba la preview en su móvil.  
6. Con la aprobación, se fusiona en develop y se despliega en pruebas.  
7. Cuando una versión está completa y probada, develop se fusiona en main y sale a producción.

**Qué se revisa en cada Pull Request:** que hace lo que dice la *issue*, que se ve bien en el móvil (con la preview de Vercel, que da una URL propia a cada Pull Request), que no incluye claves y que, si cambia la base de datos, trae su migración.

## **Fase 2: Arquitectura**

La arquitectura de la aplicación web está dividida en tres piezas fundamentales e independientes: el frontend **Angular** en la **CDN** (Red de Distribución de Contenidos) de **Vercel**, nuestra **API REST** en contenedores **Docker** detrás de **Nginx** en un servidor **Linux**, y **Supabase** como base de datos **PostgreSQL** gestionada y sistema de login.

| Decisión | Cómo lo haremos | Por qué |
| :---- | :---- | :---- |
| Arquitectura desacoplada | El frontend y la API son proyectos separados que se comunican por una API REST en formato JSON | Cada parte se despliega y se arregla por separado. |
| Frontend en Angular, compilado a archivos estáticos | Vercel publica el resultado de la compilación y lo reparte por su CDN | Una CDN sirve la app desde un punto cercano al usuario. Además, es de carga rápida en el móvil, que es nuestra prioridad |
| API REST propia en Node.js con Express | Un endpoint por recurso: pistas, disponibilidad, reservas y usuario | Es TypeScript, igual que Angular, así que los 4 podemos trabajar en las dos partes. Es ligero y muy habitual en contenedores |
| Empaquetar la API en Docker | Imagen pequeña, construida en dos etapas, que se ejecuta con un usuario sin privilegios y comprueba su propia salud | La API funciona igual en los 4 portátiles y en el servidor. |
| Docker Compose para orquestar | Un único fichero describe los servicios del servidor: Nginx, dos copias de la API y Certbot (herramientas de software libre y de código abierto) | Todo el servidor se levanta con una orden. |
| Dos copias de la API | Nginx reparte las peticiones entre las dos | Si una falla o se está actualizando, la otra sigue atendiendo |
| Servidor en Oracle Cloud | Dos máquinas virtuales Ubuntu, una de pruebas y otra de producción | Es gratis y potente: hasta 4 OCPU (capacidad de procesamiento en servicios en la nube) y 24 GB de RAM. Otras opciones, como DigitalOcean o AWS, cuestan dinero una vez pasado el periodo de prueba |
| Base de datos PostgreSQL gestionada en Supabase | Dos proyectos gratuitos: pruebas y producción | No tenemos que instalar, actualizar ni vigilar la base de datos. Es relacional, que es lo que necesitan las reservas |
| Login con Supabase Auth | Angular obtiene un token JWT (estándar abierto que representa información en formato JSON) al iniciar sesión y lo envía a la API, que lo comprueba | No guardamos ni gestionamos contraseñas, que es lo más delicado de cualquier app |
| Nunca doble reserva (principal) | Una restricción de unicidad sobre pista, fecha y hora de inicio | Aunque haya un fallo en la API, la base de datos nunca permitirá dos reservas en la misma pista a la misma hora |

**Recursos de la API REST**

| Recurso | Qué permitirá hacer | ¿Necesita login? |
| :---- | :---- | :---- |
| Pistas | Ver la lista de pistas del club | No |
| Disponibilidad | Ver las franjas libres de una pista en un día | No |
| Reservas | Crear una reserva y cancelar las propias | Sí |
| Mis reservas | Ver el historial y las próximas reservas del usuario | Sí |

**Cómo funcionará una reserva, paso a paso**

1. El usuario inicia sesión en la app con Supabase y recibe un token.  
2. Al reservar, Angular envía la petición a nuestra API junto al token.  
3. Nginx la recibe por HTTPS, comprueba que esa IP no está abusando y la pasa a una de las dos copias de la API.  
4. La API valida el token y los datos, y guarda la reserva en PostgreSQL.  
5. Si alguien cogió esa franja un segundo antes, la base de datos rechaza la reserva y la app avisa de que ya está ocupada.

![arquitectura de PadelSlot · Vercel, servidor con Docker y Nginx, Supabase][image1]

**Entorno de desarrollo**

Por una parte, cada miembro tendrá en su portátil lo mismo que el servidor: Nginx y la API con Docker Compose, y la base de datos y el login con la herramienta de Supabase, que también funciona en Docker (Supabase CLI). Supabase asegura en su documentación oficial que este entorno local es compatible con el alojado.

Por otra parte, usaremos procesadores ARM (Ampere A1), unos chips diseñados para servidores en la nube, así que las imágenes Docker se construirán para ARM. Node.js y Nginx tienen imágenes oficiales para esa arquitectura.

**Alternativas que descartamos**

* **Que Angular hable directamente con la base de datos (sin API propia).** Supabase lo permite, pero perderíamos el control de la lógica de reservas al dejar el código del navegador manejable por cualquiera. Con una API propia, toda la lógica está en un único sitio que controlamos nosotros. Además, Docker y Nginx lo hemos dado nosotros en el módulo, por lo que tenemos conocimiento previo sobre ellos.  
* **Un solo contenedor de la API.** Sería más simple, pero cada actualización cortaría el servicio y no habría balanceo, y si ese contenedor se cuelga, se cae toda la app. Con dos, Nginx manda el tráfico a la copia que funciona. Además, no cuesta nada tener dos copias.  
* **Base de datos en un contenedor nuestro.** Tendríamos que encargarnos de copias, actualizaciones y seguridad. También tendríamos que configurar usuarios, contraseñas, cifrado y accesos, por lo que un fallo nuestro dejaría los datos expuestos. Un DBaaS (Base de Datos como Servicio) lo hace por nosotros, en este caso Supabase.

## **Fase 3: Servidores web y de aplicaciones**

La gestión de las peticiones HTTP es clave para que la app funcione siempre. Nginx, en su propio contenedor Docker, será la única puerta de entrada a nuestra API, y Node.js solo se ocupará de la lógica de la aplicación.

* **Servidor web / proxy inverso. Nginx.** Recibe las peticiones de la app móvil, gestiona el certificado SSL gratuito de Let's Encrypt (HTTPS obligatorio y renovación automática) y redirige el tráfico a la API en Node.js por la red interna de Docker. Lo preferimos a Apache porque es más ligero y es el que hemos visto en el módulo.  
* **Balanceo de carga.** Nginx reparte el tráfico por turnos entre las dos copias del contenedor de la API. Si una falla, deja de usarla y la otra sigue atendiendo, así evitamos cuellos de botella y caídas del servicio.  
* **Límite de peticiones (rate limiting).** Cada IP puede hacer como máximo 10 peticiones por segundo; si se pasa, recibe el error 429\. Así protegemos la API de ataques de fuerza bruta y de denegación de servicio.  
* **Gestión de caché y CORS.** Solo se aceptan peticiones que vengan de nuestro dominio de Vercel. La lista de pistas se guarda en caché 5 minutos, pero la disponibilidad y las reservas nunca, para no mostrar como libre una hora ya reservada.  
* **El frontend se queda en Vercel.** Los archivos de Angular los sirve la CDN de Vercel, más rápida en el móvil que un único servidor. Nginx solo atiende la API.

## **Fase 4: Integración y despliegue continuos (CI/CD)**

Para esta fase vamos a utilizar **GitHub Actions**, que se encargará de comprobar que los cambios funcionan correctamente antes de publicarlos. Cada vez que se haga una Pull Request se ejecutarán unas pruebas y, cuando se aprueben y se unan los cambios, se realizará el despliegue automáticamente.

El despliegue se hará en este orden: primero se actualizará la base de datos, después la API en el servidor y, por último, el frontend en Vercel. De esta manera, no tendremos que hacer los despliegues manualmente desde nuestros ordenadores.

### **Las cuatro etapas del pipeline**

1. **Build (construcción).** En este primer paso se instalarán las dependencias necesarias, se compilará Angular y también la API. Además, se creará la imagen Docker de la API para que pueda funcionar en el servidor ARM.  
2. **Testing (pruebas automáticas).** Aquí se comprobará que todo funciona correctamente antes de publicar los cambios:  
   * **API:** se realizarán pruebas de los diferentes endpoints utilizando Jest.  
   * **Base de datos:** se iniciará una base de datos de Supabase en Docker para comprobar que los cambios y las migraciones se aplican correctamente ([Supabase CI/CD](https://supabase.com/docs/guides/cli/cicd-workflows)).  
   * **Nginx:** se comprobará que su configuración no tenga errores.  
   * **Frontend:** se harán pruebas de Angular y de Lighthouse CI para comprobar que la página funciona bien en móviles y que tiene una buena accesibilidad.  
3. **Delivery (entrega).** Cuando todas las pruebas hayan salido bien y los cambios estén aprobados, se guardará la imagen Docker de la API en GitHub Container Registry. Cada imagen tendrá una etiqueta con el identificador del commit para saber a qué versión corresponde.  
4. **Deploy (despliegue).** En este último paso se actualizará primero la base de datos, después el pipeline se conectará al servidor mediante SSH para actualizar los contenedores y, finalmente, Vercel publicará la nueva versión del frontend.

| Decisión | Cómo lo haremos | Por qué |
| :---- | :---- | :---- |
| GitHub Actions como herramienta de CI/CD | Los pipelines viven en el propio repositorio | Está integrado con GitHub, no hay que montar ningún servidor de CI (como Jenkins) y no cuesta dinero en repositorios públicos |
| Probar todo en cada Pull Request, desplegar solo al fusionar | Las pruebas corren en cada Pull Request; el despliegue, al fusionar en develop o en main | Los errores se ven antes de mezclar el código, y solo lo revisado llega a un servidor |
| Imágenes Docker en un registro, con una etiqueta por commit | GitHub Container Registry, junto al código | Cada versión queda identificada. Volver atrás es desplegar la imagen del commit anterior |
| Rolling Update para la API | Se actualiza primero una copia de la API, se comprueba que responde y después se actualiza la otra. Mientras, Nginx manda el tráfico a la que está en marcha | Despliegue sin caídas (*zero-downtime*) sin herramientas complicadas |
| Rolling Update en vez de Blue/Green | — | Blue/Green necesita un entorno completo duplicado. Con dos copias de la API, el Rolling Update consigue lo mismo con la mitad de recursos |
| Migraciones automáticas antes de actualizar la API | El pipeline aplica los cambios de tablas en el Supabase del entorno correspondiente | La API nueva siempre encuentra las tablas que espera. Supabase recomienda hacerlo desde CI/CD y no desde el portátil ([Supabase](https://supabase.com/docs/guides/cli/managing-environments)) |
| Frontend desplegado por Vercel | Cada versión de Vercel se publica aparte y recibe el tráfico cuando ya está lista | Tampoco hay caídas en el frontend, y se puede volver a la versión anterior desde el panel |
| Secretos cifrados de GitHub | Claves de Supabase, Vercel y del servidor (SSH) guardadas como secretos ([Supabase](https://supabase.com/docs/guides/resources/supabase-cli/managing-environments)) | Nunca aparecen en el código ni en los logs del pipeline |

**Qué pasa en cada momento**

| Evento en GitHub | Qué se ejecuta | Dónde acaba |
| :---- | :---- | :---- |
| Se abre o actualiza un Pull Request | Build \+ todas las pruebas \+ preview | URL temporal de Vercel |
| Fusión en develop | Lo anterior \+ imagen Docker \+ migraciones \+ despliegue | Servidor de pruebas, Supabase de pruebas y preview de Vercel |
| Fusión en main | Lo mismo, en producción | Servidor de producción, Supabase de producción y Vercel producción |

**¿Por qué seguimos este orden?**

Hemos decidido actualizar primero la base de datos, después la API y finalmente el frontend porque las tres partes están relacionadas.

La API necesita que la base de datos tenga preparadas las tablas que va a utilizar, y el frontend necesita que la API esté funcionando correctamente para poder obtener los datos.

Por eso, los cambios de la base de datos se harán de forma que no estropeen el funcionamiento de la versión anterior mientras se realiza la actualización.

**¿Qué hacemos si el despliegue sale mal?**

También hemos pensado qué hacer si aparece algún problema durante el despliegue:

* Si falla alguna prueba: el proceso se detiene automáticamente y no se publica ninguna actualización.  
* Si la nueva API no funciona: se detiene la actualización y la segunda copia sigue funcionando con la versión anterior.  
* Si encontramos un error después de publicar: podemos volver a desplegar la imagen Docker del commit anterior y recuperar la versión anterior del frontend desde Vercel.

**Límite a tener en cuenta**

Por último, hay que tener en cuenta que el plan gratuito de Vercel permite realizar hasta 100 despliegues al día, incluyendo las versiones de prueba o previews ([Vercel](https://vercel.com/docs/plans/hobby)).

Consideramos que es suficiente para nuestro proyecto, ya que somos un equipo de cuatro personas y no creemos que vayamos a necesitar tantos despliegues diariamente.

## **Fase 5: Monitorización, seguridad y calidad**

Cuando la app esté en línea vigilaremos tres cosas: los errores (con Sentry), el servidor (con Prometheus y Grafana, también en Docker) y la base de datos (con el panel de Supabase). Protegeremos cada capa por separado y tendremos una solución preparada para cada límite de los planes gratuitos.

**Monitorización**

| Decisión | Cómo lo haremos | Por qué |
| :---- | :---- | :---- |
| Sentry para los errores | Lo integramos en Angular y en la API | Nos enteramos de un fallo en el móvil de un usuario al momento, sin esperar a que nos lo cuente |
| Prometheus y Grafana para el servidor | Contenedores que recogen el uso de CPU, RAM y disco del servidor y de cada contenedor, y lo muestran en gráficas | Vemos si el servidor va al límite antes de que se caiga. Además, todo es Docker, lo que hemos estudiado |
| Logs de Nginx | Cada petición queda registrada con su ruta, su respuesta y el tiempo que tardó | Es la forma de medir la velocidad real de la API y de detectar abusos |
| Salud de los contenedores | Docker comprueba cada poco el endpoint de salud de la API y reinicia el contenedor si no responde | Muchos fallos se arreglan solos sin que tengamos que entrar al servidor |
| Panel de Supabase | Consultas lentas y logs de la base de datos, con 7 días de historial en el plan gratuito ([Automation Atlas](https://automationatlas.io/answers/supabase-free-tier-limits-2026/)) | Detectamos consultas que ralentizan la app |

**Seguridad por capas**

Si una capa falla, la siguiente sigue protegiendo los datos.

1. **Nginx:** solo HTTPS, límite de peticiones contra fuerza bruta y ataques de denegación de servicio, CORS solo para nuestro dominio y cabeceras de seguridad (Fase 3).  
2. **Servidor:** acceso SSH solo con clave (sin contraseña), cortafuegos con únicamente los puertos 22, 80 y 443 abiertos, y actualizaciones de seguridad automáticas del sistema.  
3. **Contenedores:** la API no tiene puertos abiertos al exterior (solo se llega por Nginx), se ejecuta con un usuario sin privilegios y sus claves están en un fichero de entorno que no está en Git.  
4. **Base de datos:** activaremos las *Network Restrictions* para que PostgreSQL solo acepte conexiones desde la IP de nuestro servidor, y *SSL Enforcement* para que la conexión vaya siempre cifrada ([Supabase Production Checklist](https://supabase.com/docs/guides/platform/going-into-prod)). Estas restricciones protegen PostgreSQL, pero no las APIs web de Supabase ([guía](https://nhimg.org/supabase-security-guide)). Por eso también activaremos *Row Level Security* (RLS) en todas las tablas: aunque alguien tuviera la clave pública, no podría leer ni cambiar datos.  
5. **Login:** Supabase Auth guarda las contraseñas cifradas y limita los intentos de inicio de sesión ([Supabase](https://supabase.com/docs/going-into-prod)). La API comprueba el token en cada petición que necesita login.  
6. **Cuentas del equipo:** verificación en dos pasos (MFA) en GitHub, Vercel, Supabase y Oracle Cloud. Si roban una contraseña, no basta para entrar.

**Conexión de la API a la base de datos.** Por defecto, la conexión directa de Supabase usa IPv6, y el *Session pooler* ofrece IPv4 gratis ([DBConvert](https://dbconvert.com/docs/connect-supabase/)). Usaremos el pooler, la opción segura si el servidor no tiene IPv6.

**Límites de los planes gratuitos y cómo los resolveremos**

| Límite | Solución | Por qué es suficiente |
| :---- | :---- | :---- |
| Supabase pausa el proyecto tras 7 días sin uso | Una tarea programada de GitHub Actions llamará al endpoint de salud, que consulta la base de datos, cada pocos días | El proyecto nunca pasa una semana inactivo, aunque nadie use la app |
| Supabase no hace copias automáticas en el plan gratuito | Supabase recomienda exportar los datos con su herramienta y guardar la copia fuera ([Supabase Backups](https://supabase.com/docs/guides/database/backups)). Lo hará una tarea semanal de GitHub Actions | Como mucho perderíamos una semana de reservas de prueba |
| Oracle puede recuperar máquinas gratuitas que llevan tiempo sin uso ([ThatJeffSmith](https://thatjeffsmith.com/?p=9706)) | La monitorización mantiene actividad. Y como el servidor está descrito en Docker Compose y en el pipeline, podemos rehacerlo en poco tiempo | Ningún dato importante vive en el servidor: los datos están en Supabase |
| 500 MB de base de datos | Solo guardaremos texto (usuarios, pistas y reservas), no imágenes | Son miles de reservas antes de acercarnos al límite |

**Calidad y experiencia móvil**

* **Lighthouse CI en cada Pull Request (Fase 4):** si baja la nota de rendimiento o accesibilidad, no se fusiona.  
* **Alerta en Grafana si la API tarda más de 200 ms** en responder, medido con los logs de Nginx. La latencia arruina la experiencia en el móvil.  
* **Alerta si suben los errores del servidor** (respuestas 5xx) o los bloqueos por exceso de peticiones (429), que pueden indicar un ataque.

**Qué haremos cuando salte una alerta:** el responsable de esa semana mira Sentry y Grafana, abre una *issue* con lo que ha visto y, si el fallo viene de un despliegue, vuelve a la versión anterior (Fase 4).

## **Hoja de ruta para implementarlo**

Cuando pasemos a implementar el despliegue, seguiremos este orden. Cada etapa necesita la anterior. El reparto sigue la división de la documentación: Persona 1 (Git y monitorización), Persona 2 (arquitectura), Persona 3 (Nginx y seguridad) y Persona 4 (CI/CD).

| Etapa | Qué haremos | Responsable | Estará lista cuando… |
| :---- | :---- | :---- | :---- |
| 1\. Preparar el terreno | Protección de ramas, tablero de *issues*, cuentas de Vercel, Supabase (2 proyectos) y Oracle Cloud (2 máquinas), con MFA | Persona 1 | Los 4 podemos abrir Pull Requests y entrar a todos los servicios |
| 2\. Base de datos y login | Tablas, restricción de reservas únicas, RLS y configuración de Supabase Auth, todo como migraciones | Persona 2 | Las migraciones se aplican sin errores en el proyecto de pruebas |
| 3\. API en Docker | Endpoints, imagen Docker y Docker Compose en local | Persona 2, con todos en sus ramas feature | Cualquiera levanta la API en su portátil con una orden |
| 4\. Servidor y Nginx | Máquina virtual, cortafuegos, dominio, HTTPS, balanceo, límites, CORS y caché | Persona 3 | La API responde por HTTPS desde el servidor de pruebas |
| 5\. Pipeline CI/CD | Pruebas, imágenes Docker, Rolling Update, migraciones automáticas y conexión con Vercel | Persona 4 | Una fusión en develop despliega sola en pruebas |
| 6\. Vigilancia y copias | Sentry, Prometheus y Grafana, alertas, copia semanal y tarea contra la pausa | Personas 1 y 3 | Las alertas de prueba nos llegan y hay una copia guardada |
| 7\. Salida a producción | Fusión de develop en main y pruebas en móviles reales | Todos | La app funciona en producción, con Lighthouse en verde y la API por debajo de 200 ms |

## **Por qué Angular (y no React ni Vue)**

Elegimos Angular porque es un framework completo: trae de serie todo lo que necesita PadelSlot (rutas, formularios y peticiones HTTP), usa TypeScript y obliga a los 4 a programar con la misma estructura.

**En una frase para la defensa:** con Angular los 4 escribimos el código igual, en el mismo lenguaje que la API y sin tener que elegir diez librerías antes de empezar.

| Criterio | Angular | React | Vue |
| :---- | :---- | :---- | :---- |
| Qué es | Framework completo | Librería de interfaz: el resto se añade por piezas | Framework progresivo |
| Lenguaje | TypeScript siempre | JavaScript o TypeScript | JavaScript o TypeScript |
| Rutas, formularios y HTTP | Incluidos | Librerías externas que hay que elegir | Librerías oficiales aparte |
| Estructura del proyecto | La marca Angular CLI, igual para todos | La decide cada equipo | Bastante libre |
| Quién lo mantiene | Google | Meta | Su comunidad |
| Despliegue en Vercel | Lo detecta solo | Lo detecta solo | Lo detecta solo |
| Curva de aprendizaje | Más alta | Media | Baja |

**Nuestros 6 argumentos:**

1. **Todo incluido, menos decisiones.** Router, formularios y cliente HTTP vienen con Angular. Con React tendríamos que elegir, aprender y mantener varias librerías por nuestra cuenta.  
2. **La misma estructura para los 4\.** Angular CLI crea componentes, servicios y rutas siempre de la misma forma. Cuando revisamos el Pull Request de un compañero, sabemos dónde está cada cosa (Fase 1).  
3. **TypeScript en todo el proyecto.** Angular y la API en Node.js usan el mismo lenguaje. Los tipos de una reserva se comparten y muchos errores salen al compilar, no en el móvil del usuario.  
4. **Hecho para hablar con una API REST.** Su cliente HTTP permite un *interceptor*: un único trozo de código que añade el token de Supabase a todas las peticiones. Nadie se olvida de ponerlo.  
5. **Rápido en el móvil.** Se compila a archivos estáticos optimizados, y con la carga diferida (*lazy loading*) el móvil solo descarga la pantalla que abre. Desde la versión 22 los componentes usan OnPush por defecto, que evita redibujar lo que no cambia ([InfoQ](https://infoq.com/news/2026/08/angular-v22-released)).  
6. **Estable y con soporte largo.** Angular saca una versión grande al año, y cada una tiene 24 meses de soporte. La versión 22 tiene soporte activo hasta junio de 2027 y parches de seguridad hasta junio de 2028 ([Angular](https://angular.dev/reference/releases)). PadelSlot no se quedará sin actualizaciones.

**Lo que reconocemos:** Angular cuesta más de aprender que Vue, y la primera carga puede pesar algo más. Lo compensamos con la carga diferida y con Lighthouse CI, que en cada Pull Request comprueba que el rendimiento móvil no empeora (Fase 4).

## **Por qué Vercel (y no Netlify, AWS ni nuestro Nginx)**

Elegimos Vercel porque publica Angular sin configurar nada, lo reparte por su CDN con HTTPS y crea una URL de prueba para cada Pull Request. Todo gratis.

**En una frase para la defensa:** Vercel nos da CDN, HTTPS y previews sin mantener nada, y así nuestro servidor con Nginx se dedica solo a la API.

| Criterio | Vercel | Netlify | AWS S3 \+ CloudFront | Nuestro Nginx en Oracle |
| :---- | :---- | :---- | :---- | :---- |
| Compila Angular solo | Sí | Sí | No, hay que montarlo | No, lo haría el pipeline |
| CDN repartida por el mundo | Sí | Sí | Sí | No, un solo servidor |
| HTTPS | Automático | Automático | Hay que configurar el certificado | Con Certbot |
| URL de prueba en cada Pull Request | Sí, comentada en el PR | Sí | No | No |
| Volver a la versión anterior | Un clic | Un clic | A mano | A mano |
| Mantenimiento | Ninguno | Ninguno | Alto | Nuestro |
| Coste para nosotros | 0 € (plan Hobby) | 0 € (plan gratuito) | Pago por uso al pasar la capa gratuita | 0 € |

Netlify haría casi lo mismo. Nos quedamos con Vercel por su integración con GitHub y porque tiene una guía oficial para Angular.

**Nuestros 6 argumentos:**

1. **Cero configuración.** Vercel detecta que el proyecto es Angular y aplica los ajustes de compilación correctos ([Vercel](https://vercel.com/guides/deploying-angular-with-vercel)). Como mucho añadimos un vercel.json para que las rutas de Angular funcionen al recargar la página.  
2. **CDN y HTTPS sin hacer nada.** Cada despliegue sale por su red de servidores repartidos por el mundo y con un certificado SSL que se renueva solo ([Vercel](https://vercel.com/solutions/angular)). Es lo que pide el enfoque Mobile-First: la app carga desde un punto cercano al usuario.  
3. **Una preview por Pull Request.** Cada cambio en un Pull Request tiene su propia URL, y Vercel la publica como comentario en el PR ([Vercel](https://vercel.com/docs/git/vercel-for-github)). El revisor la abre en su móvil antes de aprobar (Fase 1).  
4. **Sin caídas y con vuelta atrás inmediata.** La versión nueva solo recibe tráfico cuando está lista, y si algo falla se recupera la anterior desde el panel (Fase 4).  
5. **Cada pieza en su sitio.** Nginx se centra en la API. Si el servidor de Oracle tiene un problema, la app sigue cargando en el móvil y puede avisar al usuario, en vez de mostrar una página en blanco.  
6. **Gratis y de sobra.** El plan Hobby incluye 100 GB de transferencia y 1 millón de peticiones a la CDN al mes, 100 despliegues al día y protección contra DDoS activada por defecto ([Vercel](https://vercel.com/docs/plans/hobby)).

**Requisito para que funcione:** el repositorio debe ser público. En un repositorio privado, el plan Hobby bloquea los despliegues de commits de personas que no son el dueño de la cuenta de Vercel ([Vercel](https://vercel.com/kb/guide/commits-not-triggering-deployments)), es decir, los de los otros 3\. El proyecto de Vercel estará en la cuenta de un miembro, y GitHub Actions lo desplegará con un token guardado como secreto (Fase 4).

**Lo que reconocemos:** el plan Hobby es solo para uso personal y no comercial ([Vercel](https://vercel.com/docs/limits/fair-use-guidelines)). Si un club real usara PadelSlot, pasaríamos a Pro (20 \$ por desarrollador al mes, [Vercel](https://vercel.com/docs/plans/hobby)) o serviríamos el frontend con el Nginx que ya tenemos montado.

## **Por qué Supabase (y no Oracle ni NoSQL)**

Elegimos Supabase porque es justo lo que pide la Fase 2: una base de datos relacional gestionada (DBaaS), accesible solo desde nuestro servidor. Además nos da el login hecho, la parte más delicada de cualquier app. Mientras, nosotros dedicamos el tiempo a lo que se evalúa en el módulo: Docker y Nginx.

**En una frase para la defensa:** Supabase es PostgreSQL de verdad, gestionado y gratis, con el login y la seguridad por filas ya hechos, así que nosotros solo nos preocupamos de nuestra API.

| Criterio | Supabase | Oracle Autonomous DB | NoSQL (MongoDB) |
| :---- | :---- | :---- | :---- |
| Tipo de base de datos | Relacional (PostgreSQL) | Relacional | Documentos |
| Encaja con reservas (sin dobles reservas) | Sí, con restricciones únicas y transacciones | Sí | Hay que programarlo a mano |
| Base de datos gestionada | Sí | Sí | Sí (Atlas) |
| Login incluido | Sí (Auth con JWT) | No, hay que programarlo | No, hay que programarlo |
| Restringir el acceso a la IP del servidor | Sí (Network Restrictions) | Sí (lista de IPs) | Sí (lista de IPs) |
| Conexión desde Node.js | Una dirección de conexión de PostgreSQL | Wallet de certificados o TLS con lista de IPs | Una dirección de conexión |
| Entorno local con Docker | Su herramienta levanta base de datos y login | Imagen de Oracle Free | Imagen de MongoDB |
| Experiencia previa del equipo | Nueva, pero es SQL estándar | Sí, del curso pasado | No |
| Coste en nuestro proyecto | 0 € | 0 € | 0 € |

**¿Y Firebase?** Es la alternativa más conocida, pero su base de datos principal (Firestore) es NoSQL. Tendría el mismo problema que MongoDB: evitar dos reservas a la misma hora habría que programarlo a mano.

**Nuestros 8 argumentos:**

1. **Es PostgreSQL.** No es una base de datos "rara": es SQL estándar, así que lo que aprendimos con Oracle nos sirve. Si algún día dejamos Supabase, exportamos los datos y los llevamos a cualquier PostgreSQL, incluso a un contenedor nuestro.  
2. **Es un DBaaS de verdad.** No administramos la base de datos (parches, discos, configuración). Eso nos deja tiempo para Docker y Nginx, que es lo que se evalúa.  
3. **Nos da el login hecho y seguro.** Registro, contraseñas cifradas, tokens JWT y límite de intentos. Además permite entrar con Google o con un enlace al correo ([Supabase Auth](https://supabase.com/docs/guides/auth)). Programarlo nosotros sería más trabajo y más riesgo.  
4. **Cumple el aislamiento que pide el plan.** Con Network Restrictions, PostgreSQL solo acepta conexiones desde la IP de nuestro servidor, y siempre cifradas.  
5. **Seguridad dentro de la propia base de datos.** Con Row Level Security, las reglas de acceso viven en PostgreSQL y se aplican en cada consulta, venga de donde venga ([Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security)). Por ejemplo: cada usuario solo ve sus propias reservas. Aunque nuestra API tuviera un fallo, los datos seguirían protegidos.  
6. **Encaja con Docker y CI/CD.** La base de datos también funciona en Docker en cada portátil y en el pipeline, y sus cambios viven en Git como migraciones (Fases 1, 2 y 4).  
7. **No nos ata.** Supabase se puede instalar en un servidor propio con Docker ([Supabase](https://supabase.com/docs/guides/self-hosting)). Si el plan gratuito dejara de servirnos, podríamos llevarlo todo a nuestra máquina de Oracle.  
8. **Gratis y suficiente.** 500 MB de base de datos, 50.000 usuarios activos al mes y 2 proyectos ([Supabase](https://supabase.com/docs/pricing)): uno de pruebas y otro de producción.

**Lo que reconocemos:** Supabase es nuevo para el equipo y el plan gratuito tiene límites (pausa por inactividad y sin copias automáticas). Ya los tenemos resueltos en la Fase 5\.

## **Chuleta para la defensa**

Estas son las preguntas que probablemente nos hará el profesor, con una respuesta corta para cada una.

| Pregunta | Respuesta |
| :---- | :---- |
| ¿Dónde usáis Docker y Nginx? | En producción: Nginx y dos copias de la API funcionan con Docker Compose en nuestro servidor. También en los portátiles y en el pipeline. |
| ¿Por qué dos copias de la API? | Para que Nginx reparta la carga y para actualizarlas de una en una sin cortar el servicio (Rolling Update). |
| ¿Qué pasa si se cae un contenedor? | Nginx deja de mandarle tráfico tras 3 fallos, Docker lo reinicia solo y Sentry nos avisa. |
| ¿Cómo tenéis HTTPS? | Con un certificado gratuito de Let's Encrypt obtenido con Certbot y renovado automáticamente. Nginx redirige todo HTTP a HTTPS. |
| ¿Por qué Angular y no React? | Angular trae rutas, formularios y HTTP de serie y obliga a una estructura común. En un equipo de 4, todos escribimos igual y en TypeScript, como la API. |
| ¿Angular no es pesado para el móvil? | Lo compilamos a archivos estáticos, usamos carga diferida y Lighthouse CI frena cualquier cambio que empeore el rendimiento móvil. |
| ¿Por qué el frontend en Vercel y no en vuestro Nginx? | Son archivos estáticos: una CDN los entrega más rápido en el móvil que un solo servidor, y Vercel nos da una preview por Pull Request. Aun así, nuestro Nginx podría servirlos. |
| ¿Por qué Vercel y no Netlify? | Hacen casi lo mismo. Vercel detecta Angular solo, comenta la preview en cada Pull Request y tiene guía oficial para Angular. |
| ¿Vercel gratis se puede usar? | Sí: el plan Hobby es para proyectos personales y no comerciales ([Vercel](https://vercel.com/docs/limits/fair-use-guidelines)), como este. Si PadelSlot fuera comercial, pasaríamos a Pro (20 \$ por desarrollador al mes, [Vercel](https://vercel.com/docs/plans/hobby)). |
| ¿Y si el repositorio fuera privado? | El plan Hobby bloquearía los despliegues de los commits de los otros 3\. Por eso el repositorio es público y las claves van en secretos de GitHub. |
| El plan pide la base de datos en una red privada. ¿La tenéis? | La API está en la red interna de Docker, sin puertos abiertos. La base de datos solo acepta la IP de nuestro servidor, con conexión cifrada y RLS. |
| ¿Por qué Supabase si solo la usáis como base de datos? | Porque es un DBaaS relacional gratuito, nos da el login hecho y seguro, y se integra con Docker y con Git. |
| ¿Por qué no Firebase? | Su base de datos principal es NoSQL. Evitar dobles reservas habría que programarlo a mano; en PostgreSQL basta con una restricción única. |
| ¿Y si alguien consigue la clave pública de Supabase? | No le sirve de nada: con RLS, la base de datos solo deja a cada usuario leer y cambiar sus propias filas. |
| ¿Por qué un servidor de Oracle Cloud si descartasteis la base de datos de Oracle? | Solo usamos su máquina virtual gratuita, la más potente que hay sin pagar. Lo que descartamos fue Oracle como base de datos, porque no trae login y su conexión es más compleja. |
| ¿Qué pasa si Supabase cierra o se cae? | Es PostgreSQL estándar: con nuestras copias semanales lo montamos en otro PostgreSQL, o en el propio Supabase instalado con Docker en nuestro servidor. |
| ¿Cómo protege Nginx la API de ataques? | Con un límite de 10 peticiones por segundo por IP (error 429 si se pasa), CORS solo para nuestro dominio y solo HTTPS. |
| ¿Por qué Rolling Update y no Blue/Green? | Blue/Green duplica todo el entorno. Con dos copias de la API conseguimos lo mismo (cero caídas) con la mitad de recursos. |

## **Fuentes**

* [Angular: versiones y soporte](https://angular.dev/reference/releases)  
* [Angular v22 – InfoQ](https://infoq.com/news/2026/08/angular-v22-released)  
* [Vercel: desplegar Angular](https://vercel.com/guides/deploying-angular-with-vercel)  
* [Vercel: Angular](https://vercel.com/solutions/angular)  
* [Vercel: plan Hobby](https://vercel.com/docs/plans/hobby)  
* [Vercel: Fair Use Guidelines](https://vercel.com/docs/limits/fair-use-guidelines)  
* [Vercel for GitHub (previews por Pull Request)](https://vercel.com/docs/git/vercel-for-github)  
* [Vercel: commits que no despliegan (repositorios privados)](https://vercel.com/kb/guide/commits-not-triggering-deployments)  
* [Oracle: Always Free Resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/resourceref.htm)  
* [Docker en Oracle Cloud Free Tier – OneUptime](https://oneuptime.com/blog/post/2026-02-08-how-to-set-up-docker-on-an-oracle-cloud-free-tier-instance/markdown)  
* [Nginx como proxy inverso: TLS y rate limiting – the47network](https://the47network.com/blog/nginx-reverse-proxy.html)  
* [Nginx Reverse Proxy con Docker – Techy Owls](https://techy-owls.pages.dev/blog/nginx-reverse-proxy-docker/)  
* [Nginx en producción y Let's Encrypt – Ecosire](https://ecosire.com/blog/nginx-production-configuration)  
* [Supabase: precios y límites](https://supabase.com/docs/pricing)  
* [Supabase: Auth](https://supabase.com/docs/guides/auth)  
* [Supabase: Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)  
* [Supabase: Self-hosting con Docker](https://supabase.com/docs/guides/self-hosting)  
* [Supabase: Production Checklist](https://supabase.com/docs/guides/platform/going-into-prod)  
* [Supabase: Database Backups](https://supabase.com/docs/guides/database/backups)  
* [Supabase: CLI](https://supabase.com/docs/guides/cli/getting-started)  
* [Supabase: CI/CD workflows](https://supabase.com/docs/guides/cli/cicd-workflows)  
* [Supabase: Managing Environments](https://supabase.com/docs/guides/cli/managing-environments)  
* [Supabase security guide – NHI](https://nhimg.org/supabase-security-guide)  
* [Conectar a Supabase (pooler e IPv4) – DBConvert](https://dbconvert.com/docs/connect-supabase/)  
* [Supabase free tier limits 2026 – Automation Atlas](https://automationatlas.io/answers/supabase-free-tier-limits-2026/)

[image1]: <data:image/png;base64,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>