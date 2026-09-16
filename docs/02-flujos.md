# 2 · Los flujos de Power Automate

Tres flujos obligatorios y dos opcionales. Todos se crean desde
**Power Automate → Crear → Flujo de nube instantáneo → Solicitud HTTP recibida**.

Al guardar cada flujo, Power Automate genera la URL del disparador. Esas son las
direcciones que van en el bloque `CONFIG` de `index.html`.

> **La conexión importa.** Los conectores de SharePoint y de Excel se conectan con
> la cuenta de quien crea el flujo. Use una cuenta de servicio del área, no la
> personal de nadie: si esa persona sale, los flujos dejan de correr. Es el mismo
> error que se está corrigiendo al salir de Colab.

---

## Flujo 1 · `Autoconsumos - Verificar acceso`

Valida centro y contraseña, y entrega un token de sesión.

### Disparador

**Cuando se recibe una solicitud HTTP.** En «Esquema JSON del cuerpo de la
solicitud» pegue:

```json
{
  "type": "object",
  "properties": {
    "centro":     { "type": "string" },
    "contrasena": { "type": "string" }
  }
}
```

Método: `POST`.

### Acciones

**1. Obtener elementos** (SharePoint) — lista `AutoconsumosAccesos`

- Consulta de filtro:
  ```
  Title eq '@{replace(triggerBody()?['centro'], '''', '''''')}' and Activo eq 1
  ```
  El `replace` duplica las comillas simples. Sin eso, un centro con apóstrofo en
  el nombre rompe la consulta — y es la puerta por la que se cuela una inyección
  en OData.
- Número máximo de elementos: `1`

**2. Condición** — nómbrela `¿Credenciales válidas?`

Modo avanzado:
```
@and(
  greater(length(body('Obtener_elementos')?['value']), 0),
  equals(first(body('Obtener_elementos')?['value'])?['Clave'], triggerBody()?['contrasena'])
)
```

`equals` distingue mayúsculas de minúsculas, que es lo que se quiere en una
contraseña.

#### Rama **En caso afirmativo**

**3. Redactar** — nombre `token` — valor: `@{guid()}`

**4. Redactar** — nombre `expira` — valor: `@{addHours(utcNow(), 8)}`

> Se calculan una sola vez en un `Redactar`. Si se pusiera `addHours(utcNow(),8)`
> directamente en los dos sitios, el elemento y la respuesta quedarían con
> milisegundos distintos.

**5. Crear elemento** (SharePoint) — lista `AutoconsumosSesiones`

| Campo | Valor |
|---|---|
| `Title` | `@{outputs('token')}` |
| `Centro` | `@{first(body('Obtener_elementos')?['value'])?['Title']}` |
| `Rol` | `@{coalesce(first(body('Obtener_elementos')?['value'])?['Rol']?['Value'], 'CENTRO')}` |
| `Expira` | `@{outputs('expira')}` |

**6. Respuesta** — Código de estado `200`

Encabezados: `Content-Type` → `application/json`

Cuerpo:
```json
{
  "ok": true,
  "centro": "@{first(body('Obtener_elementos')?['value'])?['Title']}",
  "rol": "@{coalesce(first(body('Obtener_elementos')?['value'])?['Rol']?['Value'], 'CENTRO')}",
  "token": "@{outputs('token')}",
  "expira": "@{outputs('expira')}"
}
```

#### Rama **En caso negativo**

**7. Retrasar** — 2 segundos.
   Frena el probar contraseñas a lo bruto. Cuesta nada y ayuda bastante.

**8. Respuesta** — `200`, cuerpo:
```json
{ "ok": false }
```

> Se responde `200` y no `401` a propósito: el tablero lee `ok` del JSON, y un
> código de error haría que algunos navegadores no dejaran leer el cuerpo.
>
> Y se responde lo mismo si el centro no existe que si la contraseña está mal.
> Distinguirlos le diría a quien prueba cuáles centros existen.

---

## Flujo 2 · `Autoconsumos - Obtener datos`

