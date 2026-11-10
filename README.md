# PHP2Shell — análisis y laboratorio de acceso al sistema

## Introducción a PHP2Shell

### ¿Qué es PHP2Shell?

PHP2Shell es el nombre utilizado para identificar un conjunto de dos vulnerabilidades críticas de seguridad que afectan al intérprete de PHP. Fueron divulgadas públicamente el 21 de octubre de 2026 y se identifican como:

- **CVE-2026-109618:** corrupción de memoria durante el tratamiento de errores por agotamiento de memoria (*Out of Memory*, OOM).
- **CVE-2026-109619:** utilización de referencias a memoria que ya fue liberada, debido a la persistencia incorrecta de estados internos (*Use After Free*, UAF).

Ambas vulnerabilidades pueden permitir que un atacante ejecute código arbitrario en un servidor que utiliza una versión afectada de PHP, incluso sin disponer de credenciales de acceso.

Este tipo de vulnerabilidad se conoce como **Remote Code Execution (RCE)** o ejecución remota de código.

Su particularidad es que el problema no se encuentra necesariamente en el código de una aplicación web, sino en el propio intérprete de PHP.

Por lo tanto, aplicaciones desarrolladas con PHP, como WordPress, Drupal o Joomla, pueden quedar expuestas aunque su código no contenga la vulnerabilidad.

### ¿Cómo funciona PHP2Shell?

Para comprender el ataque, primero debemos considerar cómo funciona PHP cuando recibe una solicitud HTTP.

Cuando un cliente realiza una petición a una aplicación web, el servidor utiliza PHP para procesar los datos recibidos, ejecutar la lógica correspondiente y generar una respuesta.

Durante este proceso, PHP reserva memoria para almacenar variables, estructuras internas y datos de la solicitud. Una vez finalizado el procesamiento, debe liberar los recursos utilizados.

PHP2Shell aprovecha errores en la administración de esa memoria.

#### CVE-2026-109618: corrupción de memoria

Esta vulnerabilidad está relacionada con el tratamiento de errores cuando PHP agota la memoria disponible.

Normalmente, cuando una operación requiere más memoria de la permitida, PHP interrumpe su ejecución y comienza a liberar los recursos utilizados.

Sin embargo, en determinadas circunstancias, el error puede producirse mientras PHP todavía está inicializando estructuras internas.

Como consecuencia, algunas estructuras quedan incompletas. Posteriormente, durante las operaciones de limpieza, PHP puede intentar utilizarlas como si fueran válidas.

Esto puede provocar corrupción de memoria.

Un atacante que consiga controlar las condiciones que producen este comportamiento podría manipular estructuras internas del intérprete y, bajo determinadas condiciones, desviar su ejecución.

#### CVE-2026-109619: uso de memoria liberada

La segunda vulnerabilidad está relacionada con el estado interno que PHP mantiene durante el procesamiento de las solicitudes.

Algunas estructuras globales pueden conservar referencias a regiones de memoria pertenecientes a una solicitud anterior.

El problema aparece cuando esa memoria ya fue liberada, pero las referencias continúan existiendo.

Si una solicitud posterior utiliza esas referencias antes de que sean inicializadas correctamente, PHP puede acceder a memoria que ya no le pertenece.

Este comportamiento se conoce como *Use After Free* (UAF).

En determinadas condiciones, un atacante puede aprovecharlo para alterar el funcionamiento del intérprete y conseguir ejecución de código.

### ¿Cómo puede explotarse remotamente?

El ataque comienza cuando un cliente envía una solicitud HTTP especialmente construida a un servidor que utiliza una versión vulnerable de PHP.

La solicitud puede utilizar el formato `multipart/form-data`, empleado habitualmente para enviar formularios y archivos. PHP procesa estos datos antes de que la aplicación ejecute su propia lógica.

Durante ese procesamiento, el intérprete reserva memoria para almacenar los datos recibidos y las estructuras internas necesarias para administrarlos.

**El ataque busca provocar una situación anómala durante la administración de esa memoria.**

En determinadas condiciones, PHP puede liberar una región de memoria mientras todavía conserva referencias a ella. Esto produce una situación conocida como *Use After Free* (UAF), donde el programa intenta utilizar memoria que ya fue liberada.

El problema es que esa región puede haber sido reutilizada para almacenar otros datos. Si el intérprete continúa utilizando las referencias anteriores, puede interpretar el contenido de esa memoria como si todavía perteneciera a la estructura original.

