# Práctica 6: Añadir comportamientos dinámicos y validaciones

## 1. Metadatos

| Atributo         | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 52 minutos                                   |
| **Complejidad**  | Alta                                         |
| **Nivel Bloom**  | Crear (Create)                               |
| **Laboratorio**  | 06-00-01                                     |
| **Módulo**       | 6 — Comportamientos dinámicos y validaciones |

---

## 2. Descripción General

Este laboratorio añade interactividad avanzada a la aplicación construida en los laboratorios anteriores mediante el sistema de **Dynamic Actions** de Oracle APEX y la integración de JavaScript personalizado. Se trabajará progresivamente desde acciones declarativas puras (mostrar/ocultar campos, habilitar/deshabilitar botones, calcular valores) hasta la integración de la API JavaScript de APEX para realizar llamadas AJAX al servidor y validaciones híbridas cliente-servidor. Al finalizar, la aplicación contará con una experiencia de usuario reactiva, con validaciones robustas tanto en el navegador como en la base de datos.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear y configurar acciones dinámicas declarativas que modifiquen el estado de la página (mostrar/ocultar, habilitar/deshabilitar, establecer valores) en respuesta a eventos del usuario.
- [ ] Integrar código JavaScript personalizado dentro de acciones dinámicas usando `Execute JavaScript Code` y la API `apex.item()` / `apex.region()`.
- [ ] Implementar una llamada AJAX al servidor mediante `apex.server.process()` para verificar disponibilidad de un recurso en tiempo real sin recargar la página.
- [ ] Crear validaciones del lado del servidor con PL/SQL que retornen mensajes de error contextuales y específicos.
- [ ] Aplicar patrones de validación del lado del cliente con expresiones regulares en JavaScript integradas en el flujo de acciones dinámicas.

---

## 4. Prerrequisitos

### Conocimientos Previos

| Área                          | Nivel Requerido                                                          |
|-------------------------------|--------------------------------------------------------------------------|
| Oracle APEX (App Builder)     | Intermedio — haber completado Labs 04-00-01 y 05-00-01                  |
| JavaScript (ES6+)             | Básico — variables, funciones, eventos DOM, callbacks, Promises          |
| AJAX / programación asíncrona | Básico — concepto de llamada asíncrona y manejo de respuesta             |
| PL/SQL                        | Básico — bloques anónimos, cursores, excepciones, RAISE_APPLICATION_ERROR|
| Chrome DevTools               | Básico — abrir consola, inspeccionar errores de red                      |

### Acceso Requerido

- Workspace activo en Oracle APEX 23.2 (apex.oracle.com o instancia local/OCI).
- Aplicación base creada en los laboratorios 04 y 05 (con tablas `PRODUCTOS`, `PEDIDOS`, `DETALLE_PEDIDOS` y datos de muestra cargados).
- Acceso a **App Builder** con permisos de desarrollador en el workspace.
- Acceso a **SQL Workshop → SQL Commands** para crear objetos de base de datos.

---

## 5. Entorno de Laboratorio

### Configuración de Hardware Mínima

| Componente     | Mínimo                          | Recomendado                     |
|----------------|---------------------------------|---------------------------------|
| RAM            | 8 GB                            | 16 GB                           |
| Almacenamiento | 50 GB libres (SSD preferido)    | SSD 100 GB                      |
| Procesador     | Intel Core i5 8ª gen / Ryzen 5  | Intel Core i7 / Ryzen 7         |
| Monitor        | 1280×768                        | 1920×1080                       |
| Internet       | 10 Mbps                         | 25 Mbps                         |

### Software Requerido

| Software              | Versión Mínima | Uso en este Lab                              |
|-----------------------|----------------|----------------------------------------------|
| Oracle APEX           | 23.2           | Desarrollo de la aplicación                  |
| Oracle Database       | 19c / 21c XE   | Backend de datos y ejecución de PL/SQL       |
| ORDS                  | 23.2           | Servicio REST para APEX                      |
| Google Chrome         | 110+           | Pruebas y uso de DevTools                    |
| Oracle SQL Developer  | 23.1+          | Verificación de objetos de BD (opcional)     |

### Preparación del Entorno

Antes de comenzar, verifica que la aplicación base esté disponible y que los datos de muestra estén cargados. Ejecuta las siguientes consultas en **SQL Workshop → SQL Commands**:

```sql
-- Verificar que las tablas base existen y tienen datos
SELECT COUNT(*) AS total_productos FROM productos;
SELECT COUNT(*) AS total_pedidos    FROM pedidos;

-- Verificar estructura mínima de la tabla PRODUCTOS
SELECT column_name, data_type, nullable
FROM   user_tab_columns
WHERE  table_name = 'PRODUCTOS'
ORDER BY column_id;
```

Si alguna tabla no existe o está vacía, solicita al instructor los scripts de datos de muestra antes de continuar.

---

## 6. Pasos del Laboratorio

> **Nota de continuidad:** Este laboratorio trabaja sobre la aplicación construida en los Labs 04 y 05. Los nombres de página, ítems y regiones usados como ejemplo (ej. `P5_CODIGO_PRODUCTO`) deben ajustarse a los nombres reales de tu aplicación. El instructor proporcionará un snapshot de la aplicación si es necesario.

---