Devuelve la cartera que ese token tiene permitido ver.

### Disparador

```json
{
  "type": "object",
  "properties": { "token": { "type": "string" } }
}
```

### Acciones

**1. Obtener elementos** — lista `AutoconsumosSesiones`

- Consulta de filtro:
  ```
  Title eq '@{replace(triggerBody()?['token'], '''', '''''')}'
  ```
- Máximo: `1`

**2. Condición** — `¿Sesión válida?`
```
@and(
  greater(length(body('Obtener_elementos')?['value']), 0),
  greater(first(body('Obtener_elementos')?['value'])?['Expira'], utcNow())
)
```

**Rama negativa → Respuesta `200`:**
```json
{ "ok": false, "motivo": "sesion" }
```

El tablero reconoce `motivo: "sesion"` y devuelve al usuario a la pantalla de
acceso con el mensaje correspondiente. Use exactamente esa palabra.

#### Rama afirmativa

**3. Inicializar variable** — `centro` (Cadena)
```
@{first(body('Obtener_elementos')?['value'])?['Centro']}
```

**4. Inicializar variable** — `rol` (Cadena)
```
@{first(body('Obtener_elementos')?['value'])?['Rol']}
```

**4 bis. Redactar** — `MapaCentros`

La lista `Accesos` y el Excel escriben los centros distinto: `CM KENNEDY`
frente a `Kennedy`, `CM QUICENTRO SUR` frente a `Quicentro`. No es una
diferencia mecánica —no basta con quitar el «CM » y los acentos—, así que la
equivalencia va escrita a mano.

Entradas (como objeto, **sin** `fx`):

```json
{
  "CM CALDERÓN": "Calderon",
  "CM CARCELÉN": "Carcelen",
  "CM CONDADO": "Condado",
  "CM CAROLINA": "Carolina",
  "CM CUMBAYA": "Cumbaya",
  "CM CHILLOS": "Chillos",
  "CM QUICENTRO SUR": "Quicentro",
  "CM ALBORADA": "Alborada",
  "CM KENNEDY": "Kennedy",
  "CM CIUDAD CELESTE": "Ciudad Celeste",
  "CM AMAZONAS": "Amazonas"
}
```

> **Al alta de un centro nuevo hay que tocar este mapa.** Si no, esa
> supervisora entra sin problema y ve cero facturas, sin ningún mensaje que lo
> explique. Es el error más difícil de diagnosticar de todo el montaje.
>
> `Carcelen` está puesto por simetría: ese centro no tiene ninguna factura en la
> base actual, así que su escritura en el Excel no está confirmada.

**4 ter. Redactar** — `centroExcel`

```
coalesce(outputs('MapaCentros')?[outputs('centro')], outputs('centro'))
```

El `coalesce` es la red de seguridad: si entra un centro que no está en el mapa,
se usa su nombre tal cual en lugar de devolver vacío. El filtro falla hacia «no
encuentra nada», nunca hacia «ve lo de otro centro».

**5. Enumerar filas presentes en una tabla** (Excel Online (Business))

| Campo | Valor |
|---|---|
| Ubicación | El sitio de SharePoint o el OneDrive donde esté el archivo |
| Biblioteca de documentos | La que corresponda |
| Archivo | `BASE AUTOCONSUMOS.xlsx` |
| Tabla | `TablaNoLiquidado` |

> ⚠ **Active la paginación.** Menú «…» de la acción → **Configuración →
> Paginación: Activado**, umbral `100000`.
>
> Sin esto el conector devuelve **256 filas y ya**, sin error y sin aviso. El
> tablero mostraría una cartera incompleta y nadie lo notaría. Es el punto donde
> más fácil se falla al montar esto.

*(Si usó el Office Script del paso 1, aquí va **Ejecutar script** en lugar de esta
acción. El resto no cambia.)*

**6. Filtrar matriz**

- Desde: `@body('Enumerar_filas_presentes_en_una_tabla')?['value']`
- Condición, en modo avanzado:
  ```
  @or(
    equals(variables('rol'), 'ADMIN'),
    equals(item()?['Centro'], variables('centro'))
  )
  ```

