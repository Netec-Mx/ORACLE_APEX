# Práctica 8: Crear y aplicar componentes compartidos en varios módulos

## Metadatos del Laboratorio

| Atributo        | Detalle                                      |
|-----------------|----------------------------------------------|
| **Duración**    | 52 minutos                                   |
| **Complejidad** | Alta                                         |
| **Nivel Bloom** | Crear (Create)                               |
| **Laboratorio** | 08-00-01                                     |
| **Tema**        | 8.1, 8.2, 8.3 — Componentes Compartidos APEX |

---

## Descripción General

En este laboratorio construirás una base sólida de componentes reutilizables dentro de la aplicación APEX desarrollada en los laboratorios anteriores. Centralizarás listas de valores duplicadas, crearás plantillas personalizadas de región e ítem, implementarás procesos globales de aplicación para auditoría y mantenimiento, y diseñarás un plugin APEX básico que encapsule un componente de UI personalizado. Al finalizar, la aplicación seguirá el principio de "definir una vez, usar en todas partes", reduciendo la duplicación de código y estandarizando el desarrollo del equipo.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Identificar y centralizar elementos repetidos en la aplicación mediante Shared Components (LOVs, Application Items, Computations y Processes)
- [ ] Crear LOVs estáticas y dinámicas compartidas y reemplazar implementaciones duplicadas en múltiples páginas
- [ ] Desarrollar plantillas personalizadas de región e ítem que estandaricen el diseño corporativo de la aplicación
- [ ] Implementar Application Processes para lógica transversal: registro de auditoría de sesión, verificación de modo mantenimiento y carga de preferencias de usuario
- [ ] Construir y registrar un plugin APEX básico de tipo ítem para encapsular un componente de UI reutilizable

---

## Prerrequisitos

### Conocimientos Necesarios

- Haber completado los Laboratorios 04-00-01, 06-00-01 y 07-00-01
- Comprensión del sistema de plantillas de APEX Universal Theme (clases CSS, sustituciones `#BODY#`, `#TITLE#`)
- Conocimientos básicos de HTML y CSS para personalización de plantillas
- Conocimiento de JavaScript para desarrollo básico de plugins APEX
- Familiaridad con PL/SQL: procedimientos, funciones y manejo de excepciones

### Acceso Requerido

- Acceso al APEX Application Builder con la aplicación base de los laboratorios anteriores
- Permisos de desarrollador en el workspace de APEX
- Acceso a SQL Workshop para ejecutar scripts de creación de tablas
- Navegador web moderno (Chrome 110+, Firefox 110+ o Edge 110+)

---

## Entorno de Laboratorio

### Configuración de Hardware y Software

| Componente        | Requerimiento Mínimo                          |
|-------------------|-----------------------------------------------|
| RAM               | 8 GB (16 GB recomendado)                      |
| Almacenamiento    | 50 GB libres (SSD recomendado)                |
| Procesador        | Intel Core i5 8ª gen / AMD Ryzen 5 o superior |
| Navegador         | Chrome 110+ / Firefox 110+ / Edge 110+        |
| Oracle APEX       | 23.2 o superior                               |
| Oracle Database   | 19c / 21c (XE aceptable)                      |
| ORDS              | 23.2 o superior                               |

### Preparación del Entorno — Scripts SQL Iniciales

Antes de iniciar los pasos del laboratorio, ejecuta los siguientes scripts en **SQL Workshop > SQL Commands** para crear las tablas de soporte que usarán los componentes compartidos.

```sql
-- ============================================================
-- TABLA: LOG_SESIONES_APP
-- Registra cada inicio de sesión para auditoría
-- ============================================================
CREATE TABLE log_sesiones_app (
    log_id          NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    usuario         VARCHAR2(100)  NOT NULL,
    fecha_acceso    TIMESTAMP      DEFAULT SYSTIMESTAMP NOT NULL,
    ip_address      VARCHAR2(45),
    app_id          NUMBER,
    session_id      NUMBER,
    accion          VARCHAR2(50)   DEFAULT 'LOGIN'
);

-- ============================================================
-- TABLA: CONFIG_SISTEMA
-- Parámetros de configuración global (incluye modo mantenimiento)
-- ============================================================
CREATE TABLE config_sistema (
    param_clave     VARCHAR2(100)  PRIMARY KEY,
    param_valor     VARCHAR2(500)  NOT NULL,
    descripcion     VARCHAR2(200),
    fecha_modif     TIMESTAMP      DEFAULT SYSTIMESTAMP
);

-- Insertar parámetros iniciales
INSERT INTO config_sistema (param_clave, param_valor, descripcion)
VALUES ('MODO_MANTENIMIENTO', 'N', 'S=Sistema en mantenimiento, N=Sistema activo');

INSERT INTO config_sistema (param_clave, param_valor, descripcion)
VALUES ('URL_MANTENIMIENTO', 'f?p=&APP_ID.:9999:&SESSION.', 'URL de redirección en mantenimiento');

INSERT INTO config_sistema (param_clave, param_valor, descripcion)
VALUES ('ITEMS_POR_PAGINA', '15', 'Número de filas por defecto en grillas');

COMMIT;

-- ============================================================
-- TABLA: PREFERENCIAS_USUARIO
-- Almacena preferencias individuales por usuario
-- ============================================================
CREATE TABLE preferencias_usuario (
    pref_id         NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    usuario         VARCHAR2(100)  NOT NULL,
    pref_clave      VARCHAR2(100)  NOT NULL,
    pref_valor      VARCHAR2(500),
    CONSTRAINT uq_usuario_clave UNIQUE (usuario, pref_clave)
);

-- Insertar preferencias de ejemplo
INSERT INTO preferencias_usuario (usuario, pref_clave, pref_valor)
VALUES ('DEMO', 'TEMA_COLOR', 'azul');

INSERT INTO preferencias_usuario (usuario, pref_clave, pref_valor)
VALUES ('DEMO', 'IDIOMA', 'es');

COMMIT;

-- ============================================================
-- VERIFICACIÓN: confirmar que las tres tablas existen
-- ============================================================
SELECT table_name
FROM   user_tables
WHERE  table_name IN ('LOG_SESIONES_APP','CONFIG_SISTEMA','PREFERENCIAS_USUARIO')
ORDER  BY table_name;
```

**Resultado esperado de la verificación:** Deben aparecer exactamente 3 filas con los nombres de las tablas.

---

## Pasos del Laboratorio

---

