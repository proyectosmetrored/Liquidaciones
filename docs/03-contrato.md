# 3 · Contrato entre el tablero y los flujos

El JSON exacto que se intercambian. Sirve para dos cosas: montar los flujos sin
adivinar, y diagnosticar cuando algo no funciona.

Si mañana se reemplaza Power Automate por otra cosa —una función, una API, un
servicio propio— basta con respetar este contrato y el `index.html` no cambia.

---

## Reglas generales

- Todo es `POST` con `Content-Type: application/json`.
- **Siempre se responde `200`**, incluso en los errores. El resultado se indica en
  el campo `ok`. Un código de error impediría al navegador leer el cuerpo en
  algunos casos.
- Si `ok` es `false` y `motivo` es `"sesion"`, el tablero cierra la sesión y
  devuelve al usuario a la pantalla de acceso. Esa palabra es literal.
- El tablero reintenta hasta 3 veces si la llamada falla por red (2 veces en el
  marcado, para no duplicar intentos de escritura).

---

## Flujo 1 · Verificar acceso

### Envía

```json
{
  "centro": "CM KENNEDY",
  "contrasena": "la-clave-del-centro"
}
```

### Responde — correcto

```json
{
  "ok": true,
  "centro": "CM KENNEDY",
  "rol": "CENTRO",
  "token": "8f3a1c22-9b4e-4d17-a0f5-2e7c9d143b6a",
  "expira": "2026-08-25T21:30:00Z"
}
```

| Campo | Qué es |
|---|---|
| `centro` | El nombre **tal como está en la lista de accesos**, no lo que escribió el usuario. Debe coincidir con la columna `Centro` del Excel |
| `rol` | `CENTRO` (ve solo lo suyo) o `ADMIN` (ve todos) |
| `token` | Identifica la sesión. Es lo único que el tablero guarda |
| `expira` | Fecha ISO. El tablero descarta la sesión guardada si ya pasó |

### Responde — incorrecto

```json
{ "ok": false }
```

Lo mismo si el centro no existe, si está inactivo o si la contraseña está mal.

> La contraseña viaja una sola vez, en esta llamada, y nunca se guarda: ni en
> `localStorage`, ni en `sessionStorage`, ni en una variable. A partir de aquí
> solo circula el token. Es la diferencia de fondo con el tablero anterior, donde
> la clave estaba escrita dentro del archivo.

---

## Flujo 2 · Obtener datos

### Envía

```json
{ "token": "8f3a1c22-9b4e-4d17-a0f5-2e7c9d143b6a" }
```

### Responde — correcto

```json
{
  "ok": true,
  "centro": "CM KENNEDY",
  "rol": "CENTRO",
  "corte": "19/08/2026",
  "consultado": "2026-08-25T13:30:00Z",
  "origen": "BASE AUTOCONSUMOS.xlsx",
  "filas": [
    {
      "Centro": "CM KENNEDY",
      "Número factura": "017-114-000016120",
      "Fecha": "2026-07-10",
      "User Caja": "mgomez",
      "Plan | Convenio": "BMI",
      "Valor": 18.5,
      "Saldo": 18.5
    }
  ],
  "marcas": [
    {
      "factura": "017-114-000016120",
      "centro": "CM KENNEDY",
      "observacion": "Enviada al convenio el lunes.",
      "marcadoPor": "CM KENNEDY",
      "marcadoEn": "2026-08-20T15:04:00Z"
    }
  ]
}
```

| Campo | Qué es |
|---|---|
| `corte` | Fecha de modificación del Excel, en `dd/mm/aaaa`. Se muestra en el encabezado y es contra lo que se mide la antigüedad |
| `consultado` | Momento de la consulta, en ISO. Aclara que el dato es de `corte`, no de ahora |
| `origen` | Texto libre para el pie de página |
| `filas` | **Tal como salen del Excel**, sin transformar |
| `marcas` | Los marcados ya registrados |

### Sobre `filas`

El Flujo 2 **no lee el Excel**. Lee un archivo JSON ya preparado por el flujo
`Autoconsumos - Actualizar cache`, y proyecta de ahí siete claves cortas:

| Clave | Significado |
|-------|-------------|
| `c`   | Centro |
| `f`   | Número de factura |
| `d`   | Fecha de emisión |
| `u`   | Usuario de caja |
| `p`   | Plan / convenio |
| `v`   | Valor |
| `s`   | Saldo pendiente |

Tres motivos para proyectar y no entregar la fila completa:

1. **Peso.** La fila cruda trae veinte columnas más los metadatos del conector.
   Con ~5.000 facturas la respuesta superaba el límite de la puerta de enlace y
   devolvía `502` — el flujo terminaba bien, pero la respuesta no salía.
2. **Datos del paciente.** La cédula, el nombre, la historia clínica y el
   diagnóstico **no entran en la proyección**, así que nunca llegan al
   navegador. Siguen existiendo en el Excel y en el registro de gestión.
3. **Velocidad.** La consulta pasó de ~35 s a ~4 s.