Aquí es donde un centro deja de poder ver lo de otro. El filtro ocurre en el
servidor: al navegador nunca le llegan las filas ajenas.

**7. Obtener elementos** — lista `AutoconsumosSeguimiento` — renómbrela
`Obtener marcas`

- Consulta de filtro:
  ```
  @{if(equals(variables('rol'),'ADMIN'), '', concat('Centro eq ''', replace(variables('centro'), '''', ''''''), ''''))}
  ```
- Paginación activada, umbral `100000`

**8. Seleccionar** — renómbrela `Marcas`

- Desde: `@body('Obtener_marcas')?['value']`
- Asignación:

| Clave | Valor |
|---|---|
| `factura` | `@{item()?['Title']}` |
| `centro` | `@{item()?['Centro']}` |
| `observacion` | `@{item()?['Observacion']}` |
| `marcadoPor` | `@{item()?['MarcadoPor']}` |
| `marcadoEn` | `@{item()?['MarcadoEn']}` |

**9. Obtener propiedades del archivo** (SharePoint) — el `.xlsx`

Sirve para la fecha de corte: la de última modificación del archivo, no la del
reloj de quien abre el tablero. Es la corrección del hallazgo B1 del informe.

**10. Respuesta** — `200`, `Content-Type: application/json`

```json
{
  "ok": true,
  "centro": "@{variables('centro')}",
  "rol": "@{variables('rol')}",
  "corte": "@{formatDateTime(body('Obtener_propiedades_del_archivo')?['Modified'], 'dd/MM/yyyy')}",
  "consultado": "@{utcNow()}",
  "origen": "BASE AUTOCONSUMOS.xlsx",
  "filas": @{body('Filtrar_matriz')},
  "marcas": @{body('Marcas')}
}
```

> `filas` y `marcas` van **sin comillas**: son arreglos, no texto. Si se dejan
> entre comillas llegan como una cadena y el tablero no encuentra ninguna columna.

---

## Flujo 3 · `Autoconsumos - Marcar facturas`

Registra el marcado. **Solo crea. Nunca actualiza ni elimina.**

### Disparador

```json
{
  "type": "object",
  "properties": {
    "token": { "type": "string" },
    "facturas": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "factura":     { "type": "string" },
          "observacion": { "type": "string" }
        }
      }
    }
  }
}
```

### Acciones

**1 y 2.** Validar la sesión, idénticas al flujo 2 (incluida la respuesta
`{"ok": false, "motivo": "sesion"}` en la rama negativa).

**3.** Inicializar variable `centro` (Cadena), igual que antes.

**4.** Inicializar variable `grabadas` (Matriz), valor `[]`

**5.** Inicializar variable `rechazadas` (Matriz), valor `[]`

**6. Aplicar a cada uno** — sobre `@triggerBody()?['facturas']`

> ⚠ Menú «…» → **Configuración → Control de simultaneidad: Activado, Grado de
> paralelismo: 1.**
>
> En serie, no en paralelo. Con paralelismo, dos elementos del mismo lote pueden
> chocar contra la restricción de unicidad al mismo tiempo y el manejo del error
> se vuelve impredecible.

Dentro del bucle:

**6.1. Ámbito** — nómbrelo `Intento`. Dentro, una sola acción:

**Crear elemento** (SharePoint) — lista `AutoconsumosSeguimiento`

| Campo | Valor |
|---|---|
| `Title` | `@{toUpper(trim(item()?['factura']))}` |
| `Centro` | `@{variables('centro')}` |
| `Observacion` | `@{item()?['observacion']}` |
| `MarcadoPor` | `@{variables('centro')}` |
| `MarcadoEn` | `@{utcNow()}` |

**6.2. Anexar a la matriz** — `grabadas`

Menú «…» → **Configurar ejecución después → Es correcto** (solo eso).