### Paso 1 — Preparar el Proceso AJAX en la Base de Datos

**Objetivo:** Crear el Application Process que responderá a las llamadas AJAX para verificar si un código de producto ya existe en la base de datos.

#### Instrucciones

1. En **App Builder**, abre tu aplicación y navega a **Shared Components → Application Processes**.
2. Haz clic en **Create**.
3. Configura el proceso con los siguientes valores:

   | Atributo              | Valor                              |
   |-----------------------|------------------------------------|
   | **Name**              | `VERIFICAR_CODIGO_PRODUCTO`        |
   | **Sequence**          | `10`                               |
   | **Point**             | `Ajax Callback`                    |
   | **Language**          | `PL/SQL`                           |

4. En el campo **PL/SQL Code**, ingresa el siguiente bloque:

```plsql
DECLARE
    l_codigo    VARCHAR2(50) := apex_application.g_x01;
    l_count     NUMBER       := 0;
    l_resultado VARCHAR2(10);
BEGIN
    -- Validar que se recibió un parámetro
    IF l_codigo IS NULL OR TRIM(l_codigo) = '' THEN
        htp.p('{"status":"error","mensaje":"Código no proporcionado"}');
        RETURN;
    END IF;

    -- Verificar existencia del código en la tabla de productos
    SELECT COUNT(*)
    INTO   l_count
    FROM   productos
    WHERE  UPPER(codigo_producto) = UPPER(TRIM(l_codigo));

    IF l_count > 0 THEN
        l_resultado := 'ocupado';
    ELSE
        l_resultado := 'disponible';
    END IF;

    -- Retornar respuesta en formato JSON
    htp.p('{"status":"ok","resultado":"' || l_resultado || '","codigo":"' || TRIM(l_codigo) || '"}');

EXCEPTION
    WHEN OTHERS THEN
        htp.p('{"status":"error","mensaje":"' || SQLERRM || '"}');
END;
```

5. Haz clic en **Create Process**.

#### Resultado Esperado

El proceso `VERIFICAR_CODIGO_PRODUCTO` aparece en la lista de Application Processes con el punto de ejecución `Ajax Callback`.

#### Verificación

Confirma en **Shared Components → Application Processes** que el proceso está listado. El tipo debe mostrar **Ajax Callback**. No es necesario ejecutarlo directamente en este paso; se probará desde la página en el Paso 5.

---

### Paso 2 — Crear Acciones Dinámicas Declarativas: Mostrar/Ocultar Campos

**Objetivo:** Implementar una acción dinámica que muestre u oculte un campo adicional según el valor seleccionado en un Select List, sin escribir código JavaScript.

#### Instrucciones

1. En **App Builder**, navega a la página del formulario de **Productos** (generalmente la página de formulario creada en el Lab 04, por ejemplo `Página 5`).
2. Asegúrate de que la página tenga:
   - Un ítem tipo **Select List** llamado `P5_TIPO_PRODUCTO` con los valores: `FISICO`, `DIGITAL`, `SERVICIO`.
   - Un ítem tipo **Text Field** llamado `P5_PESO_KG` (visible solo para productos físicos).
   
   > Si estos ítems no existen, créalos ahora en el Page Designer antes de continuar.

3. En el **Page Designer**, haz clic derecho sobre el nodo **Dynamic Actions** en el panel izquierdo y selecciona **Create Dynamic Action**.
4. En el panel de propiedades (panel derecho), configura:

   | Atributo                 | Valor                                      |
   |--------------------------|--------------------------------------------|
   | **Name**                 | `DA - Mostrar Peso según Tipo de Producto` |
   | **Event**                | `Change`                                   |
   | **Selection Type**       | `Item(s)`                                  |
   | **Item(s)**              | `P5_TIPO_PRODUCTO`                         |
   | **Condition Type**       | `Item = Value`                             |
   | **Item**                 | `P5_TIPO_PRODUCTO`                         |
   | **Value**                | `FISICO`                                   |

5. En la rama **TRUE Actions**, haz clic en la acción predeterminada creada y configura:

   | Atributo            | Valor         |
   |---------------------|---------------|
   | **Action**          | `Show`        |
   | **Selection Type**  | `Item(s)`     |
   | **Item(s)**         | `P5_PESO_KG`  |

6. Haz clic derecho sobre la acción TRUE y selecciona **Create FALSE Action**. Configura la acción falsa:

   | Atributo            | Valor         |
   |---------------------|---------------|
   | **Action**          | `Hide`        |
   | **Selection Type**  | `Item(s)`     |
   | **Item(s)**         | `P5_PESO_KG`  |

7. **Importante:** Para que el campo esté oculto por defecto al cargar la página, selecciona el ítem `P5_PESO_KG` en el Page Designer y en sus propiedades establece:

   | Atributo               | Valor  |
   |------------------------|--------|
   | **Layout → Start New Row** | `Yes` |
   | **Template Options → Advanced → Item CSS Classes** | (dejar vacío) |

   Luego, en las propiedades del ítem, sección **Advanced**, establece `Custom Attributes` como `style="display:none;"` **o** mejor aún, crea una segunda acción dinámica para el evento `Page Load` que oculte el campo al cargar (ver sub-paso siguiente).