### Paso 1 — Auditoría de Componentes Duplicados y Creación de LOVs Compartidas

**Objetivo:** Identificar campos con listas de valores duplicadas en la aplicación y centralizarlas como LOVs compartidas (estática y dinámica).

#### Instrucciones

1. Abre el **Application Builder** y navega a tu aplicación base (la desarrollada desde el Laboratorio 04).

2. Ve a **Shared Components** haciendo clic en el ícono de cuadrícula en la barra superior de la aplicación, o desde la página de inicio de la aplicación.

3. En la sección **Application Logic**, haz clic en **List of Values**.

4. Haz clic en **Create** para crear la primera LOV.

5. Selecciona **From Scratch** y haz clic en **Next**.

6. Configura la LOV estática con los siguientes valores:

   | Campo | Valor |
   |-------|-------|
   | Name  | `LOV_S_ESTADOS_REGISTRO` |
   | Type  | Static |

7. Haz clic en **Next**. En la pantalla de valores estáticos, agrega las siguientes entradas usando el botón **Add Entry**:

   | Display Value      | Return Value |
   |--------------------|--------------|
   | Activo             | A            |
   | Inactivo           | I            |
   | Pendiente          | P            |
   | Suspendido         | S            |

8. Haz clic en **Create List of Values**.

9. Repite el proceso para crear una LOV **dinámica**. Haz clic en **Create** nuevamente.

10. Selecciona **From Scratch** → **Next**. Configura:

    | Campo | Valor |
    |-------|-------|
    | Name  | `LOV_D_DEPARTAMENTOS` |
    | Type  | Dynamic |

11. En el campo **Query**, ingresa la siguiente consulta SQL (ajusta el nombre de tabla al de tu aplicación si es diferente):

    ```sql
    SELECT nombre_departamento AS d,
           departamento_id     AS r
    FROM   departamentos
    WHERE  activo = 'S'
    ORDER  BY nombre_departamento
    ```

    > **Nota:** Si tu aplicación no tiene tabla `departamentos`, usa la siguiente consulta de ejemplo con datos sintéticos:
    > ```sql
    > SELECT column_value AS d,
    >        ROWNUM        AS r
    > FROM   TABLE(sys.odcivarchar2list(
    >            'Recursos Humanos','Tecnología',
    >            'Finanzas','Operaciones','Ventas'))
    > ORDER  BY 1
    > ```

12. Haz clic en **Create List of Values**.

13. Crea una tercera LOV dinámica para categorías:

    | Campo | Valor |
    |-------|-------|
    | Name  | `LOV_D_CATEGORIAS` |
    | Type  | Dynamic |

    ```sql
    SELECT categoria_nombre AS d,
           categoria_id     AS r
    FROM   categorias
    WHERE  activo = 'S'
    ORDER  BY categoria_nombre
    ```

14. Ahora aplica las LOVs a los campos existentes. Navega a una página de tu aplicación que tenga un campo de estado (por ejemplo, la página del formulario de empleados o registros). Haz clic en el campo de estado en el Page Designer.

15. En el panel de propiedades derecho, localiza la sección **List of Values** y cambia:
    - **Type:** Shared Component
    - **List of Values:** `LOV_S_ESTADOS_REGISTRO`
    - **Display Extra Values:** No
    - **Null Display Value:** `— Seleccione —`

16. Guarda la página con **Ctrl+S** o el botón **Save**.

17. Repite el paso 14-16 para **todos los campos de estado** en otras páginas de la aplicación, reemplazando cualquier LOV local por la compartida `LOV_S_ESTADOS_REGISTRO`.

#### Resultado Esperado

- En **Shared Components > List of Values** aparecen al menos 3 LOVs: `LOV_S_ESTADOS_REGISTRO`, `LOV_D_DEPARTAMENTOS` y `LOV_D_CATEGORIAS`.
- Los campos de estado en las páginas de la aplicación muestran las opciones correctas (Activo, Inactivo, Pendiente, Suspendido) usando la LOV compartida.

#### Verificación

Ejecuta la aplicación, navega a un formulario con campo de estado y confirma que el desplegable muestra exactamente las 4 opciones definidas en `LOV_S_ESTADOS_REGISTRO`. Verifica también en **SQL Workshop** que la LOV dinámica funciona:

```sql
-- Probar la consulta de la LOV dinámica
SELECT nombre_departamento AS d,
       departamento_id     AS r
FROM   departamentos
WHERE  activo = 'S'
ORDER  BY nombre_departamento;
```

---

### Paso 2 — Crear Application Items y Computations para Preferencias de Usuario

**Objetivo:** Definir Application Items que almacenen preferencias del usuario en la sesión de APEX y configurar Computations para cargarlos automáticamente.

#### Instrucciones

1. Desde **Shared Components**, en la sección **Application Logic**, haz clic en **Application Items**.

2. Haz clic en **Create**. Configura el primer ítem de aplicación:

   | Campo              | Valor                           |
   |--------------------|---------------------------------|
   | Name               | `APP_PREF_TEMA`                 |
   | Scope              | Application                     |
   | Session State Protection | Unrestricted             |

3. Haz clic en **Create Application Item**.

4. Repite el proceso para crear los siguientes ítems adicionales:

   | Name                    | Scope       | Session State Protection |
   |-------------------------|-------------|--------------------------|
   | `APP_PREF_IDIOMA`       | Application | Unrestricted             |
   | `APP_PREF_FILAS_PAGINA` | Application | Unrestricted             |
   | `APP_MODO_MANT`         | Application | Unrestricted             |

5. Ahora crea las **Application Computations** para cargar los valores. En **Shared Components > Application Logic**, haz clic en **Application Computations**.

6. Haz clic en **Create**. Configura la primera computation:

   | Campo                  | Valor                          |
   |------------------------|--------------------------------|
   | Item Name              | `APP_PREF_TEMA`                |
   | Sequence               | 10                             |
   | Computation Point      | After Authentication           |
   | Computation Type       | SQL Query (return single value)|

   En el campo **Computation**:
   ```sql
   SELECT NVL(pref_valor, 'azul')
   FROM   preferencias_usuario
   WHERE  usuario    = :APP_USER
     AND  pref_clave = 'TEMA_COLOR'
   ```

7. Haz clic en **Create Application Computation**.

