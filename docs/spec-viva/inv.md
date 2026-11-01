# Cuentas y acceso

## Purpose

Permitir que una persona cree una cuenta en FlowSync, entre con su email y contraseña, mantenga la sesión abierta entre visitas y consulte su perfil, de modo que el resto de la aplicación solo sea accesible para quien se ha identificado.

## Requirements

### Requirement: Formato común de las respuestas de la API

La API SHALL responder siempre en JSON bajo el prefijo `/api/v1`. Las respuestas correctas con contenido SHALL envolver su carga útil en un objeto `{ "data": ... }`, y los errores de validación, de credenciales y de autenticación SHALL devolver un objeto `{ "errors": [ ... ] }` donde cada error trae al menos `message`.

#### Scenario: Errores fuera de la validación y la autenticación
- **WHEN** un cliente pide una ruta de la API que no existe, o envía un cuerpo JSON que no se puede leer
- **THEN** la API responde en JSON (`404` o `400` respectivamente) con un objeto de error que tiene `message` pero no la clave `errors`, y fuera de producción incluye además detalles técnicos del fallo

#### Scenario: Respuesta correcta envuelta en data
- **WHEN** un cliente hace una petición válida a un endpoint de cuentas que devuelve datos (registro, inicio de sesión o perfil)
- **THEN** la respuesta lleva el contenido dentro de la clave `data`

#### Scenario: JSON aunque el cliente pida otro formato
- **WHEN** un cliente envía una petición con la cabecera `Accept: text/html` a un endpoint de la API
- **THEN** la respuesta, sea de éxito o de error, llega igualmente en JSON

#### Scenario: Errores de validación desglosados por campo
- **WHEN** un cliente envía un cuerpo que no supera la validación
- **THEN** la API responde `422` con un elemento en `errors` por cada regla incumplida, cada uno con `message`, `rule` y `field`, y con `meta` cuando la regla tiene un límite (por ejemplo `{ "min": 8 }`, `{ "max": 32 }` u `{ "otherField": "password" }`)

### Requirement: Datos públicos de un usuario

Siempre que la API devuelva un usuario SHALL exponer exactamente `id`, `fullName`, `email`, `createdAt`, `updatedAt` e `initials`, y MUST NOT exponer la contraseña ni ningún derivado de ella.

#### Scenario: Usuario con nombre de dos o más palabras
- **WHEN** la API devuelve un usuario cuyo `fullName` es "Ada Byron King"
- **THEN** `initials` vale "AB": la inicial de la primera y de la segunda palabra, en mayúsculas

#### Scenario: Usuario con nombre de una sola palabra
- **WHEN** la API devuelve un usuario cuyo `fullName` es "Ada"
- **THEN** `initials` vale "AD": las dos primeras letras del nombre, en mayúsculas

#### Scenario: Usuario sin nombre
- **WHEN** la API devuelve un usuario con `fullName` a `null` y email "ada@example.com"
- **THEN** `initials` vale "AE": la primera letra de lo que va antes de la arroba y la primera de lo que va después, en mayúsculas

#### Scenario: Sin rastro de la contraseña
- **WHEN** la API devuelve un usuario en cualquier respuesta
- **THEN** el objeto no contiene ningún campo con la contraseña

### Requirement: Registro de cuenta por API

El sistema SHALL permitir crear una cuenta con `POST /api/v1/auth/signup` sin estar autenticado, enviando `fullName`, `email`, `password` y `passwordConfirmation`. Al crearla SHALL devolver el usuario y un token de acceso, de modo que la cuenta nueva queda ya con la sesión iniciada.

#### Scenario: Registro correcto
- **WHEN** un cliente sin sesión envía un email válido y no registrado, una contraseña de entre 8 y 32 caracteres, la misma contraseña en `passwordConfirmation` y un `fullName` (texto o `null`)
- **THEN** la API responde `200` con `data.user` (los datos públicos del usuario recién creado) y `data.token` (un token opaco que empieza por `oat_`)

#### Scenario: El nombre es opcional pero la clave es obligatoria
- **WHEN** un cliente envía `fullName: null`
- **THEN** la cuenta se crea con `fullName` a `null`

