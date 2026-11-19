# 5 · El robot y los respaldos

Desde noviembre de 2026 los PDF de las facturas no los sube cada centro: los
pone un robot, bajándolos del portal de facturación electrónica **EC Datos**.
Corre cada noche y también se puede lanzar a cualquier hora. Los centros solo
revisan, agregan fotos si hace falta y entregan.

Este documento explica dónde vive cada cosa, cómo corre el robot y cómo leer
lo que deja.

---

## 5.1 Las piezas

```
  Excel BASE AUTOCONSUMOS            EC Datos (clientes.ecdatos.com)
  hojas NO LIQUIDADO y LIQUIDADO               │
         │ al guardarse                         │ cada noche
         ▼                                      ▼
  Actualizar caché  ──►  /Cache/*.json   ◄──  Robot (P:\rpa_ecdatos)
                                │              │  baja PDF y los sube
                                ▼              ▼
                     Flujos HTTP de Power Automate
                       Verificar · Obtener datos · Marcar
                       Adjuntar respaldo · Reportar corrida · Publicar archivo
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
        index.html     biblioteca Respaldos   AutoconsumosAuditoria
      (supervisoras)   /Respaldos/<Centro>/   quién hizo qué y cuándo
```

| Pieza | Dónde | Qué guarda |
|---|---|---|
| Biblioteca `Respaldos` | sitio Autoconsumos | Una carpeta por centro (`Kennedy`, `Alborada`, `Ciudad Celeste`…). Cada archivo lleva `Factura`, `Centro`, `TipoRespaldo` (PDF o FOTO), `Origen`, `Corrida`. Columnas `Centro`, `Factura` y `TipoRespaldo` **indexadas** para pasar de 5.000 archivos |
| Carpeta `Respaldos/Reportes` | sitio Autoconsumos | `LIQUIDADO.xlsx`, el Excel de solo lectura con todas las liquidadas (entregadas en el tablero + hoja LIQUIDADO del Excel base). Lo regenera el robot; Administración y Operaciones lo leen |
| `Cache/_liquidado.json` | sitio Autoconsumos | La hoja LIQUIDADO del Excel base, que «Actualizar caché» deja con la misma forma que los demás JSON del caché (tabla `TablaLiquidado`) |
| Lista `AutoconsumosAuditoria` | sitio Autoconsumos | Una fila por acción. Ver el contrato, sección «La auditoría» |
| Lista `Accesos` | sitio EquipamientoConsultorios | Usuarios de ambos sistemas. Columnas `Title`, `field_1` (clave), `CentroMedico`, `Rol`, `Ciudad`, `Sistema` |
| Robot | `P:\rpa_ecdatos` | Código, configuración, caché de PDF, logs y resultados |

## 5.2 Reglas de negocio

- **Adjuntar no es entregar.** Un PDF o una foto solo quedan guardados; la
  factura sigue pendiente hasta que alguien pulsa Entregar.
- **No se puede entregar sin PDF.** El flujo lo verifica; el botón del tablero
  dice «Falta el PDF» mientras tanto.
- **Una factura, un PDF.** Si ya tiene uno, el tablero ofrece reemplazarlo y
  queda auditoría `REEMPLAZO_PDF`; SharePoint conserva la versión anterior.
- **Fotos, las que hagan falta**, antes o después de entregar. Se reducen a
  1600 px en el navegador antes de subirlas.
- **Entregar es definitivo** y equivale a liquidar. Queda firmado con el usuario
  y la hora del servidor; nada se edita ni se borra desde el tablero.
- **El robot nunca entrega.** Su usuario tiene rol `RPA`: puede adjuntar PDF y
  reportar, nada más, y no puede abrir el tablero.
- **Un solo dueño por dato.** El Excel es de Administración y solo alimenta la
  cartera. Las entregas, los respaldos y la auditoría viven en SharePoint. Nadie
  escribe en el archivo del otro.

## 5.3 El robot

Carpeta `P:\rpa_ecdatos`, con su propio `README.md` detallado. Lo esencial:

| Archivo | Para qué |
|---|---|
| `subir_tablero.py` | La corrida: entra como `RPA`, pide la cartera, baja de EC Datos lo que no tiene PDF y lo sube de 10 en 10 |
| `descargar.py` | Baja PDF por lista de números a una carpeta, sin tocar el tablero. Sirve para preparar cargas masivas manuales |
| `exportar_liquidado.py` | Arma `LIQUIDADO.xlsx` (hojas LIQUIDADO y RESUMEN) con las entregadas del tablero y la hoja LIQUIDADO del Excel base, y lo publica con el flujo «Publicar archivo». Lo llama `subir_tablero.py` al terminar |
| `ecdatos.py` | El cliente de la API directa de EC Datos: una dirección por factura devuelve el PDF, sin iniciar sesión. Ocho descargas a la vez |
| `ecdatos_api.py` | Método anterior (entraba al portal como una persona). Ya no se usa; queda de respaldo |
| `config_tablero.py` | Direcciones de los flujos y la clave del usuario `RPA`. **No se versiona** |
| `config_ecdatos.py` | `TOKEN_API`, el identificador de la empresa en EC Datos. **No se versiona** |
| `descargas/` | Caché local de PDF. Lo que está aquí no se vuelve a pedir al portal |
| `logs/`, `resultados/` | Un log y un CSV por corrida, con nombre `tablero_RPA-aaaammdd-HHmm` |