8. Crea la segunda computation para el idioma:

   | Campo             | Valor                          |
   |-------------------|--------------------------------|
   | Item Name         | `APP_PREF_IDIOMA`              |
   | Sequence          | 20                             |
   | Computation Point | After Authentication           |
   | Computation Type  | SQL Query (return single value)|

   ```sql
   SELECT NVL(pref_valor, 'es')
   FROM   preferencias_usuario
   WHERE  usuario    = :APP_USER
     AND  pref_clave = 'IDIOMA'
   ```

9. Crea la tercera computation para filas por página:

   | Campo             | Valor                          |
   |-------------------|--------------------------------|
   | Item Name         | `APP_PREF_FILAS_PAGINA`        |
   | Sequence          | 30                             |
   | Computation Point | After Authentication           |
   | Computation Type  | SQL Query (return single value)|

   ```sql
   SELECT NVL(
       (SELECT pref_valor
        FROM   preferencias_usuario
        WHERE  usuario    = :APP_USER
          AND  pref_clave = 'ITEMS_POR_PAGINA'),
       (SELECT param_valor
        FROM   config_sistema
        WHERE  param_clave = 'ITEMS_POR_PAGINA')
   ) FROM dual
   ```

10. Crea la cuarta computation para el modo mantenimiento:

    | Campo             | Valor                          |
    |-------------------|--------------------------------|
    | Item Name         | `APP_MODO_MANT`                |
    | Sequence          | 5                              |
    | Computation Point | After Authentication           |
    | Computation Type  | SQL Query (return single value)|

    ```sql
    SELECT NVL(param_valor, 'N')
    FROM   config_sistema
    WHERE  param_clave = 'MODO_MANTENIMIENTO'
    ```

#### Resultado Esperado

- En **Shared Components > Application Items** aparecen 4 ítems: `APP_PREF_TEMA`, `APP_PREF_IDIOMA`, `APP_PREF_FILAS_PAGINA` y `APP_MODO_MANT`.
- En **Shared Components > Application Computations** aparecen 4 computations con punto de ejecución "After Authentication".

#### Verificación

Agrega temporalmente un elemento de tipo **Display Only** en cualquier página de la aplicación con el valor `&APP_PREF_TEMA.` (incluyendo el punto final). Ejecuta la aplicación y verifica que muestra el valor de la preferencia del usuario. Elimina el elemento temporal al confirmar el funcionamiento.

---

### Paso 3 — Implementar Application Processes para Lógica Transversal

**Objetivo:** Crear tres procesos globales de aplicación: registro de auditoría de sesión, verificación de modo mantenimiento y carga de preferencias extendidas.

#### Instrucciones

1. Desde **Shared Components > Application Logic**, haz clic en **Application Processes**.

2. Haz clic en **Create**. Configura el **Proceso 1: Registro de Auditoría de Sesión**:

   | Campo             | Valor                          |
   |-------------------|--------------------------------|
   | Name              | `PROC_AUDIT_LOGIN`             |
   | Sequence          | 10                             |
   | Process Point     | After Authentication           |
   | Condition Type    | (sin condición)                |

   En el campo **Process Body (PL/SQL)**:
   ```plsql
   BEGIN
       -- Registrar el acceso del usuario en la tabla de auditoría
       INSERT INTO log_sesiones_app (
           usuario,
           fecha_acceso,
           ip_address,
           app_id,
           session_id,
           accion
       ) VALUES (
           :APP_USER,
           SYSTIMESTAMP,
           OWA_UTIL.GET_CGI_ENV('REMOTE_ADDR'),
           :APP_ID,
           :APP_SESSION,
           'LOGIN'
       );
       COMMIT;
   EXCEPTION
       WHEN OTHERS THEN
           -- No interrumpir el flujo de la aplicación por error de auditoría
           NULL;
   END;
   ```

3. Haz clic en **Create Process**.

4. Crea el **Proceso 2: Verificación de Modo Mantenimiento**:

   | Campo             | Valor                          |
   |-------------------|--------------------------------|
   | Name              | `PROC_VERIF_MANTENIMIENTO`     |
   | Sequence          | 20                             |
   | Process Point     | On Load: Before Header         |
   | Condition Type    | PL/SQL Function Body           |

   En el campo **Condition Expression** (condición de ejecución):
   ```plsql
   -- Solo ejecutar si el sistema está en mantenimiento
   -- Y el usuario no es administrador
   -- Y no estamos ya en la página de mantenimiento
   RETURN (
       :APP_MODO_MANT = 'S'
       AND :APP_PAGE_ID != 9999
       AND NVL(
           (SELECT 'S'
            FROM   preferencias_usuario
            WHERE  usuario    = :APP_USER
              AND  pref_clave = 'ES_ADMIN'
              AND  pref_valor = 'S'),
           'N'
       ) = 'N'
   );
   ```

   En el campo **Process Body (PL/SQL)**:
   ```plsql
   BEGIN
       -- Redirigir al usuario a la página de mantenimiento
       APEX_UTIL.REDIRECT_URL(
           p_url => APEX_PAGE.GET_URL(
               p_page => 9999
           )
       );
   END;
   ```

5. Haz clic en **Create Process**.

6. Crea el **Proceso 3: Carga de Preferencias Extendidas**:

   | Campo             | Valor                          |
   |-------------------|--------------------------------|
   | Name              | `PROC_CARGA_PREFERENCIAS`      |
   | Sequence          | 30                             |
   | Process Point     | After Authentication           |

   En el campo **Process Body (PL/SQL)**:
   ```plsql
   DECLARE
       v_count NUMBER;
   BEGIN
       -- Verificar si el usuario tiene preferencias registradas
       SELECT COUNT(*)
       INTO   v_count
       FROM   preferencias_usuario
       WHERE  usuario = :APP_USER;
       
       -- Si no tiene preferencias, crear las predeterminadas
       IF v_count = 0 THEN
           INSERT INTO preferencias_usuario (usuario, pref_clave, pref_valor)
           VALUES (:APP_USER, 'TEMA_COLOR', 'azul');
           
           INSERT INTO preferencias_usuario (usuario, pref_clave, pref_valor)
           VALUES (:APP_USER, 'IDIOMA', 'es');
           
           INSERT INTO preferencias_usuario (usuario, pref_clave, pref_valor)
           VALUES (:APP_USER, 'ITEMS_POR_PAGINA', '15');
           
           COMMIT;
       END IF;
       
   EXCEPTION
       WHEN OTHERS THEN
           -- Registrar el error sin interrumpir la sesión
           APEX_DEBUG.ERROR(
               p_message => 'Error en PROC_CARGA_PREFERENCIAS: ' || SQLERRM
           );
   END;
   ```