```json
{
  "factura": "@{toUpper(trim(item()?['factura']))}",
  "centro": "@{variables('centro')}",
  "observacion": "@{item()?['observacion']}",
  "marcadoPor": "@{variables('centro')}",
  "marcadoEn": "@{body('Crear_elemento')?['MarcadoEn']}"
}
```

**6.3. Obtener elementos** — renómbrela `Buscar dueño`

Menú «…» → **Configurar ejecución después → Se ha producido un error** y
**Se ha agotado el tiempo de espera**. Desmarque «Es correcto».

- Lista: `AutoconsumosSeguimiento`
- Filtro: `Title eq '@{replace(toUpper(trim(item()?['factura'])), '''', '''''')}'`
- Máximo: `1`

**6.4. Anexar a la matriz** — `rechazadas` (ejecución después: **Es correcto**)

```json
{
  "factura": "@{toUpper(trim(item()?['factura']))}",
  "motivo": "ya_marcada",
  "centro": "@{first(body('Buscar_dueño')?['value'])?['Centro']}",
  "observacion": "@{first(body('Buscar_dueño')?['value'])?['Observacion']}",
  "marcadoPor": "@{first(body('Buscar_dueño')?['value'])?['MarcadoPor']}",
  "marcadoEn": "@{first(body('Buscar_dueño')?['value'])?['MarcadoEn']}"
}
```

**7. Respuesta** — `200`

```json
{
  "ok": true,
  "grabadas": @{variables('grabadas')},
  "rechazadas": @{variables('rechazadas')}
}
```

### Por qué está armado así

El `Crear elemento` **falla a propósito** cuando la factura ya está marcada,
porque la columna `Title` tiene valores únicos obligatorios. Ese fallo es la
señal: no se consulta antes de escribir, se intenta escribir y se recoge el
rechazo. Entre una consulta previa y la escritura cabría otra escritura; aquí no
cabe nada, porque quien decide es SharePoint.

El `Ámbito` existe para que ese fallo no aborte el bucle: se captura y se sigue
con la siguiente factura.

Y no hay ninguna acción de **Actualizar elemento** ni de **Eliminar elemento** en
todo el flujo. Es deliberado. Si alguien agrega una, el marcado deja de ser
definitivo.

---

## Flujo 3b · `Autoconsumos - Subir respaldo`

Recibe uno o varios PDF, los guarda en la biblioteca y marca la factura.
**Subir el respaldo es marcar**: comparte la regla de irreversibilidad del
Flujo 3, así que aquí tampoco existe ninguna acción de actualizar ni eliminar.

> **Antes de armarlo, dos cosas en SharePoint:**
> 1. En `AutoconsumosSeguimiento`, agregue la columna **`Respaldo`**
>    (una línea de texto). Ahí va el enlace al PDF.
> 2. En la biblioteca **Documentos** del sitio, cree la carpeta **`Respaldos`**.

El número de factura **no se lee del contenido del PDF**: viene del nombre del
archivo, que el tablero interpreta antes de enviar. El flujo recibe la factura
ya resuelta y no adivina nada.

### Disparador

`Cuando se recibe una solicitud HTTP`, método **POST**, «¿Quién puede
desencadenar el flujo?» → **Cualquier persona**.

```json
{
  "type": "object",
  "properties": {
    "token": { "type": "string" },
    "archivos": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "factura":     { "type": "string" },
          "nombre":      { "type": "string" },
          "observacion": { "type": "string" },
          "contenido":   { "type": "string" }
        }
      }
    }
  }
}
```

`contenido` es el PDF en base64.

### Acciones

Idénticas al Flujo 3 hasta la condición de sesión:

1. **Obtener elementos** → `AutoconsumosSesiones`, paginación 100000
2. **Filtrar matriz** `FiltroSesion`, en modo avanzado:
   ```
   @and(equals(triggerBody()?['token'], item()?['Title']), greater(ticks(item()?['Expira']), ticks(utcNow())))
   ```
