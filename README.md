# Laboratorio PHP2Shell — acceso al sistema

Laboratorio de experimentación con **PHP2Shell (CVE-2026-109618 / CVE-2026-109619)** en aplicaciones PHP desplegadas en contenedores. Las pruebas se realizan sobre servicios locales y buscan diferenciar la detección de una versión afectada de la confirmación efectiva de ejecución remota de comandos (RCE).

Referencia de las herramientas y del laboratorio original: [RedSideSecurity/php2shell](https://github.com/RedSideSecurity/php2shell).

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

## Referencias

- [Laboratorio y herramientas originales — RedSideSecurity/php2shell](https://github.com/RedSideSecurity/php2shell)
- [README del proyecto original](https://github.com/RedSideSecurity/php2shell/blob/main/README.md)
- [Imagen oficial de WordPress en Docker Hub](https://hub.docker.com/_/wordpress)

## Líneas de investigación para intentar la explotación

Las pruebas realizadas no permitieron confirmar RCE. Las siguientes son **hipótesis de trabajo**, no procedimientos ya ejecutados ni soluciones verificadas.

1. **Identificar la compilación exacta de PHP dentro de WordPress.** Contrastar la versión anunciada en `X-Powered-By` con el binario y el módulo Apache realmente cargados. La versión 8.3.33 no identifica por sí sola los offsets necesarios.
2. **Obtener y analizar los offsets del módulo de este contenedor.** El exploit necesita constantes asociadas a la compilación de `libphp.so`; hasta ahora se probaron offsets de otras compilaciones. Examinar el módulo de la imagen y contrastarlo con las herramientas de `real-vm/` del proyecto original permitiría evaluar esa diferencia.
3. **Investigar por qué falla la filtración de memoria.** Todas las variantes terminaron en `leak failed`. Conviene distinguir entre un problema de las peticiones que disparan la condición, una disposición de memoria diferente y la ausencia de la señal que espera el exploit. Los registros de Apache/PHP y el comportamiento del proceso durante cada intento podrían aportar evidencia.
4. **Comparar la configuración de Apache/PHP con la del laboratorio original.** En particular, estudiar el número de procesos trabajadores, el MPM utilizado, `memory_limit` y las opciones de `session.upload_progress`. Cambiar una condición por vez en una copia del despliegue permitiría establecer si influye en la reproducibilidad.
5. **Separar detección de explotación.** El escáner obtuvo `POTENTIAL` sin observar su señal diferencial. Analizar el comportamiento de la petición multipart en el WordPress desplegado ayudaría a determinar si el mecanismo de detección llega a activarse, independientemente de la cadena RCE.
6. **Revisar los parámetros de prueba del exploit.** El propio controlador propone ampliar los reintentos o el barrido de `htdata`. Esa posibilidad debe evaluarse después de verificar la compilación y la fase que falla, porque más intentos no corrigen por sí solos offsets incorrectos.
7. **Comparar contra un objetivo conocido del laboratorio original.** Ejecutar las mismas pruebas en una de las compilaciones donde el autor informa RCE permitiría separar un problema de las herramientas de uno específico del despliegue de WordPress.

**Criterio para avanzar:** registrar cada cambio por separado y comprobar si modifica la clasificación del escáner, la fase de filtración de memoria o la confirmación de ejecución. No considerar explotado el sistema hasta contar con una prueba reproducible de RCE.
