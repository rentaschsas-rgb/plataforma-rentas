# Arquitectura de datos — Renta 2026

Diseño de la base de datos en **Cloud Firestore** y de la sincronización con el Google Sheet.

---

## 1. Principio que gobierna todo: un dueño por campo

La sincronización bidireccional sobre el mismo campo es la forma más rápida de perder datos: dos personas editan, gana el último, nadie se entera. Por eso cada dato tiene **un solo dueño**.

| Dato | Dueño | Quién puede escribirlo |
|---|---|---|
| cédula, nombre, correo, vence, valor 2025 | **Sheet** `elaborar` | solo el sincronizador |
| paso, documentos, comentarios, pagos, **responsable**, **tareas** | **Plataforma** | los usuarios activos |
| espejo del proceso en el Sheet | Plataforma | solo el sincronizador (columnas de solo lectura para el humano) |

Consecuencia práctica: **reimportar nunca pisa el proceso**, y editar el Sheet nunca pisa un comentario. Esta regla se hace cumplir en las reglas de seguridad, no solo por convención (§6).

Tres precisiones sobre el dinero y el estado, que se descubrieron leyendo el cotizador:

- **Lo que se cobra es `PRECIO ACORDADO`**, la columna de control donde el cotizador escribe lo que el cliente aceptó, ya con descuento. `PRECIO 2026` es la cotización y la columna G es el precio de 2025: ninguno de los dos se debe. Si no hay valor acordado, el cliente aparece **sin honorarios y sin deuda**.
- **`ESTADO` fija el paso solo al crear el cliente**: `Aceptado` → Cotizó, cualquier otra cosa → Invitado. El Sheet no llega más lejos, así que dejar que mandara en cada importación devolvería hacia atrás a quien ya está en Elaboración.
- **Las filas con `ESTADO = Rechazado` no se importan.**
- **A la vista «Por vencer» solo entran los que aceptaron.** El plazo de la DIAN existe para todos, pero solo vamos a presentar la declaración de quien contrató: el resto llenaba el tablero de vencimientos ajenos. La condición es `paso >= 2 || ESTADO = Aceptado` — el Sheet o la plataforma, cualquiera de los dos basta, porque el equipo puede haber movido a alguien antes de que la hoja se actualice. Los demás siguen enteros en **Clientes**, que es la lista completa.

Todas las demás columnas de la hoja se guardan en `sheet.otras`, con su encabezado como nombre, y se ven en el expediente. Por eso la importación **necesita la fila de encabezados**: con tres columnas de plata en la hoja, adivinar por contenido escoge la equivocada.

---

## 2. Colecciones

### `/usuarios/{uid}`

La lista blanca. Sin documento aquí, no se entra.

```js
{
  correo: "johanna@…",       // debe coincidir con el correo de Google
  nombre: "Johanna",
  rol: "admin" | "operador", // admin puede borrar movimientos y sincronizar
  activo: true,
  creado: Timestamp
}
```

Se crean a mano las tres primeras veces (consola de Firebase). Dar de baja a alguien es poner `activo: false`, no borrar el documento: así se conserva la autoría de lo que escribió.

### `/config/general`

Un único documento con lo que hoy son constantes en el código. Ponerlo aquí evita volver a desplegar para cambiar un porcentaje o un paso.

```js
{
  periodo: "2026",
  pasos: ["Invitado","Cotizó","Documentos","Elaboración","Presentado","Pagado"],
  categoriasDoc: ["Certificado de ingresos","Certificados bancarios", …],
  medios: ["Bancolombia","Davivienda","Nequi","Daviplata","Efectivo"],
  reparto: [
    { id:"JL",   nombre:"Johanna + Leidy", cuota:0.70, personas:["Johanna","Leidy"] },
    { id:"JOHN", nombre:"John",            cuota:0.30, personas:["John"] }
  ],
  calendarioDian: [ { desde:1, hasta:10, fecha:"2026-08-12" }, … ]
}
```

### `/periodos/{año}/clientes/{cedula}`  ·  el expediente de cada año

**La operación cuelga del período (año gravable).** Todo lo que cambia año con año —paso, papeles, pagos, responsable— vive bajo `/periodos/{año}/…`. La cédula sigue siendo el ID del cliente dentro del período. Así, cuando entre 2026, se crea `/periodos/2026/…` y **2025 queda congelado e intacto**: nada se pisa y la historia queda consultable con el selector de año.