8. Crea una segunda acción dinámica para el estado inicial:
   - **Name:** `DA - Estado inicial campo Peso`
   - **Event:** `Page Load`
   - **TRUE Action:** `Hide` → `P5_PESO_KG`

9. Guarda los cambios con **Save** (Ctrl+S).

#### Resultado Esperado

Al ejecutar la página y cambiar el Select List a `FISICO`, el campo `P5_PESO_KG` aparece. Al seleccionar `DIGITAL` o `SERVICIO`, el campo desaparece. Al cargar la página, el campo inicia oculto.

#### Verificación

1. Haz clic en **Save and Run Page** (▶).
2. Cambia el valor de `P5_TIPO_PRODUCTO` entre las opciones.
3. Confirma que `P5_PESO_KG` aparece solo cuando se selecciona `FISICO`.
4. Abre **Chrome DevTools → Console** y verifica que no hay errores JavaScript.

---

### Paso 3 — Deshabilitar el Botón Guardar con Acción Dinámica

**Objetivo:** Implementar una acción dinámica que deshabilite el botón "Guardar" mientras el campo de nombre del producto esté vacío, y lo habilite cuando tenga contenido.

#### Instrucciones

1. En la misma página del formulario de Productos, verifica que exista:
   - Un ítem `P5_NOMBRE_PRODUCTO` (Text Field, obligatorio).
   - Un botón con el nombre estático `GUARDAR` (o el nombre que tenga en tu aplicación).

2. Crea una nueva acción dinámica:

   | Atributo              | Valor                                         |
   |-----------------------|-----------------------------------------------|
   | **Name**              | `DA - Controlar botón Guardar por Nombre`     |
   | **Event**             | `Key Release`                                 |
   | **Selection Type**    | `Item(s)`                                     |
   | **Item(s)**           | `P5_NOMBRE_PRODUCTO`                          |
   | **Condition Type**    | `Item is NOT NULL`                            |
   | **Item**              | `P5_NOMBRE_PRODUCTO`                          |

3. Configura la **TRUE Action** (campo con valor → habilitar botón):

   | Atributo            | Valor       |
   |---------------------|-------------|
   | **Action**          | `Enable`    |
   | **Selection Type**  | `Button`    |
   | **Button**          | `GUARDAR`   |

4. Configura la **FALSE Action** (campo vacío → deshabilitar botón):

   | Atributo            | Valor       |
   |---------------------|-------------|
   | **Action**          | `Disable`   |
   | **Selection Type**  | `Button`    |
   | **Button**          | `GUARDAR`   |

5. Agrega una acción dinámica adicional para el estado inicial al cargar la página:
   - **Name:** `DA - Deshabilitar Guardar en Page Load`
   - **Event:** `Page Load`
   - **TRUE Action:** `Disable` → Button → `GUARDAR`

6. Guarda los cambios.

#### Resultado Esperado

Al cargar la página (modo creación), el botón Guardar está deshabilitado. Cuando el usuario empieza a escribir en `P5_NOMBRE_PRODUCTO`, el botón se habilita. Si borra todo el contenido, el botón vuelve a deshabilitarse.

#### Verificación

1. Ejecuta la página en modo de creación de nuevo registro.
2. Confirma que el botón Guardar inicia deshabilitado (apariencia atenuada).
3. Escribe texto en el campo de nombre y confirma que el botón se activa.
4. Borra el texto y confirma que el botón vuelve a deshabilitarse.

---

### Paso 4 — Calcular Total con JavaScript en Acción Dinámica

**Objetivo:** Usar la acción `Execute JavaScript Code` y la API `apex.item()` para calcular y mostrar automáticamente el precio total cuando cambian la cantidad o el precio unitario.

#### Instrucciones

1. Navega a la página del formulario de **Detalle de Pedido** (por ejemplo `Página 8`). Verifica que existan:
   - `P8_CANTIDAD` (Number Field)
   - `P8_PRECIO_UNITARIO` (Number Field)
   - `P8_TOTAL` (Display Only o Text Field de solo lectura)

2. Crea una nueva acción dinámica:

   | Atributo              | Valor                                    |
   |-----------------------|------------------------------------------|
   | **Name**              | `DA - Calcular Total Automáticamente`    |
   | **Event**             | `Change`                                 |
   | **Selection Type**    | `Item(s)`                                |
   | **Item(s)**           | `P8_CANTIDAD,P8_PRECIO_UNITARIO`         |
   | **Condition Type**    | *(sin condición — dejar vacío)*          |

3. Configura la **TRUE Action**:

   | Atributo            | Valor                        |
   |---------------------|------------------------------|
   | **Action**          | `Execute JavaScript Code`    |

4. En el campo **Code**, ingresa el siguiente JavaScript:

```javascript
// Obtener valores de los ítems usando la API de APEX
var cantidad       = parseFloat(apex.item('P8_CANTIDAD').getValue())       || 0;
var precioUnitario = parseFloat(apex.item('P8_PRECIO_UNITARIO').getValue()) || 0;

// Calcular el total
var total = cantidad * precioUnitario;

// Formatear a 2 decimales y asignar al ítem de total
apex.item('P8_TOTAL').setValue(total.toFixed(2));

// Feedback visual: resaltar el campo si el total supera cierto umbral
if (total > 10000) {
    apex.item('P8_TOTAL').node.style.color = '#c0392b'; // Rojo para montos altos
    apex.item('P8_TOTAL').node.style.fontWeight = 'bold';
} else {
    apex.item('P8_TOTAL').node.style.color = '';
    apex.item('P8_TOTAL').node.style.fontWeight = '';
}
```

