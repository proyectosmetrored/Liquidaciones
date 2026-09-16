# 4 · Publicación

Último paso: conectar el tablero a los flujos y ponerlo a disposición de los
centros.

---

## 4.1 Configurar el archivo

Abra `index.html` con cualquier editor de texto y busque el bloque `CONFIG`, al
principio del `<script>`. Es lo único que se toca:

```js
const CONFIG = {
  VERIFICAR_URL:  "…URL del flujo 1…",
  DATOS_URL:      "…URL del flujo 2…",
  MARCAR_URL:     "…URL del flujo 3…",
  CATALOGOS_URL:  "",                    // flujo 5, opcional
  CENTROS_RESPALDO: [ "CM ALBORADA", … ],
  SESION_KEY: "autoconsumos_sesion_v1",
  MODO_DEMO: false                       // ← imprescindible
};
```

Ajuste también `CENTROS_RESPALDO` para que coincida con los centros reales, con
la misma escritura que la columna `Centro` del Excel. Si configuró el flujo 5, esa
lista solo se usa cuando el flujo no responde.

> **`MODO_DEMO: false` no es opcional.** En `true`, el tablero entra con la
> contraseña `demo` y muestra datos inventados. Publicarlo así sería peor que el
> tablero anterior.

---

## 4.2 Revisión antes de publicar

Recorra la lista completa. Cada punto corresponde a un fallo que ya ocurrió en la
versión anterior o que rompe una garantía de esta.

**Configuración**

- [ ] `MODO_DEMO` está en `false`
- [ ] Las tres URL están puestas y no tienen espacios ni saltos de línea
- [ ] `CENTROS_RESPALDO` coincide con los nombres reales de los centros

**SharePoint**

- [ ] `AutoconsumosSeguimiento.Title` tiene **valores únicos obligatorios**
- [ ] `AutoconsumosSesiones.Title` tiene valores únicos obligatorios
- [ ] Las tres listas **no heredan** permisos del sitio
- [ ] Los centros **no** tienen acceso directo a `AutoconsumosAccesos`
- [ ] `AutoconsumosSeguimiento` tiene control de versiones activado

**Flujos**

- [ ] «Enumerar filas presentes en una tabla» tiene **paginación activada**
- [ ] «Obtener marcas» tiene paginación activada
- [ ] El «Aplicar a cada uno» del flujo 3 tiene **paralelismo en 1**
- [ ] El flujo 3 **no** contiene ninguna acción de actualizar ni de eliminar
- [ ] Los flujos se conectan con una **cuenta de servicio**, no personal

**Prueba de extremo a extremo**

- [ ] Entrar con un centro real: solo se ven sus facturas
- [ ] Contar las filas: coinciden con las de la hoja `NO LIQUIDADO` de ese centro
- [ ] La fecha de corte del encabezado es la del archivo, no la de hoy
- [ ] Marcar una factura de prueba y confirmar que se bloquea
- [ ] **Entrar desde otro navegador con otro centro y ver esa misma marca**
- [ ] Intentar marcarla de nuevo: debe informar quién la tiene
- [ ] Entrar con el usuario `ADMIN`: se ven todos los centros
- [ ] Borrar de la lista el elemento de prueba

La quinta prueba es la que importa. Es justo lo que el tablero anterior no hacía
y el motivo por el que se rehízo: que lo que marca un centro lo vean los demás.

---

## 4.3 Dónde dejar el archivo

El `index.html` **no contiene datos**, así que dónde vive es menos crítico que
antes. Aun así, no conviene dejarlo suelto: publicado en un sitio de SharePoint
del área, el navegador y los flujos quedan en el mismo dominio y se evitan
problemas de permisos entre orígenes.

Opciones, de mejor a peor:

1. **Biblioteca de un sitio de SharePoint del área.** Se sube el archivo y se
   comparte el enlace. Es lo más simple y lo que mejor funciona.
2. **Un servidor web interno.** Igual de válido si ya existe uno.
3. **Enviarlo por correo o dejarlo en una carpeta compartida.** Funciona, pero
   cada persona termina con una copia distinta y actualizar significa reenviar.
   Es lo que se está tratando de dejar atrás.

Sea cual sea la opción, quien tenga el archivo **no ve ningún dato sin una
contraseña de centro**. Esa es la diferencia con el tablero de 10 MB, que se
podía leer entero con «ver código fuente».

### Si el navegador bloquea las llamadas

Si la consola muestra un error de CORS al intentar entrar, es que el navegador
rechaza la llamada al flujo por venir de otro origen. Se resuelve, en este orden:

1. Publicar el `index.html` en el mismo dominio de SharePoint (opción 1).
2. Comprobar que el flujo responde también al método `OPTIONS`.
3. Como último recurso, poner el tablero detrás de una página del sitio.

El login del que salió este tablero ya funciona con este mismo patrón en su
entorno, así que lo más probable es que no haga falta nada de esto.

---

## 4.4 Actualizar los datos

No hay nada que actualizar. El tablero lee el Excel **cada vez que alguien entra**,
y el botón `↻` junto a los filtros vuelve a consultarlo sin cerrar la sesión.

Basta con mantener el Excel al día en su sitio. Desaparecen el paso de regenerar y
el de redistribuir, que eran el cuello de botella del proceso anterior.

Lo único que conviene vigilar: que quien actualiza el Excel **no borre la tabla**
`TablaNoLiquidado` al pegar los datos nuevos. Si pega sobre el rango existente, la
tabla se conserva. Si borra la hoja y crea otra, hay que volver a crearla.

---

## 4.5 Subirlo a git

La carpeta está lista para un repositorio: un archivo, su documentación y nada
generado.

```bash
cd tablero-autoconsumos
git init
git add .
git commit -m "Tablero de autoconsumos: acceso por centro y marcado definitivo"
```

**Antes del primer `push`, revise que no se suba ninguna URL de flujo.** Las URL
de Power Automate llevan una firma (`&sig=…`) en el enlace: cualquiera que la
tenga puede llamar al flujo. En un repositorio interno el riesgo es acotado; en
uno público, no.

Las dos formas de manejarlo:

- **Versionar el archivo con las URL vacías** y completarlas al publicar. Es lo
  que viene por defecto y lo más seguro.
- **Versionarlo configurado**, solo si el repositorio es privado y del área.

El `.gitignore` incluido evita subir copias de trabajo, exportaciones y archivos
de datos por si alguien deja uno en la carpeta.

> Si alguna vez se sube una URL de flujo por error, no basta con borrarla en un
> commit posterior: queda en el historial. Hay que **regenerar la URL** desde
> Power Automate, y eso obliga a actualizar el `index.html`.