7. Haz clic en **Create Process**.

8. Verifica el orden de ejecución. En la lista de **Application Processes**, confirma que el orden de secuencia es:
   - Secuencia 10: `PROC_AUDIT_LOGIN` (After Authentication)
   - Secuencia 20: `PROC_VERIF_MANTENIMIENTO` (On Load: Before Header)
   - Secuencia 30: `PROC_CARGA_PREFERENCIAS` (After Authentication)

#### Resultado Esperado

- Tres procesos globales aparecen en **Shared Components > Application Processes**.
- Al iniciar sesión en la aplicación, se inserta un registro en `LOG_SESIONES_APP`.
- Las preferencias del usuario se inicializan automáticamente si no existen.

#### Verificación

Cierra sesión y vuelve a iniciar sesión en la aplicación. Luego ejecuta en SQL Workshop:

```sql
-- Verificar que el proceso de auditoría registró el acceso
SELECT usuario,
       TO_CHAR(fecha_acceso, 'DD/MM/YYYY HH24:MI:SS') AS fecha_acceso,
       app_id,
       accion
FROM   log_sesiones_app
ORDER  BY fecha_acceso DESC
FETCH FIRST 5 ROWS ONLY;
```

Debes ver al menos una fila con tu usuario y la acción `LOGIN`.

---

### Paso 4 — Desarrollar Plantilla de Región Personalizada

**Objetivo:** Crear una plantilla de región con estilo corporativo (panel informativo con cabecera coloreada y borde) que estandarice la apariencia de paneles en toda la aplicación.

#### Instrucciones

1. Desde **Shared Components**, en la sección **User Interface**, haz clic en **Templates**.

2. En el filtro **Component Type**, selecciona **Region** para ver solo las plantillas de región.

3. Haz clic en **Create**. Selecciona **From Scratch** y haz clic en **Next**.

4. Configura los metadatos de la plantilla:

   | Campo             | Valor                              |
   |-------------------|------------------------------------|
   | Name              | `Panel Corporativo Informativo`    |
   | Template Class    | Reports Region                     |

5. Haz clic en **Next**. En la sección de **Template Definition**, ingresa el siguiente HTML en el campo **Template**:

   ```html
   <div class="t-Region corp-panel #REGION_CSS_CLASSES#" 
        id="#REGION_STATIC_ID#" 
        #REGION_ATTRIBUTES#>
     <div class="corp-panel-header">
       <span class="corp-panel-icon">
         <span class="t-Icon #ICON_CSS_CLASSES#" aria-hidden="true"></span>
       </span>
       <h2 class="corp-panel-title">#TITLE#</h2>
       <div class="corp-panel-buttons">
         #EDIT##CLOSE##COPY##HELP##DELETE##PREVIOUS##NEXT##EXPAND#
       </div>
     </div>
     <div class="corp-panel-body">
       #BODY#
     </div>
     <div class="corp-panel-footer">
       #PREVIOUS##NEXT##DELETE##EDIT##CHANGE##CREATE##CREATE2##ADD##DELETE2##NEXT##PREVIOUS##CLOSE#
     </div>
   </div>
   ```

6. En el campo **Error Body**, ingresa:
   ```html
   <div class="corp-panel-error">
     <span class="t-Icon fa-exclamation-triangle" aria-hidden="true"></span>
     #MESSAGE#
   </div>
   ```

7. Haz clic en **Next** y luego en **Create Template**.

8. Ahora agrega los estilos CSS para la plantilla. Ve a **Shared Components > User Interface > Static Application Files** (o **Files** en versiones más recientes).

9. Haz clic en **Upload File** y crea un nuevo archivo. Si la opción es crear inline, usa **Application Static Files > Create**:

   - **File Name:** `corp-panel.css`
   - **MIME Type:** `text/css`

   En el contenido del archivo, pega el siguiente CSS:

   ```css
   /* ============================================
      Panel Corporativo Informativo — APEX Custom Template
      ============================================ */
   .corp-panel {
       border: 1px solid #0066cc;
       border-radius: 6px;
       margin-bottom: 16px;
       box-shadow: 0 2px 6px rgba(0, 102, 204, 0.15);
       overflow: hidden;
       background: #ffffff;
   }
   
   .corp-panel-header {
       background: linear-gradient(135deg, #0066cc 0%, #004499 100%);
       color: #ffffff;
       padding: 12px 16px;
       display: flex;
       align-items: center;
       gap: 10px;
   }
   
   .corp-panel-icon .t-Icon {
       font-size: 1.2rem;
       color: #ffffff;
   }
   
   .corp-panel-title {
       font-size: 1rem;
       font-weight: 600;
       color: #ffffff;
       margin: 0;
       flex: 1;
       letter-spacing: 0.3px;
   }
   
   .corp-panel-buttons {
       display: flex;
       gap: 4px;
   }
   
   .corp-panel-body {
       padding: 16px;
       min-height: 40px;
   }
   
   .corp-panel-footer {
       background: #f5f8fc;
       padding: 10px 16px;
       border-top: 1px solid #d0e4f7;
       display: flex;
       justify-content: flex-end;
       gap: 8px;
   }
   
   .corp-panel-error {
       background: #fff3f3;
       border: 1px solid #e53e3e;
       border-radius: 4px;
       padding: 10px 14px;
       color: #c53030;
       display: flex;
       align-items: center;
       gap: 8px;
       margin: 8px 16px;
   }
   ```

10. Guarda el archivo CSS.

11. Para que el CSS se cargue en la aplicación, ve a **Shared Components > User Interface > User Interface Attributes**. En la pestaña **JavaScript and CSS**, en la sección **File URLs**, agrega en el campo **CSS File URLs**:

    ```
    #APP_FILES#corp-panel.css
    ```

12. Haz clic en **Apply Changes**.

13. Para probar la plantilla, navega a cualquier página de la aplicación en el **Page Designer**. Selecciona una región existente (por ejemplo, un formulario de datos). En el panel de propiedades, en **Appearance > Template**, cambia el valor a `Panel Corporativo Informativo`.

14. Guarda y ejecuta la página para verificar la apariencia.

#### Resultado Esperado

- La plantilla `Panel Corporativo Informativo` aparece en la lista de Region Templates.
- El archivo `corp-panel.css` está disponible en Static Application Files.
- Las regiones que usan la plantilla muestran una cabecera azul corporativa con el título en blanco y el contenido en un panel con bordes y sombra.

#### Verificación