```
python subir_tablero.py                      corrida completa
python subir_tablero.py --simular            muestra qué haría, sin bajar ni subir
python subir_tablero.py --centro "Kennedy"   solo esa sucursal
python subir_tablero.py --max 20             a lo sumo 20 facturas
python subir_tablero.py --minutos 300        se detiene limpiamente al pasar ese tiempo
python subir_tablero.py --reintentar         vuelve a buscar las que tienen aviso «no encontrada»
```

Qué considera pendiente: **toda factura de la cartera sin archivo PDF en
Respaldos**. Por eso no hace falta avisarle qué se agregó al Excel: lo nuevo no
tiene PDF y entra solo en la siguiente corrida. Las que EC Datos no tiene se
dejan quietas siete días y luego se vuelven a buscar.

Se puede cortar en cualquier momento. Lo subido queda subido, lo bajado queda en
`descargas/`, y la siguiente corrida sigue con lo que falta.

### Tarea programada

Corre en el servidor **.64**, desde `P:\rpa_ecdatos`, todos los días a la
**01:00** con `--minutos 300`, con el usuario `adminbi` y su sesión abierta
(igual que la tarea del MEGA RPA en el .126). La lanza `correr_tablero.bat`, que
anota inicio y fin en `logs\tarea_programada.log`.

Tiempos (17-nov-2026): bajar 50 PDF toma 6 s; subirlos a SharePoint, unos 2 s
por archivo con 3 lotes a la vez. Una cartera entera de 4.000 facturas sin PDF
toma unas 2 horas y media la primera vez; después solo entra lo nuevo y son
minutos. Por eso el robot también se puede correr a cualquier hora, a mano:
`P:\rpa_ecdatos\correr_tablero.bat` desde cualquier PC con P:, o
`schtasks /Run /TN "Autoconsumos - Robot tablero"` en el servidor.

Orden de la noche: primero se actualiza el Excel, luego el caché (automático al
guardarse), luego el robot. Si el robot corre antes que el caché, trabaja con la
cartera anterior y lo nuevo espera un día.

## 5.4 Qué ve cada uno al día siguiente

- **La supervisora** abre el tablero y cada fila trae «📄 PDF · RPA · fecha».
  Revisa, agrega foto si quiere y pulsa Entregar. Si el robot no encontró la
  factura, la fila dice «No encontrada en EC Datos» y ahí sí adjunta a mano o
  toma foto.
- **Administración** ve lo mismo para todos los centros, con filtro por ciudad,
  y en la auditoría una fila `RPA_CORRIDA` por noche con el resumen.
- **Operaciones** ve todo y no toca nada.
- **Liquidada** es la que un centro entregó desde el tablero **o** la que
  Administración pasó a la hoja LIQUIDADO del Excel base. Las dos salen en la
  pestaña Liquidadas (la del Excel con la marca «Administración · hoja
  LIQUIDADO», sin botones) y cuentan en los indicadores y en el análisis.
- **Administración y Operaciones** tienen además, en la pestaña Liquidadas, el
  botón «Descargar LIQUIDADO.xlsx»: una fila por liquidada con los datos del
  Excel base, dónde se liquidó (Tablero / Excel base), quién entregó, cuándo,
  si tiene PDF y cuántas fotos; y una hoja RESUMEN por centro. Se regenera en
  cada corrida del robot.

## 5.5 Si algo falla

| Síntoma | Dónde mirar | Qué suele ser |
|---|---|---|
| No hay fila `RPA_CORRIDA` de anoche | `P:\rpa_ecdatos\logs\tarea_programada.log` | La tarea no corrió (sesión de adminbi cerrada), o terminó con código 2 (clave del tablero) o 3 (EC Datos) |
| Muchas `RPA_NO_ENCONTRADA` seguidas | El log, buscar `NO_ENCONTRADA` | Facturas no autorizadas aún; si son todas, cambió el token o la API de EC Datos (ver README del robot) |
| `ERROR_SUBIDA` en el CSV | El log, buscar `Lote` | El flujo Adjuntar no respondió en 150 s; relanzar |
| El tablero no carga («No se pudo obtener datos») o tarda más de 2 min | Historial del flujo «Obtener datos» | La biblioteca creció y el flujo lee de a 100: debe tener `$top` 5000 (18-nov-2026). Si pasó de 5.000 archivos, faltan las columnas indexadas |
| `LIQUIDADO.xlsx` desactualizado | `logs\tablero_<corrida>.log`, buscar `Excel` | `URL_PUBLICAR` vacía en `config_tablero.py`, o la carpeta `Reportes` no existe |
| El tablero muestra una cartera vieja (corte atrasado, faltan facturas o centros nuevos) | Historial de «Actualizar caché»: ejecuciones en rojo | Rehicieron las tablas del Excel con otro nombre o cambiaron los encabezados. Deben llamarse `TablaNoLiquidado` y `TablaLiquidado` y conservar las 15 columnas. (16-nov-2026: el archivo nuevo dejó el flujo caído tres días; se arregló leyendo las tablas por nombre) |
| Una factura que Administración ya pasó a la hoja LIQUIDADO no sale en Liquidadas | Biblioteca `Cache`: ¿existe `_liquidado.json` y es de hoy? Historial de «Actualizar caché» | La tabla `TablaLiquidado` no existe o no cubre las filas nuevas (Insertar → Tabla en la hoja LIQUIDADO), o el flujo falló al guardar el Excel |
| La fila dice «Sin PDF» pero el PDF está en la carpeta | Propiedades del archivo en Respaldos | Falta `Factura` o `TipoRespaldo`: el archivo se subió a mano, no por el flujo |
| «sin_permiso» en el log | Lista `Accesos` | La fila `RPA` no tiene `Rol = RPA` |
