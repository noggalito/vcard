# Creación y mantenimiento de archivos vCard

Una vCard es una tarjeta de presentación digital en un archivo estándar con extensión `.vcf` que permite compartir los datos de contacto personales de forma automática. Un código QR puede hacer referencia a un enlace URL donde se encuentra alojado el archivo para su fácil descarga. Este documento describe los pasos necesarios para la creación y mantenimiento de un archivo vCard versión 3.0

<p style="text-align: center"><img src="./assets/vcard.jpg" style="width: 25%;" alt="vCard" /></p>

## a.  Historial de cambios

- `[0.2]` – 2026-09-14
    - Incorporación del índice y correcciones menores
- `[0.1]` – 2026-04-18
    - Estructura inicial

## b. To-do

- Agregar ejemplos de vCard 4.0 (RFC 6350) y tabla comparativa con 3.0
- Documentar flujo de automatización con scripts bash

---

## Índice

### 1. Requisitos técnicos del formato

- [1.1 Codificación](#11-codificación)
- [1.2 Finales de línea](#12-finales-de-línea)
- [1.3 Plegado de líneas (line folding)](#13-plegado-de-líneas-line-folding)
- [1.4 Escape de caracteres especiales](#14-escape-de-caracteres-especiales)
- [1.5 Estructura obligatoria](#15-estructura-obligatoria)
- [1.6 Extensión del archivo](#16-extensión-del-archivo)

### 2. Propiedades de vCard 3.0

- [2.1 Propiedades de identificación](#21-propiedades-de-identificación)
- [2.2 Propiedades de dirección](#22-propiedades-de-dirección)
- [2.3 Propiedades de telecomunicaciones](#23-propiedades-de-telecomunicaciones)
- [2.4 Propiedades organizacionales](#24-propiedades-organizacionales)
- [2.5 Propiedades geográficas](#25-propiedades-geográficas)
- [2.6 Propiedades explicativas](#26-propiedades-explicativas)
- [2.7 Extensiones no estándar](#27-extensiones-no-estándar)
- [2.8 Propiedades a evitar en vCard 3.0](#28-propiedades-a-evitar-en-vcard-30)

### 3. Plantilla completa comentada

### 4. Plegado de líneas para PHOTO y LOGO

- [4.1 Comandos de plegado (sin script)](#41-comandos-de-plegado-sin-script)
- [4.2 Lógica del plegado](#42-lógica-del-plegado)
- [4.3 Flujo de trabajo recomendado](#43-flujo-de-trabajo-recomendado)
- [4.4 Tamaños recomendados para imágenes](#44-tamaños-recomendados-para-imágenes)
- [4.5 Incrustado vs. URI externa](#45-incrustado-vs-uri-externa)

### 5. Errores frecuentes y cómo evitarlos

- [5.1 Codificación UTF-8 doblemente convertida](#51-codificación-utf-8-doblemente-convertida)
- [5.2 Finales de línea incorrectos](#52-finales-de-línea-incorrectos)
- [5.3 Dirección duplicada](#53-dirección-duplicada)
- [5.4 GEO mal formado](#54-geo-mal-formado)
- [5.5 CHARSET=UTF-8 en cada propiedad](#55-charsetutf-8-en-cada-propiedad)
- [5.6 UID como URL](#56-uid-como-url)

### 6. Ejemplo completo

### 7. Compatibilidad entre vCard 3.0 y 4.0

### 8. Referencias

---

## 1. Requisitos técnicos del formato

### 1.1. Codificación

El archivo debe estar codificado en UTF-8 sin BOM. El BOM (bytes `EF BB BF` al inicio) haría que la primera línea no empezara exactamente con `BEGIN:VCARD`, provocando errores de parsing en clientes estrictos.

### 1.2. Finales de línea

El RFC 2426 exige finales de línea CRLF (`\r\n`). Los finales LF (`\n`, estilo Unix/Linux) son técnicamente inválidos, aunque muchos clientes los toleran. Para máxima compatibilidad, usar siempre CRLF.

En VS Code, para editar archivos `.vcf` correctamente, añadir al `settings.json` del usuario (`Ctrl+Shift+P` → "Preferences: Open User Settings (JSON)"):

```jsonc
{
  "files.associations": {
    "*.vcf": "plaintext"
  },
  "[plaintext]": {
    "files.encoding": "utf8",
    "files.eol": "\r\n",
    "files.insertFinalNewline": false,
    "files.trimTrailingWhitespace": false
  }
}
```

Verificaciones después de guardar:

1. La barra inferior derecha de VS Code debe mostrar `UTF-8` (no `UTF-8 with BOM`) y `CRLF` (no `LF`).
2. Si aparece `UTF-8 with BOM`, hacer clic → "Save with Encoding" → "UTF-8".
3. Si aparece `LF`, hacer clic y cambiar a `CRLF`.

La propiedad `files.trimTrailingWhitespace` debe estar en `false` para archivos `.vcf` porque las líneas de continuación del plegado empiezan con un espacio, y el trimming lo eliminaría.

### 1.3. Plegado de líneas (line folding)

Ninguna línea del archivo debe superar 75 octetos. Las líneas largas se pliegan añadiendo un salto de línea (CRLF) seguido de un espacio o tabulación al inicio de la línea de continuación. El espacio de continuación no forma parte del valor; es un marcador de plegado.

Ejemplo:

```vcard
PHOTO;ENCODING=b;TYPE=JPEG:/9j/4AAQSkZJRgABAQEASABIAAD/2wBDAAEB
 AQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQ
 EBAQEBAQEBAQEBAQEBAQH/2wBDAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEB
```

Cada línea de continuación empieza con un espacio (el marcador) seguido de hasta 74 caracteres de datos, para un total de 75 octetos.

### 1.4. Escape de caracteres especiales

Dentro de los valores de texto, los siguientes caracteres deben escaparse:

| Carácter | Escape | Ejemplo |
| ---------- | -------- | --------- |
| `,` (coma) | `\,` | `ADR:;;Av. Los Álamos 142\, Lima` |
| `;` (punto y coma) | `\;` | Cuando no es separador de campo |
| `\` (barra invertida) | `\\` | `NOTE:ruta C:\\Users\\...` |
| Salto de línea | `\n` | `NOTE:Línea 1\nLínea 2` |

### 1.5. Estructura obligatoria

Todo archivo vCard 3.0 debe contener al menos:

```vcard
BEGIN:VCARD
VERSION:3.0
N:<apellido>;<nombre>;<nombres adicionales>;<prefijo>;<sufijo>
FN:<nombre formateado>
END:VCARD
```

`BEGIN:VCARD`, `VERSION:3.0` y `END:VCARD` son obligatorios. `N` y `FN` son las únicas propiedades requeridas por el estándar.

### 1.6. Extensión del archivo

Se recomienda que el archivo lleve la extensión `.vcf` y el tipo MIME `text/vcard` (RFC 6350). La extensión `.vcard` también es válida según el registro IANA, pero `.vcf` es la utilizada universalmente por todos los clientes (Google Contacts, Apple Contacts, Outlook, Thunderbird, Android).

---

## 2. Propiedades de vCard 3.0

### 2.1. Propiedades de identificación

#### N (Nombre estructurado) — OBLIGATORIA

Estructura de 5 campos separados por `;`:

```vcard
N:<apellido(s)>;<nombre(s)>;<nombres adicionales>;<prefijo>;<sufijo>
```

Ejemplo:

```vcard
N:Pérez Suárez;Juan;;;
```

Los campos vacíos se dejan sin valor pero los `;` deben mantenerse. Para nombres compuestos, usar coma dentro de cada campo: `N:García López;Juan Carlos;;;`.

#### FN (Nombre formateado) — OBLIGATORIA

El nombre tal como se mostrará al usuario:

```vcard
FN:Juan Pérez
```

#### NICKNAME

Apodo o nombre informal:

```vcard
NICKNAME:Juancito
```

#### BDAY (Fecha de nacimiento)

Formato ISO 8601:

```vcard
BDAY:1985-03-15
```

También acepta solo año-mes (`1985-03`) o año (`1985`).

#### PHOTO (Fotografía)

Se puede incrustar en base64 o referenciar una URI:

```vcard
PHOTO;ENCODING=b;TYPE=JPEG:<datos base64 plegados>
```

O como URI (no soportada universalmente):

```vcard
PHOTO;VALUE=uri:https://ejemplo.com/foto.jpg
```

Tamaño recomendado: imagen JPEG de 300×300 px, menos de 100 KB antes de codificar a base64. Para el proceso de plegado, ver sección 4.

#### LOGO (Logotipo de la organización)

Misma sintaxis que PHOTO:

```vcard
LOGO;ENCODING=b;TYPE=PNG:<datos base64 plegados>
```

Tamaño recomendado: PNG de resolución moderada, menos de 64 KB antes de codificar.

### 2.2. Propiedades de dirección

#### ADR (Dirección postal)

Estructura de 7 campos separados por `;`:

```vcard
ADR;TYPE=<tipo>:;<apartado postal>;<calle>;<ciudad>;<estado/provincia>;<código postal>;<país>
```

Los tipos válidos son: `HOME`, `WORK`, `POSTAL`, `PARCEL`, `DOM` (doméstica), `INTL` (internacional). Se pueden combinar: `TYPE=WORK;TYPE=pref` (la preferida de trabajo).

Ejemplo:

```vcard
ADR;TYPE=WORK;TYPE=pref:;;742 Evergreen Terrace;Springfield;Oregon;97477;USA
```

Error común: duplicar la dirección con y sin agrupación (`item1.`). Mantener solo una por tipo.

#### Agrupación con `item1.`, `item2.`, etc

El prefijo `itemN.` vincula propiedades relacionadas entre sí. Es estándar en vCard 3.0:

```vcard
item1.ADR;TYPE=WORK;TYPE=pref:;;742 Evergreen Terrace;Springfield;Oregon;97477;USA
item1.X-ABADR:us
```

El `item1.` indica que `X-ABADR:us` (código de país de Apple Address Book) pertenece a esa dirección específica. Si hay varias direcciones, usar `item2.`, `item3.`, etc.

Uso típico con etiquetas personalizadas:

```vcard
item1.TEL;TYPE=CELL:+51964499678
item1.X-ABLabel:WhatsApp Personal
```

### 2.3. Propiedades de telecomunicaciones

#### TEL (Teléfono)

Formato E.164 internacional recomendado:

```vcard
TEL;TYPE=WORK,CELL:+14155552671
TEL;TYPE=HOME,VOICE:+51964499678
```

Tipos válidos: `HOME`, `WORK`, `CELL`, `VOICE`, `FAX`, `PAGER`, `MSG`, `pref`.

#### EMAIL

```vcard
EMAIL;TYPE=INTERNET;TYPE=WORK:contacto@empresa.com
EMAIL;TYPE=INTERNET;TYPE=HOME:correo@dominio.com
```

#### URL

```vcard
URL;TYPE=WORK:https://www.empresa.com
```

#### IMPP (Mensajería instantánea)

No es estándar en 3.0 (se introdujo en 4.0), pero se usa ampliamente:

```vcard
IMPP;TYPE=pref:xmpp:usuario@servidor.com
```

Para redes sociales, preferir `X-SOCIALPROFILE` (ver sección 2.7).

### 2.4. Propiedades organizacionales

#### ORG (Organización)

```vcard
ORG:Empresa SAC
```

Para organizaciones con divisiones: `ORG:Empresa SAC;División Norte;Departamento IT`.

#### TITLE (Cargo)

El cargo formal dentro de la organización — lo que aparecería en el organigrama o en la tarjeta de presentación corporativa:

```vcard
TITLE:Director de Tecnología
```

#### ROLE (Función)

La función que desempeña — lo que realmente hace, independientemente de cómo se llame su puesto:

```vcard
ROLE:Administración de infraestructura y seguridad
```

#### X-PROFESSION (Profesión)

La profesión o disciplina — lo que la persona es profesionalmente, independiente de dónde trabaje o qué cargo ocupe:

```vcard
X-PROFESSION:Ingeniero de sistemas
```

En resumen: `TITLE` responde a "¿cuál es tu puesto?", `ROLE` responde a "¿qué haces?", y `X-PROFESSION` responde a "¿qué eres?".

### 2.5. Propiedades geográficas

#### TZ (Zona horaria)

```vcard
TZ:-05:00
```

#### GEO (Geolocalización)

Formato: `GEO:<latitud>;<longitud>`:

```vcard
GEO:-12.0464;-77.0428
```

Errores comunes:

```vcard
GEO:geo:-5.1869;-80.6352       ❌ (prefijo "geo:" sobrante)
GEO:-5.1869,-80.6352           ❌ (coma en lugar de punto y coma)
GEO:44.7977;-106.9574,17       ❌ (tercer valor sobrante)
```

### 2.6. Propiedades explicativas

#### NOTE

Notas de texto libre. Usar `\n` para saltos de línea dentro del valor:

```vcard
NOTE:Consultoría IT\nSeguridad digital\nFuente: https://www.empresa.com/contacto.vcf
```

#### CATEGORIES

Etiquetas separadas por comas:

```vcard
CATEGORIES:tecnología,seguridad,consultoría
```

No dejar espacios después de las comas. `CATEGORIES:a, b` incluye un espacio al inicio de "b".

#### REV (Fecha de revisión)

Formato ISO 8601 UTC:

```vcard
REV:2026-04-18T15:30:00Z
```

#### UID (Identificador único)

Debe ser un identificador estable y único. El formato recomendado es UUID:

```vcard
UID:urn:uuid:f81d4fae-7dec-11d0-a765-00a0c91e6bf6
```

Generar uno nuevo en la terminal:

```bash
echo "urn:uuid:$(uuidgen | tr '[:upper:]' '[:lower:]')"
```

Error común: usar una URL como UID. Si la URL cambia, el contacto pierde su identidad persistente y los clientes lo tratan como un contacto nuevo en cada sincronización.

#### PRODID (Identificador de producto)

Identifica la aplicación que generó la vCard:

```vcard
PRODID:-//miempresa//Contactos//ES
```

#### LANG (Idioma)

```vcard
LANG:es
LANG:en
```

En vCard 3.0 no hay forma estándar de indicar cuál es el idioma preferido. En 4.0 se añade el parámetro `PREF`.

### 2.7. Extensiones no estándar

Las propiedades con prefijo `X-` son extensiones propietarias. Son válidas sintácticamente pero no tienen semántica definida en el estándar.

#### X-SOCIALPROFILE

Extensión de Apple, ampliamente soportada:

```vcard
X-SOCIALPROFILE;type=linkedin:https://www.linkedin.com/in/fulanito/
X-SOCIALPROFILE;type=github:https://github.com/fulanito
X-SOCIALPROFILE;type=twitter:https://twitter.com/fulanito
```

#### X-GENDER

En vCard 3.0 no existe `GENDER` (se introdujo en 4.0). `X-GENDER` es una extensión no estándar, pero en la práctica las implementaciones siguen los valores definidos por la propiedad `GENDER` de vCard 4.0 (RFC 6350, sección 6.2.7):

| Valor | Significado |
| ------- | ------------- |
| `M` | Male (masculino) |
| `F` | Female (femenino) |
| `O` | Other (otro) |
| `N` | None / Not applicable (no aplica) |
| `U` | Unknown (desconocido) |
| *(vacío)* | No especificado |

Uso:

```vcard
X-GENDER:M
```

Error común: usar `GENDER:M` y `X-GENDER:M` juntos. En 3.0, solo usar `X-GENDER`.

#### X-PROFESSION

```vcard
X-PROFESSION:Ingeniero de sistemas
```

#### X-ABADR (Apple Address Book)

Código de país ISO 3166-1 alpha-2 para la dirección. Apple Contacts usa este valor para formatear y mostrar la dirección según las convenciones del país correspondiente. Sin este parámetro, Apple Contacts intenta adivinar el país a partir del texto, y no siempre acierta. Siempre vincular con agrupación:

```vcard
item1.ADR;TYPE=WORK;TYPE=pref:;;742 Evergreen Terrace;Springfield;Oregon;97477;USA
item1.X-ABADR:us
```

El valor de `X-ABADR` siempre va en minúsculas (`us`, `pe`, `ec`), siguiendo la convención de Apple.

### 2.8. Propiedades a evitar en vCard 3.0

| Propiedad | Motivo |
| ----------- | -------- |
| `CLASS:PUBLIC` | Estándar en 3.0 pero ningún cliente la respeta. Eliminada en 4.0. |
| `X-KIND:individual` | No estándar en 3.0. Intento de backport de `KIND` de 4.0. Inútil. |
| `X-SOURCE:<url>` | No estándar. Semánticamente vacía. |
| `CHARSET=UTF-8` en cada propiedad | Deprecado en 3.0. El estándar asume UTF-8. Añade ruido. |
| `GENDER:M` | No existe en 3.0. Usar `X-GENDER:M`. |

---

## 3. Plantilla completa comentada

```vcard
BEGIN:VCARD
VERSION:3.0

;; === IDENTIFICACIÓN (obligatorias N y FN) ===
N:<apellido(s)>;<nombre(s)>;<nombres adicionales>;<prefijo>;<sufijo>
FN:<nombre formateado completo>
NICKNAME:<apodo>

;; === ORGANIZACIÓN ===
ORG:<nombre de la organización>
TITLE:<cargo formal>
ROLE:<función o descripción del rol>
X-PROFESSION:<profesión>

;; === TELECOMUNICACIONES ===
TEL;TYPE=WORK,CELL:<teléfono formato E.164>
TEL;TYPE=HOME,VOICE:<teléfono formato E.164>
EMAIL;TYPE=INTERNET;TYPE=WORK:<email laboral>
EMAIL;TYPE=INTERNET;TYPE=HOME:<email personal>
URL;TYPE=WORK:<URL del sitio web>

;; === DIRECCIÓN POSTAL (7 campos separados por ;) ===
;; <vacío>;<apartado postal>;<calle>;<ciudad>;<estado>;<CP>;<país>
item1.ADR;TYPE=WORK;TYPE=pref:;;<calle>;<ciudad>;<estado>;<código postal>;<país>
item1.X-ABADR:<código país ISO en minúsculas>

;; === GÉNERO (usar X- en vCard 3.0) ===
X-GENDER:<M o F>

;; === NOTAS Y CATEGORÍAS ===
NOTE:<texto libre\ncon saltos de línea usando \n>
CATEGORIES:<etiqueta1>,<etiqueta2>,<etiqueta3>

;; === REDES SOCIALES ===
X-SOCIALPROFILE;type=linkedin:<URL completa del perfil>
X-SOCIALPROFILE;type=github:<URL completa del perfil>

;; === IDIOMAS ===
LANG:<código ISO 639-1 en minúsculas>
LANG:<código ISO 639-1 en minúsculas>

;; === GEOLOCALIZACIÓN ===
TZ:<desplazamiento UTC, ej: -05:00>
GEO:<latitud>;<longitud>

;; === FOTO Y LOGO (ver sección 4 para plegado) ===
;; PHOTO;ENCODING=b;TYPE=JPEG:<base64 plegado a 75 octetos/línea>
;; LOGO;ENCODING=b;TYPE=PNG:<base64 plegado a 75 octetos/línea>

;; === METADATOS ===
BDAY:<YYYY-MM-DD>
REV:<YYYY-MM-DDThh:mm:ssZ>
UID:urn:uuid:<UUID generado con uuidgen>
PRODID:<identificador de la app generadora>

END:VCARD
```

Notas sobre la plantilla:

1. Las líneas que empiezan con `;;` son comentarios de documentación. Eliminarlas del archivo final; vCard 3.0 no define sintaxis de comentarios.
2. Para metadatos internos, usar propiedades `X-` como `X-COMMENT:esto es un comentario interno`.
3. Los campos `<entre ángulos>` son placeholders a reemplazar.
4. `PHOTO` y `LOGO` están comentados porque requieren plegado de líneas (ver sección 4).

---

## 4. Plegado de líneas para PHOTO y LOGO

Las propiedades `PHOTO` y `LOGO` con base64 incrustado producen líneas muy largas que deben plegarse a 75 octetos por línea.

### 4.1. Comandos de plegado (sin script)

Para una imagen `mi-foto.jpg` como `PHOTO`:

```bash
# 1. Generar base64 en una sola línea
base64 -w 0 mi-foto.jpg > /tmp/foto.b64

# 2. Definir la cabecera
CABECERA="PHOTO;ENCODING=b;TYPE=JPEG:"
LARGO_CABECERA=${#CABECERA}
ESPACIO_PRIMERA=$((75 - LARGO_CABECERA))

# 3. Primera línea: cabecera + primeros N caracteres de base64
#    IMPORTANTE: head -c no añade salto de línea al final de su salida,
#    por eso se usa el subshell (...; echo) para añadirlo explícitamente.
#    Sin esto, la primera línea de continuación se pegaría al final de la
#    primera línea, produciendo una línea de ~150 octetos.
(head -c "$ESPACIO_PRIMERA" /tmp/foto.b64; echo) > /tmp/foto_plegada.b64

# 4. Resto: plegado a 74 caracteres con espacio de continuación
tail -c +$((ESPACIO_PRIMERA + 1)) /tmp/foto.b64 \
  | fold -w 74 \
  | sed 's/^/ /' >> /tmp/foto_plegada.b64

# 5. Añadir la cabecera a la primera línea
sed -i "1s/^/${CABECERA}/" /tmp/foto_plegada.b64
```

Para un logo `mi-logo.png` como `LOGO`, cambiar:

```bash
CABECERA="LOGO;ENCODING=b;TYPE=PNG:"
```

Y usar `mi-logo.png` en lugar de `mi-foto.jpg`.

### 4.2. Lógica del plegado

La primera línea debe contener la cabecera (`PHOTO;ENCODING=b;TYPE=JPEG:`) más tantos caracteres base64 como quepan hasta 75 octetos totales. Cada línea siguiente empieza con un espacio (1 octeto, marcador de continuación) + 74 caracteres de datos = 75 octetos.

### 4.3. Flujo de trabajo recomendado

Escribir todo el archivo `.vcf` a mano en VS Code excepto `PHOTO` y `LOGO`. Dejar marcadores temporales:

```vcard
BEGIN:VCARD
VERSION:3.0
N:Pérez Suárez;Juan;;;
FN:Juan Pérez
...
REV:2026-04-18T15:30:00Z
...
__PHOTO_AQUI__
__LOGO_AQUI__
END:VCARD
```

Luego reemplazar los marcadores:

```bash
# Generar bloques plegados
# (usando los comandos de 4.1, guardando resultado en archivos temporales)

# Reemplazar marcadores
sed -i '/__PHOTO_AQUI__/{
    r /tmp/foto_plegada.b64
    d
}' mi-archivo.vcf

sed -i '/__LOGO_AQUI__/{
    r /tmp/logo_plegada.b64
    d
}' mi-archivo.vcf
```

### 4.4. Tamaños recomendados para imágenes

| Propiedad | Formato recomendado | Resolución | Peso máximo (antes de base64) |
| ----------- | ------------------- | ------------ | ------------------------------- |
| PHOTO | JPEG | 300×300 px | 96 KB |
| LOGO | PNG | Variable | 64 KB |

El base64 ocupa un 33% más que el binario original. Un PHOTO de 96 KB produce ~128 KB de texto en el `.vcf`. Archivos `.vcf` mayores a 1 MB pueden causar problemas en algunos clientes.

### 4.5. Incrustado vs. URI externa

| Método | Ventajas | Desventajas |
| -------- | ---------- | ------------- |
| Base64 incrustado | Máxima compatibilidad, funciona offline | Aumenta el tamaño del archivo |
| URI externa (`PHOTO;VALUE=uri:`) | Archivo ligero | No soportado por iOS/macOS, Outlook. Requiere conexión a Internet |

Para vCards profesionales que se compartirán con clientes diversos, usar siempre base64 incrustado.

---

## 5. Errores frecuentes y cómo evitarlos

### 5.1. Codificación UTF-8 doblemente convertida

**Síntoma:** los acentos se muestran como `Ãº` en lugar de `ú`.

**Causa:** el generador de vCards codifica el texto en UTF-8, pero luego vuelve a codificar los bytes resultantes como si fueran Latin-1, produciendo una secuencia de 4 bytes (`C3 83 C2 BA`) donde debería haber 2 (`C3 BA`).

**Solución:** si el archivo ya existe con doble codificación, reparar con:

```bash
iconv -f UTF-8 -t ISO-8859-1 archivo.vcf | iconv -f UTF-8 -t UTF-8 > archivo_corregido.vcf
```

La primera pasada "deshace" la codificación errónea, la segunda valida el resultado. Si el archivo se escribe a mano en VS Code con codificación UTF-8 correcta, este problema no ocurre.

### 5.2. Finales de línea incorrectos

**Síntoma:** algunos clientes (especialmente en Windows o integraciones corporativas) no parsean el archivo.

**Solución:** convertir LF a CRLF:

```bash
unix2dos mi-archivo.vcf
# o con sed:
sed -i 's/$/\r/' mi-archivo.vcf
```

### 5.3. Dirección duplicada

**Síntoma:** aparecen dos direcciones iguales del mismo tipo en el contacto.

**Causa:** incluir una `ADR` sin prefijo y otra `item1.ADR` con los mismos datos.

**Solución:** mantener solo una de las dos. Si se necesita `X-ABADR`, usar la versión con `item1.`.

### 5.4. GEO mal formado

Formato correcto: `GEO:<latitud>;<longitud>` — dos valores separados por punto y coma.

Errores comunes:

```vcard
GEO:geo:-5.1869;-80.6352       ❌ (prefijo "geo:" sobrante)
GEO:-5.1869,-80.6352           ❌ (coma en lugar de punto y coma)
GEO:44.7977;-106.9574,17       ❌ (tercer valor sobrante)
```

### 5.5. CHARSET=UTF-8 en cada propiedad

**Problema:** `FN;CHARSET=UTF-8:Juan` — el parámetro `CHARSET` fue deprecado en vCard 3.0 porque el estándar asume UTF-8 implícitamente.

**Solución:** escribir simplemente `FN:Juan`.

### 5.6. UID como URL

**Problema:** `UID:https://www.empresa.com/contacto.vcf` — si la URL cambia, se pierde la identidad del contacto.

**Solución:** usar UUID estable:

```bash
# Generar UUID
uuidgen | tr '[:upper:]' '[:lower:]'
# Resultado ejemplo: f81d4fae-7dec-11d0-a765-00a0c91e6bf6

# En la vCard:
UID:urn:uuid:f81d4fae-7dec-11d0-a765-00a0c91e6bf6
```

---

## 6. Ejemplo completo

```vcard
BEGIN:VCARD
VERSION:3.0
N:Mendoza Rivera;Ana Lucía;;;
FN:Ana Lucía Mendoza
NICKNAME:Anita
ORG:Soluciones Digitales SAC
TITLE:Directora de Tecnología
ROLE:Gestión de infraestructura cloud y seguridad
X-PROFESSION:Ingeniera de sistemas
TEL;TYPE=WORK,CELL:+14155552671
TEL;TYPE=HOME,VOICE:+51987654321
EMAIL;TYPE=INTERNET;TYPE=WORK:ana.mendoza@solucionesdigitales.com
EMAIL;TYPE=INTERNET;TYPE=HOME:analumendoza@correo.com
URL;TYPE=WORK:https://www.solucionesdigitales.com
item1.ADR;TYPE=WORK;TYPE=pref:;;Av. Javier Prado Este 4600;Lima;Lima;15023;Perú
item1.X-ABADR:pe
item2.ADR;TYPE=HOME:;;Calle Las Magnolias 280;Arequipa;Arequipa;04001;Perú
item2.X-ABADR:pe
X-GENDER:F
NOTE:Consultoría IT\nSeguridad digital\nArquitectura cloud
CATEGORIES:tecnología,seguridad,cloud
X-SOCIALPROFILE;type=linkedin:https://www.linkedin.com/in/analumendoza/
X-SOCIALPROFILE;type=github:https://github.com/analumendoza
LANG:es
LANG:en
TZ:-05:00
GEO:-12.0864;-77.0048
BDAY:1990-07-22
REV:2026-09-14T10:00:00Z
UID:urn:uuid:b7e39c4a-1f82-4d5b-9a3e-7c6d8e2f1a40
PRODID:-//Soluciones Digitales//Contactos//ES
END:VCARD
```

Este ejemplo no incluye `PHOTO` ni `LOGO` por razones de espacio. Para añadirlos, seguir el procedimiento de la sección 4. Nótese el uso de dos direcciones agrupadas con `item1.` e `item2.`, cada una con su respectivo `X-ABADR`.

---

## 7. Compatibilidad entre vCard 3.0 y 4.0

| Aspecto | vCard 3.0 (RFC 2426) | vCard 4.0 (RFC 6350) |
| --------- | --------------------- | --------------------- |
| Publicación | Septiembre 1998 | Agosto 2011 |
| Soporte en clientes | Universal (iOS, Android, Google, Outlook) | Parcial (algunos clientes aún exportan en 3.0) |
| `KIND` | No existe. Usar `X-KIND` (inútil) | Estándar: `KIND:individual` |
| `GENDER` | No existe. Usar `X-GENDER` | Estándar: `GENDER:M` |
| `PHOTO` base64 | `PHOTO;ENCODING=b;TYPE=JPEG:` | `PHOTO;ENCODING=b;MEDIATYPE=image/jpeg:` |
| `CHARSET` | Deprecado (se asume UTF-8) | No existe (siempre UTF-8) |
| `CLASS` | Estándar pero ignorada | Eliminada |
| `SOURCE` | No estándar | Estándar |
| `LANG` | Sin preferencia | Con parámetro `PREF` |

Para máxima compatibilidad, vCard 3.0 sigue siendo la elección más segura en 2026. Un contacto guardado en formato vCard 4.0 puede fallar al importarse en dispositivos Android anteriores, versiones de Outlook anteriores a 365, y versiones de iOS inferiores a 16.

---

## 8. Referencias

- RFC [2425](https://datatracker.ietf.org/doc/html/rfc2425) – A MIME Content-Type for Directory Information
- RFC [2426](https://datatracker.ietf.org/doc/html/rfc2426) – vCard MIME Directory Profile (vCard 3.0)
- RFC [6350](https://datatracker.ietf.org/doc/html/rfc6350) – vCard Format Specification (vCard 4.0, obsoleta 2425/2426)
- Página de información del RFC [2426](https://www.rfc-editor.org/info/rfc2426) (con errata, historial, etc.)
- Registro IANA del tipo MIME [text/vcard](https://www.iana.org/assignments/media-types/text/vcard)
- [CalConnect Developer Guide: vCard](https://devguide.calconnect.org/vCard/introduction/)
- [CalConnect Developer Guide: vCard 4.0](https://devguide.calconnect.org/vCard/vcard-4/)
- ISO [8601](https://www.iso.org/iso-8601-date-and-time-format.html) – Formatos de fecha y hora
- ISO [3166-1 alpha-2](https://www.iso.org/iso-3166-country-codes.html) – Códigos de país (usado en X-ABADR)
- Formato [E.164](https://www.itu.int/rec/T-REC-E.164) – Numeración telefónica internacional
- [ez-vcard](https://github.com/mangstadt/ez-vcard) (Java) – Biblioteca de parsing y validación de vCard
- [vobject](https://github.com/eventable/vobject) (Python) – Parser y validador de vCard 2.1/3.0/4.0
- [vcard](https://pypi.org/project/vcard/) (Python) – Validador estricto para vCard 3.0 (RFC 2426)
- [sabre/vobject](https://sabre.io/vobject/) (PHP) – Biblioteca con validador integrado para vCard 3.0 y 4.0

[Creación y mantenimiento de archivos vCard](https://github.com/noggalito/vcard) © 2026 by [Calú](https://github.com/calu777) in [noggalito](https://noggalito.com/) is licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)<img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;"><img src="https://mirrors.creativecommons.org/presskit/icons/sa.svg" alt="" style="max-width: 1em;max-height:1em;margin-left: .2em;">