Ejecuta la página que tiene la región con la nueva plantilla. Inspecciona el HTML con las herramientas de desarrollo del navegador (`F12`) y verifica que el elemento `div.corp-panel` existe y que los estilos del archivo CSS se están aplicando correctamente.

---

### Paso 5 — Crear Plantilla de Ítem Personalizada con Validación Visual

**Objetivo:** Desarrollar una plantilla de ítem que muestre indicadores visuales de validación (campo requerido, estado de error) para estandarizar la experiencia del usuario en formularios.

#### Instrucciones

1. Desde **Shared Components > User Interface > Templates**, filtra por **Component Type: Item**.

2. Localiza la plantilla estándar **Required** (o **Text Field** según la versión de APEX). Haz clic en ella para abrirla.

3. En lugar de modificar la plantilla estándar, haz clic en **Copy** para crear una copia. Nómbrala:
   - **New Template Name:** `Campo con Validación Corporativa`

4. Abre la copia recién creada. En el campo **Before Item**, reemplaza el contenido con:

   ```html
   <div class="corp-field-wrap #ITEM_CSS_CLASSES#" id="#CURRENT_ITEM_CONTAINER_ID#">
     <label for="#CURRENT_ITEM_NAME#" class="corp-field-label #REQUIRED#">
       #CURRENT_ITEM_LABEL#
       <span class="corp-required-mark" aria-hidden="true">*</span>
     </label>
   ```

5. En el campo **After Item**, reemplaza el contenido con:

   ```html
     <div class="corp-field-msg" id="#CURRENT_ITEM_NAME#_msg" role="alert">
       #ERROR_TEMPLATE#
     </div>
     <div class="corp-field-hint">#CURRENT_ITEM_INLINE_HELP_TEXT#</div>
   </div>
   ```

6. En el campo **Error Template**, configura:

   ```html
   <span class="corp-field-error-icon fa fa-exclamation-circle" aria-hidden="true"></span>
   <span class="corp-field-error-text">#ERROR_MESSAGE#</span>
   ```

7. Haz clic en **Apply Changes** para guardar la plantilla.

8. Agrega los estilos CSS adicionales. Abre el archivo `corp-panel.css` en Static Application Files y agrega al final:

   ```css
   /* ============================================
      Campo con Validación Corporativa — Item Template
      ============================================ */
   .corp-field-wrap {
       margin-bottom: 14px;
   }
   
   .corp-field-label {
       display: block;
       font-size: 0.875rem;
       font-weight: 600;
       color: #2d3748;
       margin-bottom: 4px;
   }
   
   /* Ocultar el asterisco por defecto cuando el campo NO es requerido */
   .corp-field-label:not(.is-required) .corp-required-mark {
       display: none;
   }
   
   /* Mostrar el asterisco en rojo cuando el campo ES requerido */
   .corp-field-label.is-required .corp-required-mark {
       color: #e53e3e;
       margin-left: 2px;
   }
   
   .corp-field-msg {
       min-height: 18px;
       margin-top: 4px;
   }
   
   .corp-field-error-icon {
       color: #e53e3e;
       margin-right: 4px;
   }
   
   .corp-field-error-text {
       font-size: 0.8rem;
       color: #e53e3e;
       font-weight: 500;
   }
   
   .corp-field-hint {
       font-size: 0.78rem;
       color: #718096;
       margin-top: 3px;
       font-style: italic;
   }
   
   /* Estado de error en el campo input */
   .corp-field-wrap.apex-item-error input,
   .corp-field-wrap.apex-item-error select,
   .corp-field-wrap.apex-item-error textarea {
       border-color: #e53e3e !important;
       box-shadow: 0 0 0 2px rgba(229, 62, 62, 0.2) !important;
   }
   ```

9. Guarda los cambios en el archivo CSS.

10. Aplica la nueva plantilla a los campos de un formulario existente. Navega a una página de formulario en el **Page Designer**. Selecciona un campo de texto (por ejemplo, el campo "Nombre"). En el panel de propiedades, en **Appearance > Item Template**, selecciona `Campo con Validación Corporativa`.

11. Asegúrate de que el campo tenga **Value Required = Yes** para probar el indicador visual de campo requerido.

12. Guarda y ejecuta la página.

#### Resultado Esperado

- La plantilla `Campo con Validación Corporativa` aparece en la lista de Item Templates.
- Los campos que usan esta plantilla muestran el asterisco rojo (`*`) cuando son requeridos.
- Al intentar guardar sin llenar un campo requerido, el mensaje de error aparece con el ícono y el texto en rojo directamente bajo el campo.

#### Verificación

Ejecuta la página del formulario. Deja vacío un campo requerido y haz clic en el botón de guardar. Verifica que el mensaje de error aparece directamente bajo el campo con el formato visual corporativo (ícono rojo + texto rojo).

---

### Paso 6 — Crear un Plugin APEX Básico de Tipo Ítem

**Objetivo:** Desarrollar y registrar un plugin de tipo ítem que encapsule un componente de UI personalizado: un campo de entrada con contador de caracteres en tiempo real.

#### Instrucciones

1. Desde **Shared Components**, en la sección **Other Components** (o busca en la barra de búsqueda), haz clic en **Plug-ins**.

2. Haz clic en **Create**. Selecciona **From Scratch**.

3. Configura los metadatos del plugin:

   | Campo                | Valor                                      |
   |----------------------|--------------------------------------------|
   | Name                 | `Texto con Contador de Caracteres`         |
   | Internal Name        | `COM.LABORATORIO.CHAR_COUNTER`             |
   | Type                 | Item                                       |
   | Category             | Text                                       |
   | Supported for        | Standard Form Items                        |

4. En la sección **Callbacks**, en el campo **Render Function Name**, escribe: `render_char_counter`

