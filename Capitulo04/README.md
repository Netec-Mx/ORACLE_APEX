# Práctica 4: Construir una aplicación de ejemplo (formularios y reportes)

## 1. Metadatos

| Atributo         | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 52 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Crear (Create)                             |
| **Tema**         | 4.1 – 4.3: Generadores, Page Designer, Universal Theme |
| **Laboratorio**  | 04-00-01                                   |

---

## 2. Descripción General

En este laboratorio construirás una aplicación funcional de **Gestión de Proyectos** en Oracle APEX utilizando el **Create Application Wizard** sobre un esquema de base de datos provisto por el instructor. Partirás de tablas existentes para generar automáticamente la estructura base de la aplicación y luego personalizarás cada página en el **Page Designer**: configurarás formularios con validaciones declarativas, crearás reportes clásicos con formato condicional, ajustarás la navegación global y aplicarás el **Universal Theme** con colores personalizados mediante el **Theme Roller**. Al finalizar, tendrás una aplicación multi-página completamente navegable y visualmente coherente.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Utilizar el **Create Application Wizard** para generar una aplicación multi-página a partir de tablas relacionadas existentes.
- [ ] Diseñar páginas en el **Page Designer** configurando regiones, layouts de columnas y orden de renderizado.
- [ ] Personalizar formularios APEX añadiendo validaciones declarativas (NOT NULL, expresión regular, rango numérico) con mensajes de error personalizados.
- [ ] Configurar un **Classic Report** con formato condicional de filas, columnas calculadas y enlace de navegación.
- [ ] Aplicar el **Universal Theme** y personalizar la paleta de colores corporativos con el **Theme Roller**.

---

## 4. Prerrequisitos

### Conocimiento previo
- Haber completado los laboratorios **01-00-01** y **03-00-01**.
- Conocimiento básico de SQL (SELECT, WHERE, JOIN).
- Familiaridad con conceptos de diseño web: formularios, tablas HTML, navegación.

### Acceso y recursos
- Workspace APEX activo con credenciales de desarrollador.
- Esquema de base de datos con las tablas de muestra instaladas (**script provisto por el instructor**). Las tablas requeridas son:

| Tabla              | Descripción                          | Filas mínimas |
|--------------------|--------------------------------------|---------------|
| `PM_PROYECTOS`     | Proyectos de la organización         | 200           |
| `PM_TAREAS`        | Tareas asociadas a proyectos         | 800           |
| `PM_EMPLEADOS`     | Empleados asignables a tareas        | 50            |
| `PM_DEPARTAMENTOS` | Departamentos de la organización     | 10            |
| `PM_CATEGORIAS`    | Categorías de proyectos              | 8             |

> **Nota:** Si el instructor aún no ha ejecutado el script de datos de muestra, detente aquí y solicítalo antes de continuar. Los reportes y gráficas del laboratorio requieren datos suficientes para ser significativos.

---

## 5. Entorno del Laboratorio

### Hardware mínimo requerido

| Componente     | Mínimo                              | Recomendado               |
|----------------|-------------------------------------|---------------------------|
| RAM            | 8 GB                                | 16 GB                     |
| Almacenamiento | 50 GB libres (SSD)                  | SSD NVMe                  |
| Procesador     | Intel Core i5 8ª gen / AMD Ryzen 5  | i7 / Ryzen 7              |
| Monitor        | 1280 × 768                          | 1920 × 1080               |
| Internet       | 10 Mbps                             | 25 Mbps+                  |

### Software requerido

| Software                  | Versión mínima | Uso en este laboratorio          |
|---------------------------|----------------|----------------------------------|
| Oracle APEX               | 23.2           | Plataforma principal             |
| Oracle Database           | 19c / 21c XE   | Backend de datos                 |
| ORDS                      | 23.2           | Servidor HTTP para APEX          |
| Google Chrome / Firefox   | 110+           | Navegador con JavaScript activo  |
| Oracle SQL Developer      | 23.1           | Verificación de esquema (opcional) |

### Verificación del entorno antes de comenzar

Abre SQL Developer o el SQL Workshop de APEX y ejecuta la siguiente consulta para confirmar que las tablas de muestra están disponibles:

```sql
SELECT table_name, num_rows
FROM   user_tables
WHERE  table_name IN (
         'PM_PROYECTOS', 'PM_TAREAS',
         'PM_EMPLEADOS', 'PM_DEPARTAMENTOS', 'PM_CATEGORIAS'
       )
ORDER BY table_name;
```

**Resultado esperado:** 5 filas devueltas. Si alguna tabla falta, contacta al instructor antes de continuar.

---

## 6. Pasos del Laboratorio

---

### Paso 1 — Explorar el esquema de base de datos en SQL Workshop

**Objetivo:** Familiarizarse con la estructura de las tablas antes de generar la aplicación, para tomar decisiones informadas en el wizard.

#### Instrucciones

1. Inicia sesión en tu workspace APEX con tus credenciales de desarrollador.
2. En la barra de navegación superior, haz clic en **SQL Workshop → Object Browser**.
3. En el panel izquierdo, expande la sección **Tables** y verifica que aparecen las cinco tablas: `PM_PROYECTOS`, `PM_TAREAS`, `PM_EMPLEADOS`, `PM_DEPARTAMENTOS`, `PM_CATEGORIAS`.
4. Haz clic sobre `PM_PROYECTOS` y revisa las columnas en la pestaña **Columns**. Anota los nombres y tipos de dato de las columnas principales (especialmente las columnas de estado, fecha y clave foránea).
5. Repite el paso 4 para `PM_TAREAS`.
6. Navega a **SQL Workshop → SQL Commands** y ejecuta la siguiente consulta para entender la distribución de datos:

