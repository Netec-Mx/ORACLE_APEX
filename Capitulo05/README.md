# Práctica 5: Implementar dashboards y reportes interactivos

---

## Metadatos del Laboratorio

| Atributo         | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 52 minutos                                   |
| **Complejidad**  | Alta                                         |
| **Nivel Bloom**  | Crear (Create)                               |
| **Laboratorio**  | 05-00-01                                     |
| **Versión APEX** | 23.2 o superior                              |
| **Prerrequisito**| Lab 04-00-01 completado                      |

---

## Descripción General

En este laboratorio transformarás la aplicación construida en el Laboratorio 04 añadiendo un módulo completo de visualización y análisis de datos. Crearás un **dashboard ejecutivo** que combina un Interactive Report con funcionalidades avanzadas, un Interactive Grid para edición masiva en línea, y un conjunto de JET Charts con datos reales. Finalizarás auditando la accesibilidad de tus componentes con Google Lighthouse y corrigiendo al menos tres problemas identificados.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Configurar un Interactive Report con filtros guardados, agrupaciones, cálculos de columna, vistas de pivote y descarga en múltiples formatos.
- [ ] Implementar un Interactive Grid con columnas editables, validaciones de celda y procesamiento de cambios múltiples mediante DML declarativo.
- [ ] Construir un dashboard ejecutivo con regiones de tipo Chart (barras, líneas, pie/donut), Calendar, Cards y métricas KPI.
- [ ] Configurar JET Charts con series múltiples, ejes secundarios, tooltips personalizados y actualización dinámica mediante refresh.
- [ ] Aplicar técnicas de accesibilidad (etiquetas ARIA, contraste, navegación por teclado) y verificarlas con Google Lighthouse.

---

## Prerrequisitos

### Conocimiento previo
- Laboratorio 04-00-01 completado: la aplicación base debe estar operativa con al menos las tablas `EMPLEADOS`, `DEPARTAMENTOS`, `PROYECTOS` y `ASIGNACIONES` (o equivalentes provistas por el instructor).
- Conocimiento de SQL para consultas de agregación: `GROUP BY`, `SUM`, `COUNT`, `AVG`, `ROLLUP`.
- Familiaridad básica con el Page Designer de APEX (paneles izquierdo, central y derecho).
- Conceptos básicos de KPI y dashboard ejecutivo.

### Acceso y datos requeridos
- Workspace de APEX activo con la aplicación del Lab 04 disponible.
- Mínimo **500 registros** en las tablas principales (idealmente 1 000+). Si no los tienes, ejecuta el script de datos de muestra provisto por el instructor antes de continuar.
- Permiso de edición sobre la aplicación (rol de desarrollador en el workspace).

> ⚠️ **Nota importante:** Si no completaste el Lab 04, solicita al instructor el *snapshot* de la aplicación correspondiente a esa etapa e impórtalo antes de comenzar.

---

## Entorno de Laboratorio

### Requisitos de hardware

| Componente     | Mínimo                        | Recomendado                   |
|----------------|-------------------------------|-------------------------------|
| RAM            | 8 GB                          | 16 GB                         |
| Almacenamiento | 50 GB libres (HDD)            | 50 GB libres (SSD)            |
| Procesador     | Intel Core i5 8ª gen / Ryzen 5| Intel Core i7 / Ryzen 7       |
| Resolución     | 1280 × 768                    | 1920 × 1080                   |

### Requisitos de software

| Software                  | Versión mínima | Uso en este lab                          |
|---------------------------|---------------|------------------------------------------|
| Oracle APEX               | 23.2          | Plataforma principal de desarrollo       |
| Oracle Database           | 19c / 21c XE  | Motor de base de datos                   |
| ORDS                      | 23.2          | Servidor de aplicaciones APEX            |
| Google Chrome             | 110+          | Navegador principal y Lighthouse         |
| Oracle SQL Developer      | 23.1          | Ejecución de scripts SQL de muestra      |
| Visual Studio Code        | 1.80+         | Edición de fragmentos JavaScript/CSS     |

### Verificación del entorno antes de comenzar

Ejecuta la siguiente consulta en SQL Developer (o en el SQL Workshop de APEX) para confirmar que tienes datos suficientes:

```sql
-- Verificar volumen de datos antes de iniciar el laboratorio
SELECT
    'EMPLEADOS'     AS tabla, COUNT(*) AS registros FROM empleados
UNION ALL
SELECT
    'DEPARTAMENTOS' AS tabla, COUNT(*) AS registros FROM departamentos
UNION ALL
SELECT
    'PROYECTOS'     AS tabla, COUNT(*) AS registros FROM proyectos
UNION ALL
SELECT
    'ASIGNACIONES'  AS tabla, COUNT(*) AS registros FROM asignaciones;
```

**Resultado esperado:** cada tabla debe tener al menos los registros indicados:

| TABLA          | REGISTROS MÍNIMOS |
|----------------|-------------------|
| EMPLEADOS      | 500               |
| DEPARTAMENTOS  | 10                |
| PROYECTOS      | 50                |
| ASIGNACIONES   | 500               |

Si alguna tabla tiene menos registros, ejecuta el script `lab05_datos_muestra.sql` provisto por el instructor antes de continuar.

---

## Pasos del Laboratorio

---

### Paso 1 — Crear la página del Dashboard Ejecutivo

**Objetivo:** Añadir una nueva página a la aplicación que servirá como contenedor del dashboard ejecutivo, con el layout de cuadrícula adecuado para múltiples regiones.

#### Instrucciones

1. Ingresa a tu workspace de APEX y abre la aplicación del Lab 04 desde el **App Builder**.
2. En la vista de páginas de la aplicación, haz clic en el botón **Create Page**.
3. En el asistente de creación de páginas, selecciona **Blank Page** y haz clic en **Next**.
4. Completa los campos del asistente:
   - **Page Number:** `10`
   - **Name:** `Dashboard Ejecutivo`
   - **Page Mode:** `Normal`
   - **Breadcrumb:** selecciona la entrada de breadcrumb raíz de tu aplicación.