El tablero acepta tanto estas claves cortas como los nombres largos originales
(ver `ALIAS` en `index.html`).

`d` se acepta como `aaaa-mm-dd`, `dd/mm/aaaa` o como número de serie de Excel.
`v` y `s` se aceptan como número o como texto, con punto o con coma decimal.

### Qué archivo se lee

La acción `rutaCache` decide:

```
ADMIN   → /Cache/_todos.json
CENTRO  → /Cache/<nombre del centro según el Excel>.json
```

El nombre sale de `centroExcel`, que traduce el centro de la sesión
(`CM KENNEDY`, como en `Accesos`) al del Excel (`Kennedy`) usando `MapaCentros`.

> **El aislamiento por centro ya no es un filtro, es físico.** Al flujo de una
> supervisora solo le llega el archivo de su centro: las filas de los demás no
> se descartan, es que nunca se leen.

**Si el archivo no existe**, `LeerCache` falla con 404. Por eso la acción
`Filas` está configurada para ejecutarse también **«Ha fallado»**, y en ese caso
devuelve una cartera vacía en vez de romper. Pasa con un centro que tiene acceso
pero todavía no tiene facturas.

---

#### El corrimiento de encabezados del Excel

En `BASE AUTOCONSUMOS (2).xlsx` la fila de encabezados perdió el rótulo
`Paciente`, así que desde la sexta columna **cada nombre quedó sobre el dato de
la columna siguiente**: `Valor` está sobre los RUC, `Saldo` sobre los valores.
Los datos están completos y en orden; lo mal escrito es la fila 1.

Se decidió no tocar el Excel. **La corrección vive en un solo sitio: la acción
`FilasCompletas` del flujo `Autoconsumos - Actualizar cache`.** Ahí cada campo
se lee del encabezado que realmente lo contiene y se le pone su nombre correcto.

De ahí en adelante —el JSON del caché, el Flujo 2, el de marcado y el de
subir respaldo— todo trabaja con nombres limpios que dicen la verdad.

> Si alguien corrige la fila 1 del Excel, lo único que hay que ajustar es
> `FilasCompletas`. Nada más depende de ese corrimiento.

### Responde — sesión vencida

```json
{ "ok": false, "motivo": "sesion" }
```

### Responde — otro error

```json
{ "ok": false, "mensaje": "No se pudo abrir el archivo de origen." }
```

El texto de `mensaje` se muestra tal cual al usuario, con un botón de reintentar.

---

## Flujo 3 · Marcar facturas

### Envía

```json
{
  "token": "8f3a1c22-9b4e-4d17-a0f5-2e7c9d143b6a",
  "facturas": [
    { "factura": "017-114-000016120", "observacion": "Enviada al convenio." },
    { "factura": "017-114-000016121", "observacion": "" }
  ]
}
```

El tablero nunca envía una factura que ya sepa marcada. Aun así, el flujo debe
manejar el caso: entre que se cargó la pantalla y se confirmó, otro centro pudo
marcarla.

### Responde

```json
{
  "ok": true,
  "grabadas": [
    {
      "factura": "017-114-000016120",
      "centro": "CM KENNEDY",
      "observacion": "Enviada al convenio.",
      "marcadoPor": "CM KENNEDY",
      "marcadoEn": "2026-08-25T13:32:11Z"
    }
  ],
  "rechazadas": [
    {
      "factura": "017-114-000016121",
      "motivo": "ya_marcada",
      "centro": "CM CAROLINA",
      "observacion": "Ya se gestionó.",
      "marcadoPor": "CM CAROLINA",
      "marcadoEn": "2026-08-24T09:15:00Z"
    }
  ]
}
```

Cada factura enviada aparece **exactamente una vez**, en `grabadas` o en
`rechazadas`.

Con `motivo: "ya_marcada"` el tablero no se limita a avisar: toma los datos del
dueño real y bloquea esa fila en pantalla. Así el usuario ve de inmediato quién la
tiene, sin recargar.

`marcadoEn` es **la hora del servidor**. Nunca la del equipo del usuario: si no,
cada quien firmaría con la hora de su reloj.

---

## Cómo se ve un problema

| Síntoma en el tablero | Causa habitual |
|---|---|
| «Al archivo de origen le faltan columnas: …» | Renombraron una columna del Excel. Agregue el nombre nuevo a `ALIAS` |
| Entra pero la tabla sale vacía | El `Filtrar matriz` compara con un nombre de centro que no coincide con el del Excel |
| Faltan facturas y nadie sabe cuáles | Paginación desactivada en «Enumerar filas»: llegaron solo 256 |
| «No se pudo consultar el origen de datos» | La URL del flujo está mal, o el flujo está apagado |
| Vuelve al acceso apenas entra | La sesión venció, o el flujo 2 no encuentra el token |
| Las filas llegan pero no se reconoce ninguna columna | `filas` se envió entre comillas en la Respuesta: llegó como texto |
| Marca dos veces la misma factura | Falta «Exigir valores únicos» en `Title`. **Corríjalo antes de publicar** |