5. En la sección **Source**, en el campo **PL/SQL Code**, ingresa el siguiente código:

   ```plsql
   -- ============================================================
   -- Plugin: Texto con Contador de Caracteres
   -- Tipo: Item Plugin
   -- Autor: Laboratorio 08
   -- ============================================================
   
   FUNCTION render_char_counter (
       p_item                IN            APEX_PLUGIN.T_ITEM,
       p_plugin              IN            APEX_PLUGIN.T_PLUGIN,
       p_param               IN            APEX_PLUGIN.T_ITEM_RENDER_PARAM,
       p_result              IN OUT NOCOPY APEX_PLUGIN.T_ITEM_RENDER_RESULT
   ) RETURN APEX_PLUGIN.T_ITEM_RENDER_RESULT
   IS
       -- Atributos configurables del plugin
       l_max_length    NUMBER  := NVL(p_item.attribute_01, 200);
       l_item_name     VARCHAR2(100) := p_item.name;
       l_item_value    VARCHAR2(32767) := p_param.value;
       l_placeholder   VARCHAR2(200) := NVL(p_item.attribute_02, '');
       
       -- Identificadores únicos para el DOM
       l_input_id      VARCHAR2(200) := p_item.name;
       l_counter_id    VARCHAR2(200) := p_item.name || '_counter';
   BEGIN
       -- Verificar si estamos en modo debug
       IF APEX_APPLICATION.G_DEBUG THEN
           APEX_PLUGIN_UTIL.DEBUG_ITEM_RENDER(
               p_item   => p_item,
               p_plugin => p_plugin
           );
       END IF;
       
       -- Renderizar el HTML del componente
       SYS.HTP.P(
           '<div class="corp-char-counter-wrap">' ||
           '  <textarea' ||
           '    id="'         || APEX_ESCAPE.HTML_ATTRIBUTE(l_input_id) || '"' ||
           '    name="'       || APEX_ESCAPE.HTML_ATTRIBUTE(l_item_name) || '"' ||
           '    class="apex-item-textarea corp-char-counter-input"' ||
           '    maxlength="'  || l_max_length || '"' ||
           '    placeholder="' || APEX_ESCAPE.HTML_ATTRIBUTE(l_placeholder) || '"' ||
           '    rows="3"' ||
           '    data-max="'   || l_max_length || '"' ||
           '    data-counter="' || APEX_ESCAPE.HTML_ATTRIBUTE(l_counter_id) || '"' ||
           '  >' || APEX_ESCAPE.HTML(l_item_value) || '</textarea>' ||
           '  <div class="corp-char-counter-bar">' ||
           '    <span id="' || APEX_ESCAPE.HTML_ATTRIBUTE(l_counter_id) || '" class="corp-char-count">0</span>' ||
           '    <span class="corp-char-sep"> / </span>' ||
           '    <span class="corp-char-max">' || l_max_length || '</span>' ||
           '    <span class="corp-char-label"> caracteres</span>' ||
           '  </div>' ||
           '</div>'
       );
       
       -- Emitir JavaScript para el comportamiento del contador
       APEX_JAVASCRIPT.ADD_ONLOAD_CODE(
           p_code =>
           '(function() {' ||
           '  var ta = document.getElementById("' || l_input_id || '");' ||
           '  var counter = document.getElementById("' || l_counter_id || '");' ||
           '  if (!ta || !counter) return;' ||
           '  function updateCount() {' ||
           '    var len = ta.value.length;' ||
           '    var max = parseInt(ta.getAttribute("data-max"), 10);' ||
           '    counter.textContent = len;' ||
           '    counter.parentElement.classList.toggle("corp-counter-warn", len > max * 0.8);' ||
           '    counter.parentElement.classList.toggle("corp-counter-danger", len >= max);' ||
           '  }' ||
           '  ta.addEventListener("input", updateCount);' ||
           '  updateCount();' || -- Inicializar con el valor actual
           '})();'
       );
       
       -- Indicar a APEX que este ítem tiene valor de sesión
       p_result.is_navigable := TRUE;
       
       RETURN p_result;
   END render_char_counter;
   ```

6. Ahora define los **Custom Attributes** del plugin. En la sección **Custom Attributes**, haz clic en **Add Attribute**:

   **Atributo 1:**
   | Campo         | Valor                              |
   |---------------|------------------------------------|
   | Attribute     | 1                                  |
   | Label         | Máximo de Caracteres               |
   | Type          | Integer                            |
   | Required      | No                                 |
   | Default Value | 200                                |

   **Atributo 2:**
   | Campo         | Valor                              |
   |---------------|------------------------------------|
   | Attribute     | 2                                  |
   | Label         | Texto de Placeholder               |
   | Type          | Text                               |
   | Required      | No                                 |

7. En la sección **CSS**, en el campo **Inline CSS**, agrega:

   ```css
   .corp-char-counter-wrap {
       width: 100%;
   }
   .corp-char-counter-input {
       width: 100%;
       resize: vertical;
   }
   .corp-char-counter-bar {
       font-size: 0.78rem;
       color: #718096;
       text-align: right;
       padding: 3px 0;
       transition: color 0.2s;
   }
   .corp-counter-warn .corp-char-count {
       color: #d69e2e;
       font-weight: 600;
   }
   .corp-counter-danger .corp-char-count {
       color: #e53e3e;
       font-weight: 700;
   }
   ```

8. Haz clic en **Apply Changes** para guardar el plugin.

9. Ahora usa el plugin en una página. Navega a una página de formulario en el **Page Designer**. Haz clic derecho en la región del formulario y selecciona **Create Page Item**.

10. Configura el nuevo ítem:

    | Campo         | Valor                                     |
    |---------------|-------------------------------------------|
    | Name          | `P_DESCRIPCION_EXTENDIDA`                 |
    | Type          | `Texto con Contador de Caracteres` (Plugin)|
    | Label         | Descripción Extendida                     |

11. En la sección **Settings** del ítem (atributos del plugin):
    - **Máximo de Caracteres:** `300`
    - **Texto de Placeholder:** `Ingrese una descripción detallada (máx. 300 caracteres)`

12. Guarda la página y ejecútala.

#### Resultado Esperado

- El plugin `COM.LABORATORIO.CHAR_COUNTER` aparece en **Shared Components > Plug-ins**.
- El campo `P_DESCRIPCION_EXTENDIDA` en el formulario muestra un textarea con un contador de caracteres en tiempo real en la esquina inferior derecha.
- El contador cambia de color (amarillo → rojo) al acercarse al límite de caracteres.

#### Verificación

Ejecuta la página con el plugin. Escribe texto en el campo de descripción extendida y verifica que:
1. El contador se actualiza en tiempo real con cada pulsación de tecla.
2. Al superar el 80% del límite (240 caracteres de 300), el contador cambia a color amarillo.
3. Al alcanzar el límite máximo, el contador cambia a rojo y no permite ingresar más caracteres.

---

### Paso 7 — Exportar Componentes Compartidos para Reutilización

**Objetivo:** Exportar los componentes compartidos creados (LOVs, plantillas y plugin) como un paquete reutilizable para otras aplicaciones del workspace.

#### Instrucciones