5. Asegúrate de que la opción **Fire on Initialization** esté configurada como `Yes` para que el cálculo ocurra también al cargar la página si ya hay valores.

6. Guarda los cambios.

#### Resultado Esperado

Al modificar la cantidad o el precio unitario, el campo `P8_TOTAL` se actualiza instantáneamente con el resultado de la multiplicación. Si el total supera 10,000, el valor se muestra en rojo y negrita.

#### Verificación

1. Ejecuta la página y abre **Chrome DevTools → Console**.
2. Ingresa `5` en cantidad y `2500` en precio unitario.
3. Confirma que `P8_TOTAL` muestra `12500.00` en color rojo.
4. Cambia la cantidad a `2` y confirma que el total cambia a `5000.00` con color normal.
5. En la consola de DevTools, ejecuta manualmente:
   ```javascript
   apex.item('P8_TOTAL').getValue();
   ```
   Debe retornar `"5000.00"`.

---

### Paso 5 — Verificación AJAX de Código de Producto con `apex.server.process()`

**Objetivo:** Implementar una llamada AJAX en tiempo real que consulte al servidor si un código de producto ya existe, mostrando retroalimentación visual inmediata al usuario sin recargar la página.

#### Instrucciones

1. En la página del formulario de **Productos** (misma del Paso 2), localiza el ítem `P5_CODIGO_PRODUCTO`.

2. Crea una nueva acción dinámica:

   | Atributo              | Valor                                           |
   |-----------------------|-------------------------------------------------|
   | **Name**              | `DA - Verificar Disponibilidad de Código AJAX`  |
   | **Event**             | `Blur` (Lost Focus)                             |
   | **Selection Type**    | `Item(s)`                                       |
   | **Item(s)**           | `P5_CODIGO_PRODUCTO`                            |
   | **Condition Type**    | `Item is NOT NULL`                              |
   | **Item**              | `P5_CODIGO_PRODUCTO`                            |

3. Configura la **TRUE Action**:

   | Atributo   | Valor                      |
   |------------|----------------------------|
   | **Action** | `Execute JavaScript Code`  |

4. En el campo **Code**, ingresa:

```javascript
// Capturar el valor del código ingresado
var codigoIngresado = apex.item('P5_CODIGO_PRODUCTO').getValue().trim();

// Referencia al contenedor de feedback (se crea dinámicamente si no existe)
var feedbackId = 'feedback_codigo_producto';
var $feedback  = $('#' + feedbackId);

if ($feedback.length === 0) {
    // Crear el elemento de feedback debajo del campo
    $feedback = $('<div id="' + feedbackId + '" style="margin-top:4px; font-size:0.85em;"></div>');
    $('#P5_CODIGO_PRODUCTO').closest('.t-Form-fieldContainer').append($feedback);
}

// Mostrar indicador de carga
$feedback.html('<span style="color:#888;">&#9679; Verificando disponibilidad...</span>');

// Llamada AJAX al Application Process creado en el Paso 1
apex.server.process(
    'VERIFICAR_CODIGO_PRODUCTO',   // Nombre del Application Process
    {
        x01: codigoIngresado       // Pasar el código como parámetro x01
    },
    {
        success: function(data) {
            // Parsear la respuesta JSON del servidor
            var respuesta;
            try {
                respuesta = (typeof data === 'string') ? JSON.parse(data) : data;
            } catch(e) {
                $feedback.html('<span style="color:#e74c3c;">&#10007; Error al procesar respuesta</span>');
                return;
            }

            if (respuesta.status === 'ok') {
                if (respuesta.resultado === 'disponible') {
                    $feedback.html('<span style="color:#27ae60;">&#10003; Código disponible</span>');
                    // Habilitar el botón Guardar si estaba deshabilitado por esta razón
                    apex.item('P5_CODIGO_PRODUCTO').node.style.borderColor = '#27ae60';
                } else {
                    $feedback.html('<span style="color:#e74c3c;">&#10007; Código ya existe. Ingresa uno diferente.</span>');
                    apex.item('P5_CODIGO_PRODUCTO').node.style.borderColor = '#e74c3c';
                    // Enfocar el campo para que el usuario lo corrija
                    apex.item('P5_CODIGO_PRODUCTO').node.focus();
                }
            } else {
                $feedback.html('<span style="color:#e67e22;">&#9888; ' + (respuesta.mensaje || 'Error desconocido') + '</span>');
            }
        },
        error: function(jqXHR, textStatus, errorThrown) {
            $feedback.html('<span style="color:#e74c3c;">&#10007; Error de conexión: ' + textStatus + '</span>');
            console.error('Error AJAX:', textStatus, errorThrown);
        },
        dataType: 'text'   // Recibir respuesta como texto para parsear manualmente
    }
);
```

