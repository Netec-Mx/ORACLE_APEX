# Práctica 2: Diagnóstico de rendimiento básico

## Metadatos del Laboratorio

| Atributo         | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 44 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Crear (Create)                             |
| **Laboratorio**  | 02-00-01                                   |
| **Módulo**       | 2 — Arquitectura, Ciclo de Vida y Rendimiento |

---

## Descripción General

En este laboratorio explorarás la arquitectura interna de Oracle APEX desde la perspectiva del rendimiento. Partirás de una aplicación deliberadamente construida con problemas comunes —consultas SQL sin índices, regiones que cargan datos innecesarios y páginas sin paginación— y utilizarás las herramientas de diagnóstico integradas de APEX (Debug Mode, Activity Log, APEX Monitor) junto con las DevTools del navegador para localizar y cuantificar cada cuello de botella. Al finalizar, habrás aplicado correcciones concretas y medido su impacto, desarrollando un flujo de trabajo sistemático de diagnóstico que podrás reutilizar en cualquier proyecto APEX.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Activar el modo Debug de APEX e interpretar el log de ejecución para identificar las operaciones más costosas en tiempo dentro del ciclo de vida de una página.
- [ ] Utilizar el Activity Log y el APEX Monitor para analizar patrones de carga y tiempos de respuesta históricos de la aplicación.
- [ ] Identificar y corregir al menos tres problemas de rendimiento: consulta SQL ineficiente, región sin caché y reporte sin paginación.
- [ ] Aplicar `EXPLAIN PLAN` en Oracle SQL Developer para validar el uso de índices antes y después de la optimización.
- [ ] Medir el impacto de cada corrección utilizando la pestaña Network de las DevTools del navegador y comparar los tiempos de carga.

---

## Prerrequisitos

### Conocimientos requeridos

- Haber completado el Laboratorio 01-00-01 o tener experiencia básica navegando el APEX App Builder.
- Conocimientos de SQL: escritura de consultas `SELECT`, cláusulas `WHERE`, `JOIN` y uso básico de `EXPLAIN PLAN`.
- Familiaridad con las DevTools del navegador: saber abrir la pestaña **Network/Red** y leer tiempos de carga de solicitudes.
- Comprensión del modelo de tres capas de APEX (navegador → ORDS → APEX Engine) estudiado en la Lección 2.1.

### Acceso requerido

- Workspace APEX activo con permisos de desarrollador.
- Esquema de base de datos asociado al workspace con las tablas de muestra cargadas (mínimo 1 000 registros en la tabla `LAB_PEDIDOS`).
- Acceso a Oracle SQL Developer conectado al mismo esquema.
- Navegador Google Chrome 110+ (recomendado para este laboratorio por sus DevTools).
- *(Opcional)* Acceso a APEX Administration Services para visualizar el Activity Log global; si el entorno es `apex.oracle.com`, el instructor proveerá capturas de pantalla de referencia.

---

## Entorno del Laboratorio

### Hardware mínimo recomendado

| Componente       | Mínimo                         | Recomendado                    |
|------------------|--------------------------------|--------------------------------|
| RAM              | 8 GB                           | 16 GB                          |
| Almacenamiento   | 50 GB libres (SSD)             | 100 GB SSD                     |
| Procesador       | Intel Core i5 8ª gen / Ryzen 5 | Intel Core i7 / Ryzen 7        |
| Resolución       | 1280 × 768                     | 1920 × 1080                    |

### Software requerido

| Software                  | Versión mínima | Propósito en este laboratorio              |
|---------------------------|----------------|--------------------------------------------|
| Oracle APEX               | 23.2           | Plataforma principal                       |
| Oracle Database           | 19c / 21c XE   | Motor de base de datos                     |
| ORDS                      | 23.2           | Listener HTTP                              |
| Oracle SQL Developer      | 23.1           | EXPLAIN PLAN y análisis de consultas       |
| Google Chrome             | 110+           | DevTools Network para medir tiempos        |
| Visual Studio Code        | 1.80+          | Edición de scripts SQL (opcional)          |

### Preparación del entorno — Scripts de configuración

Antes de comenzar los pasos del laboratorio, el instructor o el estudiante debe ejecutar el siguiente script en el esquema asociado al workspace. Este script crea las tablas de muestra con problemas de rendimiento intencionales.

