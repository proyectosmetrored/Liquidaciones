# Tablero de Autoconsumos

Consulta de la cartera pendiente de autoconsumos, respaldo de cada factura con
su PDF y registro de entrega por centro.

Reemplaza al tablero anterior, que era un HTML de 10 MB con las 26 000 facturas
incrustadas, una contraseña escrita en el propio código y el seguimiento guardado
en el navegador de cada persona.

## Qué cambia

| | Antes | Ahora |
|---|---|---|
| Los datos | Dentro del archivo, 10 MB | Se piden al abrir; el archivo va vacío |
| El acceso | `metro` / `2026` en el JavaScript | Usuario y contraseña, validados en el servidor |
| Qué ve cada uno | Todos veían todo | Cada centro ve solo su cartera |
| El PDF de la factura | Lo buscaba cada centro | **Lo pone un robot cada noche desde EC Datos** |
| El seguimiento | En el navegador de cada quien | En SharePoint, visible para todos |
| Corregir una entrega | Se podía borrar sin rastro | **No se puede: es definitiva y queda firmada** |
| Quién hizo qué | Nadie lo sabía | Lista de auditoría: cada PDF, foto y entrega con usuario y hora |
| La fecha de corte | La del reloj del equipo | La de última modificación del Excel |

## Qué ve cada rol

El rol sale de la columna `Rol` de la lista de accesos y define la pantalla
completa, no solo los datos.

| | Centro (`CENTRO`) | Administración (`ADMIN`) | Operaciones (`OPERACIONES`) | Robot (`RPA`) |
|---|---|---|---|---|
| Facturas | Solo las de su centro | Todos los centros | Todos los centros | Todos los centros |
| Adjuntar PDF y fotos | Sí | Sí | — | Solo PDF |
| Entregar | Sí | Sí | — | — |
| Filtros | Sí | Sí, más ciudad y centro | Sí, más ciudad y centro | — |
| Indicadores, análisis, CSV | — | Sí | Indicadores y análisis | — |
| Entrar al tablero | Sí | Sí | Sí | **No** |

> El límite real no es que los botones estén ocultos: al navegador de un centro
> **nunca le llegan las filas ni los archivos de otro**. El filtro ocurre en el
> flujo.

## Cómo funciona

```
  Excel BASE AUTOCONSUMOS            EC Datos (portal de facturación)
  hoja NO LIQUIDADO                            │
         │ al guardarse                         │ cada noche
         ▼                                      ▼
  Actualizar caché  ──►  /Cache/*.json   ◄──  Robot (P:\rpa_ecdatos)
                                │              │  baja los PDF y los sube
                                ▼              ▼
                     Flujos HTTP de Power Automate
                                │
              ┌─────────────────┼──────────────────┐
              ▼                 ▼                  ▼
        index.html     biblioteca Respaldos   AutoconsumosAuditoria
```

El navegador nunca habla con el Excel ni con SharePoint directamente. Todo pasa
por flujos de Power Automate:

1. **Verificar acceso** — recibe usuario y contraseña, devuelve un *token* de
   sesión con el rol. La contraseña no vuelve a viajar.
2. **Actualizar caché** — se dispara cuando el Excel se modifica y deja un JSON
   por centro. Nadie lo espera.
3. **Obtener datos** — con el token, devuelve la cartera del centro, sus
   entregas, sus respaldos (PDF y fotos) y los avisos del robot.
4. **Adjuntar respaldo** — guarda PDF y fotos en la biblioteca `Respaldos`, en
   la carpeta del centro. **No entrega.** Lo usan el tablero y el robot.
5. **Marcar facturas** — registra la entrega. Exige que la factura tenga PDF.
   **Solo crea: nunca modifica ni borra.**
6. **Reportar corrida** — el robot deja el resumen de cada noche y la lista de
   facturas que EC Datos no tenía.

### El flujo de una factura