5. En la sección **Navigation**, activa **Use Breadcrumb** y **Use Navigation**. Establece el **Navigation Preference** en `Create a new navigation menu entry` con la etiqueta `Dashboard`.
6. Haz clic en **Create Page**.
7. Una vez en el **Page Designer** de la página 10, localiza el panel derecho (propiedades de página) y configura:
   - **CSS Classes:** `apex-dashboard-page`
   - En la sección **Appearance**, selecciona el **Page Template** `No Tabs with Side Bar` (o el equivalente disponible en tu tema).

#### Resultado esperado
La página 10 aparece en el árbol de páginas de la aplicación. Al ejecutarla, muestra una página en blanco con el título "Dashboard Ejecutivo" y la entrada correspondiente en el menú de navegación.

#### Verificación
Haz clic en el botón **Save and Run Page** (ícono ▶). El navegador debe abrir la página sin errores JavaScript en la consola del navegador (F12 → Console).

---

### Paso 2 — Implementar el Interactive Report con funcionalidades avanzadas

**Objetivo:** Crear un Interactive Report completo sobre la tabla de empleados, habilitando todas las funcionalidades avanzadas que APEX ofrece: filtros guardados, agrupaciones, cálculos de columna, pivotes y descarga en múltiples formatos.

#### Instrucciones

1. En el **Page Designer** de la página 10, haz clic derecho sobre **Content Body** en el panel izquierdo y selecciona **Add Region**.
2. En el panel derecho, configura la región:
   - **Title:** `Análisis de Empleados`
   - **Type:** `Interactive Report`
3. En la propiedad **Source → SQL Query**, introduce la siguiente consulta:

```sql
SELECT
    e.employee_id                               AS "ID",
    e.first_name || ' ' || e.last_name          AS "Nombre Completo",
    d.department_name                           AS "Departamento",
    j.job_title                                 AS "Puesto",
    e.salary                                    AS "Salario",
    e.hire_date                                 AS "Fecha Contratación",
    TRUNC(MONTHS_BETWEEN(SYSDATE, e.hire_date) / 12) AS "Años Antigüedad",
    e.commission_pct * 100                      AS "Comisión %",
    CASE
        WHEN e.salary < 5000  THEN 'Bajo'
        WHEN e.salary < 10000 THEN 'Medio'
        ELSE 'Alto'
    END                                         AS "Rango Salarial",
    e.email                                     AS "Email",
    l.city                                      AS "Ciudad"
FROM
    employees   e
    JOIN departments d ON e.department_id = d.department_id
    JOIN jobs         j ON e.job_id        = j.job_id
    JOIN locations    l ON d.location_id   = l.location_id
ORDER BY
    e.last_name, e.first_name
```

> 📝 **Nota:** Adapta los nombres de tabla y columna a los de tu esquema si difieren de los del esquema HR estándar de Oracle.

4. En la sección **Attributes** del Interactive Report (panel derecho, pestaña **Attributes**), configura:
   - **Search Bar:** `Yes`
   - **Actions Menu:** `Yes`
   - **Show Actions Menu:** `Yes`
   - **Download:** `Yes`
   - **Download Formats:** activa `CSV`, `HTML`, `Excel`, `PDF`
   - **Saved Reports:** `Yes`
   - **Enable Users to Save Public Reports:** `Yes` (requiere rol de desarrollador en producción; en lab está permitido)
   - **Subscriptions:** `Yes`

5. Configura las columnas del IR. En el panel izquierdo, expande la región y selecciona la columna **Salario**:
   - **Column Alignment:** `Right`
   - **Format Mask:** `$999,999.00`
   - **Heading Alignment:** `Center`

6. Selecciona la columna **Fecha Contratación**:
   - **Format Mask:** `DD/MM/YYYY`

7. Selecciona la columna **Comisión %**:
   - **Format Mask:** `FM990.00`

8. Configura un **reporte guardado por defecto** para que los usuarios vean inicialmente los empleados ordenados por departamento con un agrupamiento visual. En el panel de Attributes del IR, busca la sección **Default Report Settings** y establece:
   - **Default Report Type:** `Alternative`
   - Haz clic en **Save** para guardar los cambios.

9. Ejecuta la página (▶) y en el reporte en tiempo de ejecución, usa el menú **Actions** para:
   - Agregar una agrupación por `Departamento` (Actions → Format → Control Break → Departamento).
   - Agregar un cálculo de columna: suma del `Salario` (Actions → Data → Aggregate → Sum sobre Salario).
   - Guardar esta vista como reporte primario: Actions → Report → Save Report → `Vista por Departamento` → marcar **Default** → Save.

#### Resultado esperado
El Interactive Report muestra la lista de empleados con agrupación por departamento, suma de salarios por grupo y las opciones de descarga disponibles en el menú Actions. El reporte guardado "Vista por Departamento" aparece como opción en el selector de reportes.

#### Verificación
- Descarga el reporte en formato Excel: Actions → Download → Excel. Verifica que el archivo se descarga correctamente y contiene los datos.
- Verifica que el menú Actions → Report muestra el reporte guardado "Vista por Departamento".
- En el menú Actions → Data, comprueba que la opción Pivot está disponible y permite crear una vista pivote sobre la columna `Rango Salarial`.

---

### Paso 3 — Implementar el Interactive Grid para edición masiva

**Objetivo:** Añadir un Interactive Grid que permita a los usuarios editar múltiples registros de proyectos directamente en la cuadrícula, con validaciones de celda y procesamiento de cambios en lote.

#### Instrucciones

1. En el **Page Designer** de la página 10, añade una segunda región debajo del Interactive Report:
   - Haz clic derecho sobre **Content Body** → **Add Region**.
   - **Title:** `Gestión de Proyectos (Edición Masiva)`
   - **Type:** `Interactive Grid`

2. En **Source → SQL Query**, introduce:

```sql
SELECT
    p.project_id,
    p.project_name,
    p.status,
    p.start_date,
    p.end_date,
    p.budget,
    p.actual_cost,
    p.budget - p.actual_cost           AS presupuesto_restante,
    d.department_name                  AS departamento,
    p.manager_id,
    e.first_name || ' ' || e.last_name AS nombre_gerente
FROM
    projects    p
    JOIN departments d ON p.department_id = d.department_id
    JOIN employees   e ON p.manager_id    = e.employee_id
```

