# Tablero de Autoconsumos

Consulta de la cartera pendiente de autoconsumos y registro de gestión por centro.

Reemplaza al tablero anterior, que era un HTML de 10 MB con las 26 000 facturas
incrustadas, una contraseña escrita en el propio código y el seguimiento guardado
en el navegador de cada persona.

## Qué cambia

| | Antes | Ahora |
|---|---|---|
| Los datos | Dentro del archivo, 10 MB | Se piden al abrir; el archivo pesa 70 KB y va vacío |
| El acceso | `metro` / `2026` en el JavaScript | Contraseña por centro, validada en el servidor |
| Qué ve cada uno | Todos veían todo | Cada centro ve solo su cartera |
| El seguimiento | En el navegador de cada quien | En SharePoint, visible para todos |
| Corregir un marcado | Se podía borrar sin rastro | **No se puede: es definitivo y queda firmado** |
| La fecha de corte | La del reloj del equipo | La de última modificación del Excel |
| Sin internet | Los gráficos no cargaban | No aplica: el tablero necesita conexión para los datos |

## Qué ve cada rol

El rol sale de la columna `Rol` de la lista de accesos y define la pantalla
completa, no solo los datos.

| | Supervisora de centro (`CENTRO`) | Administración (`ADMIN`) |
|---|---|---|
| Facturas que recibe | Solo las de su centro | Todos los centros |
| Filtros | Sí | Sí, más el filtro por centro |
| Tabla y marcado | Sí | Sí |
| Indicadores | — | Sí |
| Pestaña de análisis y gráficos | — | Sí |
| Resumen comparativo por centro | — | Sí |
| Descargar a CSV | — | Sí |

La supervisora entra a **consultar y registrar**: filtros, tabla y marcado. Nada
más. Las descargas quedan reservadas a Administración, así que la cartera no sale
del tablero por iniciativa de un centro.

> El límite real no es que los botones estén ocultos, sino que al navegador de un
> centro **nunca le llegan las filas de otro**: el filtro ocurre en el flujo. Lo
> que sí puede hacer una supervisora es copiar de la pantalla lo que ya está
> viendo, que es su propia cartera.

## Cómo funciona

```
  Excel en SharePoint                 Listas de SharePoint
  (hoja NO LIQUIDADO)                 (accesos, sesiones, seguimiento)
         │                                      │
         │ al guardarse                         │
         ▼                                      │
  Actualizar cache  ──►  un JSON por centro     │
  (~40 s, nadie espera)      en /Cache          │
                                 │              │
                                 ▼              ▼
                          Flujos con disparador HTTP
                                      │
                                      ▼
                                 index.html
                                 (navegador)
```

El navegador nunca habla con el Excel ni con SharePoint directamente. Todo pasa
por flujos de Power Automate:

1. **Verificar acceso** — recibe centro y contraseña, devuelve un *token* de
   sesión. La contraseña se compara en el servidor y no vuelve a viajar nunca más.
2. **Actualizar cache** — se dispara cuando el Excel se modifica. Lee las ~4.700
   filas, corrige los encabezados y deja un archivo JSON por centro. Es un
   proceso de fondo: ninguna persona lo espera.
3. **Obtener datos** — recibe el token y lee **solo el archivo del centro que
   preguntó**, más los marcados registrados.
4. **Marcar facturas** — recibe el token y las facturas a marcar. Guarda la
   factura completa, no solo su número. **Solo crea: nunca modifica ni borra.**
5. **Subir respaldo** — recibe PDF, los adjunta al registro y marca la factura.
   Hereda la misma regla: si ya estaba marcada, la rechaza.

### Por qué hay un caché

Leer el Excel completo tarda unos 35 segundos, y no depende del tamaño de la
cartera de cada centro: para encontrar las 1.156 facturas de Kennedy hay que
recorrer las 4.721 igual.

Separando la lectura de la consulta, **el tablero abre en unos 4 segundos** en
lugar de 35. La contrapartida es que un cambio en el Excel tarda uno o dos
minutos en reflejarse — lo que cambia minuto a minuto, que son los marcados, se
lee en vivo y aparece al instante.

### Qué se guarda al marcar

Una fila de `AutoconsumosSeguimiento` no es un puntero al Excel: es el registro
completo. Lleva las quince columnas de la factura tal como estaban en el momento
de marcarla, la observación, quién la marcó, cuándo, y el PDF adjunto al propio
elemento.

Si el Excel se regenera o esa factura desaparece de la base, **el registro se
sostiene solo**.

## Puesta en marcha

Los pasos están en `docs/`, en el orden en que hay que hacerlos:

| Documento | Qué se hace |
|---|---|
| [`docs/01-sharepoint.md`](docs/01-sharepoint.md) | Preparar el Excel y crear las tres listas |
| [`docs/02-flujos.md`](docs/02-flujos.md) | Crear los flujos de Power Automate, acción por acción |
| [`docs/03-contrato.md`](docs/03-contrato.md) | El JSON exacto que intercambian el tablero y los flujos |
| [`docs/04-publicacion.md`](docs/04-publicacion.md) | Dónde publicar el `index.html` y qué revisar antes |

Cuando los flujos existan, se pegan sus tres direcciones en el bloque `CONFIG`,
al principio del `<script>` de `index.html`, y se pone `MODO_DEMO` en `false`:

```js
const CONFIG = {
  VERIFICAR_URL: "https://…/workflows/…/triggers/manual/paths/invoke?…",
  DATOS_URL:     "https://…",
  MARCAR_URL:    "https://…",
  …
  MODO_DEMO: false          // ← imprescindible antes de publicar
};
```

## Probarlo ahora mismo

El archivo viene con `MODO_DEMO: true`. Ábralo con doble clic y entre con
cualquier centro de la lista y la contraseña `demo`. Para ver el tablero como lo
vería Administración, entre con `ADMINISTRACIÓN` / `demo`.

Los datos son inventados y no se escribe nada en ninguna parte. Sirve para
recorrer la herramienta y decidir, no para trabajar.

## El marcado es definitivo

Es la regla de fondo de esta versión y conviene que quede clara antes de
capacitar a nadie:

- Marcar una factura **crea un registro y nada más**. No hay forma de editarlo ni
  de borrarlo desde el tablero, y el flujo tampoco lo permite.
- Queda firmado con el centro, la fecha y la hora del servidor.
- La observación se escribe **antes** de confirmar. Después queda congelada.
- El tablero pide confirmación en una ventana aparte, listando las facturas, y
  avisa que la acción no se puede deshacer.
- Si dos centros intentan marcar la misma factura a la vez, gana el primero. Al
  segundo se le informa quién la marcó y cuándo, y su intento no altera nada.

Esa garantía no depende solo del código del tablero: la columna del número de
factura en la lista de SharePoint tiene **valores únicos obligatorios**, así que
el segundo intento falla en la base de datos, no en el navegador.

## Estructura

```
autoconsumos-tablero/
├── index.html          El tablero completo. Sin datos, sin librerías externas.
├── README.md           Este archivo.
├── .gitignore          Impide que entren datos, respaldos y credenciales.
├── docs/
│   ├── 01-sharepoint.md    Preparar el Excel y las tres listas
│   ├── 02-flujos.md        Los flujos, acción por acción
│   ├── 03-contrato.md      Lo que intercambian el tablero y los flujos
│   └── 04-publicacion.md   Dónde publicar y qué revisar antes
└── presentacion/
    └── entrega-2026-09-16.html   Presentación de entrega. Se abre con doble clic.
```

## Antes de trabajar con este repositorio

El `index.html` versionado viene **sin las direcciones de los flujos y en modo
demostración**. Es deliberado: esas direcciones llevan una firma (`&sig=`) que
funciona como llave del disparador, y quien la tenga puede llamar al flujo.

Para levantarlo en local, ábralo con doble clic y entre con cualquier centro y la
contraseña `demo`. Para ver la vista completa, entre con `ADMINISTRACIÓN` / `demo`.

Para publicar, siga [`docs/04-publicacion.md`](docs/04-publicacion.md): se pegan
las direcciones en el bloque `CONFIG` y se pone `MODO_DEMO: false`. Ese archivo
configurado **no se vuelve a subir**.

Un solo archivo, sin dependencias, sin compilación. Se edita con cualquier
editor de texto y se publica copiándolo.

## Lo que todavía no resuelve

Puntos abiertos, para no dar por cerrado lo que no lo está:

- **Las contraseñas se guardan en texto plano** en la lista de accesos. Power
  Automate no tiene función de cifrado, así que la protección real es restringir
  los permisos de esa lista. La solución de fondo es autenticar con Entra ID.
- **Un centro podría marcar la factura de otro** si alguien construye la llamada
  a mano. El registro siempre queda a su nombre, así que es rastreable, pero no
  está bloqueado. Se cierra validando la factura contra el Excel dentro del flujo.
- **El Excel vive en un OneDrive personal.** Si esa persona sale de la
  organización, el tablero deja de actualizarse. Conviene moverlo a un sitio de
  SharePoint del área antes de poner esto en producción.
- **Solo se lee la hoja `NO LIQUIDADO`.** El tablero no muestra lo liquidado ni el
  porcentaje de liquidación. Añadir la segunda hoja es una acción más en el flujo.