**Aquí aparece la posibilidad de explotación.**

Un atacante que consiga influir sobre el contenido de esa región podría hacer que PHP interprete datos manipulados como estructuras internas válidas.

En el caso de PHP2Shell, la explotación busca manipular una estructura denominada `HashTable`, utilizada internamente por PHP para administrar arrays, y alterar una referencia a una función que se ejecuta durante la destrucción de esa estructura.

Si el procedimiento tiene éxito, el intérprete termina invocando una función distinta de la prevista, lo que puede permitir la ejecución de comandos del sistema operativo.

Esta situación se conoce como *Remote Code Execution* (RCE).

Sin embargo, **existe una diferencia entre provocar la vulnerabilidad y conseguir explotarla**.

El atacante necesita controlar con suficiente precisión la disposición y el contenido de la memoria. Si no lo consigue, el proceso puede finalizar inesperadamente o simplemente no producir el resultado buscado.

Por eso, una instalación que utiliza una versión vulnerable de PHP no necesariamente puede explotarse mediante cualquier prueba de concepto disponible. La explotación también depende de las condiciones concretas del entorno y de la capacidad del ataque para manipular las estructuras internas del intérprete.

### ¿Qué puede conseguir un atacante?

Si la explotación tiene éxito, el atacante puede ejecutar código dentro del contexto del proceso PHP comprometido.

Esto puede permitir:

- Ejecutar comandos del sistema operativo con los permisos del proceso PHP.
- Leer archivos accesibles para ese proceso.
- Acceder a información de configuración y credenciales de aplicaciones.
- Modificar archivos sobre los que el proceso tenga permisos de escritura.
- Interactuar con otros servicios a los que el proceso pueda acceder.
- Interrumpir el funcionamiento del servicio.

El alcance de una explotación depende de los permisos del usuario que ejecuta PHP y de las restricciones existentes en el sistema operativo.

Por ejemplo, en un servidor Apache sobre GNU/Linux, el proceso puede ejecutarse con un usuario de servicio como `www-data`.

Conseguir ejecutar comandos como `www-data` no significa obtener automáticamente permisos de administrador o `root`.

### Versiones afectadas

Las vulnerabilidades fueron corregidas el 21 de octubre de 2026 mediante actualizaciones de seguridad de PHP.

| Rama PHP | Primera versión corregida |
| -------- | ------------------------- |
| PHP 8.2  | 8.2.35                    |
| PHP 8.3  | 8.3.36                    |
| PHP 8.4  | 8.4.27                    |
| PHP 8.5  | 8.5.12                    |

Las versiones anteriores también pueden estar afectadas. Las ramas que ya finalizaron su ciclo de soporte no recibieron necesariamente una actualización correctiva.

En distribuciones GNU/Linux, además, debe considerarse que los mantenedores pueden incorporar correcciones de seguridad sin modificar el número principal de versión del paquete.

Por ese motivo, la versión anunciada por un servidor es un indicador útil, pero no siempre resulta suficiente para determinar su estado de seguridad.

### ¿Por qué es importante esta vulnerabilidad?

PHP2Shell permite estudiar una categoría de vulnerabilidades que resulta especialmente relevante en la seguridad de aplicaciones web: los errores de administración de memoria dentro de los componentes que ejecutan las aplicaciones.

A diferencia de vulnerabilidades como SQL Injection o Cross-Site Scripting (XSS), donde el problema suele encontrarse en la forma en que una aplicación procesa determinados datos, PHP2Shell afecta al funcionamiento interno del intérprete.

Esto introduce una distinción fundamental: **una aplicación puede estar correctamente desarrollada y, aun así, encontrarse expuesta por una vulnerabilidad en uno de sus componentes subyacentes**.

También permite comprender por qué identificar una versión vulnerable, detectar un comportamiento anómalo y conseguir ejecutar código son resultados diferentes que deben comprobarse de manera independiente.