- `/periodos/{año}/clientes/{cedula}` — el expediente (abajo), con sus subcolecciones `documentos`, `hitos`, `notas`.
- `/periodos/{año}/movimientos/{id}` — los pagos de ese año.
- `/config/{año}` — calendario DIAN, precios y demás de ese año (el calendario cambia cada año).
- La app trae un **selector de período** (por defecto el vigente) que corta las escuchas y re-suscribe contra el año elegido.
- La estructura anterior de nivel raíz (`/clientes`, `/movimientos`) **se conserva de respaldo**. Un botón admin **«Traer datos anteriores»** copia (no mueve) esos datos al período activo; es idempotente y no pisa lo que ya exista.

**La cédula es el ID del documento.** Es la llave natural, ya existe en el Sheet, y hace que la sincronización sea idempotente: escribir dos veces la misma fila no duplica nada.

```js
{
  // ── Territorio del Sheet ─────────────────────────────
  sheet: {
    nombre: "Marina Salgado Ríos",
    correo: "marina@example.com",       // puede traer varios separados por coma
    vence: "2026-08-28",           // ISO; si viene vacío se deriva del calendario DIAN
    honorarios: 380000,            // columna PRECIO ACORDADO. 0 = todavía no aceptó
    estado: "Aceptado",            // columna ESTADO: Enviado | Aceptado | (vacío)
    grupo: "A",
    responsable: "Johanna",
    otras: {                       // el resto de columnas de la hoja, tal cual
      "CIUDAD": "Medellín",
      "FECHA RESPUESTA": "2026-07-14"
    }
  },

  // ── Territorio de la plataforma ──────────────────────
  paso: 5,                          // 1..6, paso actual
  responsable: "Johanna",           // quién lleva el cliente; se EDITA en la
                                    // plataforma (arranca del Sheet, luego no lo pisa)
  tareas: {                         // listas de chequeo por paso (mapa {tarea:true})
    "Colorear reporte": true,
    "Tomar pantallazos": true
  },

  // ── Derivados (los mantiene la Cloud Function) ───────
  resumen: {
    docsTotal: 9,                   // cuenta ítems, no categorías
    docsRecibidos: 7,
    docsCompletos: false,
    abonado: 380000,
    saldo: 0
  },

  actualizado: Timestamp,
  actualizadoPor: "uid…"
}
```

`resumen` está **desnormalizado a propósito**: las tablas de lista necesitan ordenar y filtrar por saldo y por documentos faltantes. Sin esto, pintar 86 filas exigiría leer 86 subcolecciones — lento y caro. Lo recalcula una Cloud Function cada vez que cambia un documento o un movimiento.

#### `/clientes/{cedula}/documentos/{docId}`

Un documento por **papel concreto**, no por categoría. Así "Certificados bancarios" puede ser tres filas: Davivienda, AV Villas, Bancolombia.

```js
{
  categoria: "Certificados bancarios",  // una de config.categoriasDoc
  nombre: "Banco Davivienda",           // lo escribe el usuario; vacío si la categoría no se desagrega
  recibido: true,
  nota: "Llegó por correo el 18",
  creado: Timestamp,
  creadoPor: "uid…"
}
```

Una categoría está completa cuando **todos** sus ítems tienen `recibido: true`, y se considera pendiente si no tiene ningún ítem todavía.

**Las categorías las decide cada cliente, no son fijas.** Antes se le daban las mismas seis a todos; ahora la lista sale de lo que respondió en el cotizador. Todo cliente arranca con los tres universales —**Cédula, RUT y Clave de la DIAN actualizada**— y a eso se le suman las categorías que activan sus respuestas (mapeo en `REGLAS_DOC`/`REGLA_ACTIVOS` dentro de `index.html`, confirmado por el equipo): p. ej. `Salario = Sí` → «Certificado de ingresos y retenciones (laboral)», `Cantidad Activos ≠ vacío` → «Relación de bienes». El expediente pinta solo las categorías que tiene, y un selector permite agregar a mano cualquiera que falte.

**Categorías con subdocumentos fijos.** Algunas categorías nacen con ítems concretos en vez de un genérico suelto (mapa `SUBDOCS` en `index.html`). Hoy la única es **Clave de la DIAN actualizada**, que trae tres: *Descarga del reporte*, *Descarga de renta sugerida* y *Descarga de facturas electrónicas*.