```sql
SELECT
    p.nombre_proyecto,
    COUNT(t.id_tarea)   AS total_tareas,
    p.estado_proyecto,
    p.fecha_inicio,
    p.fecha_fin_estimada
FROM pm_proyectos p
LEFT JOIN pm_tareas t ON t.id_proyecto = p.id_proyecto
GROUP BY p.nombre_proyecto, p.estado_proyecto,
         p.fecha_inicio, p.fecha_fin_estimada
ORDER BY total_tareas DESC
FETCH FIRST 10 ROWS ONLY;
```

#### Salida esperada
Una tabla con los 10 proyectos con más tareas, sus estados y fechas. Esto confirma que los datos de muestra están correctamente cargados.

#### Verificación
✅ Las 5 tablas aparecen en Object Browser.
✅ La consulta devuelve al menos 5 filas con datos variados en la columna `estado_proyecto`.

---

### Paso 2 — Crear la aplicación base con el Create Application Wizard

**Objetivo:** Usar el generador de aplicaciones de APEX para crear la estructura multi-página de forma declarativa, aprovechando las tablas del esquema.

#### Instrucciones

1. En la barra superior de APEX, haz clic en **App Builder**.
2. Haz clic en el botón **Create** (ícono de más, color azul).
3. Selecciona **New Application** en el diálogo que aparece.
4. En la sección **Name**, escribe:
   - **Application Name:** `Sistema de Gestión de Proyectos`
   - **Application ID:** deja el valor generado automáticamente.
5. En la sección **Appearance**, confirma que el tema seleccionado es **Universal Theme**. Deja el estilo en `Vita` por ahora (lo cambiaremos con el Theme Roller más adelante).
6. En la sección **Pages**, haz clic en **Add Page** para agregar las páginas de la aplicación. Agrega las siguientes páginas en orden:

   **Página 1 — Dashboard (Home):**
   - Haz clic en **Add Page → Dashboard**.
   - **Page Name:** `Dashboard`
   - Haz clic en **Add Page** para confirmar.

   **Página 2 — Reporte de Proyectos:**
   - Haz clic en **Add Page → Interactive Report**.
   - **Page Name:** `Proyectos`
   - **Table or View:** selecciona `PM_PROYECTOS` en el desplegable.
   - Activa la opción **Include Form Page** para generar automáticamente el formulario de edición asociado.
   - **Form Page Name:** `Detalle de Proyecto`
   - Haz clic en **Add Page**.

   **Página 3 — Reporte de Tareas:**
   - Haz clic en **Add Page → Interactive Report**.
   - **Page Name:** `Tareas`
   - **Table or View:** selecciona `PM_TAREAS`.
   - Activa **Include Form Page**.
   - **Form Page Name:** `Detalle de Tarea`
   - Haz clic en **Add Page**.

   **Página 4 — Empleados:**
   - Haz clic en **Add Page → Report with Form**.
   - **Page Name:** `Empleados`
   - **Table or View:** selecciona `PM_EMPLEADOS`.
   - Haz clic en **Add Page**.

7. En la sección **Features**, activa las siguientes opciones:
   - ✅ **Access Control** (control de acceso básico)
   - ✅ **Activity Reporting** (reporte de actividad)
   - ✅ **Breadcrumb** (navegación de migas de pan — se activa automáticamente con el tema)

8. En la sección **Settings**, deja los valores por defecto:
   - **Authentication:** Application Express Accounts
   - **Language:** Spanish (si está disponible en tu instancia)

9. Revisa el resumen en la parte inferior del wizard. Deberías ver 7 páginas listadas (Dashboard + 2 reportes + 2 formularios + Empleados + formulario de Empleados).

10. Haz clic en el botón **Create Application** (botón azul en la parte inferior derecha).

11. APEX generará la aplicación. Cuando termine, haz clic en **Run Application** para ver el resultado inicial.

12. Inicia sesión con tus credenciales de workspace. Navega por las páginas generadas para familiarizarte con la estructura base.

#### Salida esperada
Una aplicación funcional con menú lateral izquierdo que contiene los ítems: Dashboard, Proyectos, Tareas, Empleados. Los reportes muestran datos de las tablas. Los formularios permiten editar registros.

#### Verificación
✅ La aplicación se ejecuta sin errores.
✅ El menú lateral muestra los 4 ítems de navegación.
✅ El reporte de Proyectos muestra registros de `PM_PROYECTOS`.
✅ Al hacer clic en un registro del reporte, se abre el formulario de edición.

---

### Paso 3 — Explorar el Page Designer y configurar el layout de la página Dashboard

**Objetivo:** Familiarizarse con el entorno del Page Designer y configurar el layout de columnas y el orden de renderizado de regiones en la página de inicio.

#### Instrucciones

1. Desde el App Builder, haz clic en el nombre de tu aplicación para acceder a su lista de páginas.
2. Haz clic en la página **1 - Dashboard** para abrirla en el **Page Designer**.
3. Observa los tres paneles principales del Page Designer:
   - **Panel izquierdo (Rendering):** árbol de componentes de la página.
   - **Panel central (Layout):** representación visual de la cuadrícula de la página.
   - **Panel derecho (Properties):** propiedades del componente seleccionado.
4. En el panel izquierdo, expande **Body** para ver las regiones existentes generadas por el wizard.
5. Haz clic en la región **Recently Updated Projects** (o el nombre equivalente generado). En el panel derecho, localiza la sección **Layout**:
   - Cambia **Start New Row** a `Yes`.
   - Cambia **Column** a `1`.
   - Cambia **Column Span** a `6` (la región ocupará 6 de las 12 columnas del grid).
6. Si existe una segunda región de estadísticas, selecciónala y configura:
   - **Start New Row:** `No`
   - **Column Span:** `6`