```sql
-- ============================================================
-- SCRIPT DE PREPARACIÓN: Lab 02-00-01
-- Ejecutar como el usuario del esquema del workspace APEX
-- ============================================================

-- 1. Tabla principal de pedidos (sin índices intencionales)
CREATE TABLE lab_pedidos (
    pedido_id     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_nom   VARCHAR2(100)  NOT NULL,
    producto_cod  VARCHAR2(20)   NOT NULL,
    cantidad      NUMBER(10,2)   NOT NULL,
    estado        VARCHAR2(20)   DEFAULT 'PENDIENTE',
    fecha_pedido  DATE           DEFAULT SYSDATE,
    region        VARCHAR2(50),
    vendedor_id   NUMBER
);

-- 2. Tabla de detalle de pedidos
CREATE TABLE lab_detalle_pedido (
    detalle_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    pedido_id     NUMBER         NOT NULL,
    linea_num     NUMBER         NOT NULL,
    descripcion   VARCHAR2(200),
    precio_unit   NUMBER(12,2),
    CONSTRAINT fk_detalle_ped FOREIGN KEY (pedido_id)
        REFERENCES lab_pedidos(pedido_id)
);

-- 3. Insertar 2000 registros de muestra en lab_pedidos
BEGIN
    FOR i IN 1..2000 LOOP
        INSERT INTO lab_pedidos (
            cliente_nom, producto_cod, cantidad, estado,
            fecha_pedido, region, vendedor_id
        ) VALUES (
            'Cliente_' || LPAD(i, 4, '0'),
            'PROD-' || MOD(i, 50),
            ROUND(DBMS_RANDOM.VALUE(1, 500), 2),
            CASE MOD(i, 4)
                WHEN 0 THEN 'PENDIENTE'
                WHEN 1 THEN 'PROCESADO'
                WHEN 2 THEN 'ENVIADO'
                ELSE 'CANCELADO'
            END,
            SYSDATE - DBMS_RANDOM.VALUE(0, 365),
            CASE MOD(i, 5)
                WHEN 0 THEN 'NORTE'
                WHEN 1 THEN 'SUR'
                WHEN 2 THEN 'ESTE'
                WHEN 3 THEN 'OESTE'
                ELSE 'CENTRO'
            END,
            MOD(i, 20) + 1
        );
    END LOOP;
    COMMIT;
END;
/

-- 4. Insertar registros en detalle (3 líneas por pedido aprox.)
BEGIN
    FOR i IN 1..6000 LOOP
        INSERT INTO lab_detalle_pedido (pedido_id, linea_num, descripcion, precio_unit)
        SELECT pedido_id,
               MOD(i, 3) + 1,
               'Detalle item ' || i,
               ROUND(DBMS_RANDOM.VALUE(10, 1000), 2)
        FROM   lab_pedidos
        WHERE  ROWNUM = 1
        AND    pedido_id = MOD(i, 2000) + 1;
    END LOOP;
    COMMIT;
END;
/

-- 5. Vista con función lenta (simula carga costosa)
CREATE OR REPLACE VIEW lab_v_pedidos_lenta AS
SELECT p.pedido_id,
       p.cliente_nom,
       p.producto_cod,
       p.cantidad,
       p.estado,
       p.fecha_pedido,
       p.region,
       -- Subconsulta correlacionada intencional (costosa)
       (SELECT COUNT(*) FROM lab_detalle_pedido d
        WHERE d.pedido_id = p.pedido_id) AS num_lineas,
       -- Función de cadena costosa aplicada en cada fila
       UPPER(TRIM(p.cliente_nom)) || ' [' || p.region || ']' AS cliente_display
FROM   lab_pedidos p;

COMMIT;
```

> **Nota:** Si el script reporta errores de objeto ya existente (`ORA-00955`), ejecuta primero `DROP TABLE lab_detalle_pedido CASCADE CONSTRAINTS;` y `DROP TABLE lab_pedidos CASCADE CONSTRAINTS;` antes de volver a correrlo.

---

## Pasos del Laboratorio

---

### Paso 1 — Crear la aplicación de prueba con problemas de rendimiento

**Objetivo:** Construir una aplicación APEX mínima que contenga los tres antipatrones de rendimiento que diagnosticaremos: reporte sobre vista lenta, región estática sin caché y reporte sin paginación.

**Instrucciones:**

1. Inicia sesión en tu workspace APEX y navega a **App Builder → Create → New Application**.

2. Configura la aplicación con los siguientes valores:
   - **Name:** `Lab02 - Diagnóstico de Rendimiento`
   - **Appearance:** Universal Theme (Vita)
   - Deja las demás opciones en sus valores predeterminados y haz clic en **Create Application**.

3. Una vez creada la aplicación, abre el **Page Designer** de la página `1` (Home).

4. En la región existente (o crea una nueva de tipo **Classic Report**), configura:
   - **Title:** `Reporte de Pedidos (sin optimizar)`
   - **Type:** Classic Report
   - **SQL Query:**
     ```sql
     SELECT pedido_id,
            cliente_display,
            producto_cod,
            cantidad,
            estado,
            fecha_pedido,
            region,
            num_lineas
     FROM   lab_v_pedidos_lenta
     ORDER BY fecha_pedido DESC
     ```

5. En las propiedades de la región, sección **Attributes**, configura:
   - **Pagination Type:** `No Pagination (Show All Rows)`
   - **Maximum Row Count:** `2000`

   > Esto fuerza que el reporte cargue los 2 000 registros de una sola vez, sin paginación.

