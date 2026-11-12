# 3 · Contrato entre el tablero y los flujos

El JSON exacto que se intercambian el `index.html`, el robot y los flujos de
Power Automate. Sirve para montar o reconstruir los flujos sin adivinar, y para
diagnosticar cuando algo no funciona.

Versión vigente desde noviembre de 2026: acceso por usuario, respaldos en una
biblioteca aparte, entrega solo con PDF, y el robot como un usuario más.

Los flujos ya no se arman a mano: cada uno se genera con un programa en
`flujos/construir_*.py` que produce un paquete `.zip` para «Importar paquete».
El código de esos programas es la definición de referencia; este documento la
explica.

---

## Reglas generales

- Todo es `POST` con `Content-Type: application/json`.
- **Siempre se responde `200`**, incluso en los errores. El resultado va en `ok`.
- Si `ok` es `false` y `motivo` es `"sesion"`, el tablero cierra la sesión y
  vuelve a la pantalla de acceso. Esa palabra es literal.
- Si `ok` es `false` y `motivo` es `"sin_permiso"`, el rol de la sesión no puede
  hacer esa operación.
- Las fechas van en ISO 8601 con zona (`2026-11-11T16:22:18Z`) y son **la hora
  del servidor**, nunca la del equipo del usuario.

## Roles

Salen de la columna `Rol` de la lista `Accesos` (sitio EquipamientoConsultorios).

| Rol | Quién | Ve | Puede |
|---|---|---|---|
| `CENTRO` | Supervisora de un centro | Solo su centro | Adjuntar PDF y fotos, entregar |
| `ADMIN` | Administración | Todos los centros | Lo mismo que CENTRO, en cualquier centro, más análisis y CSV |
| `OPERACIONES` | Operaciones | Todos los centros | Solo mirar |
| `RPA` | El robot | Todos los centros | Adjuntar PDF y reportar corridas. **No puede entrar al tablero ni entregar** |

## Nombres de centro

Conviven dos escrituras y los flujos traducen entre ellas con `MapaCentros`:

| En `Accesos` y en Seguimiento | En el Excel, el caché y las carpetas de Respaldos |
|---|---|
| `CM KENNEDY` | `Kennedy` |
| `CM CIUDAD CELESTE` | `Ciudad Celeste` |
| `CM QUICENTRO SUR` | `Quicentro` |
| … | … |

---

## Flujo 1 · Verificar acceso

### Envía

```json
{ "usuario": "CM KENNEDY", "centro": "CM KENNEDY", "contrasena": "…" }
```

`usuario` es el `Title` de la fila en `Accesos`. El tablero manda el mismo valor
en `centro` por compatibilidad. Solo se aceptan filas con `Sistema` igual a
`AUTOCONSUMOS` o `AMBOS`.

### Responde — correcto

```json
{
  "ok": true,
  "usuario": "CM KENNEDY",
  "centro": "CM KENNEDY",
  "ciudad": "Quito",
  "rol": "CENTRO",
  "token": "8f3a1c22-9b4e-4d17-a0f5-2e7c9d143b6a",
  "expira": "2026-11-12T03:30:00Z"
}
```

`centro` es la columna `CentroMedico` de la fila, o el `Title` si está vacía.
El tablero guarda `usuario`, `centro`, `ciudad`, `rol`, `token` y `expira`;
la contraseña no se guarda en ninguna parte.

Si `rol` es `RPA`, el tablero rechaza la entrada: ese usuario es solo para el
robot.

### Responde — incorrecto

```json
{ "ok": false }
```

---

## Flujo 2 · Obtener datos

### Envía

```json
{ "token": "…" }
```

### Responde — correcto