5. Agrega también una acción dinámica para limpiar el feedback cuando el usuario modifique el campo:
   - **Name:** `DA - Limpiar feedback código`
   - **Event:** `Key Release`
   - **Item(s):** `P5_CODIGO_PRODUCTO`
   - **TRUE Action:** `Execute JavaScript Code`
   - **Code:**
   ```javascript
   $('#feedback_codigo_producto').html('');
   apex.item('P5_CODIGO_PRODUCTO').node.style.borderColor = '';
   ```

6. Guarda los cambios.

#### Resultado Esperado

Al escribir un código en `P5_CODIGO_PRODUCTO` y salir del campo (Tab o clic fuera), aparece un mensaje debajo del campo indicando si el código está disponible (verde) o ya existe (rojo). El borde del campo cambia de color según el resultado.

#### Verificación

1. Ejecuta la página y escribe un código que **ya exista** en la tabla de productos (puedes consultarlo con `SELECT codigo_producto FROM productos WHERE ROWNUM = 1`).
2. Sal del campo con Tab y confirma el mensaje en rojo.
3. Escribe un código nuevo que no exista (ej. `PROD-TEST-9999`) y confirma el mensaje en verde.
4. En **Chrome DevTools → Network**, filtra por `XHR` y confirma que se realizó una llamada a la URL de APEX con el parámetro `p_process=VERIFICAR_CODIGO_PRODUCTO`.

---

### Paso 6 — Validación del Lado del Cliente con Expresión Regular

**Objetivo:** Agregar validación de formato en tiempo real para el campo de correo electrónico del proveedor usando JavaScript dentro de una acción dinámica.

#### Instrucciones

1. Navega a la página del formulario de **Proveedores** (o la página que contenga datos de contacto en tu aplicación). Localiza el ítem `P6_EMAIL_PROVEEDOR`.

2. Crea una nueva acción dinámica:

   | Atributo              | Valor                                     |
   |-----------------------|-------------------------------------------|
   | **Name**              | `DA - Validar Formato Email Proveedor`    |
   | **Event**             | `Blur` (Lost Focus)                       |
   | **Selection Type**    | `Item(s)`                                 |
   | **Item(s)**           | `P6_EMAIL_PROVEEDOR`                      |
   | **Condition Type**    | `Item is NOT NULL`                        |
   | **Item**              | `P6_EMAIL_PROVEEDOR`                      |

3. Configura la **TRUE Action** con `Execute JavaScript Code`:

```javascript
var email   = apex.item('P6_EMAIL_PROVEEDOR').getValue().trim();
// Expresión regular estándar para validación de email
var regexEmail = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;

var $campo    = $('#P6_EMAIL_PROVEEDOR');
var feedbackId = 'feedback_email_proveedor';
var $feedback  = $('#' + feedbackId);

if ($feedback.length === 0) {
    $feedback = $('<div id="' + feedbackId + '" style="margin-top:4px; font-size:0.85em;"></div>');
    $campo.closest('.t-Form-fieldContainer').append($feedback);
}

if (regexEmail.test(email)) {
    $feedback.html('<span style="color:#27ae60;">&#10003; Formato de email válido</span>');
    $campo.css('border-color', '#27ae60');
    // Remover error de APEX si existía
    apex.message.clearErrors();
} else {
    $feedback.html('<span style="color:#e74c3c;">&#10007; Formato inválido. Ejemplo: usuario@dominio.com</span>');
    $campo.css('border-color', '#e74c3c');
    // Mostrar error usando la API de mensajes de APEX
    apex.message.showErrors([{
        type:       'error',
        location:   'inline',
        pageItem:   'P6_EMAIL_PROVEEDOR',
        message:    'El formato del correo electrónico no es válido.',
        unsafe:     false
    }]);
}
```

4. Agrega una acción dinámica para limpiar la validación al editar:
   - **Event:** `Key Release`
   - **Item(s):** `P6_EMAIL_PROVEEDOR`
   - **Action:** `Execute JavaScript Code`
   ```javascript
   $('#feedback_email_proveedor').html('');
   $('#P6_EMAIL_PROVEEDOR').css('border-color', '');
   apex.message.clearErrors();
   ```

5. Guarda los cambios.

#### Resultado Esperado

Al salir del campo de email con un formato incorrecto (ej. `usuario@`, `noesunmail`, `@dominio.com`), aparece un mensaje de error en rojo y el borde del campo cambia a rojo. Con un email válido, aparece confirmación verde.

#### Verificación

1. Prueba con los siguientes valores y confirma el comportamiento esperado:

   | Valor de prueba        | Resultado esperado |
   |------------------------|--------------------|
   | `juan@empresa.com`     | ✓ Válido (verde)   |
   | `juan@empresa`         | ✗ Inválido (rojo)  |
   | `@empresa.com`         | ✗ Inválido (rojo)  |
   | `juan.garcia@co.mx`    | ✓ Válido (verde)   |
   | `sin espacios @x.com`  | ✗ Inválido (rojo)  |

---

### Paso 7 — Validaciones del Lado del Servidor con PL/SQL

**Objetivo:** Crear validaciones de página en APEX que ejecuten lógica PL/SQL en el servidor al hacer submit, retornando mensajes de error específicos y contextuales.

#### Instrucciones

1. En la página del formulario de **Productos** (Paso 2), navega a la sección **Validations** en el Page Designer (panel izquierdo, bajo el nodo de la región del formulario o directamente bajo el nodo Validations de la página).