1. Desde la página de inicio de la aplicación en el **Application Builder**, haz clic en el menú de opciones (ícono de engranaje o menú `...`) y selecciona **Export / Import**.

2. Selecciona la opción **Export** y elige **Export Application Components**.

3. En la pantalla de selección de componentes, marca los siguientes elementos:

   - [ ] **List of Values:** `LOV_S_ESTADOS_REGISTRO`, `LOV_D_DEPARTAMENTOS`, `LOV_D_CATEGORIAS`
   - [ ] **Templates:** `Panel Corporativo Informativo`, `Campo con Validación Corporativa`
   - [ ] **Plug-ins:** `Texto con Contador de Caracteres`
   - [ ] **Application Items:** `APP_PREF_TEMA`, `APP_PREF_IDIOMA`, `APP_PREF_FILAS_PAGINA`, `APP_MODO_MANT`

4. Haz clic en **Export** y guarda el archivo generado con el nombre:
   ```
   componentes_compartidos_lab08_v1.sql
   ```

5. Abre el archivo exportado en un editor de texto (Visual Studio Code) para verificar su estructura. Busca las siguientes secciones en el SQL generado:

   ```sql
   -- Debe contener bloques como:
   -- wwv_flow_api.create_list_of_values (para LOVs)
   -- wwv_flow_api.create_template (para plantillas)
   -- wwv_flow_api.create_plugin (para el plugin)
   ```

6. Para simular la importación en otra aplicación, crea una **nueva aplicación vacía** en el workspace:
   - Ve al **App Builder > Create** → **New Application**
   - Nombre: `App de Prueba Importación Lab08`
   - Acepta los valores predeterminados y crea la aplicación.

7. En la nueva aplicación, ve a **Shared Components > Export / Import > Import**.

8. Selecciona el archivo `componentes_compartidos_lab08_v1.sql` y haz clic en **Next**.

9. Confirma la importación y verifica que los componentes aparecen en la nueva aplicación.

#### Resultado Esperado

- El archivo `componentes_compartidos_lab08_v1.sql` se descarga correctamente y contiene los bloques SQL de los componentes exportados.
- Al importar en la aplicación de prueba, todos los componentes (LOVs, plantillas, plugin) aparecen disponibles en sus respectivas secciones de **Shared Components**.

#### Verificación

En la aplicación de prueba, ve a **Shared Components > List of Values** y confirma que `LOV_S_ESTADOS_REGISTRO` está disponible. Ve a **Templates** y confirma que `Panel Corporativo Informativo` está en la lista. Ve a **Plug-ins** y confirma que `Texto con Contador de Caracteres` está registrado.

---

## Validación y Pruebas Finales

Ejecuta las siguientes verificaciones para confirmar que todos los componentes del laboratorio funcionan correctamente de forma integrada.

### Prueba 1 — Verificación de LOVs en Múltiples Páginas

```sql
-- Confirmar que las LOVs compartidas están definidas en la aplicación
SELECT lov_name,
       lov_type,
       source_type
FROM   apex_application_lovs
WHERE  application_id = :APP_ID
  AND  lov_name IN (
       'LOV_S_ESTADOS_REGISTRO',
       'LOV_D_DEPARTAMENTOS',
       'LOV_D_CATEGORIAS'
  )
ORDER  BY lov_name;
```

**Resultado esperado:** 3 filas, una por cada LOV creada.

### Prueba 2 — Verificación de Application Items y Computations

```sql
-- Verificar Application Items creados
SELECT item_name,
       item_scope,
       protection_level
FROM   apex_application_items
WHERE  application_id = :APP_ID
  AND  item_name LIKE 'APP_%'
ORDER  BY item_name;
```

**Resultado esperado:** Al menos 4 filas con los ítems `APP_MODO_MANT`, `APP_PREF_FILAS_PAGINA`, `APP_PREF_IDIOMA` y `APP_PREF_TEMA`.

### Prueba 3 — Verificación del Proceso de Auditoría

```sql
-- Verificar registros de auditoría generados
SELECT COUNT(*) AS total_accesos,
       MIN(fecha_acceso) AS primer_acceso,
       MAX(fecha_acceso) AS ultimo_acceso
FROM   log_sesiones_app
WHERE  usuario = :APP_USER;
```

**Resultado esperado:** `total_accesos` debe ser mayor a 0 (al menos el acceso de esta sesión de laboratorio).

### Prueba 4 — Verificación del Plugin

Navega a la página que contiene el campo `P_DESCRIPCION_EXTENDIDA`. Abre las herramientas de desarrollo del navegador (`F12 > Console`) y ejecuta:

```javascript
// Verificar que el plugin está activo
var ta = document.querySelector('.corp-char-counter-input');
var counter = document.querySelector('.corp-char-count');
console.log('Plugin activo:', ta !== null && counter !== null);
console.log('Límite máximo:', ta ? ta.getAttribute('maxlength') : 'N/A');
```

**Resultado esperado:** `Plugin activo: true` y `Límite máximo: 300`.

---

## Solución de Problemas

### Problema 1 — La LOV Dinámica Muestra Error "ORA-00942: table or view does not exist"

**Síntoma:** Al ejecutar la aplicación y navegar a una página con un campo que usa `LOV_D_DEPARTAMENTOS` o `LOV_D_CATEGORIAS`, aparece un error de base de datos indicando que la tabla no existe.

**Causa:** La consulta SQL de la LOV referencia una tabla (`departamentos` o `categorias`) que no existe en el esquema del workspace de APEX, o el nombre de la tabla es diferente al esperado.

**Solución:**
1. Ve a **SQL Workshop > Object Browser** y verifica los nombres exactos de las tablas disponibles en el esquema.
2. Regresa a **Shared Components > List of Values** y edita la LOV problemática.
3. Ajusta la consulta SQL para usar el nombre correcto de la tabla:
   ```sql
   -- Verificar tablas disponibles en SQL Workshop
   SELECT table_name
   FROM   user_tables
   ORDER  BY table_name;
   ```
4. Si la tabla no existe, usa la consulta alternativa con `sys.odcivarchar2list` proporcionada en el Paso 1, instrucción 11.
5. Haz clic en el botón **Test** dentro del editor de la LOV para validar la consulta antes de guardar.

---

### Problema 2 — El Plugin de Contador de Caracteres No Actualiza el Contador en Tiempo Real

**Síntoma:** El campo del plugin se renderiza correctamente con el textarea y el contador, pero el número de caracteres no cambia al escribir. El contador permanece en "0" o en el valor inicial.