6. Agrega una segunda región de tipo **Static Content** con:
   - **Title:** `Información del Sistema`
   - **HTML Content:**
     ```html
     <p><strong>Aplicación:</strong> Lab02 Diagnóstico de Rendimiento</p>
     <p><strong>Versión:</strong> 1.0</p>
     <p><strong>Entorno:</strong> Laboratorio de Práctica</p>
     <p><strong>Fecha de carga:</strong> &APP_DATE_TIME.</p>
     ```
   - **Cache:** asegúrate de que la opción de caché esté **desactivada** (valor por defecto en regiones nuevas).

7. Haz clic en **Save** y luego en **Run Page** para ejecutar la página.

**Resultado esperado:** La página carga lentamente (puede tardar entre 3 y 15 segundos dependiendo del hardware) y muestra los 2 000 registros del reporte sin paginación. La región de contenido estático también aparece en la parte superior.

**Verificación:** Confirma que la URL de la página en ejecución tiene el formato:
```
https://[tu-servidor]/ords/f?p=[APP_ID]:1:[SESSION_ID]
```
Y que el reporte muestra exactamente 2 000 filas en la parte inferior de la tabla.

---

### Paso 2 — Activar el modo Debug y analizar el log de ejecución

**Objetivo:** Utilizar el Debug Mode de APEX para obtener el log detallado de ejecución de la página e identificar las operaciones más costosas en tiempo.

**Instrucciones:**

1. Con la aplicación en ejecución en el navegador, modifica la URL para activar el modo Debug. Localiza el parámetro de debug en la URL y cámbialo:

   **URL sin debug:**
   ```
   https://[servidor]/ords/f?p=[APP_ID]:1:[SESSION]
   ```
   **URL con debug activado:**
   ```
   https://[servidor]/ords/f?p=[APP_ID]:1:[SESSION]:::YES
   ```
   
   Alternativamente, en el **Developer Toolbar** (barra inferior que aparece cuando ejecutas desde App Builder), haz clic en el botón **Debug** → **Enable Debug**.

2. Recarga la página con el debug activo. Observa que la página tarda más en cargar (el modo debug agrega overhead de logging).

3. Una vez que la página termine de cargar, haz clic en el botón **View Debug** en el Developer Toolbar (ícono de documento con lupa) o navega a:
   ```
   App Builder → [Tu Aplicación] → Utilities → Debug Messages
   ```

4. En la pantalla de Debug Messages, selecciona la entrada más reciente (la que acaba de generarse). Haz clic en ella para ver el log detallado.

5. Examina el log y busca las siguientes columnas:
   - **Elapsed**: tiempo acumulado desde el inicio de la solicitud.
   - **Execution**: tiempo de cada operación individual.
   - **Message**: descripción de la operación.

6. Ordena el log por la columna **Execution** de mayor a menor. Identifica y anota en la siguiente tabla las tres operaciones con mayor tiempo de ejecución:

   | # | Mensaje de la operación | Tiempo (ms) |
   |---|------------------------|-------------|
   | 1 | *(anota aquí)*         |             |
   | 2 | *(anota aquí)*         |             |
   | 3 | *(anota aquí)*         |             |

7. Busca específicamente entradas que contengan los textos:
   - `"fetch rows"` o `"execute"` — relacionadas con la consulta SQL del reporte.
   - `"render region"` — tiempo de renderizado de cada región.
   - `"show"` o `"process"` — tiempo total de la solicitud.

**Resultado esperado:** El log muestra decenas de entradas. La operación más costosa debería ser la ejecución de la consulta SQL del reporte (`fetch rows` de `lab_v_pedidos_lenta`), típicamente representando más del 70% del tiempo total. Deberías observar un tiempo total de solicitud (`Elapsed` en la última línea) superior a 3 000 ms.

**Verificación:** Confirma que puedes identificar claramente en el log:
- La línea con `"execute"` o `"fetch rows"` para la región del reporte con el mayor tiempo.
- Al menos una entrada de `"render region"` para cada una de las dos regiones.
- La línea final con el tiempo total `"stop apex_engine"`.

---

### Paso 3 — Medir tiempos de carga con las DevTools del navegador

**Objetivo:** Usar la pestaña Network de Chrome DevTools para medir el tiempo de respuesta HTTP de la solicitud de página y establecer una línea base de rendimiento antes de las optimizaciones.

**Instrucciones:**

1. Abre las **Chrome DevTools** presionando `F12` o `Ctrl+Shift+I` (Windows/Linux) / `Cmd+Option+I` (Mac).

2. Navega a la pestaña **Network** (Red).

3. Asegúrate de que el botón de grabación (círculo rojo) esté activo. Marca la casilla **Disable cache** para evitar que el caché del navegador interfiera con las mediciones.