2. Haz clic derecho en **Validations** y selecciona **Create Validation**.

3. Crea la primera validación — **Precio no negativo:**

   | Atributo                  | Valor                                          |
   |---------------------------|------------------------------------------------|
   | **Name**                  | `VAL - Precio Unitario No Negativo`            |
   | **Sequence**              | `10`                                           |
   | **Type**                  | `PL/SQL Function (returning Error Text)`       |
   | **Associated Item**       | `P5_PRECIO_UNITARIO`                           |
   | **Error Message**         | *(se retorna desde el bloque PL/SQL)*          |
   | **When Button Pressed**   | `GUARDAR`                                      |

   **PL/SQL Code:**
   ```plsql
   DECLARE
       l_precio NUMBER;
   BEGIN
       l_precio := TO_NUMBER(:P5_PRECIO_UNITARIO);
       
       IF l_precio < 0 THEN
           RETURN 'El precio unitario no puede ser negativo. Valor ingresado: ' || l_precio;
       END IF;
       
       IF l_precio > 999999.99 THEN
           RETURN 'El precio unitario excede el máximo permitido (999,999.99).';
       END IF;
       
       RETURN NULL; -- NULL significa que la validación pasó
   EXCEPTION
       WHEN VALUE_ERROR THEN
           RETURN 'El precio unitario debe ser un número válido.';
   END;
   ```

4. Crea la segunda validación — **Código de producto único (validación de servidor):**

   | Atributo                  | Valor                                              |
   |---------------------------|----------------------------------------------------|
   | **Name**                  | `VAL - Código de Producto Único`                   |
   | **Sequence**              | `20`                                               |
   | **Type**                  | `PL/SQL Function (returning Error Text)`           |
   | **Associated Item**       | `P5_CODIGO_PRODUCTO`                               |
   | **When Button Pressed**   | `GUARDAR`                                          |

   **PL/SQL Code:**
   ```plsql
   DECLARE
       l_count  NUMBER;
       l_codigo VARCHAR2(50) := UPPER(TRIM(:P5_CODIGO_PRODUCTO));
       l_id     NUMBER       := :P5_PRODUCTO_ID; -- PK del registro actual (NULL en creación)
   BEGIN
       IF l_codigo IS NULL THEN
           RETURN 'El código de producto es obligatorio.';
       END IF;
       
       -- Verificar unicidad excluyendo el registro actual (para edición)
       SELECT COUNT(*)
       INTO   l_count
       FROM   productos
       WHERE  UPPER(codigo_producto) = l_codigo
         AND  (l_id IS NULL OR producto_id <> l_id);
       
       IF l_count > 0 THEN
           RETURN 'El código "' || :P5_CODIGO_PRODUCTO || '" ya está registrado. '
               || 'Por favor ingresa un código único.';
       END IF;
       
       RETURN NULL;
   EXCEPTION
       WHEN OTHERS THEN
           RETURN 'Error al verificar el código: ' || SQLERRM;
   END;
   ```

5. Crea la tercera validación — **Fecha de vigencia coherente:**

   | Atributo                  | Valor                                              |
   |---------------------------|----------------------------------------------------|
   | **Name**                  | `VAL - Fecha Vigencia Mayor a Hoy`                 |
   | **Sequence**              | `30`                                               |
   | **Type**                  | `PL/SQL Function (returning Error Text)`           |
   | **Associated Item**       | `P5_FECHA_VIGENCIA`                                |
   | **When Button Pressed**   | `GUARDAR`                                          |
   | **Condition Type**        | `Item is NOT NULL`                                 |
   | **Condition Item**        | `P5_FECHA_VIGENCIA`                                |

   **PL/SQL Code:**
   ```plsql
   DECLARE
       l_fecha DATE;
   BEGIN
       l_fecha := TO_DATE(:P5_FECHA_VIGENCIA, 'DD/MM/YYYY');
       
       IF l_fecha < TRUNC(SYSDATE) THEN
           RETURN 'La fecha de vigencia no puede ser anterior a hoy ('
               || TO_CHAR(SYSDATE, 'DD/MM/YYYY') || ').';
       END IF;
       
       RETURN NULL;
   EXCEPTION
       WHEN OTHERS THEN
           RETURN 'Formato de fecha inválido. Use el formato DD/MM/YYYY.';
   END;
   ```

6. Guarda todos los cambios.

#### Resultado Esperado

Al intentar guardar un producto con precio negativo, código duplicado o fecha de vigencia pasada, el formulario muestra mensajes de error específicos junto a cada campo afectado, sin procesar el guardado.

#### Verificación

1. Intenta guardar un producto con `P5_PRECIO_UNITARIO = -50`. Confirma el mensaje de error junto al campo.
2. Intenta guardar con el código de un producto existente. Confirma el mensaje indicando que el código ya está registrado.
3. Intenta guardar con una fecha de vigencia de ayer. Confirma el mensaje de fecha inválida.
4. Llena todos los campos correctamente y confirma que el guardado procede sin errores.

---

### Paso 8 — Acción Dinámica de Confirmación Personalizada

**Objetivo:** Reemplazar el diálogo de confirmación estándar del navegador por una confirmación personalizada usando la API de APEX antes de eliminar un registro.

#### Instrucciones