**Listas de chequeo por paso.** Un paso puede tener tareas internas que cumplir (`CHECKLIST_PASO`), guardadas por cliente en `tareas`. Hoy:
- **Documentos:** colorear reporte, tomar pantallazos, verificar documentos contra reporte, verificar contra pantallazos, solicitar faltantes. La tarea *Colorear reporte* muestra un tooltip con la guía de colores (`GUIA_COLORES`): verde = saldos de bancos, amarillo = ingresos, morado = patrimonio, azul = retenciones.
- **Elaboración:** ingresar papeles al ayuda renta, verificar valores contra reporte.

Las respuestas viven en una hoja aparte del Sheet, **RESPUESTAS** (una fila por renta, escrita por el cotizador; los valores son «Sí»/«No»). Como el mapa `sheet` es territorio del Sheet y las reglas prohíben que el navegador lo toque, las respuestas **no** se guardan en el cliente: se traducen directo a documentos. El flujo es el botón **«Actualizar desde RESPUESTAS»** del modal de importar — se pega la hoja RESPUESTAS, se cruza por cédula contra los clientes ya importados, y para cada uno se **agregan** las categorías que falten sin tocar nada ya recibido (`fusionarDocs`). Es idempotente: volver a pegar no duplica. Como los documentos solo se sincronizan con el expediente cargado, el proceso trae en tandas los expedientes que falten antes de mezclar.

Además, **si quien lo corre es admin**, ese mismo flujo **pisa el honorario** con la columna **Precio Neto** de RESPUESTAS cuando trae valor y difiere del actual — así se corrige un pago equivocado sin reimportar `elaborar`. El preview avisa cuántos pagos cambiarán antes de confirmar.

**Honorarios: override de plataforma.** `sheet.honorarios` es el valor del Sheet (PRECIO ACORDADO), pero la plataforma puede fijar uno propio en `honorarioManual` (null = usar el del Sheet; **0 es válido**, para un cliente regalado). El helper `honorariosDe(c)` es la única fuente de verdad para cobros/saldos. Se edita en el bloque *Cobro* del expediente (solo admin) con un ↺ para volver al valor del Sheet. RESPUESTAS escribe `honorarioManual`, **pero respeta `honorarioFijo`**: si el honorario se fijó a mano, RESPUESTAS no lo pisa (así el $0 regalado no se revierte al re-pegar).

Nota: la columna **RESPONSABLE** de `elaborar` es un `TRUE`/`FALSE` (marca de contacto del grupo), **no** un nombre; por eso `responsable` **no** se hereda del Sheet ni en la importación ni en la migración: arranca vacío y se asigna en la plataforma.

**Recordatorio de documentos: se envía solo.** El bloque *Documentos* del expediente trae el botón **«Enviar recordatorio»**. El correo lo manda el **Apps Script del Sheet** (`recordatorioDocumentos` en `apps_script.js`), no el navegador: la consola es un HTML estático y no puede enviar correo. Sale con el mismo vestido y desde la misma cuenta que la invitación del cotizador, con **copia a `rentas.chsas`**. Lleva, en este orden: los **días que faltan** (o los que lleva vencido, en rojo), la lista de lo pendiente, y el **cargo por entrega tardía**.

- La lista nombra la **categoría**, y si dentro de ella ya llegó algo, aclara entre paréntesis qué falta (`Certificados bancarios (Bancolombia)`): pedir la categoría entera hacía que el cliente reenviara lo que ya había mandado.
- El **nombre va completo**. En la hoja está como «APELLIDO1 APELLIDO2 NOMBRE», así que saludar por la primera palabra saluda por el apellido (`nombreLargo`, no `primerNombre`).
- Se le escribe a **todos** los correos de la celda del Sheet. Sin correo, el botón queda deshabilitado y lo dice.
- Deja **constancia en el expediente**: un comentario en el paso *Documentos* con `tipo:'solicitud'`, y marca la tarea *Solicitar documentos faltantes*. De ese comentario sale el «Última solicitud enviada el …» de la barra — por eso **no hizo falta un campo nuevo** para eso.
- **«Copiar el texto»** deja la misma carta en texto plano, para WhatsApp.

**Por qué el endpoint público no es un buzón abierto.** El `doPost` del Apps Script atiende a cualquiera que conozca la URL, así que:

1. **El destinatario no viaja en la petición**: se busca por cédula en `elaborar`. El endpoint no sirve para escribirle a nadie que no sea ya cliente.
2. **El texto tampoco viaja**: se arma en el servidor. De afuera solo entra *cuáles* documentos faltan, validados contra el catálogo `CATEGORIAS_DOC` (que debe seguir a `ORDEN_CAT` de `index.html`).
3. **Copia a la oficina en cada envío y tope diario** (`TOPE_RECORDATORIOS_DIA`, 60 — Gmail gratis da 100 destinatarios/día). Un abuso se ve el mismo día y se corta solo.

El `token` compartido es un cerrojo de cortesía: la consola es HTML público y quien la lea puede sacarlo. Lo que protege de verdad son los tres puntos de arriba.

### El cargo por entrega tardía — `$70.000`

**No es la sanción de la DIAN**, es la cláusula de gestión extemporánea del contrato que el cliente aceptó en el cotizador: «*aplicar un cargo adicional por gestión extemporánea de SETENTA MIL PESOS ($70.000 COP) por cada proceso afectado*», junto con el compromiso de entregar los papeles **una semana antes** del vencimiento. Los dos números viven en `MULTA_DOCS` y `DIAS_ENTREGA`, en `index.html` y en `apps_script.js`; en el cotizador está el mismo `MULTA_DOCS`, **informativo** — no entra en el precio cotizado, y por eso la plataforma lo suma aparte.

- Se aplica **en el mismo acto en que se le avisa**, nunca antes: el correo que sale le dice al cliente que se le está cobrando. Cobrar en silencio sería peor que no cobrar.
- La condición es *pasó la fecha de entrega* **y** *siguen faltando papeles*. La decide el **servidor** (tiene la fecha del Sheet, que es la buena) y la devuelve en `recargo`; la plataforma la obedece.
- Se cobra **una sola vez**, aunque se le insista después.
- Vive en `recargoDocs` (monto) y `recargoDocsFecha`. Un **admin** puede quitarlo con el ↺ del bloque *Cobro*, y queda anotado: perdonar $70.000 es una decisión de la que alguien va a preguntar.
- **`honorariosDe` sigue siendo solo el precio**; lo que se debe es **`cobroDe` = honorarios + cargo**, y es lo que usan el saldo, la cartera, la vista de pagos y el recibo. Separarlos evita que editar los honorarios a mano se coma el cargo, o que el cargo se duplique al reimportar.

#### `/clientes/{cedula}/hitos/{n}`

Un documento por paso, `n` de `1` a `6`.

```js
{
  completado: true,
  fecha: "2026-07-14",
  por: "uid…"
}
```

#### `/clientes/{cedula}/notas/{notaId}`

Comentarios. **`paso` es lo que permite comentar dentro de cada etapa** en vez de una sola bitácora suelta.

```js
{
  texto: "El banco no ha dado el certificado de retenciones.",
  paso: 3,                    // null = comentario general del expediente
  origen: "equipo" | "cliente",
  autor: "uid…",              // null si viene del cotizador
  creado: Timestamp
}
```

### `/movimientos/{movId}`

Colección **de primer nivel**, no subcolección de cliente. La conciliación pregunta "¿cuánto recibió John este mes?", que cruza todos los clientes; si colgara de cada cliente habría que barrer 86 subcolecciones.

```js
{
  fecha: "2026-07-15",
  cedula: "10000001",         // "" si aún no se identifica al cliente
  monto: 380000,
  quien: "Johanna",           // una de config.reparto[].personas
  medio: "Bancolombia",
  referencia: "Transf. 4471",
  conciliado: false,
  creado: Timestamp,
  creadoPor: "uid…"
}
```

### `/sincronizaciones/{id}`

Bitácora de cada importación. Sirve para responder "¿por qué cambió este dato?".

```js
{ fuente:"sheets", filas:86, nuevos:2, cuando:Timestamp, por:"uid…" }
```

---

## 2 bis. Dos filtros de la interfaz que no viven en Firestore

**La semana del calendario es pulsable.** En *Por vencer*, tocar una de las 12 semanas deja en la tabla de abajo solo los que vencen en ella; tocarla otra vez lo quita. Las **cifras de arriba y el calendario siguen contando todo**: son el panorama, y cambiarlos con el filtro haría imposible saber contra qué se está comparando. La semana escogida (`semanaSel`) **no se recuerda entre sesiones** — es una mirada de un momento — y *Limpiar filtros* la suelta.