7. Para agregar una nueva región de bienvenida en la parte superior, haz clic derecho sobre **Body** en el árbol del panel izquierdo → **Create Region**.
8. Configura la nueva región:
   - **Title:** `Bienvenido al Sistema de Gestión de Proyectos`
   - **Type:** `Static Content`
   - **Template:** `Hero` (busca esta opción en el desplegable de Template)
   - **Sequence:** `10` (para que aparezca primero)
   - En la pestaña **Source → HTML Content**, escribe:
     ```html
     <p>Utiliza el menú lateral para navegar entre proyectos, tareas y empleados.
     Este sistema te permite gestionar el ciclo de vida completo de tus proyectos.</p>
     ```
9. Haz clic en **Save** (botón de disco en la barra superior del Page Designer).
10. Haz clic en el botón **Save and Run Page** (triángulo verde) para ver el resultado.

#### Salida esperada
La página Dashboard muestra: (1) una región Hero con el mensaje de bienvenida en la parte superior, (2) las regiones de estadísticas ocupando cada una la mitad del ancho de la página.

#### Verificación
✅ La región Hero aparece en la parte superior con el texto de bienvenida.
✅ Las regiones de estadísticas se muestran en dos columnas lado a lado.
✅ No hay errores de JavaScript en la consola del navegador (F12 → Console).

---

### Paso 4 — Personalizar el formulario de Proyectos con tipos de ítem específicos

**Objetivo:** Modificar el formulario de edición de proyectos para usar tipos de ítem apropiados (Select List, Date Picker, Text Area) y configurar valores por defecto y condiciones de visibilidad.

#### Instrucciones

1. Desde la lista de páginas de la aplicación, haz clic en la página del formulario **Detalle de Proyecto** (generalmente la página 3 o 4, verifica el número en tu instancia).
2. En el árbol del panel izquierdo, expande **Body → [nombre de la región del formulario]** para ver todos los ítems generados.
3. **Configurar el ítem de Estado del Proyecto como Select List:**
   - Haz clic sobre el ítem `P_ESTADO_PROYECTO` (o el nombre generado para la columna de estado).
   - En el panel derecho, cambia **Type** de `Text Field` a `Select List`.
   - En la sección **List of Values**, selecciona **Type:** `Static Values`.
   - Haz clic en el ícono de edición junto a **Static Values** y agrega los siguientes valores:

     | Display Value        | Return Value  |
     |----------------------|---------------|
     | En Planificación     | PLANIFICACION |
     | En Progreso          | EN_PROGRESO   |
     | En Revisión          | EN_REVISION   |
     | Completado           | COMPLETADO    |
     | Cancelado            | CANCELADO     |

   - Activa **Display Null Value:** `Yes` con el texto `-- Selecciona un estado --`.

4. **Configurar los ítems de fecha como Date Picker:**
   - Haz clic sobre el ítem `P_FECHA_INICIO`.
   - Cambia **Type** a `Date Picker`.
   - En **Format Mask**, escribe: `DD/MM/YYYY`.
   - En **Default → Type**, selecciona `Expression` y en **PL/SQL Expression** escribe: `SYSDATE`.
   - Repite para `P_FECHA_FIN_ESTIMADA`, pero sin valor por defecto.

5. **Configurar el ítem de descripción como Text Area:**
   - Haz clic sobre el ítem `P_DESCRIPCION` (o equivalente).
   - Cambia **Type** a `Textarea`.
   - En **Settings → Number of Rows**, escribe `4`.

6. **Configurar el ítem de presupuesto con formato numérico:**
   - Haz clic sobre el ítem `P_PRESUPUESTO`.
   - Verifica que **Type** sea `Number Field`.
   - En **Settings → Number Alignment**, selecciona `Right`.
   - En **Format Mask**, escribe: `999G999G990D00` (formato numérico con separadores).

7. **Agregar condición de visibilidad para el campo de fecha de cierre real:**
   - Haz clic sobre el ítem `P_FECHA_CIERRE_REAL` (si existe en el esquema).
   - En la sección **Server-side Condition**:
     - **Type:** `Item = Value`
     - **Item:** `P_ESTADO_PROYECTO`
     - **Value:** `COMPLETADO`
   - Esto hará que el campo solo sea visible cuando el estado sea "Completado".

8. **Reorganizar los ítems en el layout del formulario:**
   - En el panel central (Layout), arrastra los ítems para organizarlos en dos columnas:
     - Columna izquierda: Nombre del Proyecto, Categoría, Estado, Responsable.
     - Columna derecha: Fecha de Inicio, Fecha Fin Estimada, Presupuesto, Descripción (ancho completo al final).
   - Para cada ítem, ajusta **Column Span** en el panel derecho según sea necesario.

9. Haz clic en **Save and Run Page**.

#### Salida esperada
El formulario muestra el campo de estado como un menú desplegable con 5 opciones. Los campos de fecha muestran un selector de calendario al hacer clic. El campo de descripción es un área de texto multilínea. El campo de fecha de cierre real solo aparece cuando se selecciona "Completado" en el estado.

#### Verificación
✅ El Select List de estado muestra las 5 opciones configuradas.
✅ Al hacer clic en el campo de fecha, aparece el calendario del Date Picker.
✅ El campo `P_FECHA_CIERRE_REAL` desaparece cuando el estado no es "COMPLETADO".
✅ El campo de presupuesto muestra el formato numérico con decimales.

---

### Paso 5 — Agregar validaciones declarativas al formulario de Proyectos

**Objetivo:** Implementar validaciones del lado del servidor para garantizar la integridad de los datos ingresados en el formulario, sin escribir código PL/SQL manual.

#### Instrucciones

1. Permanece en la página del formulario **Detalle de Proyecto** en el Page Designer.
2. En el árbol del panel izquierdo, localiza la sección **Validating** (debajo de Processing). Haz clic derecho → **Create Validation**.

3. **Validación 1 — Nombre de proyecto obligatorio:**
   - **Name:** `VAL_NOMBRE_REQUERIDO`
   - **Type:** `Item is NOT NULL`
   - **Item:** `P_NOMBRE_PROYECTO`
   - **Error Message:** `El nombre del proyecto es obligatorio. Por favor, ingresa un nombre descriptivo.`
   - **Error Display Location:** `Inline with Field and in Notification`
   - Haz clic en **Apply Changes**.