3. En los **Attributes** del Interactive Grid, configura:
   - **Editable:** `Yes`
   - **Edit Mode:** `Row` *(permite editar fila completa al hacer doble clic)*
   - **Add Row Allowed:** `Yes`
   - **Delete Allowed:** `Yes`
   - **Save Button Label:** `Guardar Cambios`

4. En **Source**, establece:
   - **Primary Key Column 1:** `PROJECT_ID`
   - **Allowed Operations:** `Add Row`, `Update Row`, `Delete Row`

5. Configura las columnas editables. En el panel izquierdo, expande la región del IG y configura cada columna:

   | Columna              | Editable | Tipo de ítem        | Obligatorio |
   |----------------------|----------|---------------------|-------------|
   | PROJECT_ID           | No       | —                   | —           |
   | PROJECT_NAME         | Sí       | Text Field          | Sí          |
   | STATUS               | Sí       | Select List         | Sí          |
   | START_DATE           | Sí       | Date Picker         | Sí          |
   | END_DATE             | Sí       | Date Picker         | No          |
   | BUDGET               | Sí       | Number Field        | Sí          |
   | ACTUAL_COST          | Sí       | Number Field        | No          |
   | PRESUPUESTO_RESTANTE | No       | —                   | —           |
   | DEPARTAMENTO         | No       | —                   | —           |
   | MANAGER_ID           | No       | —                   | —           |
   | NOMBRE_GERENTE       | No       | —                   | —           |

6. Para la columna **STATUS**, configura la lista de valores. Selecciona la columna en el panel izquierdo y en el panel derecho:
   - **Type:** `Select List`
   - **List of Values Type:** `Static Values`
   - Añade los valores: `Planificado`, `En Progreso`, `Completado`, `Cancelado`

7. Añade una **validación de celda** para la columna `END_DATE`. Haz clic derecho sobre la región del IG → **Add Validation**:
   - **Name:** `VAL_FECHA_FIN`
   - **Type:** `PL/SQL Expression`
   - **PL/SQL Expression:**
     ```sql
     :END_DATE IS NULL OR :END_DATE >= :START_DATE
     ```
   - **Error Message:** `La fecha de fin no puede ser anterior a la fecha de inicio.`
   - **Column:** `END_DATE`

8. Añade una segunda validación para el presupuesto:
   - **Name:** `VAL_PRESUPUESTO`
   - **Type:** `PL/SQL Expression`
   - **PL/SQL Expression:**
     ```sql
     :BUDGET > 0
     ```
   - **Error Message:** `El presupuesto debe ser mayor a cero.`
   - **Column:** `BUDGET`

9. Haz clic en **Save** y ejecuta la página.

#### Resultado esperado
El Interactive Grid muestra la lista de proyectos. Al hacer doble clic en una fila, las columnas configuradas como editables se vuelven interactivas. Al intentar guardar con una fecha de fin anterior a la de inicio, aparece el mensaje de error de validación directamente en la celda correspondiente.

#### Verificación
- Edita el nombre de tres proyectos simultáneamente y haz clic en **Guardar Cambios**. Verifica en la base de datos que los tres cambios se persistieron:
  ```sql
  SELECT project_id, project_name, status
  FROM   projects
  WHERE  project_id IN (1, 2, 3); -- Ajusta los IDs según tus datos
  ```
- Intenta introducir una fecha de fin anterior a la de inicio y confirma que la validación impide el guardado.

---

### Paso 4 — Construir las regiones de métricas KPI

**Objetivo:** Añadir una banda de métricas KPI en la parte superior del dashboard usando regiones de tipo **Cards** configuradas como indicadores de valor único.

#### Instrucciones

1. En el **Page Designer**, añade una nueva región **encima** del Interactive Report (arrástrala en el panel izquierdo hasta la posición superior de Content Body):
   - **Title:** `Métricas Clave`
   - **Type:** `Cards`
   - **Template:** `Cards` (selecciona la plantilla que muestra tarjetas horizontales)

2. En **Source → SQL Query**, introduce:

```sql
SELECT
    'Total Empleados'                           AS card_title,
    TO_CHAR(COUNT(*))                           AS card_value,
    'fa-users'                                  AS card_icon,
    'u-color-1'                                 AS card_color,
    'Plantilla activa'                          AS card_subtitle
FROM employees
UNION ALL
SELECT
    'Salario Promedio',
    TO_CHAR(ROUND(AVG(salary), 2), 'FM$999,999.00'),
    'fa-dollar',
    'u-color-2',
    'Promedio global'
FROM employees
UNION ALL
SELECT
    'Proyectos Activos',
    TO_CHAR(COUNT(*)),
    'fa-briefcase',
    'u-color-3',
    'En progreso'
FROM projects
WHERE status = 'En Progreso'
UNION ALL
SELECT
    'Departamentos',
    TO_CHAR(COUNT(*)),
    'fa-building',
    'u-color-4',
    'Unidades organizativas'
FROM departments
```

3. En los **Attributes** de la región Cards, configura:
   - **Title Column:** `CARD_TITLE`
   - **Body Column:** `CARD_VALUE`
   - **Icon Source:** `Icon Class Column`
   - **Icon Class Column:** `CARD_ICON`
   - **Subtitle Column:** `CARD_SUBTITLE`
   - **Card Color Column:** `CARD_COLOR`
   - **Layout:** `Horizontal (Grid)` con **Columns:** `4`

4. En la sección **Appearance** de la región, establece:
   - **Template:** `Cards`
   - **Template Options:** activa `Compact` para reducir el tamaño de las tarjetas.
   - **CSS Classes:** `lab-kpi-cards`

5. Haz clic en **Save**.

#### Resultado esperado
La parte superior del dashboard muestra cuatro tarjetas KPI en una fila horizontal con los valores de total de empleados, salario promedio, proyectos activos y número de departamentos, cada una con su ícono y color distintivo.