Referencia de las herramientas y del laboratorio original: [RedSideSecurity/php2shell](https://github.com/RedSideSecurity/php2shell).

## Herramientas de RedSide Security para analizar PHP2Shell

### Presentación del proyecto

El equipo de **RedSide Security** desarrolló un conjunto de herramientas para estudiar, detectar y comprobar la explotación de las vulnerabilidades PHP2Shell (CVE-2026-109618 y CVE-2026-109619).

El proyecto se encuentra disponible públicamente en GitHub:

https://github.com/RedSideSecurity/php2shell

Su propósito es proporcionar un entorno de experimentación que permita comprender cómo se manifiestan estas vulnerabilidades y bajo qué condiciones pueden aprovecharse para conseguir ejecución remota de comandos (RCE).

A diferencia de una demostración preparada para garantizar un resultado exitoso, el proyecto busca reproducir condiciones similares a las de un servidor real. Para ello, evita utilizar archivos previamente preparados, directorios compartidos con el sistema anfitrión o vulnerabilidades adicionales que faciliten artificialmente la explotación.

### ¿Qué incluye el repositorio?

El proyecto organiza sus herramientas en tres componentes principales.

**1. Laboratorio de servidores PHP**

Proporciona cuatro entornos basados en Apache y distintas versiones de PHP:

| Versión | Resultado documentado por el autor |
|---|---|
| PHP 8.2.34 | Vulnerabilidad detectada, sin RCE mediante la técnica utilizada |
| PHP 8.3.35 | RCE confirmada |
| PHP 8.4.26 | RCE confirmada |
| PHP 8.5.11 | RCE confirmada |

Estos entornos permiten observar cómo una misma técnica puede producir resultados diferentes según la versión y las condiciones de ejecución.

**2. Herramientas de detección**

Incluye un escáner denominado `p2s_scan.py`, diseñado para identificar indicios de vulnerabilidad mediante solicitudes HTTP especialmente construidas.

El escáner analiza las respuestas del servidor y clasifica los resultados según las evidencias obtenidas.

Su objetivo es determinar si existen señales compatibles con la vulnerabilidad, sin intentar ejecutar comandos en el sistema.

**3. Herramientas de explotación y verificación**

El repositorio incluye herramientas para comprobar si las condiciones detectadas pueden aprovecharse para ejecutar comandos.

Entre ellas se encuentra `p2s.py`, que automatiza diferentes etapas del procedimiento, como la identificación de la versión de PHP, la selección de offsets de memoria y la comprobación de ejecución.

También proporciona herramientas para trabajar con compilaciones específicas de PHP y analizar los parámetros necesarios para reproducir el ataque.

### ¿Qué diferencia a este laboratorio?

Uno de los aspectos más importantes del proyecto es que distingue entre tres resultados:

- **Detección:** identificar indicios compatibles con una versión vulnerable de PHP.
- **Verificación:** comprobar si es posible ejecutar una operación mediante la vulnerabilidad.
- **Explotación:** demostrar que el atacante puede ejecutar comandos y obtener resultados del sistema comprometido.

Esta distinción resulta fundamental para interpretar correctamente las pruebas.

Un servidor puede utilizar una versión vulnerable de PHP y, sin embargo, no permitir que una prueba de concepto concreta consiga ejecutar comandos.

Esto no demuestra que el servidor sea invulnerable. Significa que la explotación no pudo confirmarse bajo las condiciones utilizadas.

Por esa razón, las herramientas de RedSide Security resultan útiles tanto para reproducir ataques exitosos como para investigar las causas de una explotación fallida.

## Laboratorio 1 — WordPress con PHP 8.3.33

### 1. Contexto y elección de la imagen

El laboratorio de RedSideSecurity prepara **cuatro objetivos Apache/PHP**, uno por rama (8.2, 8.3, 8.4 y 8.5). Según el README original, las compilaciones utilizadas y sus resultados son:

| PHP en el laboratorio original | Resultado publicado |
| --- | --- |
| 8.2.34 | Detectado por el escáner, sin RCE mediante la cadena ensayada |
| 8.3.35 | RCE confirmada |
| 8.4.26 | RCE confirmada |
| 8.5.11 | RCE confirmada |

Para probar el comportamiento en una aplicación real en vez de la página mínima del laboratorio original, buscamos una **imagen oficial de WordPress con Apache y una versión de PHP perteneciente a alguna de las ramas afectadas**. Elegimos una imagen con **PHP 8.3.33**, anterior a la versión corregida 8.3.36 que señala el README original.

La imagen queda fijada por digest para evitar que una etiqueta móvil cambie la compilación:

```text
wordpress@sha256:65919a9ca10940feb10d9400fead0d639bf86241f47c91e2b9ea4703aa8452cf
```

**Distinción importante:** PHP **8.3.33 no es la misma compilación** que PHP 8.3.35, la utilizada por el laboratorio original para esa rama. La coincidencia de rama y el hecho de ser anterior al parche **no implican que los offsets ni el exploit publicado funcionen sin ajustes**.

### 2. Despliegue de WordPress

Archivo de este repositorio: [`compose-wordpress-php8.3.33.yaml`](compose-wordpress-php8.3.33.yaml).

Incluye WordPress/Apache y MariaDB. No declara volúmenes en Compose; esto no excluye los volúmenes anónimos que puedan declarar las propias imágenes. El puerto HTTP se publica solamente en la interfaz local, `127.0.0.1:8080`.

Desde el directorio del repositorio:

```bash
docker compose -f compose-wordpress-php8.3.33.yaml up -d
```

Acceso HTTP: `http://127.0.0.1:8080/`.

### 3. Prueba de detección

Se utilizó el escáner del proyecto original, ejecutado desde su directorio:

```bash
python3 scanner/p2s_scan.py -u http://127.0.0.1:8080/
```

**Salida obtenida:**

```text
[*] 1 target(s), 5 thread(s)

POTENTIAL  http://127.0.0.1:8080/index.php [PHP/8.3.33]
           no crash seen; banner reports unpatched PHP/8.3.33 (verify manually)

VULNERABLE: 0  POTENTIAL: 1  SAFE: 0  UNKNOWN: 0  ERROR: 0
```

**Interpretación:** el servidor informa una versión de PHP anterior a la corrección. Sin embargo, el escáner **no observó la señal de fallo que busca** (`no crash seen`), por lo que clasificó el objetivo como `POTENTIAL`, **no** como `VULNERABLE`. El resultado no acredita por sí mismo que se pueda ejecutar código remotamente.

### 4. Prueba de RCE sin archivo de prueba

Se ejecutó el controlador del proyecto original con `--no-poc`, que intenta comprobar ejecución mediante un retraso temporal, sin generar el archivo de demostración de lectura HTTP:

```bash
python3 p2s.py -u http://127.0.0.1:8080/ --no-poc
```

**Salida obtenida:**

La cabecera HTTP de respuesta `X-Powered-By` puede indicar la tecnología que genera la respuesta. En este caso, `X-Powered-By: PHP/8.3.33` informa la versión de PHP utilizada por el servidor; es el dato que el programa emplea para identificarla, pero no demuestra por sí solo que la instalación sea explotable.

```text
[1] fingerprint: X-Powered-By: PHP/8.3.33 -> version=8.3.33
[2] offsets: 2 candidate build(s); htdata swept +-2 page(s)
[3] try ubuntu 24.04 (8.3.6-0ubuntu0.24.04.13): zif=0x2371c0 dtor=0x30dbc0 htdata=0x6a0d8 ... no (leak failed)
[3] try ubuntu 24.04 (8.3.6-0ubuntu0.24.04.13): zif=0x2371c0 dtor=0x30dbc0 htdata=0x6b0d8 ... no (leak failed)
[3] try ubuntu 24.04 (8.3.6-0ubuntu0.24.04.13): zif=0x2371c0 dtor=0x30dbc0 htdata=0x690d8 ... no (leak failed)
[3] try ubuntu 24.04 (8.3.6-0ubuntu0.24.04.13): zif=0x2371c0 dtor=0x30dbc0 htdata=0x6c0d8 ... no (leak failed)
[3] try ubuntu 24.04 (8.3.6-0ubuntu0.24.04.13): zif=0x2371c0 dtor=0x30dbc0 htdata=0x680d8 ... no (leak failed)
[3] try official php:8.3-apache (debian trixie): zif=0x3ac080 dtor=0x49ec50 htdata=0x6a0d8 ... no (leak failed)
[3] try official php:8.3-apache (debian trixie): zif=0x3ac080 dtor=0x49ec50 htdata=0x6b0d8 ... no (leak failed)
[3] try official php:8.3-apache (debian trixie): zif=0x3ac080 dtor=0x49ec50 htdata=0x690d8 ... no (leak failed)
[3] try official php:8.3-apache (debian trixie): zif=0x3ac080 dtor=0x49ec50 htdata=0x6c0d8 ... no (leak failed)
[3] try official php:8.3-apache (debian trixie): zif=0x3ac080 dtor=0x49ec50 htdata=0x680d8 ... no (leak failed)
[-] not confirmed on any candidate this run (raise --retries/--sweep, or target may be patched / wrong build).
```

**Interpretación de la salida:**

1. **Fingerprint:** la herramienta identifica PHP 8.3.33 a partir de `X-Powered-By`.
2. **Offsets:** selecciona dos compilaciones candidatas disponibles en su base de datos: Ubuntu 24.04 (PHP 8.3.6 empaquetado por Ubuntu) y `php:8.3-apache` sobre Debian Trixie.
3. **Barrido:** para cada candidata ensaya cinco variantes de `htdata`, conservando los offsets `zif` y `dtor` correspondientes.
4. **`leak failed`:** las diez pruebas fallan en la fase de filtración de direcciones de memoria. **No se llega a confirmar la ejecución del comando de prueba.**
5. **Resultado final:** `not confirmed on any candidate this run`; la ejecución remota **no fue demostrada** en este intento.

Los offsets dependen de la **compilación concreta** de `libphp.so`, no solo de la versión mayor y menor de PHP. El laboratorio original también utiliza una configuración específica de Apache, con un único proceso trabajador para facilitar un estado de memoria predecible. Ambas diferencias podrían influir en el resultado, pero **no se determinó la causa del fallo con estas salidas**.

### 5. Conclusiones de esta primera experiencia

- Se desplegó el objetivo WordPress/Apache con una imagen identificada como **PHP 8.3.33**.
- El escáner devolvió **`POTENTIAL`**, no `VULNERABLE`.
- La prueba de ejecución devolvió **`leak failed` en las diez combinaciones** y terminó sin confirmar RCE.
- La versión identificada se encuentra en el rango que el README original considera sin corregir, pero **esto no equivale a haber demostrado explotación**.
- No se verificó aún si el problema se debe a offsets distintos, al comportamiento de memoria del servidor o a otra condición.

**Estado del laboratorio WordPress:** versión potencialmente afectada identificada; ejecución remota de comandos **no confirmada**.

### 6. Líneas de investigación para intentar la explotación

Las pruebas realizadas no permitieron confirmar RCE. Las siguientes son **hipótesis de trabajo**, no procedimientos ya ejecutados ni soluciones verificadas.

1. **Identificar la compilación exacta de PHP dentro de WordPress.** Contrastar la versión anunciada en `X-Powered-By` con el binario y el módulo Apache realmente cargados. La versión 8.3.33 no identifica por sí sola los offsets necesarios.
2. **Obtener y analizar los offsets del módulo de este contenedor.** El exploit necesita constantes asociadas a la compilación de `libphp.so`; hasta ahora se probaron offsets de otras compilaciones. Examinar el módulo de la imagen y contrastarlo con las herramientas de `real-vm/` del proyecto original permitiría evaluar esa diferencia.
3. **Investigar por qué falla la filtración de memoria.** Todas las variantes terminaron en `leak failed`. Conviene distinguir entre un problema de las peticiones que disparan la condición, una disposición de memoria diferente y la ausencia de la señal que espera el exploit. Los registros de Apache/PHP y el comportamiento del proceso durante cada intento podrían aportar evidencia.
4. **Comparar la configuración de Apache/PHP con la del laboratorio original.** En particular, estudiar el número de procesos trabajadores, el MPM utilizado, `memory_limit` y las opciones de `session.upload_progress`. Cambiar una condición por vez en una copia del despliegue permitiría establecer si influye en la reproducibilidad.
5. **Separar detección de explotación.** El escáner obtuvo `POTENTIAL` sin observar su señal diferencial. Analizar el comportamiento de la petición multipart en el WordPress desplegado ayudaría a determinar si el mecanismo de detección llega a activarse, independientemente de la cadena RCE.
6. **Revisar los parámetros de prueba del exploit.** El propio controlador propone ampliar los reintentos o el barrido de `htdata`. Esa posibilidad debe evaluarse después de verificar la compilación y la fase que falla, porque más intentos no corrigen por sí solos offsets incorrectos.
7. **Comparar contra un objetivo conocido del laboratorio original.** Ejecutar las mismas pruebas en una de las compilaciones donde el autor informa RCE permitiría separar un problema de las herramientas de uno específico del despliegue de WordPress.

**Criterio para avanzar:** registrar cada cambio por separado y comprobar si modifica la clasificación del escáner, la fase de filtración de memoria o la confirmación de ejecución. No considerar explotado el sistema hasta contar con una prueba reproducible de RCE.