4. Recarga la página de la aplicación APEX con `Ctrl+R` (o `Cmd+R` en Mac). Espera a que la página cargue completamente.

5. En la lista de solicitudes de red, busca la solicitud principal de la página APEX. Será la primera solicitud de tipo `document` con la URL que contiene `f?p=`.

6. Haz clic sobre esa solicitud y examina la pestaña **Timing** en el panel derecho. Anota los siguientes valores:

   | Métrica                    | Valor (ms) |
   |----------------------------|------------|
   | **TTFB** (Time to First Byte) |          |
   | **Content Download**       |            |
   | **Total Time**             |            |

7. En la barra inferior de DevTools, anota también:
   - **Requests**: número total de solicitudes HTTP.
   - **Transferred**: datos transferidos.
   - **DOMContentLoaded**: tiempo hasta que el DOM está listo.
   - **Load**: tiempo total de carga de la página.

8. Toma una captura de pantalla o anota estos valores. Serán tu **línea base** para comparar después de las optimizaciones.

**Resultado esperado:** El TTFB debería ser alto (>2 000 ms) debido a la consulta costosa en el servidor. El `DOMContentLoaded` será elevado por la cantidad de HTML generado para 2 000 filas.

**Verificación:** Confirma que el TTFB registrado es superior a 1 500 ms. Si es menor, verifica que el caché del navegador esté desactivado (`Disable cache` marcado) y que estés midiendo la solicitud correcta (`f?p=...`).

---

### Paso 4 — Analizar la consulta SQL con EXPLAIN PLAN

**Objetivo:** Usar Oracle SQL Developer para obtener el plan de ejecución de la consulta del reporte e identificar operaciones de Full Table Scan que confirman la ausencia de índices.

**Instrucciones:**

1. Abre **Oracle SQL Developer** y conéctate al esquema del workspace APEX.

2. Abre una nueva hoja de trabajo SQL (`Ctrl+Shift+N`) y escribe la consulta base del reporte:

   ```sql
   EXPLAIN PLAN FOR
   SELECT pedido_id,
          cliente_display,
          producto_cod,
          cantidad,
          estado,
          fecha_pedido,
          region,
          num_lineas
   FROM   lab_v_pedidos_lenta
   ORDER BY fecha_pedido DESC;
   ```

3. Ejecuta el `EXPLAIN PLAN` con `F10` o el botón **Explain Plan** (ícono de árbol) en SQL Developer.

4. Examina el plan de ejecución en la pestaña **Explain Plan**. Busca las siguientes operaciones:
   - `TABLE ACCESS FULL` — indica un escaneo completo de tabla (sin índice).
   - `SORT ORDER BY` — ordenamiento en memoria de todos los registros.
   - Apariciones de `lab_detalle_pedido` con `TABLE ACCESS FULL` — confirma que la subconsulta correlacionada escanea la tabla completa por cada fila padre.

5. Anota el **Cost** total que aparece en la raíz del árbol del plan.

6. Ahora ejecuta también el plan para la subconsulta correlacionada de forma aislada:

   ```sql
   EXPLAIN PLAN FOR
   SELECT COUNT(*)
   FROM   lab_detalle_pedido d
   WHERE  d.pedido_id = 1;
   ```

   Observa que también genera un `TABLE ACCESS FULL` en `lab_detalle_pedido`.

7. Verifica estadísticas de la tabla ejecutando:

   ```sql
   SELECT table_name,
          num_rows,
          last_analyzed
   FROM   user_tables
   WHERE  table_name IN ('LAB_PEDIDOS', 'LAB_DETALLE_PEDIDO');
   ```

   Si `last_analyzed` es `NULL`, recopila estadísticas:

   ```sql
   BEGIN
       DBMS_STATS.GATHER_TABLE_STATS(
           ownname => USER,
           tabname => 'LAB_PEDIDOS'
       );
       DBMS_STATS.GATHER_TABLE_STATS(
           ownname => USER,
           tabname => 'LAB_DETALLE_PEDIDO'
       );
   END;
   /
   ```

**Resultado esperado:** El plan de ejecución muestra `TABLE ACCESS FULL` en `LAB_PEDIDOS` y en `LAB_DETALLE_PEDIDO`. El costo total es alto (típicamente >500 para 2 000 registros con subconsulta correlacionada). La subconsulta correlacionada aparece como un nodo hijo que se ejecuta una vez por cada fila de `LAB_PEDIDOS`.

**Verificación:** Confirma que el texto `TABLE ACCESS FULL` aparece al menos dos veces en el plan de ejecución. Si ves `INDEX RANGE SCAN` o `INDEX UNIQUE SCAN`, significa que ya existen índices y debes verificar que estás conectado al esquema correcto.

---

### Paso 5 — Corrección 1: Optimizar la consulta SQL con índices y reescritura

**Objetivo:** Crear índices adecuados y reescribir la consulta del reporte para eliminar la subconsulta correlacionada, midiendo la mejora en el plan de ejecución.