#### Verificación
Ejecuta la página y comprueba que los cuatro valores coinciden con los resultados de estas consultas de verificación:
```sql
SELECT COUNT(*) FROM employees;
SELECT ROUND(AVG(salary), 2) FROM employees;
SELECT COUNT(*) FROM projects WHERE status = 'En Progreso';
SELECT COUNT(*) FROM departments;
```

---

### Paso 5 — Crear JET Charts: Gráfica de Barras con serie múltiple

**Objetivo:** Añadir una gráfica de barras JET que muestre la distribución salarial por departamento con dos series (salario promedio y salario máximo), incluyendo eje secundario y tooltips personalizados.

#### Instrucciones

1. Añade una nueva región en **Content Body**:
   - **Title:** `Distribución Salarial por Departamento`
   - **Type:** `Chart`

2. En **Attributes**, configura:
   - **Chart Type:** `Bar`
   - **Orientation:** `Vertical`
   - **Stack:** `No`

3. El asistente crea automáticamente una **Serie 1**. Selecciónala en el panel izquierdo y configura:
   - **Name:** `Salario Promedio`
   - **Source → SQL Query:**
     ```sql
     SELECT
         d.department_name  AS label,
         ROUND(AVG(e.salary), 2) AS value,
         d.department_id    AS series_id
     FROM
         employees   e
         JOIN departments d ON e.department_id = d.department_id
     GROUP BY
         d.department_name,
         d.department_id
     ORDER BY
         AVG(e.salary) DESC
     ```
   - **Label Column:** `LABEL`
   - **Value Column:** `VALUE`
   - **Color:** `#0572CE` (azul Oracle)
   - **Assigned To Y2 Axis:** `No`

4. Añade una segunda serie. Haz clic derecho sobre la región → **Add Series**:
   - **Name:** `Salario Máximo`
   - **Source → SQL Query:**
     ```sql
     SELECT
         d.department_name  AS label,
         MAX(e.salary)      AS value,
         d.department_id    AS series_id
     FROM
         employees   e
         JOIN departments d ON e.department_id = d.department_id
     GROUP BY
         d.department_name,
         d.department_id
     ORDER BY
         MAX(e.salary) DESC
     ```
   - **Label Column:** `LABEL`
   - **Value Column:** `VALUE`
   - **Color:** `#E95B0C` (naranja Oracle)
   - **Assigned To Y2 Axis:** `Yes` *(esto crea el eje secundario)*

5. Configura los ejes. En el panel izquierdo, selecciona **Axes → Y Axis**:
   - **Title:** `Salario Promedio (USD)`
   - **Format:** `$#,##0`
   - **Min Value:** `0`

6. Selecciona **Axes → Y2 Axis**:
   - **Title:** `Salario Máximo (USD)`
   - **Format:** `$#,##0`
   - **Opposite:** `Yes`

7. Configura el **Tooltip** personalizado. En los Attributes de la región Chart:
   - **Tooltip:** `Custom`
   - **Tooltip HTML Expression:**
     ```html
     <strong>{SERIES_NAME}</strong><br>
     Departamento: {LABEL}<br>
     Valor: {VALUE}
     ```

8. En la sección **Legend**:
   - **Show Legend:** `Yes`
   - **Position:** `Bottom`

9. Haz clic en **Save**.

#### Resultado esperado
La gráfica de barras muestra dos series de barras agrupadas por departamento. El eje izquierdo corresponde al salario promedio y el eje derecho al salario máximo. Al pasar el cursor sobre una barra, aparece el tooltip personalizado con el nombre de la serie, el departamento y el valor.

#### Verificación
Compara visualmente los valores del gráfico con esta consulta:
```sql
SELECT
    d.department_name,
    ROUND(AVG(e.salary), 2) AS salario_promedio,
    MAX(e.salary)           AS salario_maximo
FROM
    employees   e
    JOIN departments d ON e.department_id = d.department_id
GROUP BY
    d.department_name
ORDER BY
    AVG(e.salary) DESC;
```

---

### Paso 6 — Crear JET Charts: Gráfica de Líneas de tendencia y Pie/Donut

**Objetivo:** Añadir dos gráficas adicionales: una de líneas para mostrar la tendencia de contrataciones por año y una de tipo Donut para mostrar la distribución de empleados por rango salarial.

#### Instrucciones

**Gráfica de Líneas — Tendencia de Contrataciones**

1. Añade una nueva región en Content Body (posiciónala en una columna diferente usando el layout de cuadrícula; ajusta la **Grid Column** a `1` y el **Column Span** a `6` para ocupar la mitad del ancho):
   - **Title:** `Tendencia de Contrataciones por Año`
   - **Type:** `Chart`

2. Configura la serie:
   - **Name:** `Contrataciones`
   - **Chart Type:** `Line`
   - **Source → SQL Query:**
     ```sql
     SELECT
         TO_CHAR(hire_date, 'YYYY') AS anio,
         COUNT(*)                   AS total_contrataciones
     FROM
         employees
     GROUP BY
         TO_CHAR(hire_date, 'YYYY')
     ORDER BY
         TO_CHAR(hire_date, 'YYYY')
     ```
   - **Label Column:** `ANIO`
   - **Value Column:** `TOTAL_CONTRATACIONES`
   - **Color:** `#00B0CA`
   - **Line Type:** `Curved`
   - **Marker Displayed:** `Yes`
   - **Marker Shape:** `Circle`

3. Configura el eje Y:
   - **Title:** `Número de Contrataciones`
   - **Min Value:** `0`
   - **Format:** `#,##0`

4. Configura el eje X:
   - **Title:** `Año`

**Gráfica Donut — Distribución por Rango Salarial**

5. Añade otra región (en la segunda columna, **Grid Column** `7`, **Column Span** `6`):
   - **Title:** `Distribución por Rango Salarial`
   - **Type:** `Chart`