4. **Validación 2 — Fecha fin posterior a fecha inicio:**
   - Crea una nueva validación.
   - **Name:** `VAL_FECHAS_COHERENTES`
   - **Type:** `PL/SQL Expression`
   - **PL/SQL Expression:**
     ```plsql
     TO_DATE(:P_FECHA_FIN_ESTIMADA, 'DD/MM/YYYY') >=
     TO_DATE(:P_FECHA_INICIO, 'DD/MM/YYYY')
     ```
   - **Error Message:** `La fecha de fin estimada debe ser igual o posterior a la fecha de inicio.`
   - **Error Display Location:** `Inline with Field and in Notification`
   - **Associated Item:** `P_FECHA_FIN_ESTIMADA`
   - En **Condition**, agrega:
     - **Type:** `Item is NOT NULL`
     - **Item:** `P_FECHA_FIN_ESTIMADA`
   - Haz clic en **Apply Changes**.

5. **Validación 3 — Presupuesto positivo:**
   - Crea una nueva validación.
   - **Name:** `VAL_PRESUPUESTO_POSITIVO`
   - **Type:** `PL/SQL Expression`
   - **PL/SQL Expression:**
     ```plsql
     :P_PRESUPUESTO > 0
     ```
   - **Error Message:** `El presupuesto debe ser un valor mayor a cero.`
   - **Error Display Location:** `Inline with Field and in Notification`
   - **Associated Item:** `P_PRESUPUESTO`
   - En **Condition**:
     - **Type:** `Item is NOT NULL`
     - **Item:** `P_PRESUPUESTO`
   - Haz clic en **Apply Changes**.

6. **Validación 4 — Formato de código de proyecto (expresión regular):**
   - Crea una nueva validación (asumiendo que existe un campo `P_CODIGO_PROYECTO`).
   - **Name:** `VAL_CODIGO_FORMATO`
   - **Type:** `Item Matches Regular Expression`
   - **Item:** `P_CODIGO_PROYECTO`
   - **Regular Expression:** `^[A-Z]{2,4}-[0-9]{3,6}$`
   - **Error Message:** `El código debe tener el formato: 2-4 letras mayúsculas, guion y 3-6 dígitos. Ejemplo: PROJ-001`
   - **Error Display Location:** `Inline with Field and in Notification`
   - Haz clic en **Apply Changes**.

7. Haz clic en **Save** en la barra del Page Designer.

8. **Probar las validaciones:**
   - Ejecuta la aplicación y navega al formulario de un proyecto.
   - Borra el nombre del proyecto y haz clic en **Save**. Verifica que aparece el mensaje de error de la validación 1.
   - Ingresa una fecha de fin anterior a la de inicio. Verifica el mensaje de la validación 2.
   - Ingresa un presupuesto negativo (-100). Verifica el mensaje de la validación 3.
   - Ingresa un código con formato incorrecto (por ejemplo, `abc123`). Verifica el mensaje de la validación 4.

#### Salida esperada
Al intentar guardar con datos inválidos, aparecen mensajes de error en rojo directamente junto a cada campo afectado y en la notificación superior de la página. Los mensajes son los textos personalizados configurados.

#### Verificación
✅ Cada validación muestra su mensaje de error personalizado.
✅ Los errores aparecen tanto inline (junto al campo) como en la notificación superior.
✅ Al corregir los datos y guardar nuevamente, el registro se guarda exitosamente.

---

### Paso 6 — Crear un Classic Report con formato condicional y columnas calculadas

**Objetivo:** Reemplazar el Interactive Report de Proyectos por un Classic Report enriquecido con formato condicional de filas, una columna calculada y un enlace de navegación al formulario.

> **Nota pedagógica:** En una aplicación real, los Interactive Reports ofrecen más flexibilidad para el usuario final. Este paso usa Classic Report para demostrar el control preciso del desarrollador sobre el formato de presentación, que es un concepto clave del tema 4.2.

#### Instrucciones

1. Desde la lista de páginas, abre la página **Proyectos** en el Page Designer.
2. En el árbol del panel izquierdo, haz clic derecho sobre **Body → Create Region**.
3. Configura la nueva región:
   - **Title:** `Resumen de Proyectos`
   - **Type:** `Classic Report`
   - **Source → SQL Query:**

     ```sql
     SELECT
         p.id_proyecto,
         p.codigo_proyecto,
         p.nombre_proyecto,
         d.nombre_departamento,
         c.nombre_categoria,
         p.estado_proyecto,
         p.fecha_inicio,
         p.fecha_fin_estimada,
         p.presupuesto,
         COUNT(t.id_tarea)                                    AS total_tareas,
         SUM(CASE WHEN t.estado_tarea = 'COMPLETADA'
                  THEN 1 ELSE 0 END)                          AS tareas_completadas,
         ROUND(
             SUM(CASE WHEN t.estado_tarea = 'COMPLETADA'
                      THEN 1 ELSE 0 END) * 100.0
             / NULLIF(COUNT(t.id_tarea), 0), 1
         )                                                    AS pct_completado,
         CASE
             WHEN p.estado_proyecto = 'COMPLETADO'    THEN 'success'
             WHEN p.estado_proyecto = 'CANCELADO'     THEN 'danger'
             WHEN p.estado_proyecto = 'EN_PROGRESO'   THEN 'warning'
             WHEN p.estado_proyecto = 'EN_REVISION'   THEN 'info'
             ELSE 'default'
         END                                                  AS css_estado
     FROM   pm_proyectos    p
     JOIN   pm_departamentos d ON d.id_departamento = p.id_departamento
     JOIN   pm_categorias   c ON c.id_categoria    = p.id_categoria
     LEFT JOIN pm_tareas    t ON t.id_proyecto     = p.id_proyecto
     GROUP BY
         p.id_proyecto, p.codigo_proyecto, p.nombre_proyecto,
         d.nombre_departamento, c.nombre_categoria,
         p.estado_proyecto, p.fecha_inicio,
         p.fecha_fin_estimada, p.presupuesto
     ORDER BY p.fecha_inicio DESC
     ```