**Instrucciones:**

1. En SQL Developer, crea los índices necesarios:

   ```sql
   -- Índice para la columna de ordenamiento principal
   CREATE INDEX idx_lab_pedidos_fecha
       ON lab_pedidos(fecha_pedido DESC);

   -- Índice para la subconsulta correlacionada (FK)
   CREATE INDEX idx_lab_detalle_pedido_id
       ON lab_detalle_pedido(pedido_id);

   -- Índice compuesto para filtros frecuentes por estado y región
   CREATE INDEX idx_lab_pedidos_estado_region
       ON lab_pedidos(estado, region);
   ```

2. Crea una nueva vista optimizada que reemplaza la subconsulta correlacionada con un `JOIN` agregado:

   ```sql
   CREATE OR REPLACE VIEW lab_v_pedidos_optimizada AS
   SELECT p.pedido_id,
          UPPER(TRIM(p.cliente_nom)) || ' [' || p.region || ']' AS cliente_display,
          p.producto_cod,
          p.cantidad,
          p.estado,
          p.fecha_pedido,
          p.region,
          NVL(d.num_lineas, 0) AS num_lineas
   FROM   lab_pedidos p
   LEFT JOIN (
       SELECT pedido_id,
              COUNT(*) AS num_lineas
       FROM   lab_detalle_pedido
       GROUP BY pedido_id
   ) d ON p.pedido_id = d.pedido_id;
   ```

3. Verifica el nuevo plan de ejecución:

   ```sql
   EXPLAIN PLAN FOR
   SELECT pedido_id,
          cliente_display,
          producto_cod,
          cantidad,
          estado,
          fecha_pedido,
          region,
          num_lineas
   FROM   lab_v_pedidos_optimizada
   ORDER BY fecha_pedido DESC;
   ```

   Ejecuta `EXPLAIN PLAN` en SQL Developer y confirma que ahora aparecen operaciones de `INDEX RANGE SCAN` o `INDEX FULL SCAN` en lugar de `TABLE ACCESS FULL`.

4. Regresa al **Page Designer** de APEX (página 1 de tu aplicación). Selecciona la región del reporte y actualiza la consulta SQL:

   ```sql
   SELECT pedido_id,
          cliente_display,
          producto_cod,
          cantidad,
          estado,
          fecha_pedido,
          region,
          num_lineas
   FROM   lab_v_pedidos_optimizada
   ORDER BY fecha_pedido DESC
   ```

5. Guarda los cambios con **Save**.

**Resultado esperado:** El nuevo `EXPLAIN PLAN` muestra `INDEX RANGE SCAN` sobre `IDX_LAB_PEDIDOS_FECHA` y un `HASH JOIN` o `NESTED LOOPS` para el join con el subquery agregado, en lugar de la subconsulta correlacionada. El costo total del plan debe reducirse al menos un 40% respecto al plan original.

**Verificación:** Ejecuta ambos `EXPLAIN PLAN` (vista lenta y vista optimizada) en paralelo en SQL Developer y compara los costos en la raíz del árbol. Documenta la diferencia porcentual.

---

### Paso 6 — Corrección 2: Configurar paginación en el reporte

**Objetivo:** Activar la paginación en el Classic Report para que solo se carguen y rendericen los registros visibles, reduciendo dramáticamente el tiempo de generación de HTML y la transferencia de datos.

**Instrucciones:**

1. En el **Page Designer**, selecciona la región del reporte `Reporte de Pedidos (sin optimizar)`.

2. En el panel de propiedades de la derecha, localiza la sección **Attributes**.

3. Cambia los siguientes valores:
   - **Pagination Type:** `Row Ranges X to Y (with next and previous links)`
   - **Rows Per Page:** `15`
   - **Maximum Row Count:** `500`

4. Renombra la región a `Reporte de Pedidos (optimizado)` para reflejar el cambio.

5. Guarda los cambios con **Save**.

6. Ejecuta la página (`Run Page`) y verifica que ahora el reporte muestra solo 15 filas con controles de paginación en la parte inferior (`1 - 15` con flechas de navegación).

7. Abre las DevTools de Chrome (pestaña **Network**) con **Disable cache** activo y recarga la página. Anota los nuevos valores:

   | Métrica                    | Valor ANTES (ms) | Valor DESPUÉS (ms) |
   |----------------------------|------------------|--------------------|
   | **TTFB**                   | *(del Paso 3)*   |                    |
   | **Content Download**       | *(del Paso 3)*   |                    |
   | **DOMContentLoaded**       | *(del Paso 3)*   |                    |
   | **Load**                   | *(del Paso 3)*   |                    |

**Resultado esperado:** El tiempo de carga total de la página debe reducirse significativamente. El `Content Download` debería disminuir en más del 90% (de HTML con 2 000 filas a HTML con 15 filas). El `DOMContentLoaded` también debería bajar notablemente.