**Etapas visibles.** Una fila de pastillas sobre la tabla, en *Por vencer* y en *Clientes*, enciende y apaga cada paso. Es lo contrario del filtro de la columna *Paso*, que deja ver **una**: aquí se apagan las que estorban y queda el resto. Se guarda en `localStorage` (`renta2026:etapas`), porque cada quien mira siempre las mismas; *Limpiar filtros* también las vuelve a encender, o el botón *Ver todas*. El conteo del pie dice cuántas hay ocultas, para que una tabla corta nunca sea un misterio.

---

## 3. Índices compuestos

Firestore los exige explícitamente para consultas con más de un campo:

| Colección | Campos |
|---|---|
| `clientes` | `paso` ASC, `sheet.vence` ASC |
| `clientes` | `resumen.saldo` DESC, `sheet.nombre` ASC |
| `movimientos` | `quien` ASC, `fecha` DESC |
| `movimientos` | `conciliado` ASC, `fecha` DESC |

Con 86 clientes cabe leer la colección entera de una vez y filtrar en el navegador — que es lo que hace la interfaz hoy. Los índices importan cuando esto crezca a varios miles.

---

## 4. Sincronización con el Sheet

### Sheet → Firestore (datos del cliente)

Un disparador en Apps Script que, al editar `elaborar` y también una vez al día, escribe con `merge` **solo el mapa `sheet`** de cada cliente. Nunca toca `paso`, `documentos` ni `notas`.

Autenticación: cuenta de servicio de Firebase, con su JWT firmado desde Apps Script contra la API REST de Firestore.

### Firestore → Sheet (espejo de estados) — **implementado, automático**

**Sin Cloud Functions y sin plan Blaze.** El Apps Script del Sheet lee `/periodos/{PERIODO_SYNC}/clientes` por la API REST de Firestore y escribe en `elaborar` cuatro columnas espejo: **PRESENTADO** (paso ≥ 5), **PAGADO** (paso ≥ 6), **SALDO PLATAFORMA** y **ACTUALIZADO PLATAFORMA**. Funciones en `apps_script.js`: `actualizarEstadosEnSheet()` (el trabajo), `leerClientesFirestore()` (REST + paginación), `syncEstadosTrigger()` (disparador).

**Autenticación sin clave de cuenta de servicio.** Se usa `ScriptApp.getOAuthToken()` con el scope `https://www.googleapis.com/auth/datastore`. Como el dueño del script (`rentas.chsas`) es propietario del proyecto, la API REST lo autoriza por **IAM y omite las reglas de seguridad** (las reglas solo aplican al SDK de Firebase con token de Firebase Auth, no a la REST con token de Google). Requisito: agregar ese scope al manifiesto `appsscript.json`.

Menú «🧾 Rentas 2026»: **Actualizar estados desde la plataforma** (manual), **Activar / Desactivar sincronización automática** (disparador de tiempo cada 15 min, `everyMinutes(15)`).

Esas columnas son un **espejo de solo lectura**: si alguien las edita a mano, la siguiente corrida las vuelve a poner — su dueño es la plataforma. Solo se pisan las filas cuya cédula existe en la plataforma; las demás quedan intactas. La columna **ACTUALIZADO PLATAFORMA** marca la frescura.

---

## 5. Autenticación

Firebase Auth con **correo y contraseña**. Las cuentas las crea un admin desde la consola de Firebase; **el registro abierto queda deshabilitado**, así nadie se crea una cuenta solo.

Dos capas:

1. **Autenticación:** el usuario existe en Firebase Auth y sabe su contraseña.
2. **Autorización:** además debe existir `/usuarios/{uid}` con `activo: true`. Las reglas de seguridad lo verifican del lado del servidor en cada lectura y escritura.

La segunda capa es la que realmente protege, y es la que permite dar de baja a alguien sin borrar su cuenta: `activo: false` y queda por fuera conservando la autoría de lo que escribió.

**Al crear cada usuario en la consola, hay que copiar su UID** y crear con ese UID el documento en `/usuarios`. Es el paso que se olvida y produce el síntoma «entro pero no veo nada».

Como es contraseña y no Google, conviene: contraseñas largas y distintas por persona, y tener presente que el restablecimiento llega al correo registrado — así que ese correo debe ser uno al que la persona realmente tenga acceso.

---

## 6. Reglas de seguridad