4. **Configurar columnas del reporte:**
   - En el árbol del panel izquierdo, expande la región **Resumen de Proyectos → Columns**.
   - Haz clic en la columna `ID_PROYECTO`:
     - **Type:** `Hidden Column` (esta columna no se muestra, se usa para el enlace).
   - Haz clic en la columna `NOMBRE_PROYECTO`:
     - En **Link**, activa **Link Column: Link to Custom Target**.
     - **Target → Type:** `Page in this Application`
     - **Target → Page:** número de la página del formulario Detalle de Proyecto.
     - **Set Items → Name:** `P_ID_PROYECTO` → **Value:** `#ID_PROYECTO#`
     - **Link Text:** `#NOMBRE_PROYECTO#`
   - Haz clic en la columna `PCT_COMPLETADO`:
     - **Heading:** `% Completado`
     - **Format Mask:** `999G990D0"%"`
   - Haz clic en la columna `PRESUPUESTO`:
     - **Format Mask:** `$999G999G990D00`
     - **Number Alignment:** `Right`
   - Haz clic en la columna `CSS_ESTADO`:
     - **Type:** `Hidden Column` (se usa solo para el formato condicional).

5. **Configurar formato condicional de filas:**
   - Selecciona la región **Resumen de Proyectos**.
   - En el panel derecho, ve a la sección **Appearance → CSS Classes**.
   - Escribe: `t-Report--altRowsDefault`
   - Ahora, en el panel izquierdo, haz clic en la columna `ESTADO_PROYECTO`:
     - En **Column Formatting → HTML Expression**, escribe:
       ```html
       <span class="t-Badge t-Badge--&CSS_ESTADO.">
         &ESTADO_PROYECTO.
       </span>
       ```

6. **Configurar opciones de descarga del reporte:**
   - Selecciona la región **Resumen de Proyectos**.
   - En el panel derecho, ve a **Attributes → Report → Enable Users To**.
   - Activa: ✅ **Download** → selecciona formatos: `CSV`, `Excel`, `PDF`.

7. Haz clic en **Save and Run Page**.

#### Salida esperada
El reporte muestra una tabla con colores de badge diferenciados por estado (verde para Completado, rojo para Cancelado, amarillo para En Progreso). Los nombres de proyecto son enlaces clicables que abren el formulario de edición. La columna de porcentaje muestra valores con un decimal y símbolo %. Los botones de descarga aparecen en la barra de herramientas del reporte.

#### Verificación
✅ Los badges de estado muestran colores distintos según el valor.
✅ Al hacer clic en el nombre de un proyecto, se abre el formulario con los datos correctos.
✅ El porcentaje completado muestra valores entre 0% y 100%.
✅ Los botones de descarga CSV y Excel están visibles y funcionales.

---

### Paso 7 — Personalizar el formulario de Tareas con ítem File Browse y Select List dinámico

**Objetivo:** Agregar un ítem de carga de archivos al formulario de tareas y configurar un Select List dinámico que obtenga sus valores desde la tabla `PM_EMPLEADOS`.

#### Instrucciones

1. Abre la página **Detalle de Tarea** en el Page Designer.
2. **Agregar ítem File Browse para adjuntar archivos:**
   - En el árbol del panel izquierdo, haz clic derecho sobre la región del formulario → **Create Page Item**.
   - **Name:** `P_ARCHIVO_ADJUNTO`
   - **Type:** `File Browse`
   - **Label:** `Adjuntar Documento`
   - En **Settings**:
     - **Storage Type:** `Table APEX_APPLICATION_FILES`
     - **Allowed File Types:** `.pdf,.doc,.docx,.xlsx,.jpg,.png`
     - **Maximum File Size (MB):** `5`
   - **Layout → Sequence:** colócalo al final del formulario.

3. **Configurar el Select List dinámico para Empleado Asignado:**
   - Haz clic sobre el ítem `P_ID_EMPLEADO_ASIGNADO` (o el nombre correspondiente al campo de empleado).
   - Cambia **Type** a `Select List`.
   - En **List of Values**:
     - **Type:** `SQL Query`
     - **SQL Query:**
       ```sql
       SELECT e.nombre_completo AS display_value,
              e.id_empleado     AS return_value
       FROM   pm_empleados e
       WHERE  e.activo = 'S'
       ORDER BY e.nombre_completo
       ```
   - **Display Null Value:** `Yes`
   - **Null Display Value:** `-- Sin asignar --`

4. **Agregar validación de prioridad con rango numérico:**
   - En la sección **Validating**, crea una nueva validación:
   - **Name:** `VAL_PRIORIDAD_RANGO`
   - **Type:** `PL/SQL Expression`
   - **PL/SQL Expression:**
     ```plsql
     :P_PRIORIDAD BETWEEN 1 AND 5
     ```
   - **Error Message:** `La prioridad debe ser un valor entre 1 (más alta) y 5 (más baja).`
   - **Associated Item:** `P_PRIORIDAD`
   - **Condition → Type:** `Item is NOT NULL`
   - **Condition → Item:** `P_PRIORIDAD`

5. Haz clic en **Save and Run Page**.

6. **Probar la carga de archivos:**
   - Abre el formulario de una tarea existente.
   - En el campo "Adjuntar Documento", haz clic en **Choose File** y selecciona un archivo PDF de prueba (menos de 5 MB).
   - Haz clic en **Save**.
   - Vuelve a abrir el registro y verifica que aparece un enlace al archivo adjunto.

#### Salida esperada
El formulario de tareas muestra un selector de archivos con restricción de tipos. El campo de empleado asignado muestra un menú desplegable con los nombres de empleados activos. Al intentar guardar una prioridad fuera del rango 1-5, aparece el mensaje de error.