3. **Inicializar variable** `grabadas` (Matriz, `[]`) — en el tronco, no dentro de la condición
4. **Inicializar variable** `rechazadas` (Matriz, `[]`) — igual
5. **Condición** `length(body('FiltroSesion'))` es mayor que `0`

#### Rama negativa

**Respuesta** 200 · `Content-Type: application/json`

```json
{ "ok": false, "motivo": "sesion" }
```

#### Rama afirmativa

**Redactar** `centro`
```
first(body('FiltroSesion'))?['Centro']
```

**Obtener elementos** `MarcasExistentes` → `AutoconsumosSeguimiento`, paginación 100000

**Aplicar a cada uno** `PorArchivo` — Salidas seleccionadas: `triggerBody()?['archivos']`

> ⚙ Configuración → **Control de simultaneidad: Activado, paralelismo 1.**
> Sin esto dos archivos del lote pueden chocar sobre la misma factura.

Dentro del bucle:

**Filtrar matriz** `YaMarcada`
- Desde: `outputs('MarcasExistentes')?['body/value']`
- Consulta, en modo avanzado:
  ```
  @equals(toUpper(trim(items('PorArchivo')?['factura'])), item()?['Title'])
  ```

**Condición** `length(body('YaMarcada'))` es mayor que `0`

**Rama verdadera** — ya estaba marcada. No se sube el archivo: un PDF por
factura, una sola verdad.

**Anexar a la variable de matriz** `rechazadas`:
```json
{
  "factura": "@{toUpper(trim(items('PorArchivo')?['factura']))}",
  "motivo": "ya_marcada",
  "centro": "@{first(body('YaMarcada'))?['Centro']}",
  "observacion": "@{first(body('YaMarcada'))?['Observacion']}",
  "marcadoPor": "@{first(body('YaMarcada'))?['MarcadoPor']}",
  "marcadoEn": "@{first(body('YaMarcada'))?['MarcadoEn']}",
  "respaldo": "@{first(body('YaMarcada'))?['Respaldo']}"
}
```

**Rama falsa** — se puede grabar. Tres acciones:

**1. Redactar** `sello` → `utcNow()`

**2. Crear archivo** (SharePoint)

| Campo | Valor |
|---|---|
| Dirección del sitio | `https://metroredec.sharepoint.com/sites/Autoconsumos` |
| Ruta de la carpeta | `/Documentos compartidos/Respaldos` |
| Nombre del archivo | `@{toUpper(trim(items('PorArchivo')?['factura']))}.pdf` |
| Contenido del archivo | `@{base64ToBinary(items('PorArchivo')?['contenido'])}` |

> `base64ToBinary` es imprescindible. Sin esa función el PDF se guarda como
> texto y el archivo no abre.
>
> El archivo se nombra con la factura, no con el nombre original. Así el
> respaldo es localizable por sí mismo y dos personas no pueden subir
> `escaneo.pdf` encima del otro.

**3. Crear elemento** (SharePoint) → `AutoconsumosSeguimiento`

| Campo | Valor |
|---|---|
| Título | `toUpper(trim(items('PorArchivo')?['factura']))` |
| Centro | `outputs('centro')` |
| Observacion | `items('PorArchivo')?['observacion']` |
| MarcadoPor | `outputs('centro')` |
| MarcadoEn | `outputs('sello')` |
| Respaldo | `body('Crear_archivo')?['{Link}']` |

> Si `{Link}` no aparece en el contenido dinámico, use **Ruta de acceso**
> (`Path`). El tablero acepta las dos: si recibe una ruta relativa la completa
> con `CONFIG.SITIO_SHAREPOINT`.

**4. Anexar a la variable de matriz** `grabadas`:
```json
{
  "factura": "@{toUpper(trim(items('PorArchivo')?['factura']))}",
  "centro": "@{outputs('centro')}",
  "observacion": "@{items('PorArchivo')?['observacion']}",
  "marcadoPor": "@{outputs('centro')}",
  "marcadoEn": "@{outputs('sello')}",
  "respaldo": "@{body('Crear_archivo')?['{Link}']}"
}
```