6. Configura la serie:
   - **Name:** `Empleados`
   - **Chart Type:** `Donut`
   - **Source → SQL Query:**
     ```sql
     SELECT
         CASE
             WHEN salary < 5000  THEN 'Bajo (< $5,000)'
             WHEN salary < 10000 THEN 'Medio ($5,000–$9,999)'
             WHEN salary < 15000 THEN 'Alto ($10,000–$14,999)'
             ELSE 'Ejecutivo ($15,000+)'
         END AS rango,
         COUNT(*) AS total
     FROM
         employees
     GROUP BY
         CASE
             WHEN salary < 5000  THEN 'Bajo (< $5,000)'
             WHEN salary < 10000 THEN 'Medio ($5,000–$9,999)'
             WHEN salary < 15000 THEN 'Alto ($10,000–$14,999)'
             ELSE 'Ejecutivo ($15,000+)'
         END
     ORDER BY
         MIN(salary)
     ```
   - **Label Column:** `RANGO`
   - **Value Column:** `TOTAL`

7. En los **Attributes** del Chart Donut:
   - **Show Legend:** `Yes`
   - **Legend Position:** `Right`
   - **Center Label:** `Custom`
   - **Center Label Text:** `Empleados`

8. Configura una **Acción Dinámica** para refrescar ambas gráficas cuando cambie un filtro global (esto se completará en el Paso 8). Por ahora, haz clic en **Save**.

#### Resultado esperado
La página muestra dos gráficas adicionales: la de líneas con la evolución de contrataciones año a año (con puntos marcadores en cada año), y la Donut con los cuatro segmentos de rango salarial, leyenda a la derecha y etiqueta central.

#### Verificación
Ejecuta la página y verifica que:
- La gráfica de líneas muestra al menos 5 puntos de datos (años distintos).
- La gráfica Donut tiene 4 segmentos y el total de todos los segmentos coincide con `SELECT COUNT(*) FROM employees`.

---

### Paso 7 — Añadir la región de Calendar

**Objetivo:** Implementar una región de Calendar que muestre los proyectos por fecha de inicio, permitiendo visualizar la distribución temporal de los proyectos en el mes.

#### Instrucciones

1. Añade una nueva región en Content Body:
   - **Title:** `Calendario de Proyectos`
   - **Type:** `Calendar`

2. En **Source → SQL Query**, introduce:

```sql
SELECT
    p.project_id,
    p.project_name                             AS event_name,
    p.start_date                               AS start_date,
    NVL(p.end_date, p.start_date + 30)         AS end_date,
    CASE p.status
        WHEN 'En Progreso' THEN 'apex-cal-green'
        WHEN 'Completado'  THEN 'apex-cal-blue'
        WHEN 'Cancelado'   THEN 'apex-cal-red'
        ELSE                    'apex-cal-orange'
    END                                        AS css_class,
    'Gerente: ' || e.first_name || ' '
        || e.last_name                         AS tooltip_text
FROM
    projects  p
    JOIN employees e ON p.manager_id = e.employee_id
WHERE
    p.start_date IS NOT NULL
```

3. En los **Attributes** del Calendar, configura:
   - **Start Date Column:** `START_DATE`
   - **End Date Column:** `END_DATE`
   - **Display Column:** `EVENT_NAME`
   - **CSS Class Column:** `CSS_CLASS`
   - **Tooltip Column:** `TOOLTIP_TEXT`

4. En la sección **Settings**:
   - **View:** `Month` (vista mensual por defecto)
   - **Display Navigation:** `Yes`
   - **Show Weekend:** `Yes`
   - **Show Time:** `No`

5. Para añadir un **link** al hacer clic en un evento del calendario, en la sección **Link**:
   - **Target:** `URL`
   - **URL:** `f?p=&APP_ID.:20:&SESSION.::NO::P20_PROJECT_ID:#PROJECT_ID#`
   
   > 📝 Reemplaza `20` con el número de la página de detalle de proyectos de tu aplicación (del Lab 04).

6. Haz clic en **Save**.

#### Resultado esperado
La región del calendario muestra los proyectos distribuidos en el mes, con colores diferentes según el estado (verde = en progreso, azul = completado, rojo = cancelado, naranja = planificado). Al pasar el cursor sobre un evento, aparece el tooltip con el nombre del gerente.

#### Verificación
- Navega al mes donde tengas más proyectos registrados usando los controles de navegación del calendario.
- Confirma que los colores corresponden correctamente a los estados haciendo clic en un proyecto "En Progreso" (debe ser verde) y verificando su estado en la base de datos.

---

### Paso 8 — Configurar actualización dinámica de gráficas con Dynamic Actions

**Objetivo:** Implementar una acción dinámica que refresque todas las gráficas del dashboard cuando el usuario seleccione un departamento específico desde un elemento de filtro global, demostrando la actualización dinámica mediante refresh.

#### Instrucciones

1. Añade un **ítem de página** en la parte superior del Content Body (antes de las KPI cards). Haz clic derecho sobre la región de Métricas Clave → **Add Page Item**:
   - **Name:** `P10_DEPARTAMENTO_FILTRO`
   - **Type:** `Select List`
   - **Label:** `Filtrar por Departamento:`
   - **List of Values Type:** `SQL Query`
   - **SQL Query para LOV:**
     ```sql
     SELECT department_name AS display_value,
            department_id   AS return_value
     FROM   departments
     ORDER BY department_name
     ```
   - **Display Extra Values:** `Yes`
   - **Null Display Value:** `-- Todos los Departamentos --`
   - **Null Return Value:** `%`

2. Modifica la consulta SQL de la **gráfica de barras** (Paso 5) para incluir el filtro. Selecciona la Serie 1 de la gráfica de barras y actualiza la consulta:

```sql
SELECT
    d.department_name           AS label,
    ROUND(AVG(e.salary), 2)     AS value,
    d.department_id             AS series_id
FROM
    employees   e
    JOIN departments d ON e.department_id = d.department_id
WHERE
    d.department_id = NVL(TO_NUMBER(:P10_DEPARTAMENTO_FILTRO),
                          d.department_id)
    OR :P10_DEPARTAMENTO_FILTRO = '%'
GROUP BY
    d.department_name,
    d.department_id
ORDER BY
    AVG(e.salary) DESC
```

> Aplica el mismo filtro `WHERE` a la Serie 2 (Salario Máximo) de la misma gráfica.

3. Añade la misma cláusula `WHERE` a la consulta de la gráfica Donut:

```sql
SELECT
    CASE
        WHEN salary < 5000  THEN 'Bajo (< $5,000)'
        WHEN salary < 10000 THEN 'Medio ($5,000–$9,999)'
        WHEN salary < 15000 THEN 'Alto ($10,000–$14,999)'
        ELSE 'Ejecutivo ($15,000+)'
    END AS rango,
    COUNT(*) AS total
FROM
    employees
WHERE
    department_id = NVL(TO_NUMBER(:P10_DEPARTAMENTO_FILTRO),
                        department_id)
    OR :P10_DEPARTAMENTO_FILTRO = '%'
GROUP BY
    CASE
        WHEN salary < 5000  THEN 'Bajo (< $5,000)'
        WHEN salary < 10000 THEN 'Medio ($5,000–$9,999)'
        WHEN salary < 15000 THEN 'Alto ($10,000–$14,999)'
        ELSE 'Ejecutivo ($15,000+)'
    END
ORDER BY
    MIN(salary)
```

4. Crea la **Acción Dinámica**. Haz clic derecho sobre el ítem `P10_DEPARTAMENTO_FILTRO` en el panel izquierdo → **Create Dynamic Action**:
   - **Name:** `DA_REFRESCAR_GRAFICAS`
   - **Event:** `Change`
   - **Selection Type:** `Item(s)`
   - **Item(s):** `P10_DEPARTAMENTO_FILTRO`

5. En la acción **True** de la Dynamic Action, configura la primera acción:
   - **Action:** `Refresh`
   - **Selection Type:** `Region`
   - **Region:** `Distribución Salarial por Departamento`

6. Añade una segunda acción True (clic en **+ Add Action**):
   - **Action:** `Refresh`
   - **Selection Type:** `Region`
   - **Region:** `Distribución por Rango Salarial`

7. Añade una tercera acción True:
   - **Action:** `Refresh`
   - **Selection Type:** `Region`
   - **Region:** `Métricas Clave`

8. Haz clic en **Save**.

#### Resultado esperado
Al cambiar el valor del selector de departamento, las tres regiones (gráfica de barras, gráfica donut y tarjetas KPI) se actualizan automáticamente sin recargar la página completa, mostrando los datos filtrados para el departamento seleccionado.

#### Verificación
- Selecciona un departamento específico y verifica que la gráfica de barras muestra solo ese departamento.
- Selecciona "-- Todos los Departamentos --" y confirma que vuelven a aparecer todos los departamentos.
- Abre las DevTools del navegador (F12 → Network) y confirma que al cambiar el filtro se realizan llamadas XHR/Fetch al servidor (indicando que es una actualización AJAX, no una recarga completa de página).

---

### Paso 9 — Auditoría y corrección de accesibilidad con Google Lighthouse

**Objetivo:** Auditar la accesibilidad del dashboard con Google Lighthouse e implementar al menos tres correcciones para los problemas identificados.

#### Instrucciones

**Ejecutar la auditoría inicial**

1. Con el dashboard ejecutivo abierto en Chrome (en modo de ejecución, no en el Page Designer), presiona **F12** para abrir las DevTools.
2. Navega a la pestaña **Lighthouse** (si no está visible, haz clic en `>>` para ver más pestañas).
3. Configura la auditoría:
   - **Mode:** `Navigation`
   - **Device:** `Desktop`
   - Desmarca todas las categorías excepto **Accessibility**.
4. Haz clic en **Analyze page load** y espera a que complete (puede tomar 30-60 segundos).
5. Revisa el reporte. Anota el **puntaje inicial de accesibilidad** (típicamente entre 60-80 en un dashboard APEX sin ajustes de accesibilidad).
6. Identifica y anota los tres problemas de mayor impacto. Los problemas más comunes en APEX son:
   - Imágenes sin atributo `alt`.
   - Elementos de formulario sin etiqueta `label` asociada.
   - Contraste de color insuficiente en textos sobre fondos de color.
   - Elementos interactivos sin nombre accesible.

**Corrección 1: Añadir etiquetas ARIA a las regiones de gráficas**

7. En el **Page Designer**, selecciona la región de la gráfica de barras `Distribución Salarial por Departamento`.
8. En el panel derecho, navega a **Advanced → Custom Attributes**:
   - Añade el atributo: `aria-label` con valor `Gráfica de barras: distribución salarial promedio y máxima por departamento`
   - Añade el atributo: `role` con valor `img`
9. Repite para la gráfica Donut:
   - `aria-label`: `Gráfica donut: distribución de empleados por rango salarial`
   - `role`: `img`
10. Repite para la gráfica de líneas:
    - `aria-label`: `Gráfica de líneas: tendencia de contrataciones por año`
    - `role`: `img`

**Corrección 2: Mejorar el contraste del selector de filtro**

11. En el **Page Designer**, selecciona el ítem `P10_DEPARTAMENTO_FILTRO`.
12. En **Appearance → CSS Classes**, añade la clase `apex-high-contrast-label`.
13. En la sección **Advanced → Custom Attributes**, añade:
    - `aria-describedby`: `P10_DEPARTAMENTO_FILTRO_HELP`
14. Añade un ítem de tipo **Display Only** o **Hidden** con el ID `P10_DEPARTAMENTO_FILTRO_HELP` y el texto: `Seleccione un departamento para filtrar todos los gráficos del dashboard`.

**Corrección 3: Añadir texto alternativo a los íconos KPI**

15. Selecciona la región de **Métricas Clave** (Cards).
16. En la sección **Attributes → Accessibility**, configura:
    - **Card Title Attribute:** `aria-label`
17. En el **Page Designer**, navega a **Page → JavaScript → Execute when Page Loads** y añade el siguiente fragmento para enriquecer los iconos con texto accesible:

```javascript
// Añadir aria-hidden a íconos decorativos en tarjetas KPI
// y texto visible para lectores de pantalla
document.querySelectorAll('.lab-kpi-cards .fa').forEach(function(icon) {
    icon.setAttribute('aria-hidden', 'true');
    icon.setAttribute('focusable', 'false');
});

// Asegurar que las tarjetas sean navegables por teclado
document.querySelectorAll('.lab-kpi-cards .t-Card').forEach(function(card) {
    if (!card.getAttribute('tabindex')) {
        card.setAttribute('tabindex', '0');
    }
    // Añadir rol de artículo para lectores de pantalla
    card.setAttribute('role', 'article');
});
```