Esbozo. La parte importante es la penúltima: **hace cumplir el reparto de autoridad del §1**.

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {

    function activo() {
      return request.auth != null
        && exists(/databases/$(db)/documents/usuarios/$(request.auth.uid))
        && get(/databases/$(db)/documents/usuarios/$(request.auth.uid)).data.activo == true;
    }
    function esAdmin() {
      return activo()
        && get(/databases/$(db)/documents/usuarios/$(request.auth.uid)).data.rol == 'admin';
    }

    match /usuarios/{uid} { allow read: if activo(); allow write: if esAdmin(); }
    match /config/{doc}   { allow read: if activo(); allow write: if esAdmin(); }

    match /clientes/{cedula} {
      allow read: if activo();
      // Nadie desde la app puede tocar el mapa `sheet`: ese lo escribe
      // la cuenta de servicio, que no pasa por estas reglas.
      allow update: if activo()
        && request.resource.data.sheet == resource.data.sheet;
      allow create, delete: if false;

      match /{sub=**} { allow read, write: if activo(); }
    }

    match /movimientos/{id} {
      allow read, create: if activo();
      allow update: if activo();
      allow delete: if esAdmin();
    }

    match /sincronizaciones/{id} { allow read: if activo(); allow write: if false; }
  }
}
```

Nota sobre `resource.data.sheet`: las escrituras hechas con la cuenta de servicio del Admin SDK **omiten las reglas por completo**, así que el sincronizador sí puede escribir ese mapa. Las reglas solo aplican a los tres navegadores.

### El límite de 20 lecturas por petición

Firestore permite **20 lecturas de documentos desde las reglas por petición**, y cada escritura de un lote evalúa las reglas por separado. Con estas reglas, que leen `/usuarios` para saber quién eres, un lote de 86 clientes pedía 86 lecturas y **el servidor rechazaba el lote entero**. Ese fue el motivo real de que la importación no escribiera nada durante días.

Dos consecuencias de diseño:

1. `activo()` y `esAdmin()` hacen **un solo `get()`**, no `exists()` más dos `get()`. Si el documento no existe, `get()` devuelve null, la regla falla y deniega — que es lo correcto.
2. La aplicación escribe en **tandas de 8 operaciones** (`POR_TANDA`), no en un lote único. Si una tanda falla, las anteriores ya quedaron y el aviso dice cuántas de cuántas se guardaron. La importación es idempotente, así que repetirla arregla el resto.

---

## 7. Costo

| Servicio | Uso esperado | Plan |
|---|---|---|
| Firestore | ~90 documentos, unos miles de lecturas/día | Gratis (50k lecturas/día) |
| Authentication | 3 usuarios, correo y contraseña | Gratis |
| Hosting | un HTML | Gratis (10 GB/mes) |
| Cloud Functions | **no se usan** | — |

**Todo cae en el plan Spark. No hace falta tarjeta.** Ese fue el motivo de hacer el espejo con un botón en el Sheet en vez de un disparador de Firestore.

Consecuencia de no tener Cloud Functions: el campo `resumen` (§2) **lo calcula y escribe la propia interfaz** cada vez que cambia un documento o un movimiento, no un servidor. Funciona bien con tres usuarios; si alguna vez quedara desfasado, se recalcula al abrir el cliente.

---

## 8. Decisiones tomadas

- **Cuenta dueña:** `rentas.chsas@gmail.com`. Proyecto **`rentas-786af`** («Rentas»).
- **Autenticación:** correo y contraseña, sin registro abierto.
- **Espejo hacia el Sheet:** botón «Actualizar» en el menú del Sheet. Sin Blaze.
- **Calendario DIAN 2026:** tomado del Calendario Tributario 2026, por los **dos últimos dígitos** de la cédula, del 12 de agosto al 26 de octubre de 2026.

### Qué queda abierto

- **Reparto interno del 70%.** Hoy Johanna y Leidy van como bolsa común. Si se divide, se separan en dos entradas de `config.reparto` y el cálculo sale solo.
- **Modelo por año gravable** (`/periodos/{año}/…`) adoptado el 27 de julio de 2026, con la base casi vacía. El año activo (`PERIODO`) y la lista `PERIODOS` están en `index.html`; para abrir 2026 se agrega ahí y se carga su calendario DIAN. Etiqueta = **año gravable** (lo de hoy es **2025**, se declara ago–oct 2026).
- **Responsable por cliente:** Johanna, Leidy o John (`RESPONSABLES` en `index.html`), editable en la plataforma, por período.
- **El mapeo respuesta → documento** (`REGLAS_DOC` en `index.html`) lo confirmó el equipo el 27 de julio de 2026. Si la DIAN cambia un requisito, se ajusta esa tabla y se despliega — no depende del Sheet.