#### Scenario: Falta la clave del nombre
- **WHEN** un cliente omite por completo la clave `fullName` del cuerpo
- **THEN** la API responde `422` con un error de regla `required` sobre el campo `fullName`

#### Scenario: Nombre en blanco
- **WHEN** un cliente envía como `fullName` una cadena formada solo por espacios
- **THEN** la cuenta se crea con `fullName` a `null`

#### Scenario: Email ya registrado
- **WHEN** un cliente intenta registrarse con un email que ya pertenece a otra cuenta
- **THEN** la API responde `422` con un error de regla `database.unique` sobre el campo `email` y no crea ninguna cuenta

#### Scenario: Email con formato inválido
- **WHEN** un cliente envía como `email` un texto que no es una dirección de correo válida
- **THEN** la API responde `422` con un error de regla `email` sobre el campo `email`

#### Scenario: Email demasiado largo
- **WHEN** un cliente envía un `email` de más de 254 caracteres
- **THEN** la API responde `422` con un error sobre el campo `email` y no crea ninguna cuenta

#### Scenario: Campos enviados vacíos o solo con espacios
- **WHEN** un cliente envía `email`, `password` o `passwordConfirmation` como cadena vacía o formada solo por espacios
- **THEN** la API los trata como ausentes y responde `422` con un error de regla `required` sobre cada uno

#### Scenario: Espacios alrededor de los valores
- **WHEN** un cliente envía un valor de texto (incluida la contraseña) con espacios al principio o al final
- **THEN** la API descarta esos espacios antes de validar y guardar, de modo que la contraseña queda registrada sin ellos

#### Scenario: Email que solo difiere en mayúsculas
- **WHEN** un cliente se registra con un email que coincide con el de una cuenta existente salvo en el uso de mayúsculas y minúsculas
- **THEN** la API lo considera un email distinto y crea una cuenta nueva

#### Scenario: Contraseña demasiado corta o demasiado larga
- **WHEN** un cliente envía una `password` de menos de 8 caracteres o de más de 32
- **THEN** la API responde `422` con un error de regla `minLength` (con `meta.min` 8) o `maxLength` (con `meta.max` 32) sobre `password`, y el mismo tipo de error sobre `passwordConfirmation` si también la incumple

#### Scenario: Las contraseñas no coinciden
- **WHEN** un cliente envía `password` y `passwordConfirmation` válidas en longitud pero distintas entre sí
- **THEN** la API responde `422` con un error de regla `sameAs` sobre el campo `passwordConfirmation`

#### Scenario: Cuerpo vacío
- **WHEN** un cliente envía un cuerpo sin ningún campo
- **THEN** la API responde `422` con un error de regla `required` por cada uno de `fullName`, `email`, `password` y `passwordConfirmation`

### Requirement: Inicio de sesión por API

El sistema SHALL permitir iniciar sesión con `POST /api/v1/auth/login` enviando `email` y `password`. Con credenciales correctas SHALL devolver el usuario y un token de acceso nuevo; con credenciales incorrectas MUST NOT revelar si el email existe.

#### Scenario: Credenciales correctas
- **WHEN** un cliente envía el email y la contraseña de una cuenta existente
- **THEN** la API responde `200` con `data.user` (datos públicos del usuario) y `data.token` (un token opaco nuevo que empieza por `oat_`)

#### Scenario: Contraseña incorrecta o email desconocido
- **WHEN** un cliente envía un email bien formado con una contraseña que no le corresponde, o un email que no pertenece a ninguna cuenta
- **THEN** la API responde `400` con `errors: [{ "message": "Invalid user credentials" }]`, idéntico en ambos casos y sin indicar el campo

#### Scenario: Email con distinto uso de mayúsculas
- **WHEN** un cliente envía el email de una cuenta existente escrito con mayúsculas y minúsculas distintas a las del registro
- **THEN** la API lo trata como credenciales incorrectas y responde `400`

#### Scenario: Cuerpo mal formado
- **WHEN** un cliente envía un `email` que no es una dirección válida o no envía `password`
- **THEN** la API responde `422` con un error por cada campo incumplido (reglas `email` o `required`)