#### Verificación
✅ El Select List de empleados muestra nombres en orden alfabético.
✅ El campo File Browse acepta archivos PDF y rechaza tipos no permitidos.
✅ La validación de prioridad funciona correctamente para valores fuera de rango.

---

### Paso 8 — Configurar la navegación global: Navigation Menu y Breadcrumb

**Objetivo:** Personalizar el menú lateral de navegación y configurar el breadcrumb para que refleje la jerarquía de páginas de la aplicación.

#### Instrucciones

1. En el App Builder, ve a tu aplicación y haz clic en **Shared Components** (ícono de engranaje o en el menú superior de la aplicación).
2. En la sección **Navigation and Search**, haz clic en **Navigation Menu**.
3. Verás el menú **Desktop Navigation Menu**. Haz clic en él para editarlo.
4. **Reorganizar y personalizar los ítems del menú:**
   - Verifica que los ítems generados automáticamente existen: Dashboard, Proyectos, Tareas, Empleados.
   - Haz clic en el ítem **Dashboard** para editarlo:
     - **Icon:** escribe `fa-home` (ícono de casa de Font APEX).
     - Haz clic en **Apply Changes**.
   - Haz clic en el ítem **Proyectos**:
     - **Icon:** `fa-folder`
     - Haz clic en **Apply Changes**.
   - Haz clic en el ítem **Tareas**:
     - **Icon:** `fa-check-square-o`
     - Haz clic en **Apply Changes**.
   - Haz clic en el ítem **Empleados**:
     - **Icon:** `fa-users`
     - Haz clic en **Apply Changes**.

5. **Agregar un separador y un ítem de administración:**
   - Haz clic en **Create Entry**.
   - **Type:** `Navigation Menu Entry`
   - **Name:** `Administración`
   - **Target:** deja en blanco (será un ítem padre).
   - **Icon:** `fa-cog`
   - **Sequence:** `50`
   - Haz clic en **Create Navigation Menu Entry**.
   - Crea otro ítem hijo:
     - **Name:** `Categorías`
     - **Target:** página de gestión de categorías (si existe) o escribe `#` temporalmente.
     - **Parent Entry:** `Administración`
     - **Icon:** `fa-tag`
     - Haz clic en **Create Navigation Menu Entry**.

6. **Configurar el Breadcrumb:**
   - Regresa a **Shared Components → Breadcrumbs**.
   - Haz clic en **Breadcrumb** (el breadcrumb principal).
   - Verifica que cada página de la aplicación tiene su entrada de breadcrumb con el padre correcto:
     - Dashboard → (sin padre / raíz)
     - Proyectos → Dashboard
     - Detalle de Proyecto → Proyectos
     - Tareas → Dashboard
     - Detalle de Tarea → Tareas
   - Si alguna entrada falta o tiene el padre incorrecto, haz clic en ella y corrige el campo **Parent Entry**.

7. **Verificar la navegación en tiempo de ejecución:**
   - Ejecuta la aplicación.
   - Navega desde Dashboard → Proyectos → haz clic en un proyecto para abrir el formulario.
   - Verifica que el breadcrumb muestra: `Dashboard / Proyectos / Detalle de Proyecto`.
   - Haz clic en "Proyectos" en el breadcrumb para regresar al reporte.

#### Salida esperada
El menú lateral muestra los 4 ítems principales con iconos. El ítem "Administración" es un menú colapsable con "Categorías" como subítem. El breadcrumb muestra la ruta de navegación completa en cada página y los enlaces son funcionales.

#### Verificación
✅ Cada ítem del menú lateral muestra su ícono Font APEX.
✅ El submenú "Administración" se expande/colapsa al hacer clic.
✅ El breadcrumb en el formulario muestra 3 niveles de navegación.
✅ Los enlaces del breadcrumb navegan a la página correcta.

---

### Paso 9 — Aplicar el Universal Theme y personalizar con Theme Roller

**Objetivo:** Personalizar la apariencia visual de la aplicación usando el Theme Roller para aplicar una paleta de colores corporativa, sin modificar las plantillas base.

#### Instrucciones

1. **Acceder al Theme Roller desde la aplicación en ejecución:**
   - Ejecuta tu aplicación en el navegador.
   - En la barra de desarrollo de APEX (barra azul en la parte inferior del navegador), haz clic en el ícono de **lápiz/edición rápida**.
   - En el menú emergente, selecciona **Theme Roller**.
   - Se abrirá el panel del Theme Roller en el lado derecho del navegador.

2. **Seleccionar un estilo base:**
   - En la sección **Style**, cambia el estilo de `Vita` a `Vita - Slate` para ver el efecto inmediato.
   - Observa cómo cambia la paleta de colores en tiempo real.
   - Vuelve a `Vita` como punto de partida para la personalización.

3. **Personalizar el color primario:**
   - En la sección **Global Colors**, localiza **Custom** o **Override CSS Variables**.
   - Haz clic en el cuadro de color junto a **Header Background Color** (o `--ut-header-bg-color`).
   - Ingresa el valor hexadecimal: `#1A3A5C` (azul marino corporativo).
   - Observa cómo la cabecera de la aplicación cambia al nuevo color.

4. **Ajustar el color de acento (botones y enlaces):**
   - Localiza **Body Link Color** o el control de color primario de acción.
   - Cambia el valor a: `#0066CC` (azul brillante para contraste).

5. **Cambiar la fuente base:**
   - Localiza la sección **Typography** o **Font**.
   - Si el Theme Roller de tu instancia lo permite, cambia el **Body Font** a `"Segoe UI", sans-serif`.

6. **Configurar el estilo de la barra de navegación lateral:**
   - Localiza **Navigation Background** o `--ut-nav-bg-color`.
   - Cambia el valor a: `#0D2137` (azul oscuro para el menú lateral).
   - Cambia **Navigation Text Color** a: `#FFFFFF` (texto blanco para contraste).