**Verificación:** Confirma que:
1. El reporte muestra exactamente 15 filas.
2. Existen controles de paginación funcionales (puedes navegar a la página 2).
3. El tiempo `DOMContentLoaded` en DevTools es menor que el registrado en el Paso 3.

---

### Paso 7 — Corrección 3: Activar caché en la región de contenido estático

**Objetivo:** Configurar el caché de región en APEX para la región de contenido estático, evitando que el motor regenere HTML idéntico en cada solicitud del usuario.

**Instrucciones:**

1. En el **Page Designer**, selecciona la región `Información del Sistema` (la de contenido estático).

2. En el panel de propiedades, localiza la sección **Cache**.

3. Configura las siguientes propiedades:
   - **Caching:** `Cached`
   - **Cache By:** `By User` *(o `By Session` si el contenido es igual para todos los usuarios)*
   - **Cache Timeout Seconds:** `3600` (1 hora)

   > **Nota conceptual:** Cuando una región tiene caché activado, APEX almacena el HTML generado en la tabla `APEX_COLLECTIONS` o en la caché de sesión. En solicitudes subsecuentes, en lugar de ejecutar el renderizado completo, devuelve directamente el HTML almacenado. Esto es especialmente efectivo para regiones cuyo contenido no cambia frecuentemente.

4. Guarda los cambios con **Save**.

5. Ejecuta la página por primera vez (la caché se construye). Luego recarga la página una segunda vez.

6. Activa el modo Debug nuevamente (modifica la URL añadiendo `:::YES` al final) y recarga la página.

7. Abre el **View Debug** y busca las entradas relacionadas con la región `Información del Sistema`. En la segunda carga (con caché), deberías ver un mensaje similar a:
   ```
   "region cached, skipping render"
   ```
   o un tiempo de ejecución de `0 ms` o muy cercano a cero para esa región.

**Resultado esperado:** En el log de debug, la región de contenido estático muestra un tiempo de ejecución de `0 ms` o aparece marcada como `"from cache"` en la segunda carga. El tiempo de la primera carga es mayor (construye la caché), pero las cargas subsecuentes son notablemente más rápidas para esa región.

**Verificación:** Compara las entradas del log de debug para la región `Información del Sistema` entre la primera y segunda carga:
- Primera carga: tiempo de renderizado > 0 ms.
- Segunda carga: tiempo de renderizado ≈ 0 ms o mensaje de caché.

---

### Paso 8 — Medición final y comparación de resultados

**Objetivo:** Ejecutar una medición final completa con todas las optimizaciones aplicadas y comparar cuantitativamente el antes y el después utilizando el Debug Mode y las DevTools.

**Instrucciones:**

1. Con todas las correcciones aplicadas (índices, vista optimizada, paginación y caché), ejecuta la página en modo normal (sin debug).

2. Abre las DevTools de Chrome (pestaña **Network**, **Disable cache** activo) y recarga la página tres veces consecutivas. Anota el promedio de los tiempos.

3. Activa el modo Debug y recarga la página una vez más. Abre el log de debug y anota:
   - Tiempo total de la solicitud (última línea del log).
   - Tiempo de la operación `"execute"` o `"fetch rows"` del reporte.
   - Tiempo de la región de contenido estático (debe ser ≈ 0 ms por caché).

4. Completa la tabla de comparación final:

   | Métrica                         | Antes de optimizar | Después de optimizar | Mejora (%) |
   |---------------------------------|--------------------|----------------------|------------|
   | Tiempo total debug (ms)         |                    |                      |            |
   | Tiempo fetch rows reporte (ms)  |                    |                      |            |
   | TTFB navegador (ms)             |                    |                      |            |
   | DOMContentLoaded (ms)           |                    |                      |            |
   | Datos transferidos (KB)         |                    |                      |            |

5. Desactiva el modo Debug modificando la URL (reemplaza `YES` por `NO` en el parámetro de debug, o usa el botón **Disable Debug** en el Developer Toolbar).

**Resultado esperado:** La tabla de comparación debería mostrar mejoras significativas en todas las métricas:
- Tiempo total de debug: reducción esperada del 60–80%.
- Datos transferidos: reducción esperada del 85–95% (de ~500 KB a ~30–50 KB con paginación).
- TTFB: reducción del 50–70% gracias a la consulta optimizada.

**Verificación:** Confirma que al menos tres de las cinco métricas de la tabla muestran una mejora superior al 50%.

---

## Validación y Pruebas

Una vez completados todos los pasos, realiza las siguientes verificaciones de integridad para confirmar que las optimizaciones son correctas y no introdujeron errores funcionales:

### Prueba 1 — Integridad de datos del reporte