#### Scenario: Varias sesiones simultáneas
- **WHEN** la misma cuenta inicia sesión dos veces y obtiene dos tokens
- **THEN** ambos tokens son válidos a la vez, y cerrar la sesión con uno no invalida el otro

### Requirement: Acceso autenticado mediante token

Los endpoints bajo `/api/v1/account` SHALL exigir un token de acceso válido en la cabecera `Authorization: Bearer <token>`. Los tokens SHALL seguir siendo válidos hasta que se cierre la sesión con ellos; no caducan por tiempo.

#### Scenario: Petición sin token
- **WHEN** un cliente llama a un endpoint de `/api/v1/account` sin cabecera `Authorization`
- **THEN** la API responde `401` con `errors: [{ "message": "Unauthorized access" }]`

#### Scenario: Token inválido o revocado
- **WHEN** un cliente llama a un endpoint de `/api/v1/account` con un token inventado o con uno cuya sesión ya se cerró
- **THEN** la API responde `401` con `errors: [{ "message": "Unauthorized access" }]`

### Requirement: Consulta del perfil por API

El sistema SHALL devolver los datos del usuario autenticado en `GET /api/v1/account/profile`.

#### Scenario: Perfil con token válido
- **WHEN** un cliente llama a `GET /api/v1/account/profile` con un token válido
- **THEN** la API responde `200` con `data` conteniendo los datos públicos del usuario dueño del token

### Requirement: Cierre de sesión por API

El sistema SHALL permitir cerrar la sesión con `POST /api/v1/account/logout`, invalidando únicamente el token usado en esa petición.

#### Scenario: Cierre de sesión correcto
- **WHEN** un cliente llama a `POST /api/v1/account/logout` con un token válido
- **THEN** la API responde `200` con `{ "message": "Logged out successfully" }` (sin envoltorio `data`), y a partir de ese momento ese token recibe `401` en cualquier endpoint de `/api/v1/account`

#### Scenario: Cierre de sesión con un token ya revocado
- **WHEN** un cliente repite `POST /api/v1/account/logout` con un token cuya sesión ya cerró
- **THEN** la API responde `401` con `errors: [{ "message": "Unauthorized access" }]`

### Requirement: Navegación según el estado de la sesión

La aplicación web SHALL ofrecer tres pantallas en este vertical: inicio de sesión (`/login`), registro (`/register`) y perfil (`/profile`). El perfil SHALL estar reservado a personas con sesión iniciada, y las pantallas de inicio de sesión y registro SHALL estar reservadas a personas sin sesión.

#### Scenario: Visitante sin sesión intenta ver el perfil
- **WHEN** una persona sin sesión abre `/profile`
- **THEN** la aplicación la lleva a `/login`

#### Scenario: Persona con sesión abre login o registro
- **WHEN** una persona con la sesión iniciada abre `/login` o `/register`
- **THEN** la aplicación la lleva a `/profile`

#### Scenario: Dirección desconocida
- **WHEN** una persona abre cualquier dirección que no sea `/login`, `/register` ni `/profile` (incluida la raíz `/`)
- **THEN** la aplicación la lleva a `/profile`, y desde ahí a `/login` si no tiene sesión

#### Scenario: Comprobando una sesión recordada
- **WHEN** una persona abre cualquier pantalla y el navegador tiene recordada una sesión que aún se está comprobando con el servidor
- **THEN** ve solo un indicador de carga ("Cargando…" para lectores de pantalla) y no se la redirige hasta que termina la comprobación

### Requirement: Pantalla de registro

La aplicación web SHALL ofrecer en `/register` un formulario para crear cuenta, bajo el título "FlowSync", con el encabezado "Crea tu cuenta" y la descripción "Regístrate para empezar a organizar el trabajo del equipo.".

#### Scenario: Contenido del formulario
- **WHEN** una persona sin sesión abre `/register`
- **THEN** ve los campos "Nombre completo (opcional)", "Email", "Contraseña" (con la ayuda "Entre 8 y 32 caracteres.") y "Repite la contraseña", el botón "Crear cuenta" y el enlace "¿Ya tienes cuenta? Inicia sesión", que lleva a `/login`