7. **Guardar los cambios:**
   - Haz clic en el botón **Save** en la parte superior del panel Theme Roller.
   - En el diálogo de confirmación, escribe el nombre del estilo: `Corporativo Azul`.
   - Haz clic en **Save**.

8. **Verificar la accesibilidad del contraste:**
   - Con los nuevos colores aplicados, verifica visualmente que el texto en la cabecera y el menú lateral es legible (texto blanco sobre fondo azul oscuro).
   - Si el contraste no es suficiente, ajusta los colores de texto hasta lograr una relación de contraste mínima de 4.5:1 (estándar WCAG AA).

9. **Explorar las Template Options en una región:**
   - Regresa al Page Designer de la página **Proyectos**.
   - Haz clic sobre la región del Classic Report **Resumen de Proyectos**.
   - En el panel derecho, haz clic en el botón **Template Options** (junto al campo Template).
   - Explora las opciones disponibles y activa:
     - **Style:** `Standard`
     - **Borders:** `All Borders`
     - **Stripe Rows:** actívalo si está disponible.
   - Haz clic en **OK** y luego **Save and Run Page**.

#### Salida esperada
La aplicación muestra una cabecera azul marino, menú lateral azul oscuro con texto blanco, y botones en azul brillante. El estilo guardado aparece como "Corporativo Azul" en el Theme Roller. La región del reporte muestra bordes en todas las celdas y filas alternadas con color diferente.

#### Verificación
✅ La cabecera muestra el color `#1A3A5C`.
✅ El menú lateral muestra texto blanco sobre fondo `#0D2137`.
✅ El estilo "Corporativo Azul" aparece en la lista de estilos del Theme Roller.
✅ Las Template Options del reporte muestran el efecto de bordes y filas alternadas.

---

## 7. Validación y Pruebas Finales

Antes de dar por completado el laboratorio, ejecuta las siguientes pruebas de validación integral:

### Lista de verificación final

```
PRUEBA 1: Navegación completa
□ Acceder al Dashboard desde el menú lateral.
□ Navegar a Proyectos → hacer clic en un proyecto → abrir formulario.
□ Verificar que el breadcrumb muestra "Dashboard / Proyectos / Detalle de Proyecto".
□ Regresar a Proyectos usando el breadcrumb.
□ Navegar a Tareas → abrir una tarea → verificar breadcrumb.

PRUEBA 2: Validaciones del formulario de Proyectos
□ Intentar guardar sin nombre → verificar mensaje de error.
□ Ingresar fecha fin anterior a fecha inicio → verificar mensaje de error.
□ Ingresar presupuesto negativo → verificar mensaje de error.
□ Ingresar código con formato incorrecto → verificar mensaje de error.
□ Corregir todos los errores y guardar exitosamente → verificar redirección al reporte.

PRUEBA 3: Reporte clásico con formato condicional
□ Verificar que proyectos con estado COMPLETADO muestran badge verde.
□ Verificar que proyectos CANCELADOS muestran badge rojo.
□ Verificar que el enlace en el nombre del proyecto abre el formulario correcto.
□ Descargar el reporte en formato CSV y verificar que el archivo se descarga.

PRUEBA 4: Formulario de Tareas
□ Abrir el formulario de una tarea nueva.
□ Verificar que el Select List de empleados muestra nombres en orden alfabético.
□ Intentar adjuntar un archivo de tipo no permitido (ej: .exe) → verificar rechazo.
□ Ingresar prioridad = 6 → verificar mensaje de error de rango.

PRUEBA 5: Apariencia y tema
□ Verificar que los colores corporativos están aplicados en toda la aplicación.
□ Verificar que el menú lateral muestra íconos junto a cada ítem.
□ Verificar que el submenú "Administración" se expande correctamente.
□ Verificar que la aplicación es usable en una ventana reducida a 1280x768.
```

### Consulta de verificación en base de datos

Ejecuta la siguiente consulta en SQL Workshop para confirmar que los datos guardados durante las pruebas son correctos:

```sql
SELECT
    p.id_proyecto,
    p.nombre_proyecto,
    p.estado_proyecto,
    p.fecha_inicio,
    p.fecha_fin_estimada,
    p.presupuesto,
    (SELECT COUNT(*) FROM pm_tareas t WHERE t.id_proyecto = p.id_proyecto) AS tareas
FROM pm_proyectos p
WHERE p.fecha_inicio >= SYSDATE - 30
ORDER BY p.id_proyecto DESC
FETCH FIRST 5 ROWS ONLY;
```

**Resultado esperado:** Los registros creados o modificados durante el laboratorio aparecen con datos coherentes (fechas válidas, presupuesto positivo, estado en los valores permitidos).

---

## 8. Resolución de Problemas

### Problema 1: El formato condicional del Classic Report no aplica los colores de los badges

**Síntomas:** La columna `ESTADO_PROYECTO` muestra el texto plano del estado (por ejemplo, "EN_PROGRESO") en lugar de un badge con color. La expresión HTML en la configuración de la columna parece correcta pero no se renderiza como HTML.

**Causa:** Por defecto, APEX escapa el contenido HTML en las columnas de reportes para prevenir ataques XSS. Si la opción de escape está activa, la expresión HTML se muestra como texto literal en lugar de renderizarse.

**Solución:**
1. En el Page Designer, haz clic sobre la columna `ESTADO_PROYECTO` del Classic Report.
2. En el panel derecho, localiza la sección **Security**.
3. Cambia **Escape Special Characters** de `Yes` a `No`.
4. Haz clic en **Save and Run Page**.

> ⚠️ **Advertencia de seguridad:** Desactivar el escape de HTML solo es seguro cuando el contenido de la columna proviene de una fuente controlada (como un CASE WHEN en tu propia consulta SQL), no de entrada directa del usuario. En este caso es seguro porque el HTML lo genera la consulta, no el usuario.