#### Y al final, fuera del bucle

**Respuesta** 200 · `Content-Type: application/json`

```json
{
  "ok": true,
  "grabadas": "@variables('grabadas')",
  "rechazadas": "@variables('rechazadas')"
}
```

### El Flujo 2 también cambia

Para que el 📎 aparezca al recargar, la acción `Marcas` del Flujo 2 necesita una
fila más en su mapa:

| Clave | Valor |
|---|---|
| `respaldo` | `item()?['Respaldo']` |

Sin eso el enlace se ve al subirlo pero desaparece al refrescar la página.

### Comprobar que quedó bien

1. Suba un PDF de una factura pendiente → la fila queda con 🔒 y 📎
2. **Suba el mismo PDF otra vez** → debe rechazarlo diciendo quién la marcó,
   y en la biblioteca debe haber **un solo archivo**, no dos
3. Recargue con ↻ → el 📎 debe seguir ahí
4. Entre con otro centro → debe ver la factura bloqueada

El punto 2 es el que prueba que el respaldo hereda la irreversibilidad.

## Flujo 4 · `Autoconsumos - Limpiar sesiones` *(opcional pero recomendable)*

**Disparador: Periodicidad**, una vez al día.

1. **Obtener elementos** — `AutoconsumosSesiones`
   Filtro: `Expira lt '@{utcNow()}'` · paginación activada
2. **Aplicar a cada uno** sobre `@body('Obtener_elementos')?['value']`
   → **Eliminar elemento**, Id: `@{item()?['ID']}`

Sin esto la lista crece sin parar y las consultas se van poniendo lentas.

---

## Flujo 5 · `Autoconsumos - Catálogo de centros` *(opcional)*

Llena el desplegable de la pantalla de acceso sin tener que editar el HTML cada
vez que se agrega un centro.

**Disparador HTTP**, esquema `{}`.

1. **Obtener elementos** — `AutoconsumosAccesos`
   Filtro: `Activo eq 1 and Rol eq 'CENTRO'`
2. **Seleccionar** — Desde `@body('Obtener_elementos')?['value']`,
   en modo texto: `@item()?['Title']`
3. **Respuesta** — `200`:
   ```json
   { "ok": true, "centros": @{body('Seleccionar')} }
   ```

Su URL va en `CATALOGOS_URL`. Si se deja vacío, el tablero usa la lista fija de
`CENTROS_RESPALDO`.

> Este flujo responde sin pedir credenciales, así que expone los nombres de los
> centros. No es información sensible, pero conviene saberlo.

---

## Comprobar que quedó bien

Antes de tocar el `index.html`, pruebe cada flujo con el «Probar» de Power
Automate o desde una terminal:

```bash
# 1 · Acceso
curl -X POST "URL_DEL_FLUJO_1" -H "Content-Type: application/json" \
     -d '{"centro":"CM KENNEDY","contrasena":"la-clave"}'
# Debe responder: {"ok":true,"centro":"CM KENNEDY","rol":"CENTRO","token":"…","expira":"…"}

# 2 · Datos, con el token que devolvió el anterior
curl -X POST "URL_DEL_FLUJO_2" -H "Content-Type: application/json" \
     -d '{"token":"EL-TOKEN"}'
# Revise: que "filas" traiga MÁS de 256 elementos si la cartera es mayor,
# y que TODOS sean del centro que entró.

# 3 · Marcado, con una factura de prueba
curl -X POST "URL_DEL_FLUJO_3" -H "Content-Type: application/json" \
     -d '{"token":"EL-TOKEN","facturas":[{"factura":"001-001-000000001","observacion":"prueba"}]}'
# La primera vez: "grabadas" con un elemento.
# Repita el MISMO comando: debe caer en "rechazadas" con motivo "ya_marcada".
```

Esa última prueba —correr el mismo marcado dos veces— es la que confirma que la
inmutabilidad funciona de verdad. Hágala antes de dar el tablero por listo, y
borre después el elemento de prueba de la lista.
