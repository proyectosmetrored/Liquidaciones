# 1 · Preparar SharePoint

Primer paso. Sin esto, los flujos del paso 2 no tienen dónde leer ni escribir.

---

## 1.1 El Excel de origen

Hoy está en el OneDrive personal de `ybasantes`:

```
https://metroredec-my.sharepoint.com/:x:/g/personal/ybasantes_metrored_med_ec/…
```

**Recomendación antes de empezar:** muévalo a una biblioteca de un sitio de
SharePoint del área. Un archivo en un OneDrive personal desaparece cuando esa
persona sale de la organización, y la conexión del flujo queda atada a su cuenta.
Es exactamente el riesgo que se quiso quitar al dejar Colab.

El resto de esta guía funciona igual en cualquiera de los dos sitios; solo cambia
lo que se elige en el campo «Ubicación» del conector de Excel.

### Convertir la hoja en tabla

El conector de Excel de Power Automate **no lee hojas sueltas: lee tablas.**

1. Abra el archivo en Excel (escritorio o web).
2. Vaya a la hoja **`NO LIQUIDADO`**.
3. Seleccione el rango completo, cabecera incluida.
4. **Insertar → Tabla**, con «La tabla tiene encabezados» marcado.
5. Con la tabla seleccionada: **Diseño de tabla → Nombre de la tabla**, escriba
   `TablaNoLiquidado`.
6. Guarde.

> Si el archivo lo regenera un proceso automático que borra la tabla en cada
> actualización, use la alternativa del final de este documento.

### Columnas que el tablero espera

| Columna | Obligatoria | Nombres alternativos que también reconoce |
|---|---|---|
| `Centro` | Sí | Centro Médico, Sucursal |
| `Número factura` | Sí | Factura, Nro factura, No factura, Documento |
| `Fecha` | Sí | Fecha emisión, Fecha factura |
| `Valor` | Sí | Valor total, Total, Valor factura |
| `Saldo` | Sí | Saldo pendiente, Pendiente, Valor pendiente |
| `User Caja` | No | Caja, Usuario caja, Cajero |
| `Plan \| Convenio` | No | Plan convenio, Plan, Convenio |

El tablero reconoce los nombres sin distinguir mayúsculas ni tildes. Si mañana
renombran una columna, se agrega el nombre nuevo a la lista `ALIAS` dentro de
`index.html` y listo — no hay que tocar los flujos.

Las columnas que sobran se ignoran sin avisar. **No incluya `Paciente` ni
`Nombre cliente`**: no hacen falta para gestionar la factura y son el dato más
sensible de toda la base.

---

## 1.2 Lista `AutoconsumosAccesos`

Quién puede entrar y con qué contraseña.

**Crear:** sitio de SharePoint → **Nuevo → Lista → Lista en blanco**.

| Columna | Tipo | Notas |
|---|---|---|
| `Title` | Línea de texto | Nombre del centro, **exactamente** como aparece en la columna `Centro` del Excel |
| `Clave` | Línea de texto | La contraseña |
| `Rol` | Opción | Valores `CENTRO` y `ADMIN`. Predeterminado `CENTRO` |
| `Activo` | Sí/No | Predeterminado `Sí`. Poner en `No` desactiva el acceso sin borrar nada |

Un elemento por centro, más uno con `Rol = ADMIN` para el área que debe ver todos
los centros (por ejemplo `ADMINISTRACIÓN` o `CARTERA`).

> ⚠ **Las contraseñas quedan en texto plano.** Power Automate no tiene función de
> cifrado ni de hash, así que la única protección real es el permiso de la lista.
>
> **Rompa la herencia de permisos** (Configuración de la lista → Permisos para
> esta lista → Dejar de heredar permisos) y deje solo:
> - la cuenta con la que se conectan los flujos → **Control total**
> - el equipo de Tecnología → **Lectura**
> - nadie más
>
> Los centros **no** deben tener acceso a esta lista. No lo necesitan: entran por
> el flujo, no por SharePoint.

---