#### Scenario: Registro correcto
- **WHEN** una persona rellena el formulario con datos válidos y pulsa "Crear cuenta"
- **THEN** el botón muestra "Creando cuenta…" y queda deshabilitado mientras espera, la cuenta se crea, la sesión queda iniciada y recordada en el navegador, y la persona aterriza en `/profile`

#### Scenario: Nombre vacío
- **WHEN** una persona deja "Nombre completo" vacío o solo con espacios y envía el formulario
- **THEN** la cuenta se crea sin nombre

#### Scenario: Contraseñas distintas detectadas antes de enviar
- **WHEN** una persona escribe valores diferentes en "Contraseña" y "Repite la contraseña" y pulsa "Crear cuenta"
- **THEN** aparece "Las contraseñas no coinciden." bajo "Repite la contraseña" y no se envía nada al servidor

#### Scenario: Errores de validación del servidor
- **WHEN** el servidor rechaza el registro por datos inválidos
- **THEN** cada error aparece en castellano bajo su campo (sustituyendo la ayuda de la contraseña si el error es de ese campo), sin aviso general, con estos textos: "Ese email ya está registrado. Inicia sesión en su lugar.", "Introduce una dirección de email válida.", "Las contraseñas no coinciden.", "Falta rellenar el email/la contraseña/la confirmación de la contraseña.", "la contraseña/la confirmación de la contraseña debe tener al menos 8 caracteres." o "la contraseña/la confirmación de la contraseña no puede superar los 32 caracteres."; cualquier otra regla incumplida se muestra como "Revisa <el campo>."

#### Scenario: Error de validación sobre un dato que no está en pantalla
- **WHEN** el servidor rechaza el envío con algún error sobre un campo que el formulario no muestra
- **THEN** el mensaje del primer error aparece como aviso general sobre el formulario, además de los errores que sí tienen campo en pantalla

#### Scenario: Fallo inesperado en el propio navegador
- **WHEN** el envío falla por un motivo que no es una respuesta del servidor ni la falta de conexión
- **THEN** la persona ve el aviso general "Algo ha ido mal. Inténtalo de nuevo."

#### Scenario: Solo el primer error por campo
- **WHEN** el servidor devuelve varios errores sobre el mismo campo
- **THEN** bajo ese campo se muestra únicamente el primero

#### Scenario: Servidor inalcanzable
- **WHEN** una persona envía el formulario y el servidor no responde
- **THEN** ve un aviso general en rojo sobre el formulario con "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." y el botón vuelve a estar disponible

#### Scenario: Error inesperado del servidor
- **WHEN** una persona envía el formulario y el servidor falla con un error no previsto
- **THEN** ve un aviso general con "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento."

#### Scenario: Nuevo intento limpia los errores
- **WHEN** una persona vuelve a pulsar "Crear cuenta" tras un intento fallido
- **THEN** los errores del intento anterior desaparecen mientras se procesa el nuevo

### Requirement: Pantalla de inicio de sesión

La aplicación web SHALL ofrecer en `/login` un formulario de acceso, bajo el título "FlowSync", con el encabezado "Inicia sesión" y la descripción "Entra con tu cuenta para volver a tus tareas.".

#### Scenario: Contenido del formulario
- **WHEN** una persona sin sesión abre `/login`
- **THEN** ve los campos "Email" y "Contraseña", el botón "Entrar" y el enlace "¿Aún no tienes cuenta? Crea una", que lleva a `/register`

#### Scenario: Inicio de sesión correcto
- **WHEN** una persona introduce el email y la contraseña de su cuenta y pulsa "Entrar"
- **THEN** el botón muestra "Entrando…" y queda deshabilitado mientras espera, la sesión queda iniciada y recordada en el navegador, y la persona aterriza en `/profile`

#### Scenario: Credenciales incorrectas
- **WHEN** una persona introduce un email o una contraseña que no corresponden a ninguna cuenta y pulsa "Entrar"
- **THEN** ve un aviso general en rojo con "El email o la contraseña no son correctos." y puede volver a intentarlo