```sql
-- Ejecutar en SQL Developer
-- Verificar que ambas vistas devuelven el mismo número de registros
SELECT COUNT(*) AS total_lenta      FROM lab_v_pedidos_lenta;
SELECT COUNT(*) AS total_optimizada FROM lab_v_pedidos_optimizada;

-- Verificar que los num_lineas son consistentes entre vistas
SELECT l.pedido_id,
       l.num_lineas AS lineas_lenta,
       o.num_lineas AS lineas_optimizada,
       CASE WHEN l.num_lineas = o.num_lineas THEN 'OK' ELSE 'DIFERENCIA' END AS resultado
FROM   lab_v_pedidos_lenta      l
JOIN   lab_v_pedidos_optimizada o ON l.pedido_id = o.pedido_id
WHERE  l.num_lineas != o.num_lineas
FETCH FIRST 10 ROWS ONLY;
```

**Resultado esperado:** Ambos `COUNT(*)` devuelven `2000`. La consulta de diferencias no devuelve filas (0 registros).

### Prueba 2 — Verificación de índices creados

```sql
-- Confirmar que los tres índices existen
SELECT index_name,
       table_name,
       status,
       uniqueness
FROM   user_indexes
WHERE  table_name IN ('LAB_PEDIDOS', 'LAB_DETALLE_PEDIDO')
ORDER BY table_name, index_name;
```

**Resultado esperado:** La consulta devuelve al menos 4 filas: la clave primaria de cada tabla más los 3 índices creados en el Paso 5.

### Prueba 3 — Verificación de paginación funcional

En el navegador, con la aplicación en ejecución:
1. Confirma que la primera página del reporte muestra exactamente 15 filas.
2. Haz clic en el control de siguiente página y confirma que carga las filas 16–30.
3. Navega directamente a la última página y confirma que el número de filas es ≤ 15.

### Prueba 4 — Verificación de caché de región

```sql
-- Verificar en la base de datos que la caché de APEX registra la región
-- (requiere acceso al esquema APEX o a vistas de monitoreo del workspace)
SELECT page_id,
       region_name,
       cached_when,
       cache_timeout_secs
FROM   apex_application_page_regions
WHERE  application_id = :APP_ID
AND    caching = 'BY_USER_BY_SESSION';
```

> Si no tienes acceso a `apex_application_page_regions`, verifica visualmente en el Debug Log que la segunda carga de la región muestra tiempo ≈ 0 ms.

---

## Solución de Problemas

### Problema 1 — El modo Debug no genera entradas en el log / la página muestra error al activar debug

**Síntoma:** Al agregar `:::YES` a la URL, la página muestra un error HTTP 404 o redirige al login, o el log de debug aparece vacío después de ejecutar la página.

**Causa probable:** El parámetro de debug se está añadiendo en la posición incorrecta de la URL de APEX. La URL de APEX tiene un formato posicional estricto: `f?p=APP:PAGE:SESSION:REQUEST:DEBUG:CLEAR_CACHE:ITEMS:VALUES:PRINTER_FRIENDLY`. El parámetro de debug está en la posición 5, no al final de la URL. Otra causa común es que la sesión APEX expiró mientras se editaba la URL manualmente.

**Solución:**
1. En lugar de editar la URL manualmente, usa el **Developer Toolbar** que aparece en la parte inferior de la página cuando ejecutas desde App Builder. Haz clic en el botón **Debug** → **Enable Debug**. APEX construirá la URL correcta automáticamente.
2. Si el Developer Toolbar no aparece, verifica que en App Builder → Application Properties → Security, la opción **Allow Debug** esté configurada como `Yes`.
3. Si la sesión expiró, regresa a App Builder, ejecuta la página nuevamente con el botón **Run Page**, y desde ahí activa el debug con el Developer Toolbar.

---

### Problema 2 — El EXPLAIN PLAN sigue mostrando TABLE ACCESS FULL después de crear los índices

**Síntoma:** Después de ejecutar el `CREATE INDEX` en el Paso 5 y volver a ejecutar el `EXPLAIN PLAN`, el plan de ejecución sigue mostrando `TABLE ACCESS FULL` en `LAB_PEDIDOS` o `LAB_DETALLE_PEDIDO`, sin usar los nuevos índices.

**Causa probable:** El optimizador de Oracle (CBO) no tiene estadísticas actualizadas de las tablas y estima que un Full Table Scan es más eficiente que usar el índice (esto ocurre cuando las estadísticas están desactualizadas o son nulas). También puede ocurrir si la consulta no tiene una cláusula `WHERE` selectiva que haga al índice ventajoso (para `ORDER BY` sobre tablas pequeñas, el optimizador puede preferir un Full Scan + Sort).

**Solución:**
1. Recopila estadísticas actualizadas de ambas tablas e índices:
   ```sql
   BEGIN
       DBMS_STATS.GATHER_TABLE_STATS(
           ownname     => USER,
           tabname     => 'LAB_PEDIDOS',
           cascade     => TRUE  -- incluye estadísticas de índices
       );
       DBMS_STATS.GATHER_TABLE_STATS(
           ownname     => USER,
           tabname     => 'LAB_DETALLE_PEDIDO',
           cascade     => TRUE
       );
   END;
   /
   ```