**Causa:** El JavaScript del plugin no se está ejecutando correctamente. Esto puede deberse a un conflicto con el modo de carga de scripts de APEX, o a que el `APEX_JAVASCRIPT.ADD_ONLOAD_CODE` se ejecuta antes de que el DOM del plugin esté completamente renderizado.

**Solución:**
1. Abre las herramientas de desarrollo del navegador (`F12 > Console`) y verifica si hay errores de JavaScript. Busca mensajes como `Cannot read properties of null`.
2. Si el error indica que `ta` o `counter` son `null`, el problema es de timing. Edita el plugin en **Shared Components > Plug-ins** y modifica la llamada `ADD_ONLOAD_CODE` para usar un pequeño retraso:
   ```plsql
   APEX_JAVASCRIPT.ADD_ONLOAD_CODE(
       p_code =>
       'setTimeout(function() {' ||
       '  var ta = document.getElementById("' || l_input_id || '");' ||
       '  var counter = document.getElementById("' || l_counter_id || '");' ||
       '  if (!ta || !counter) return;' ||
       '  function updateCount() {' ||
       '    var len = ta.value.length;' ||
       '    var max = parseInt(ta.getAttribute("data-max"), 10);' ||
       '    counter.textContent = len;' ||
       '    counter.parentElement.classList.toggle("corp-counter-warn", len > max * 0.8);' ||
       '    counter.parentElement.classList.toggle("corp-counter-danger", len >= max);' ||
       '  }' ||
       '  ta.addEventListener("input", updateCount);' ||
       '  updateCount();' ||
       '}, 100);'
   );
   ```
3. Si el problema persiste, verifica que la página no tenga habilitado el modo **Partial Page Refresh** que podría interferir con la inicialización del plugin. En el **Page Designer**, revisa **Page > JavaScript > Execute when Page Loads** y asegúrate de que no haya código conflictivo.

---

## Limpieza del Entorno

Al finalizar el laboratorio, realiza las siguientes acciones de limpieza para mantener el entorno ordenado.

### Eliminar la Aplicación de Prueba de Importación

1. En el **App Builder**, localiza la aplicación `App de Prueba Importación Lab08`.
2. Haz clic en el menú de opciones y selecciona **Delete Application**.
3. Confirma la eliminación.

### Desactivar el Modo Mantenimiento (si fue activado durante pruebas)

```sql
-- Asegurar que el sistema NO está en modo mantenimiento
UPDATE config_sistema
SET    param_valor   = 'N',
       fecha_modif   = SYSTIMESTAMP
WHERE  param_clave   = 'MODO_MANTENIMIENTO';

COMMIT;

-- Verificar
SELECT param_clave, param_valor
FROM   config_sistema
WHERE  param_clave = 'MODO_MANTENIMIENTO';
```

### Limpiar Registros de Auditoría de Prueba (Opcional)

```sql
-- Opcional: limpiar registros de prueba de la sesión del laboratorio
-- PRECAUCIÓN: Solo ejecutar en entorno de laboratorio, no en producción
DELETE FROM log_sesiones_app
WHERE  fecha_acceso < SYSTIMESTAMP - INTERVAL '1' HOUR
  AND  accion = 'LOGIN';

COMMIT;
```

### Revertir Campos de Prueba Temporales

Si durante el laboratorio agregaste elementos temporales (como el Display Only para verificar `APP_PREF_TEMA`), regresa al **Page Designer** de las páginas correspondientes y elimina esos elementos temporales.

---

## Resumen

En este laboratorio construiste una base sólida de componentes reutilizables que transforma la arquitectura de la aplicación APEX, pasando de un modelo con código duplicado a uno centralizado y mantenible.

### Logros del Laboratorio

| Componente Creado                          | Ubicación en APEX                    | Beneficio Principal                              |
|--------------------------------------------|--------------------------------------|--------------------------------------------------|
| LOV_S_ESTADOS_REGISTRO (estática)          | Shared Components > LOVs             | Valores de estado unificados en toda la app      |
| LOV_D_DEPARTAMENTOS / LOV_D_CATEGORIAS     | Shared Components > LOVs             | Datos de catálogo sincronizados con la BD        |
| APP_PREF_TEMA / IDIOMA / FILAS / MODO_MANT | Shared Components > Application Items| Estado de sesión centralizado y persistente      |
| 4 Application Computations                 | Shared Components > Computations     | Carga automática de preferencias post-login      |
| PROC_AUDIT_LOGIN                           | Shared Components > App Processes    | Trazabilidad de accesos para auditoría           |
| PROC_VERIF_MANTENIMIENTO                   | Shared Components > App Processes    | Control de acceso durante mantenimiento          |
| PROC_CARGA_PREFERENCIAS                    | Shared Components > App Processes    | Inicialización automática de preferencias        |
| Panel Corporativo Informativo              | Shared Components > Templates        | Diseño corporativo consistente en regiones       |
| Campo con Validación Corporativa           | Shared Components > Templates        | UX mejorada en formularios con feedback visual   |
| Plugin: Texto con Contador de Caracteres   | Shared Components > Plug-ins         | Componente UI encapsulado y reutilizable         |

### Conceptos Clave Aplicados

- **Principio DRY (Don't Repeat Yourself):** Las LOVs compartidas eliminan la definición repetida de consultas idénticas en múltiples páginas.
- **Separación de responsabilidades:** Los Application Processes manejan lógica transversal (auditoría, mantenimiento) sin contaminar la lógica de negocio de páginas individuales.
- **Extensibilidad mediante plugins:** El sistema de plugins de APEX permite encapsular componentes UI complejos con su propio HTML, CSS y JavaScript, haciéndolos portables entre aplicaciones.
- **Exportabilidad:** Los componentes compartidos pueden exportarse e importarse independientemente de la aplicación completa, facilitando la construcción de librerías corporativas.

### Recursos Adicionales

- [Oracle APEX Documentation — Shared Components](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/understanding-shared-components.html)
- [APEX Plugin Development Guide](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/understanding-plug-ins.html)
- [Universal Theme — Template Reference](https://apex.oracle.com/pls/apex/r/apex_pm/ut/template-types)
- [APEX_PLUGIN API Reference](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_PLUGIN.html)
- [APEX_JAVASCRIPT Package Reference](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_JAVASCRIPT.html)

---
*Laboratorio 08-00-01 — Curso Oracle APEX Intermedio — Versión para APEX 23.2*