```json
{
  "ok": true,
  "usuario": "CM KENNEDY",
  "centro": "CM KENNEDY",
  "rol": "CENTRO",
  "corte": "14/09/2026",
  "consultado": "2026-11-11T20:34:00Z",
  "origen": "BASE AUTOCONSUMOS.xlsx",
  "filas": [
    { "c": "Kennedy", "f": "013-111-000024730", "d": "2026-04-20",
      "u": "MARIA … | 2026/04/20 16:48:40 | Autoconsumos: 30.50",
      "p": "Acuerdos Comerciales | PLAN NORMAL", "v": 30.5, "s": 30.5 }
  ],
  "marcas": [
    { "factura": "013-111-000024730", "centro": "CM KENNEDY",
      "observacion": "prueba", "marcadoPor": "CM KENNEDY",
      "marcadoEn": "2026-09-14T16:10:00Z",
      "respaldo": "Respaldos/Kennedy/013-111-000024730.pdf", "id": 12 }
  ],
  "respaldos": [
    { "f": "013-111-000024730", "t": "PDF", "n": "013-111-000024730.pdf",
      "u": "Respaldos/Kennedy/013-111-000024730.pdf",
      "o": "RPA", "e": "2026-11-11T16:37:44Z" },
    { "f": "013-111-000024730", "t": "FOTO",
      "n": "013-111-000024730_foto_20261111-162215.jpg",
      "u": "Respaldos/Kennedy/013-111-000024730_foto_20261111-162215.jpg",
      "o": "CM KENNEDY", "e": "2026-11-11T16:22:18Z" }
  ],
  "avisos": [
    { "f": "013-113-000033174", "c": "Kennedy",
      "e": "2026-11-11T07:05:36Z", "k": "RPA-20261111-0200" }
  ]
}
```

| Bloque | Qué es | Quién lo ve |
|---|---|---|
| `filas` | La cartera, proyectada del caché: `c` centro, `f` factura, `d` fecha, `u` caja, `p` plan, `v` valor, `s` saldo | CENTRO: solo su archivo `/Cache/<Centro>.json`. Los demás: `/Cache/_todos.json` |
| `marcas` | Las entregas registradas en `AutoconsumosSeguimiento` | CENTRO: solo las suyas |
| `respaldos` | Archivos de la biblioteca `Respaldos`: `f` factura, `t` `PDF` o `FOTO`, `n` nombre, `u` ruta completa, `o` quién lo subió, `e` cuándo | CENTRO: solo su carpeta |
| `avisos` | Facturas que el robot no encontró en EC Datos (`RPA_NO_ENCONTRADA` en auditoría): `f` factura, `c` centro, `e` cuándo, `k` corrida | CENTRO: solo su centro |

El enlace al archivo se arma en el tablero como `<sitio>/<u>` codificado. El
tablero solo muestra el aviso de una factura si sigue sin PDF.

### Responde — sesión vencida

```json
{ "ok": false, "motivo": "sesion" }
```

---

## Flujo 3 · Marcar facturas (entregar)

### Envía

```json
{
  "token": "…",
  "facturas": [
    { "factura": "013-111-000024730", "observacion": "Entregada a Administración." }
  ]
}
```

### Responde

```json
{
  "ok": true,
  "usuario": "CM KENNEDY",
  "grabadas": [
    { "factura": "013-111-000024730", "centro": "CM KENNEDY",
      "observacion": "…", "marcadoPor": "CM KENNEDY",
      "marcadoEn": "2026-11-11T16:46:00Z",
      "respaldo": "Respaldos/Kennedy/013-111-000024730.pdf" }
  ],
  "rechazadas": [
    { "factura": "013-111-000024731", "motivo": "sin_pdf" }
  ]
}
```

Reglas, en el orden en que se aplican:

| Caso | Resultado |
|---|---|
| Rol `OPERACIONES` o `RPA` | `{ "ok": false, "motivo": "sin_permiso" }` y no se graba nada |
| La factura ya está en Seguimiento | rechazada `ya_marcada`, con `centro`, `marcadoPor`, `marcadoEn` del dueño real |
| No está en el caché, o es CENTRO y pertenece a otro centro | rechazada `fuera_de_cartera` |
| No tiene un archivo `PDF` en Respaldos | rechazada `sin_pdf` |
| Todo bien | grabada: `Centro` = centro dueño (`CM KENNEDY`), `MarcadoPor` = usuario de la sesión, `Respaldo` = ruta del PDF; auditoría `ENTREGO` |

Cada factura enviada aparece exactamente una vez, en `grabadas` o en
`rechazadas`. El flujo **solo crea** filas; nunca modifica ni borra. La unicidad
de `Title` en Seguimiento es lo que impide dos entregas de la misma factura.

---

## Flujo 4 · Adjuntar respaldo

Lo usan el tablero (carga masiva de PDF, botón Foto) y el robot.

### Envía

```json
{
  "token": "…",
  "corrida": "RPA-20261112-0100",
  "archivos": [
    { "factura": "013-111-000024730", "tipo": "PDF",
      "nombre": "013-111-000024730.pdf", "contenido": "<base64>",
      "reemplazar": false },
    { "factura": "013-111-000024730", "tipo": "FOTO",
      "nombre": "foto.jpg", "contenido": "<base64>" }
  ]
}
```