2. Vuelve a ejecutar el `EXPLAIN PLAN`. Con estadísticas actualizadas, el optimizador debería reconocer que el índice `IDX_LAB_PEDIDOS_FECHA` es más eficiente para el `ORDER BY fecha_pedido DESC`.
3. Si el problema persiste para la subconsulta correlacionada (en la vista lenta), confirma que estás analizando la vista `lab_v_pedidos_lenta` y no la optimizada. La vista lenta siempre mostrará `TABLE ACCESS FULL` en `lab_detalle_pedido` porque la subconsulta correlacionada no puede usar el índice de manera eficiente para cada fila individual.

---

## Limpieza del Entorno

Después de completar el laboratorio, ejecuta los siguientes pasos para mantener el entorno ordenado:

### Conservar para laboratorios futuros

La aplicación `Lab02 - Diagnóstico de Rendimiento` y las tablas `LAB_PEDIDOS` / `LAB_DETALLE_PEDIDO` **deben conservarse** si el instructor indica que serán utilizadas en laboratorios posteriores (Laboratorios 04–10 construyen sobre una aplicación base progresiva).

### Limpiar objetos temporales (solo si el instructor lo indica)

```sql
-- Ejecutar SOLO si el instructor confirma que no se necesitarán más
-- Eliminar vistas
DROP VIEW lab_v_pedidos_lenta;
DROP VIEW lab_v_pedidos_optimizada;

-- Eliminar índices (las tablas se conservan)
DROP INDEX idx_lab_pedidos_fecha;
DROP INDEX idx_lab_detalle_pedido_id;
DROP INDEX idx_lab_pedidos_estado_region;
```

### Desactivar el modo Debug

Asegúrate de que el modo Debug esté desactivado en la aplicación antes de finalizar:

1. Ejecuta la aplicación desde App Builder.
2. En el Developer Toolbar, haz clic en **Debug** → **Disable Debug**.
3. Confirma que la URL ya no contiene `YES` en la posición del parámetro de debug.

> **Importante:** Dejar el modo Debug activo en producción es un riesgo de seguridad, ya que expone información interna del motor APEX a usuarios con acceso a la URL.

---

## Resumen

En este laboratorio completaste un ciclo completo de diagnóstico y optimización de rendimiento en Oracle APEX, aplicando directamente los conceptos de la arquitectura de tres capas estudiados en la Lección 2.1:

| Actividad realizada                         | Componente APEX involucrado          | Herramienta utilizada         |
|---------------------------------------------|--------------------------------------|-------------------------------|
| Activación y análisis del Debug Log         | APEX Engine (motor de renderizado)   | APEX Debug Mode / View Debug  |
| Medición de TTFB y tiempos de carga         | ORDS (capa de servidor web)          | Chrome DevTools Network       |
| Análisis de plan de ejecución SQL           | Oracle Database (motor de datos)     | SQL Developer EXPLAIN PLAN    |
| Creación de índices y reescritura de vista  | Oracle Database                      | SQL Developer                 |
| Configuración de paginación                 | APEX Engine (renderizado de región)  | APEX Page Designer            |
| Activación de caché de región               | APEX Engine (caché de sesión)        | APEX Page Designer            |

### Conceptos clave consolidados

- El **APEX Engine** genera HTML dinámicamente en cada solicitud leyendo metadatos del repositorio APEX y ejecutando las consultas SQL de cada región. Las consultas ineficientes impactan directamente el TTFB.
- El **modo Debug** de APEX expone el log interno de ejecución del motor, permitiendo identificar con precisión milimétrica qué operación consume más tiempo dentro del ciclo de vida de la página.
- La **paginación** es la optimización de mayor impacto cuando el problema es la cantidad de datos transferidos (reduce el HTML generado en >90% para tablas grandes).
- El **caché de regiones** es efectivo para contenido estático o semestático, eliminando el costo de renderizado en cargas subsecuentes.
- Los **índices** son fundamentales cuando el problema es el tiempo de ejecución de la consulta SQL (TTFB alto). Sin estadísticas actualizadas, el optimizador puede ignorarlos.

### Recursos adicionales

- [Oracle APEX Debug Mode — Documentación oficial](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/debugging-an-application.html)
- [Oracle APEX Performance Tuning Best Practices](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/managing-application-performance.html)
- [Oracle Database SQL Tuning Guide — EXPLAIN PLAN](https://docs.oracle.com/en/database/oracle/oracle-database/21/tgsql/generating-and-displaying-execution-plans.html)
- [APEX Region Caching Reference](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/understanding-region-caching.html)
- [Chrome DevTools Network Analysis](https://developer.chrome.com/docs/devtools/network/)

---