**Corrección 4: Mejorar la accesibilidad del Interactive Grid**

18. Selecciona la región del Interactive Grid en el Page Designer.
19. En **Advanced → Custom Attributes**, añade:
    - `aria-label`: `Tabla editable de gestión de proyectos. Use Tab para navegar entre celdas y Enter para editar.`
20. Para cada columna editable del IG, selecciona la columna y en **Advanced → Custom Attributes** añade:
    - `aria-required`: `true` (solo para las columnas marcadas como obligatorias).

**Re-ejecutar la auditoría**

21. Guarda todos los cambios (**Save**) y ejecuta la página nuevamente.
22. Repite la auditoría de Lighthouse siguiendo los pasos 1-5.
23. Compara el nuevo puntaje con el inicial y documenta las mejoras.

#### Resultado esperado
El puntaje de accesibilidad en Lighthouse mejora al menos **10 puntos** respecto al puntaje inicial. Los tres problemas identificados ya no aparecen como errores críticos en el reporte.

#### Verificación
- El reporte de Lighthouse muestra el puntaje mejorado.
- Usando solo el teclado (Tab, Enter, flechas), es posible navegar por todas las tarjetas KPI y llegar al selector de filtro.
- Con un lector de pantalla (NVDA en Windows o VoiceOver en Mac), al enfocar una gráfica se anuncia la descripción ARIA configurada.

---

## Validación y Pruebas Finales

Antes de dar por concluido el laboratorio, realiza las siguientes verificaciones integrales:

### Lista de verificación del dashboard completo

```sql
-- Verificación 1: Confirmar que el IR tiene al menos un reporte guardado
SELECT
    application_id,
    page_id,
    report_name,
    report_type
FROM
    apex_application_page_ir_rpt
WHERE
    application_id = :APP_ID
    AND page_id    = 10;
```

```sql
-- Verificación 2: Confirmar que los cambios del IG se persisten correctamente
-- (ejecutar DESPUÉS de hacer cambios de prueba en el IG)
SELECT
    project_id,
    project_name,
    status,
    last_updated_by,
    last_update_date
FROM
    projects
WHERE
    last_update_date >= TRUNC(SYSDATE)
ORDER BY
    last_update_date DESC
FETCH FIRST 10 ROWS ONLY;
```

### Prueba funcional del flujo completo

| # | Acción a probar                                                    | Resultado esperado                                              |
|---|--------------------------------------------------------------------|-----------------------------------------------------------------|
| 1 | Cargar la página 10 del dashboard                                  | Todas las regiones cargan sin errores en consola JS             |
| 2 | Cambiar filtro a un departamento específico                        | Las 3 regiones se refrescan vía AJAX                            |
| 3 | En el IR: Actions → Format → Control Break → Departamento          | Los datos se agrupan por departamento con subtotales            |
| 4 | En el IR: Actions → Download → Excel                               | Se descarga un archivo `.xlsx` con los datos                    |
| 5 | En el IG: doble clic en una fila → editar nombre → Guardar Cambios | El cambio se persiste en la BD                                  |
| 6 | En el IG: introducir fecha fin anterior a fecha inicio             | Aparece mensaje de validación en la celda END_DATE              |
| 7 | Pasar cursor sobre barra en gráfica de barras                      | Aparece tooltip con nombre de serie, departamento y valor       |
| 8 | Hacer clic en evento del calendario                                | Navega a la página de detalle del proyecto                      |
| 9 | Auditoría Lighthouse Accessibility                                 | Puntaje ≥ puntaje inicial + 10 puntos                           |
| 10| Navegación completa con teclado (Tab + Enter)                      | Es posible llegar a todos los controles interactivos            |

---

## Solución de Problemas

### Problema 1: Las gráficas JET muestran "No data found" aunque la consulta SQL retorna datos

**Síntoma:** La región de tipo Chart aparece vacía con el mensaje "No data found" o similar, pero al ejecutar la misma consulta SQL en el SQL Workshop de APEX, esta retorna registros correctamente.

**Causa probable:** Las columnas en la consulta SQL del Chart tienen alias que no coinciden exactamente (incluyendo mayúsculas/minúsculas) con los valores configurados en los campos **Label Column** y **Value Column** de la serie. APEX es sensible a las mayúsculas en los alias de columna cuando los mapea a las propiedades del gráfico. Otra causa frecuente es que el ítem de sesión referenciado en el `WHERE` (por ejemplo, `:P10_DEPARTAMENTO_FILTRO`) tenga un valor `NULL` en lugar del valor comodín `'%'`, lo que hace que la condición `NVL(TO_NUMBER(:P10_DEPARTAMENTO_FILTRO), ...)` falle si el ítem no está inicializado.

**Solución:**
1. Verifica que los alias de columna en la consulta SQL estén en **MAYÚSCULAS** (p. ej., `AS LABEL`, `AS VALUE`) y que coincidan exactamente con los valores en Label Column y Value Column.
2. Inicializa el ítem `P10_DEPARTAMENTO_FILTRO` con el valor `%` en el proceso **Before Header** de la página:
   - En el Page Designer, navega a **Processing → Pre-Rendering → Before Header**.
   - Añade un proceso de tipo **Set Value**:
     - **Item:** `P10_DEPARTAMENTO_FILTRO`
     - **Value Type:** `Static Value`
     - **Value:** `%`
     - **When button pressed:** *(vacío, siempre)*
     - **Condition:** `Item is NULL` sobre `P10_DEPARTAMENTO_FILTRO`
3. Guarda, ejecuta y verifica que las gráficas muestran datos.

---

### Problema 2: El Interactive Grid no guarda los cambios y muestra el error "ORA-02291: integrity constraint violated"

**Síntoma:** Al editar filas en el Interactive Grid y hacer clic en "Guardar Cambios", aparece un error rojo en la parte superior de la región con el mensaje `ORA-02291: integrity constraint (SCHEMA.FK_PROJECTS_DEPT) violated - parent key not found` o similar.