1. En la página de **listado de Productos** (Interactive Report o Interactive Grid), localiza el botón o enlace de **Eliminar** registro.

2. Si el botón de eliminar está en la página de formulario, selecciónalo en el Page Designer.

3. En las propiedades del botón `ELIMINAR`, en la sección **Behavior**, establece:

   | Atributo                    | Valor |
   |-----------------------------|-------|
   | **Action**                  | `Defined by Dynamic Action` |
   | **Execute Validations**     | `No`  |

4. Crea una nueva acción dinámica:

   | Atributo              | Valor                                      |
   |-----------------------|--------------------------------------------|
   | **Name**              | `DA - Confirmar Eliminación de Producto`   |
   | **Event**             | `Click`                                    |
   | **Selection Type**    | `Button`                                   |
   | **Button**            | `ELIMINAR`                                 |

5. Configura la **TRUE Action** con `Execute JavaScript Code`:

```javascript
// Obtener el nombre del producto para personalizar el mensaje
var nombreProducto = apex.item('P5_NOMBRE_PRODUCTO').getValue() || 'este producto';
var codigoProducto = apex.item('P5_CODIGO_PRODUCTO').getValue() || '';

// Usar la API de diálogo de confirmación de APEX (más elegante que window.confirm)
apex.message.confirm(
    'Estás a punto de eliminar el producto:<br><br>' +
    '<strong>' + $('<div>').text(nombreProducto).html() + '</strong>' +
    (codigoProducto ? ' (Código: ' + $('<div>').text(codigoProducto).html() + ')' : '') +
    '<br><br>Esta acción <strong>no se puede deshacer</strong>. ¿Deseas continuar?',
    function(okPressed) {
        if (okPressed) {
            // El usuario confirmó: hacer submit de la página con el request ELIMINAR
            apex.submit({ request: 'ELIMINAR' });
        }
        // Si el usuario canceló, simplemente no hacer nada
    }
);
```

6. Guarda los cambios.

#### Resultado Esperado

Al hacer clic en el botón Eliminar, aparece un diálogo modal de APEX (no el `window.confirm` nativo del navegador) con el nombre y código del producto a eliminar, y botones "Aceptar" / "Cancelar". Solo al confirmar se procesa la eliminación.

#### Verificación

1. Abre un registro de producto existente.
2. Haz clic en el botón Eliminar.
3. Confirma que aparece el diálogo modal de APEX con el nombre del producto.
4. Haz clic en **Cancelar** y verifica que el registro no se elimina.
5. Haz clic nuevamente en Eliminar y confirma con **Aceptar**. Verifica que el registro se elimina y la página redirige correctamente.

---

## 7. Validación y Pruebas Integrales

Una vez completados todos los pasos, realiza las siguientes pruebas de regresión para confirmar que todos los comportamientos funcionan correctamente en conjunto:

### Lista de Verificación Final

```
[ ] PASO 2: El campo P5_PESO_KG se muestra/oculta correctamente según P5_TIPO_PRODUCTO
[ ] PASO 2: El campo P5_PESO_KG inicia oculto al cargar la página (Page Load)
[ ] PASO 3: El botón GUARDAR inicia deshabilitado en modo creación
[ ] PASO 3: El botón GUARDAR se habilita al escribir en P5_NOMBRE_PRODUCTO
[ ] PASO 4: P8_TOTAL se calcula automáticamente al cambiar cantidad o precio
[ ] PASO 4: El total en rojo/negrita aparece cuando supera 10,000
[ ] PASO 5: La verificación AJAX muestra feedback de disponibilidad de código
[ ] PASO 5: La llamada AJAX es visible en DevTools → Network → XHR
[ ] PASO 6: La validación de email detecta formatos incorrectos en tiempo real
[ ] PASO 6: apex.message.showErrors() muestra el error inline junto al campo
[ ] PASO 7: Las 3 validaciones de servidor retornan mensajes específicos
[ ] PASO 7: Las validaciones de servidor NO bloquean registros válidos
[ ] PASO 8: El diálogo de confirmación muestra el nombre del producto
[ ] PASO 8: Cancelar en el diálogo NO elimina el registro
```

### Prueba de Escenario Completo

Ejecuta el siguiente escenario de punta a punta:

1. Crea un nuevo producto con tipo `FISICO`, completa todos los campos incluyendo peso.
2. Intenta guardarlo con el mismo código de un producto existente → debe fallar con mensaje específico.
3. Cambia el código a uno nuevo y único → debe guardarse correctamente.
4. Cambia el tipo a `DIGITAL` → el campo de peso debe ocultarse automáticamente.
5. Navega al detalle de pedido, ingresa cantidad y precio → el total debe calcularse.
6. Regresa al formulario de producto y elimina el registro recién creado → confirma el diálogo personalizado.

---

## 8. Resolución de Problemas

### Problema 1: La llamada AJAX no retorna datos o retorna error 404

**Síntomas:**
- El feedback de verificación de código muestra "Error de conexión" o no aparece nada.
- En **Chrome DevTools → Network**, la llamada XHR muestra código de estado `404` o `0`.
- La consola muestra: `Error AJAX: error` o `parsererror`.

**Causa probable:**
El nombre del Application Process en la llamada JavaScript no coincide exactamente con el nombre registrado en APEX, o el proceso no está configurado con el punto de ejecución `Ajax Callback`.