---

### Problema 2: El Select List de empleados en el formulario de Tareas aparece vacío o muestra un error ORA-

**Síntomas:** Al abrir el formulario de Detalle de Tarea, el campo "Empleado Asignado" muestra el mensaje "-- Sin asignar --" como única opción, o aparece un error de tipo `ORA-00942: table or view does not exist` o `ORA-01017: invalid username/password`.

**Causa:** La consulta SQL del List of Values hace referencia a `PM_EMPLEADOS` pero el usuario de la sesión de APEX no tiene permisos SELECT sobre esa tabla, o la tabla no existe en el esquema correcto. Esto ocurre frecuentemente cuando el esquema de la aplicación APEX es diferente al esquema donde están las tablas.

**Solución:**
1. Ve a **SQL Workshop → SQL Commands** y ejecuta:
   ```sql
   SELECT COUNT(*) FROM pm_empleados WHERE activo = 'S';
   ```
   - Si devuelve un error, la tabla no existe en el esquema actual. Verifica con el instructor que ejecutaste el script de datos de muestra en el esquema correcto.
   - Si devuelve un número, la tabla existe. Continúa al siguiente paso.

2. Verifica el esquema asociado a la aplicación: en App Builder → tu aplicación → **Edit Application Properties** → verifica el campo **Parsing Schema**. Debe coincidir con el esquema donde están las tablas.

3. Si el esquema es diferente, tienes dos opciones:
   - **Opción A:** Cambiar el Parsing Schema de la aplicación al esquema correcto (requiere permisos de administrador).
   - **Opción B:** Crear un sinónimo en el esquema actual:
     ```sql
     CREATE SYNONYM pm_empleados FOR otro_esquema.pm_empleados;
     ```
4. Después de corregir el esquema o crear el sinónimo, regresa al Page Designer y haz clic en **Save and Run Page** para refrescar el List of Values.

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, **no elimines la aplicación** ya que será la base de los laboratorios 05 al 10. Sin embargo, realiza las siguientes acciones de limpieza:

1. **Exportar la aplicación como respaldo:**
   - En App Builder, haz clic en el menú de la aplicación (tres puntos o ícono de engranaje).
   - Selecciona **Export / Import → Export**.
   - Selecciona **Application** y haz clic en **Export**.
   - Guarda el archivo `.sql` generado con el nombre: `lab04_gestion_proyectos_backup.sql`.
   - Almacena el archivo en tu carpeta de trabajo del curso.

2. **Eliminar archivos de prueba adjuntos:**
   - En SQL Workshop → SQL Commands, ejecuta:
     ```sql
     DELETE FROM apex_application_files
     WHERE  created_on < SYSDATE
       AND  created_on > SYSDATE - 1/24;  -- archivos de la última hora
     COMMIT;
     ```
   - Esto elimina los archivos de prueba cargados durante el laboratorio sin afectar datos de producción.

3. **Registrar el número de la aplicación:**
   - En App Builder, anota el **Application ID** de tu aplicación (visible en la lista de aplicaciones o en la URL del Page Designer).
   - Necesitarás este número en el Laboratorio 05.

4. **Cerrar sesiones activas:**
   - Cierra la pestaña del navegador donde está ejecutándose la aplicación.
   - No cierres la sesión del App Builder si continuarás con el siguiente laboratorio.

---

## 10. Resumen

En este laboratorio completaste la construcción de una aplicación multi-página de Gestión de Proyectos en Oracle APEX, aplicando el enfoque **declarativo** que caracteriza a la plataforma. Los logros principales fueron:

| Componente construido                    | Técnica aplicada                                    |
|------------------------------------------|-----------------------------------------------------|
| Aplicación base con 7 páginas            | Create Application Wizard                           |
| Layout de Dashboard en 2 columnas        | Page Designer → Layout → Column Span                |
| Formulario con Select List, Date Picker  | Page Item Types + List of Values estático           |
| Validaciones: NOT NULL, fechas, regex    | Validations declarativas en Page Designer           |
| Classic Report con formato condicional   | HTML Expression + CSS Badge classes                 |
| Select List dinámico desde tabla         | List of Values con SQL Query                        |
| Menú lateral con íconos y submenús       | Shared Components → Navigation Menu                 |
| Breadcrumb jerárquico                    | Shared Components → Breadcrumbs                     |
| Paleta de colores corporativa            | Theme Roller → CSS Custom Properties                |

### Conceptos clave aprendidos

- Los **generadores de APEX** crean metadatos, no código fuente. La aplicación existe como definición en el repositorio y se renderiza dinámicamente en cada petición.
- Las **plantillas** definen el *cómo* renderizar; los metadatos definen el *qué* mostrar. Las **Template Options** permiten variaciones sin modificar la plantilla base.
- Las **validaciones declarativas** son más mantenibles que el código PL/SQL manual para casos estándar (NOT NULL, rangos, expresiones regulares).
- El **Universal Theme** usa CSS Custom Properties, lo que permite personalización global mediante el Theme Roller sin tocar código.
- **Nunca modificar plantillas base del tema:** siempre trabajar con copias o con Template Options.

### Recursos adicionales

- [Oracle APEX Documentation — Creating Applications](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/creating-applications.html)
- [Universal Theme — Template Options Reference](https://apex.oracle.com/pls/apex/r/apex_pm/ut/template-options)
- [Theme Roller Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/using-theme-roller.html)
- [Oracle APEX Classic Report Configuration](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/managing-classic-report-attributes.html)
- [Font APEX Icons Reference](https://apex.oracle.com/pls/apex/r/apex_pm/ut/icons)

> **Próximo laboratorio:** En el **Laboratorio 05-00-01** agregarás dashboards interactivos con JET Charts, reportes interactivos avanzados y un calendario de tareas sobre esta misma aplicación. Asegúrate de tener el respaldo exportado en este paso antes de continuar.

---
LAB_END---