**Causa probable:** El Interactive Grid está intentando hacer un UPDATE directo sobre la tabla `PROJECTS` con un valor de `DEPARTMENT_ID` que no existe en la tabla `DEPARTMENTS`, o está intentando actualizar una columna que es clave foránea con un valor inválido. Esto ocurre frecuentemente cuando la columna `DEPARTMENT_ID` está incluida en la fuente del IG pero no está configurada correctamente como no editable, o cuando los datos de muestra tienen inconsistencias referencial.

**Solución:**
1. En el Page Designer, selecciona la región del Interactive Grid y verifica que las columnas de clave foránea (`DEPARTMENT_ID`, `MANAGER_ID`) están configuradas como **Editable: No** en sus propiedades de columna.
2. Si necesitas permitir la edición del departamento, usa un **Select List** con una LOV que retorne solo los `DEPARTMENT_ID` válidos:
   ```sql
   -- LOV para columna DEPARTMENT_ID en el IG
   SELECT department_name AS d, department_id AS r
   FROM   departments
   ORDER BY department_name
   ```
3. Verifica la integridad de los datos de muestra:
   ```sql
   -- Detectar proyectos con department_id inválido
   SELECT p.project_id, p.project_name, p.department_id
   FROM   projects p
   WHERE  NOT EXISTS (
       SELECT 1 FROM departments d
       WHERE  d.department_id = p.department_id
   );
   ```
4. Si la consulta anterior retorna registros, corrígelos antes de continuar:
   ```sql
   -- Asignar un departamento válido a proyectos huérfanos
   UPDATE projects
   SET    department_id = (SELECT MIN(department_id) FROM departments)
   WHERE  department_id NOT IN (SELECT department_id FROM departments);
   COMMIT;
   ```

---

## Limpieza del Entorno

Al finalizar el laboratorio, realiza las siguientes acciones para mantener el entorno ordenado:

1. **Guardar la aplicación como exportación de respaldo:**
   - En el App Builder, selecciona tu aplicación.
   - Ve a **Export/Import → Export Application**.
   - Descarga el archivo `.sql` y guárdalo con el nombre `LAB05_BACKUP_[TU_NOMBRE].sql`.

2. **Eliminar reportes guardados de prueba innecesarios** (si creaste reportes de prueba adicionales durante el laboratorio):
   - Ejecuta la página 10, ve al Interactive Report.
   - Actions → Report → **Manage** → elimina los reportes de prueba que no sean el reporte principal "Vista por Departamento".

3. **Revertir datos de prueba del Interactive Grid** (si modificaste registros de proyectos durante las pruebas):
   ```sql
   -- Revertir cambios de prueba si es necesario
   -- (Solo ejecutar si el instructor lo indica)
   -- ROLLBACK; -- Solo funciona si no hiciste COMMIT
   
   -- Si ya se hizo COMMIT, restaura desde el backup del instructor
   -- o usa Flashback Query si está disponible:
   SELECT * FROM projects AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR)
   WHERE  project_id IN (1, 2, 3); -- Ajusta IDs según tus pruebas
   ```

4. **Cerrar sesiones activas** en el workspace de APEX si estás en un entorno compartido.

---

## Resumen

En este laboratorio implementaste un módulo completo de visualización y análisis de datos en Oracle APEX 23.2. Los logros principales fueron:

| Componente                  | Lo que construiste                                                                 |
|-----------------------------|------------------------------------------------------------------------------------|
| **Interactive Report**      | Reporte con agrupaciones, cálculos, pivote, descarga multi-formato y vistas guardadas |
| **Interactive Grid**        | Cuadrícula editable con validaciones de celda y DML declarativo en lote           |
| **Métricas KPI**            | Cuatro tarjetas de valor único con íconos y colores diferenciados                 |
| **JET Chart — Barras**      | Gráfica con dos series, eje secundario y tooltips personalizados                  |
| **JET Chart — Líneas**      | Tendencia temporal con marcadores y escala automática                             |
| **JET Chart — Donut**       | Distribución porcentual con etiqueta central y leyenda                            |
| **Calendar**                | Visualización temporal de proyectos con colores por estado y links de detalle     |
| **Dynamic Actions**         | Actualización AJAX de múltiples regiones desde un filtro global                   |
| **Accesibilidad**           | Etiquetas ARIA, navegación por teclado y mejora de puntaje Lighthouse             |

### Conceptos clave aplicados

- Las regiones en APEX son los bloques fundamentales de construcción: cada tipo tiene un propósito específico y la elección correcta impacta directamente la experiencia del usuario.
- El **Interactive Report** y el **Interactive Grid** difieren fundamentalmente en su propósito: el primero es para análisis y exploración de datos, el segundo para edición directa en tabla.
- Los **JET Charts** de Oracle permiten visualizaciones avanzadas completamente declarativas, con soporte para series múltiples, ejes secundarios y tooltips personalizados.
- Las **Dynamic Actions** con acción `Refresh` permiten construir dashboards reactivos sin escribir JavaScript complejo.
- La accesibilidad no es opcional: los atributos ARIA, el contraste adecuado y la navegación por teclado son requisitos fundamentales en aplicaciones empresariales.

### Recursos adicionales

- [Oracle APEX 23.2 — Interactive Reports Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/managing-interactive-reports.html)
- [Oracle APEX 23.2 — Interactive Grids Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/using-interactive-grids-in-your-application.html)
- [Oracle JET Chart Types Reference](https://www.oracle.com/webfolder/technetwork/jet/jetCookbook.html)
- [Google Lighthouse Accessibility Audit Guide](https://developer.chrome.com/docs/lighthouse/accessibility/)
- [WCAG 2.1 Quick Reference](https://www.w3.org/WAI/WCAG21/quickref/)
- **Siguiente laboratorio:** Lab 06-00-01 — Formularios dinámicos con validaciones avanzadas y lógica de negocio PL/SQL.

---
*Laboratorio 05-00-01 — Curso Oracle APEX Nivel Intermedio — Versión 1.0*