1. Administración la pone en la hoja NO LIQUIDADO del Excel.
2. El tablero la muestra al centro, sin PDF: el botón dice «Falta el PDF».
3. Esa noche el robot la baja de EC Datos y la deja en `Respaldos/<Centro>/`.
4. Al día siguiente la fila trae «📄 PDF · RPA». La supervisora la revisa, agrega
   fotos si hace falta y pulsa **Entregar**. Si el robot no la encontró, la fila
   lo dice y ahí sí ella adjunta el PDF a mano o toma una foto.
5. La entrega queda en SharePoint, firmada, y equivale a liquidada: la factura
   sale de la pestaña **Pendientes** y pasa a **Liquidadas**, donde solo se mira.

## Puesta en marcha

Los pasos están en `docs/`:

| Documento | Qué hay |
|---|---|
| [`docs/01-sharepoint.md`](docs/01-sharepoint.md) | Las listas y el Excel (construcción inicial, con nota de lo que cambió) |
| [`docs/02-flujos.md`](docs/02-flujos.md) | Los flujos acción por acción (construcción inicial, con nota) |
| [`docs/03-contrato.md`](docs/03-contrato.md) | **Vigente.** El JSON exacto que intercambian el tablero, el robot y los flujos |
| [`docs/04-publicacion.md`](docs/04-publicacion.md) | Dónde publicar y qué revisar antes |
| [`docs/05-robot-y-respaldos.md`](docs/05-robot-y-respaldos.md) | **Vigente.** El robot, la biblioteca de respaldos, la auditoría y las reglas |

Los flujos ya no se arman a mano: se generan con `flujos/construir_*.py` (fuera
de este repositorio) y se importan como paquete.

## La entrega es definitiva

Es la regla de fondo y conviene que quede clara antes de capacitar a nadie:

- Entregar una factura **crea un registro y nada más**. No hay forma de editarlo
  ni de borrarlo desde el tablero, y el flujo tampoco lo permite.
- No se puede entregar sin PDF. Adjuntar un PDF o una foto **no** entrega.
- Queda firmada con el usuario, la fecha y la hora del servidor.
- La observación se escribe **antes** de confirmar. Después queda congelada.
- Si dos personas intentan entregar la misma factura a la vez, gana la primera.
  La segunda ve quién la entregó y cuándo.

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
│   ├── 01-sharepoint.md         Listas y Excel
│   ├── 02-flujos.md             Flujos, acción por acción
│   ├── 03-contrato.md           Lo que intercambian tablero, robot y flujos
│   ├── 04-publicacion.md        Dónde publicar y qué revisar antes
│   └── 05-robot-y-respaldos.md  El robot y los respaldos
└── presentacion/
    └── entrega-2026-09-16.html  Presentación de entrega.
```

Fuera del repositorio: el robot en `P:\rpa_ecdatos` y los generadores de flujos
en `flujos/` de la carpeta de trabajo.

## Sobre este repositorio

El `index.html` versionado **está configurado**: lleva las direcciones de los
flujos en `CONFIG` y `MODO_DEMO: false`, porque es lo que sirve GitHub Pages.
Esas direcciones llevan una firma (`&sig=`) que funciona como llave del
disparador. Se decidió publicarlas así a sabiendas; si alguna vez hay que
revocarlas, se regenera la URL en Power Automate y se vuelve a publicar.

Un solo archivo, sin dependencias, sin compilación. Se edita con cualquier
editor de texto y se publica con un `git push`.

## Lo que todavía no resuelve

- **Las contraseñas se guardan en texto plano** en la lista de accesos. La
  protección real es el permiso de esa lista. La solución de fondo es autenticar
  con Entra ID.
- **Administración todavía no tiene las liquidadas en Excel.** En el tablero
  ya están en la pestaña Liquidadas; falta un `LIQUIDADO.xlsx` aparte que se
  regenere cada noche para Administración. El Excel base no se toca.