`corrida` lo manda el robot; el tablero lo deja vacío. Se envía de 10 en 10:
el flujo tarda unos 4 segundos por archivo y debe responder antes de 120.

### Responde

```json
{
  "ok": true,
  "usuario": "RPA",
  "grabados": [
    { "factura": "013-111-000024730", "tipo": "PDF",
      "nombre": "013-111-000024730.pdf",
      "ruta": "/Respaldos/Kennedy/013-111-000024730.pdf",
      "origen": "RPA", "fecha": "2026-11-12T06:12:00Z" }
  ],
  "rechazados": [
    { "factura": "017-113-000029610", "motivo": "ya_tiene_pdf",
      "origen": "CM ALBORADA", "fecha": "2026-11-10T19:40:22Z",
      "ruta": "/Respaldos/Alborada/017-113-000029610.pdf" }
  ]
}
```

| Caso | Resultado |
|---|---|
| Rol `OPERACIONES` | rechazado `sin_permiso` |
| Factura no está en el caché, o es CENTRO y es de otro centro | rechazado `fuera_de_cartera` |
| `PDF` y ya existe uno, sin `reemplazar` | rechazado `ya_tiene_pdf`, con quién y cuándo lo subió |
| `PDF` y ya existe uno, con `reemplazar: true` | se sobreescribe (SharePoint guarda la versión anterior); auditoría `REEMPLAZO_PDF` |
| `PDF` nuevo | se guarda como `/Respaldos/<Centro>/<factura>.pdf`; auditoría `SUBIO_PDF` |
| `FOTO` | se guarda como `<factura>_foto_<aaaammdd-HHmmss>.<ext>`; varias por factura; auditoría `SUBIO_FOTO` |

Cada archivo guardado queda con las columnas `Factura`, `Centro`,
`TipoRespaldo`, `Origen` (el usuario) y `Corrida`. Adjuntar **nunca entrega** la
factura.

---

## Flujo 5 · Reportar corrida

Solo el robot, al terminar.

### Envía

```json
{
  "token": "…",
  "corrida": "RPA-20261112-0100",
  "bajadas": 120, "subidas": 118, "errores": 0,
  "noEncontradas": [ { "factura": "013-113-000033174", "centro": "Kennedy" } ]
}
```

### Responde

```json
{ "ok": true, "corrida": "RPA-20261112-0100", "avisos": 1 }
```

Crea en `AutoconsumosAuditoria` una fila `RPA_CORRIDA` con el resumen y una
fila `RPA_NO_ENCONTRADA` por factura. Con otro rol responde
`{ "ok": false, "motivo": "sin_permiso" }`.

---

## La auditoría

Lista `AutoconsumosAuditoria`. Una fila por acción; nadie la edita.

| `Accion` | Quién la produce | `Title` |
|---|---|---|
| `SUBIO_PDF` | Adjuntar | factura |
| `REEMPLAZO_PDF` | Adjuntar con `reemplazar` | factura |
| `SUBIO_FOTO` | Adjuntar | factura |
| `ENTREGO` | Marcar | factura |
| `RPA_CORRIDA` | Reportar corrida | corrida |
| `RPA_NO_ENCONTRADA` | Reportar corrida | factura |

Columnas comunes: `Quien` (usuario), `Rol`, `Centro`, `Archivo`, `Detalle`,
`Corrida`, `Cuando`.

---

## Cómo se ve un problema

| Síntoma en el tablero | Causa habitual |
|---|---|
| «Ese usuario es solo para el robot» | Se intentó entrar con el usuario RPA |
| Vuelve al acceso apenas entra | La sesión venció, o el flujo 2 no encuentra el token |
| «Falta el PDF» en todas las filas | El robot no ha corrido, o la biblioteca Respaldos no responde |
| Al adjuntar dice «fuera de tu cartera» | La factura no está en el caché, o es de otro centro |
| Al entregar dice «no tiene PDF» aunque se ve el PDF | El archivo no tiene `TipoRespaldo = PDF` o `Factura` vacía en sus propiedades |
| `WorkflowTriggerIsNotEnabled` | El flujo se importó pero no se activó |
| La carga masiva tarda y falla | Lotes de más de 10 archivos o archivos muy pesados: el flujo pasa de 120 s |