**Solución:**
1. Ve a **Shared Components → Application Processes** y confirma que el proceso se llama exactamente `VERIFICAR_CODIGO_PRODUCTO` (sin espacios extra, respetando mayúsculas).
2. Verifica que el atributo **Point** del proceso sea `Ajax Callback` (no `On Load - Before Header` ni ningún otro).
3. En el código JavaScript, confirma que el primer argumento de `apex.server.process()` coincide letra por letra con el nombre del proceso.
4. Si usas apex.oracle.com, verifica que tu sesión no haya expirado (los procesos AJAX requieren sesión activa).
5. Prueba la llamada directamente desde la consola de DevTools:
   ```javascript
   apex.server.process('VERIFICAR_CODIGO_PRODUCTO', {x01: 'TEST'}, {
       success: function(d){ console.log('Respuesta:', d); },
       dataType: 'text'
   });
   ```

---

### Problema 2: Las validaciones de servidor se ejecutan pero el mensaje aparece en la parte superior de la página, no junto al campo

**Síntomas:**
- Al hacer submit con datos inválidos, el mensaje de error aparece en el área de notificaciones general de la página (encabezado), no inline junto al campo específico.
- El campo afectado no muestra ningún indicador visual de error.

**Causa probable:**
El atributo **Associated Item** de la validación no está configurado, o está configurado con el nombre incorrecto del ítem. APEX necesita esta asociación para saber junto a qué campo mostrar el error inline.

**Solución:**
1. En el **Page Designer**, selecciona la validación problemática (bajo el nodo **Validations** de la página).
2. En el panel de propiedades, sección **Error**, verifica que **Associated Item** tenga el nombre exacto del ítem de página correspondiente (ej. `P5_PRECIO_UNITARIO`).
3. Adicionalmente, verifica que el ítem de página tenga habilitado el template de error inline: selecciona el ítem, y en **Template Options** confirma que **Inline Validation Errors** no esté deshabilitado.
4. Si el template de la página no soporta errores inline, considera cambiar el **Error Display Location** de la validación a `Inline with Field and in Notification`.
5. Guarda y vuelve a probar.

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, realiza las siguientes acciones para dejar el entorno ordenado:

### Verificar que no quedaron objetos temporales

```sql
-- Verificar que no hay objetos de prueba residuales en la BD
SELECT object_name, object_type
FROM   user_objects
WHERE  object_name LIKE '%TEST%'
   OR  object_name LIKE '%TEMP%'
ORDER BY object_type, object_name;
```

### Eliminar registros de prueba

```sql
-- Eliminar productos de prueba creados durante el laboratorio
DELETE FROM productos
WHERE  codigo_producto LIKE 'PROD-TEST-%'
   OR  codigo_producto LIKE 'TEST-%';

COMMIT;
```

### Guardar snapshot de la aplicación

1. En **App Builder**, navega a tu aplicación.
2. Haz clic en **Export / Import → Export**.
3. Selecciona **Application** y descarga el archivo `.sql` de exportación.
4. Guarda el archivo con el nombre: `app_lab06_completado_[tu_nombre].sql`.

> **Importante:** Este snapshot será el punto de partida para el Laboratorio 07. Consérvalo en un lugar seguro.

---

## 10. Resumen

En este laboratorio implementaste un conjunto completo de comportamientos dinámicos e interactivos en tu aplicación APEX, progresando desde lo declarativo hasta la integración programática:

| Técnica Implementada                          | Herramienta APEX Utilizada                        |
|-----------------------------------------------|---------------------------------------------------|
| Mostrar/ocultar campos según selección        | Dynamic Action: `Show` / `Hide` con condición     |
| Habilitar/deshabilitar botón según campo      | Dynamic Action: `Enable` / `Disable`              |
| Cálculo automático de totales                 | `Execute JavaScript Code` + `apex.item()` API     |
| Verificación AJAX de disponibilidad           | `apex.server.process()` + Application Process     |
| Validación de formato con regex               | `Execute JavaScript Code` + `apex.message` API    |
| Validaciones de servidor con mensajes claros  | Page Validations con PL/SQL                       |
| Confirmación personalizada antes de eliminar  | `apex.message.confirm()` API                      |

Los patrones aprendidos en este laboratorio son reutilizables en cualquier aplicación APEX: la arquitectura **Evento → Condición → Acción** es el fundamento de toda interactividad avanzada en la plataforma. La combinación de acciones declarativas con JavaScript personalizado y llamadas AJAX te permite construir experiencias de usuario modernas sin sacrificar la robustez de las validaciones en el servidor.

### Recursos Adicionales

- [Oracle APEX JavaScript API Reference](https://docs.oracle.com/en/database/oracle/apex/23.2/aexjs/)
- [Dynamic Actions — APEX Documentation 23.2](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/managing-dynamic-actions.html)
- [apex.server Namespace Reference](https://docs.oracle.com/en/database/oracle/apex/23.2/aexjs/apex.server.html)
- [APEX Page Validations Guide](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/understanding-validations.html)
- [Chrome DevTools — Network Tab](https://developer.chrome.com/docs/devtools/network/)

---
*Lab 06-00-01 — Curso Oracle APEX Intermedio — Versión 1.0 para APEX 23.2*