#### Scenario: Datos mal formados
- **WHEN** una persona deja vacío el email o la contraseña, o escribe un email con formato inválido, y pulsa "Entrar"
- **THEN** ve el error en castellano bajo el campo correspondiente ("Falta rellenar el email.", "Falta rellenar la contraseña.", "Introduce una dirección de email válida.") y ningún aviso general nuevo; si ya se estaba mostrando el motivo de una sesión perdida, ese aviso sigue visible

#### Scenario: Servidor inalcanzable o con fallo
- **WHEN** una persona pulsa "Entrar" y el servidor no responde o falla de forma inesperada
- **THEN** ve el aviso general "No se pudo conectar con el servidor. Comprueba que el backend está arrancado." o "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento.", respectivamente

### Requirement: Sesión recordada entre visitas

La aplicación web SHALL recordar la sesión en el navegador tras registrarse o iniciar sesión, y al volver a abrirla SHALL comprobar con el servidor que la sesión sigue siendo válida antes de dar acceso. Si la sesión recordada se pierde, la pantalla de inicio de sesión SHALL explicar el motivo.

#### Scenario: Sesión recordada válida
- **WHEN** una persona que inició sesión cierra la pestaña o recarga la página y vuelve a abrir la aplicación
- **THEN** tras el indicador de carga entra directamente con la sesión iniciada, sin volver a escribir sus credenciales

#### Scenario: Sesión recordada rechazada por el servidor
- **WHEN** una persona abre la aplicación con una sesión recordada que el servidor ya no reconoce
- **THEN** la aplicación olvida esa sesión, la lleva a `/login` y muestra el aviso "Tu sesión ha caducado. Vuelve a iniciar sesión."

#### Scenario: Servidor caído al restaurar la sesión
- **WHEN** una persona abre la aplicación con una sesión recordada y el servidor no responde
- **THEN** la lleva a `/login` con el aviso "No se pudo conectar con el servidor. Comprueba que el backend está arrancado.", pero conserva la sesión recordada, de modo que al recargar con el servidor ya disponible vuelve a entrar sin credenciales

#### Scenario: Error inesperado al restaurar la sesión
- **WHEN** una persona abre la aplicación con una sesión recordada y el servidor falla de forma inesperada
- **THEN** la lleva a `/login` con el aviso "Algo ha ido mal en el servidor. Inténtalo de nuevo en un momento." y conserva la sesión recordada

#### Scenario: El error del intento actual tiene prioridad
- **WHEN** en `/login` se está mostrando el motivo de la pérdida de sesión y la persona hace un intento de acceso que falla con un aviso general
- **THEN** se muestra el aviso del intento actual en lugar del motivo anterior

#### Scenario: El motivo desaparece al entrar
- **WHEN** tras perder la sesión la persona inicia sesión o se registra con éxito
- **THEN** el motivo de la pérdida de sesión deja de mostrarse

### Requirement: Pantalla de perfil

La aplicación web SHALL mostrar en `/profile` los datos de la persona con sesión iniciada y SHALL permitirle cerrar la sesión.

#### Scenario: Datos visibles
- **WHEN** una persona con sesión abre `/profile`
- **THEN** ve un círculo con sus iniciales, su nombre completo (o "Sin nombre" si no tiene), su email, la línea "Miembro desde" con la fecha de alta en formato largo en castellano (por ejemplo "1 de noviembre de 2026") y el botón "Cerrar sesión"

#### Scenario: Cerrar sesión
- **WHEN** una persona pulsa "Cerrar sesión"
- **THEN** la sesión se olvida en el navegador y la persona aterriza de inmediato en `/login` sin ningún aviso, sin esperar al servidor, al que se pide en segundo plano que invalide la sesión

#### Scenario: Cerrar sesión sin conexión con el servidor
- **WHEN** una persona pulsa "Cerrar sesión" y el servidor no responde o rechaza la petición
- **THEN** la sesión se cierra igualmente en el navegador, la persona aterriza en `/login` y no ve ningún error