## 1.3 Lista `AutoconsumosSesiones`

Las sesiones abiertas. La escribe el flujo de acceso y la leen los otros dos.

| Columna | Tipo | Notas |
|---|---|---|
| `Title` | Línea de texto | El token (un GUID). **Exigir valores únicos: Sí** |
| `Centro` | Línea de texto | |
| `Rol` | Línea de texto | `CENTRO` o `ADMIN` |
| `Expira` | Fecha y hora | Con hora incluida |

Para poder exigir valores únicos hay que indexar la columna primero:
Configuración de la lista → `Title` → **Exigir valores únicos: Sí** (SharePoint
crea el índice solo y pide confirmación).

Mismos permisos que la lista anterior: solo la cuenta de los flujos escribe.

---

## 1.4 Lista `AutoconsumosSeguimiento`

El registro de gestión. **Es la lista que no se puede modificar.**

| Columna | Tipo | Notas |
|---|---|---|
| `Title` | Línea de texto | Número de factura, en mayúsculas. **Exigir valores únicos: Sí** ← lo más importante de todo |
| `Centro` | Línea de texto | Centro dueño de la factura |
| `Observacion` | Varias líneas de texto | Texto sin formato, sin control de versiones |
| `MarcadoPor` | Línea de texto | Centro que registró el marcado |
| `MarcadoEn` | Fecha y hora | Hora del servidor, no del equipo del usuario |

### Por qué «valores únicos» en `Title` es la pieza clave

Es lo que hace que el marcado sea realmente irreversible y a prueba de carreras.
Si dos centros confirman la misma factura en el mismo segundo, el segundo
`Crear elemento` **falla en SharePoint**, no en el navegador. El flujo captura ese
fallo, consulta quién la tiene y se lo informa al segundo usuario. No hay forma de
que queden dos registros ni de que el segundo pise al primero.

Sin esa restricción, la inmutabilidad dependería de que el flujo consulte antes de
escribir — y entre la consulta y la escritura cabe otra escritura.

### Permisos

1. Configuración de la lista → Permisos → **Dejar de heredar permisos**.
2. Deje:
   - cuenta de los flujos → **Colaborar**
   - todos los demás → **Lectura** (o ninguno: el tablero lee por el flujo)
3. Configuración de versiones → **Crear versiones: Sí**. Si alguien con permisos
   llegara a editar un elemento a mano, queda registrado quién y cuándo.

> Sin romper la herencia, cualquiera con permiso de edición en el sitio puede
> abrir la lista y cambiar un marcado. El código del tablero no lo impide: lo
> impiden los permisos. Este paso no es opcional.

---

## Alternativa: leer la hoja sin convertirla en tabla

Si no se puede o no conviene crear la tabla, use la acción **Ejecutar script**
(Office Scripts) en lugar de «Enumerar filas presentes en una tabla».

Cree el script en Excel en la web (**Automatizar → Nuevo script**), llámelo
`LeerNoLiquidado` y pegue:

```typescript
function main(workbook: ExcelScript.Workbook): object[] {
  const hoja = workbook.getWorksheet("NO LIQUIDADO");
  if (!hoja) throw new Error("No existe la hoja NO LIQUIDADO.");

  const valores = hoja.getUsedRange().getValues();
  if (valores.length < 2) return [];

  const cabeceras = valores[0].map(c => String(c).trim());
  const filas: object[] = [];

  for (let i = 1; i < valores.length; i++) {
    const fila: { [clave: string]: string | number | boolean } = {};
    cabeceras.forEach((h, j) => { if (h) fila[h] = valores[i][j]; });
    filas.push(fila);
  }
  return filas;
}
```

Devuelve exactamente la misma forma que el conector de tablas: un arreglo de
objetos con las cabeceras como claves. En el flujo del paso 2 se cambia una sola
acción; el resto queda igual.

Dos avisos: las fechas llegan como número de serie de Excel (el tablero ya lo
convierte solo) y Office Scripts tiene un límite de ejecución, así que con bases
muy grandes conviene la tabla.
