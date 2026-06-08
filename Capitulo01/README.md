# Práctica 1: Crear primer workspace y aplicación básica.

## 1. Metadatos

| Atributo | Valor |
|---|---|
| **Duración estimada** | 32 minutos |
| **Complejidad** | Fácil |
| **Nivel Bloom** | Crear (*Create*) |
| **Laboratorio número** | 01-00-01 |
| **Módulo** | 1 — Fundamentos de Oracle APEX |
| **Versión APEX objetivo** | 23.2 |

---

## 2. Descripción General

Este laboratorio introduce al estudiante al ecosistema Oracle APEX partiendo desde cero conceptual hasta la primera aplicación funcional. Aplicando los conceptos de arquitectura estudiados en la lección 1.1 —motor de renderizado PL/SQL, ORDS como listener HTTP y esquema de datos de negocio— el estudiante creará un workspace, explorará las áreas principales del entorno de desarrollo y construirá una aplicación básica conectada al esquema **HR** de Oracle. Al finalizar, habrá completado el ciclo completo: acceder a datos reales, visualizarlos en un reporte y editar un registro a través de un formulario generado automáticamente.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio, el estudiante será capaz de:

- [ ] **Identificar** los componentes de la arquitectura de APEX (ORDS, motor PL/SQL, esquema de datos) observando cómo se manifiestan en el entorno de desarrollo real.
- [ ] **Crear y configurar** un workspace de APEX asociado a un esquema de base de datos Oracle existente.
- [ ] **Navegar** con fluidez por las áreas principales de APEX: App Builder, SQL Workshop y Administration.
- [ ] **Construir** una aplicación básica usando el asistente de creación, con al menos una página de reporte interactivo y una página de formulario CRUD.
- [ ] **Verificar** la conexión entre la aplicación y las tablas del esquema HR visualizando y editando datos en tiempo real.

---

## 4. Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| SQL básico (`SELECT`, `INSERT`, `UPDATE`, `DELETE`) | Conocimiento funcional |
| Concepto de tablas, esquemas y filas en Oracle Database | Familiaridad básica |
| Uso de navegador web (Chrome, Firefox o Edge) | Usuario habitual |
| Conceptos de arquitectura web (cliente-servidor, HTTP) | Comprensión conceptual |

### Acceso y recursos necesarios

| Recurso | Detalle |
|---|---|
| Cuenta en **apex.oracle.com** o instancia local/VM con APEX 23.2 | Obligatorio — ver Sección 5 |
| Esquema **HR** disponible en la base de datos (o esquema de demostración provisto por el instructor) | Obligatorio |
| Navegador web Chrome 110+, Firefox 110+ o Edge 110+ con JavaScript habilitado | Obligatorio |
| Acceso a **APEX Administration Services** (cuenta `internal`) | Necesario para crear el workspace |

> **Nota para el instructor:** Si los estudiantes trabajan en `apex.oracle.com`, el workspace ya estará creado y el esquema de datos será el esquema personal asignado. En ese caso, el estudiante puede omitir el **Paso 1** (creación del workspace) e iniciar desde el **Paso 2**. Asegúrese de que el esquema HR o un esquema equivalente con datos de empleados esté disponible antes de comenzar.

---

## 5. Entorno de Laboratorio

### Hardware mínimo requerido

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 10 GB | 50 GB SSD |
| Procesador | Intel Core i5 8ª gen / AMD Ryzen 5 | i7 / Ryzen 7 |
| Resolución de pantalla | 1280 × 768 | 1920 × 1080 |
| Conexión a Internet | 10 Mbps | 25 Mbps |

### Software requerido

| Software | Versión | Uso en este laboratorio |
|---|---|---|
| Oracle APEX | 23.2 o superior | Plataforma principal de desarrollo |
| Oracle Database | 19c / 21c (XE aceptable) | Motor de datos y esquema HR |
| ORDS (Oracle REST Data Services) | 23.2 o superior | Listener HTTP entre navegador y APEX |
| Navegador web | Chrome/Firefox/Edge 110+ | Interfaz de usuario de APEX |
| Oracle SQL Developer *(opcional)* | 23.1 o superior | Verificación de objetos en el esquema HR |

### Verificación del entorno antes de comenzar

Antes de iniciar el laboratorio, confirme que el entorno está operativo ejecutando las siguientes verificaciones:

**Opción A — Entorno en la nube (apex.oracle.com):**
1. Abra su navegador y navegue a `https://apex.oracle.com/en/`.
2. Haga clic en **Sign In** y acceda con sus credenciales de workspace ya provistas.
3. Si puede ver el **App Builder**, el entorno está listo. Continúe desde el **Paso 2**.

**Opción B — Instalación local o VM:**
```
# Verificar que ORDS esté activo (ejecutar en terminal del servidor)
curl -I http://localhost:8080/ords/

# Respuesta esperada: HTTP/1.1 200 OK  (o redirección 302)
```

```sql
-- Verificar que el esquema HR existe y tiene datos (ejecutar en SQL*Plus o SQL Developer)
SELECT COUNT(*) FROM hr.employees;
-- Resultado esperado: 107 (número de empleados en el esquema HR estándar)
```

> **Si el esquema HR no existe o no tiene datos**, ejecute el script de instalación provisto por el instructor (`hr_main.sql`) o siga las instrucciones de la sección de Solución de Problemas al final de este documento.

---

## 6. Pasos del Laboratorio

> **Tiempo sugerido por sección:**
> - Paso 1 (Crear workspace): ~8 min
> - Paso 2 (Explorar el entorno): ~7 min
> - Paso 3 (Crear la aplicación): ~10 min
> - Paso 4 (Verificar la aplicación): ~7 min

---

### Paso 1: Crear y configurar el workspace de APEX

**Objetivo:** Acceder a APEX Administration Services y crear un workspace asociado al esquema HR, comprendiendo que este workspace es el contenedor lógico donde residirán todas las aplicaciones del estudiante.

> **Si usa apex.oracle.com:** Este paso ya fue completado por la plataforma. Anote su nombre de workspace, usuario y contraseña, y continúe en el **Paso 2**.

#### Instrucciones

1. Abra su navegador y navegue a la URL de administración de APEX. En una instalación local estándar con ORDS, la URL es:
   ```
   http://localhost:8080/ords/apex_admin
   ```
   En una VM provista por el instructor, use la IP de la VM:
   ```
   http://<IP_DE_LA_VM>:8080/ords/apex_admin
   ```

2. En la pantalla de inicio de sesión de **APEX Administration Services**, ingrese:
   - **Username:** `admin` *(o el usuario administrativo provisto por el instructor)*
   - **Password:** La contraseña de administración de APEX configurada durante la instalación.
   
   Haga clic en **Sign In**.

   > **Contexto arquitectónico:** Está accediendo al motor de administración de APEX, que reside en el esquema `APEX_230100` dentro de Oracle Database. Las acciones que realice aquí modificarán filas en las tablas de metadatos de APEX.

3. En la pantalla de **APEX Administration Services**, localice y haga clic en el botón **Create Workspace** (puede estar en la sección *Manage Workspaces* o directamente visible en el panel principal).

4. Se abrirá el asistente de creación de workspace. Complete los campos de la siguiente manera:

   **Pantalla: Identify Workspace**
   | Campo | Valor a ingresar |
   |---|---|
   | **Workspace Name** | `LAB_ESTUDIANTE` *(o use su nombre, ej: `LAB_JUAN`)* |
   | **Workspace ID** | *(dejar en blanco — se asignará automáticamente)* |

   Haga clic en **Next**.

5. **Pantalla: Identify Schema**
   
   Aquí se define qué esquema de base de datos usará el workspace como *parsing schema* (esquema de análisis). Este es el esquema contra el cual se ejecutarán todas las consultas SQL de las aplicaciones.

   | Campo | Valor a ingresar |
   |---|---|
   | **Re-use existing schema?** | Seleccione **Yes** |
   | **Schema Name** | Seleccione o escriba `HR` |

   Haga clic en **Next**.

   > **Nota:** Si el esquema HR no aparece en la lista, puede ser que no tenga los permisos necesarios o que no esté desbloqueado. Consulte la sección de Solución de Problemas.

6. **Pantalla: Identify Administrator**
   
   Cree el usuario administrador del workspace:

   | Campo | Valor a ingresar |
   |---|---|
   | **Administrator Username** | `ADMIN` |
   | **Administrator Password** | `Oracle_APEX#2024` *(o la contraseña que indique el instructor)* |
   | **Confirm Password** | *(repita la contraseña)* |
   | **Administrator Email** | `admin@laboratorio.local` |

   Haga clic en **Next**.

7. **Pantalla: Confirm Request**
   
   Revise el resumen de configuración. Debe mostrar:
   - Workspace: `LAB_ESTUDIANTE`
   - Schema: `HR`
   - Administrator: `ADMIN`
   
   Haga clic en **Create Workspace**.

8. Aparecerá un mensaje de confirmación: *"Workspace created successfully"*. Haga clic en **Done**.

#### Resultado esperado

El workspace `LAB_ESTUDIANTE` ha sido creado y aparece en la lista de workspaces administrados. El sistema está listo para recibir el primer inicio de sesión de desarrollador.

#### Verificación

- En la lista de workspaces de Administration Services, localice `LAB_ESTUDIANTE` y confirme que el esquema asociado es `HR`.
- Haga clic en el ícono de engranaje o en el nombre del workspace para ver sus propiedades y verificar que el estado es **Active**.

---

### Paso 2: Iniciar sesión en el workspace y explorar el entorno

**Objetivo:** Iniciar sesión en el workspace recién creado e identificar las tres áreas principales de APEX (App Builder, SQL Workshop, Administration), relacionándolas con los componentes de arquitectura estudiados en la lección 1.1.

#### Instrucciones

1. Navegue a la URL del workspace de APEX. En instalaciones locales con ORDS:
   ```
   http://localhost:8080/ords/
   ```
   O en la URL provista por el instructor. Verá la pantalla de inicio de sesión del workspace.

2. Ingrese las credenciales del workspace:
   | Campo | Valor |
   |---|---|
   | **Workspace** | `LAB_ESTUDIANTE` |
   | **Username** | `ADMIN` |
   | **Password** | `Oracle_APEX#2024` |

   Haga clic en **Sign In**.

3. Al iniciar sesión por primera vez, APEX puede solicitar que cambie la contraseña. Si es así, establezca una contraseña que cumpla los requisitos de seguridad y continúe.

4. Observe la **pantalla de inicio del workspace** (Home). Identifique los tres bloques principales:

   | Área | Ícono/Sección | Propósito |
   |---|---|---|
   | **App Builder** | Ícono de bloques/aplicación | Crear y gestionar aplicaciones APEX |
   | **SQL Workshop** | Ícono de base de datos/SQL | Ejecutar SQL, gestionar objetos de BD |
   | **Team Development** | Ícono de equipo | Gestión de tareas y bugs (no se usa en este lab) |
   | **Administration** | Ícono de engranaje (esquina superior) | Configurar workspace, usuarios y seguridad |

   > **Conexión con la arquitectura:** El **App Builder** es la interfaz para trabajar con los metadatos de aplicaciones almacenados en el esquema `APEX_230100`. El **SQL Workshop** le da acceso directo al esquema de datos `HR` a través del motor de análisis SQL de APEX (que pasa por ORDS antes de llegar a Oracle Database).

5. Haga clic en **SQL Workshop** y luego en **SQL Commands**. Se abrirá un editor SQL en línea.

6. Ejecute la siguiente consulta para confirmar que el workspace tiene acceso al esquema HR:
   ```sql
   SELECT table_name, num_rows
   FROM   user_tables
   ORDER BY table_name;
   ```
   Haga clic en el botón **Run** (o presione `Ctrl+Enter`).

7. Observe los resultados. Debe ver las tablas del esquema HR, incluyendo: `COUNTRIES`, `DEPARTMENTS`, `EMPLOYEES`, `JOB_HISTORY`, `JOBS`, `LOCATIONS`, `REGIONS`.

8. Ejecute una segunda consulta para verificar los datos:
   ```sql
   SELECT employee_id,
          first_name,
          last_name,
          email,
          job_id,
          salary
   FROM   employees
   WHERE  ROWNUM <= 10
   ORDER BY employee_id;
   ```

9. Regrese a la pantalla de inicio del workspace haciendo clic en el logotipo de APEX en la esquina superior izquierda o en el enlace **Home**.

10. Explore brevemente **Administration** (ícono de engranaje en la barra superior derecha → **Manage Users and Groups**). Observe que puede crear usuarios adicionales para el workspace. **No realice cambios en este momento.**

#### Resultado esperado

- El comando SQL en el paso 6 devuelve 7 tablas del esquema HR.
- La consulta del paso 8 devuelve 10 filas de empleados con sus datos.
- Puede navegar entre App Builder, SQL Workshop y Administration sin errores.

#### Verificación

Confirme que la tabla `EMPLOYEES` tiene el número correcto de registros:
```sql
SELECT COUNT(*) AS total_empleados FROM employees;
-- Resultado esperado: 107
```

Si el resultado es 107, el esquema HR está correctamente instalado y el workspace tiene acceso completo.

---

### Paso 3: Crear la aplicación básica con el asistente

**Objetivo:** Utilizar el asistente de creación de aplicaciones de APEX para generar automáticamente una aplicación funcional con páginas de reporte y formulario conectadas a la tabla `EMPLOYEES` del esquema HR.

#### Instrucciones

1. Desde la pantalla de inicio del workspace, haga clic en **App Builder**.

2. Haga clic en el botón **Create** (botón azul, esquina superior derecha o centro de la pantalla si no hay aplicaciones aún).

3. En la pantalla **Create an Application**, seleccione la opción **New Application** (no *From a File*, no *From a Spreadsheet*).

4. Se abrirá el asistente principal de creación de aplicaciones. Complete los campos de la sección **Name**:

   | Campo | Valor |
   |---|---|
   | **Name** | `Gestión de Empleados HR` |
   | **Appearance** | Seleccione el tema **Redwood Light** (o el tema predeterminado disponible) |
   | **Application ID** | *(dejar en blanco — se asignará automáticamente)* |

5. En la sección **Pages**, el asistente muestra las páginas que se incluirán en la aplicación. Por defecto incluye una página de inicio (*Home*). Necesitamos agregar páginas para la tabla EMPLOYEES. Haga clic en **Add Page**.

6. En el diálogo **Add Page**, seleccione el tipo de página **Interactive Report** y complete:

   | Campo | Valor |
   |---|---|
   | **Page Type** | Interactive Report |
   | **Page Name** | `Empleados` |
   | **Table or View** | Seleccione `EMPLOYEES` de la lista desplegable |

   Haga clic en **Add Page**. El asistente automáticamente crea **dos páginas**: el reporte interactivo (lista) y el formulario de edición (detalle).

   > **¿Qué ocurre internamente?** El asistente está insertando filas en las tablas de metadatos del esquema `APEX_230100`. Está definiendo: una página de reporte que ejecutará `SELECT * FROM EMPLOYEES`, y una página de formulario con campos correspondientes a cada columna de la tabla. Todo esto sin escribir una línea de código.

7. Verifique que en la lista de páginas del asistente ahora aparecen:
   - **1 — Home** (página de inicio)
   - **2 — Empleados** (Interactive Report)
   - **3 — Empleados** (Form — creada automáticamente junto con la página 2)

8. Desplácese hacia abajo en el asistente para revisar la sección **Features** (Características). Asegúrese de que las siguientes opciones estén **marcadas**:

   | Característica | Estado recomendado |
   |---|---|
   | **Install Progressive Web App** | Desmarcar (no necesario para este lab) |
   | **Activity Reporting** | Dejar como está (opcional) |
   | **Access Control** | Desmarcar (simplifica el lab inicial) |

9. Revise la sección **Settings** al final del asistente:

   | Campo | Valor |
   |---|---|
   | **Authentication** | Application Express Accounts *(predeterminado)* |
   | **Language** | English *(o Spanish si está disponible)* |

10. Haga clic en el botón **Create Application** (botón azul en la parte inferior o superior del asistente).

11. APEX procesará la solicitud durante unos segundos. Verá una barra de progreso mientras el motor crea todas las páginas y objetos de la aplicación.

12. Al finalizar, será redirigido automáticamente a la **página de propiedades de la aplicación** dentro del App Builder, donde verá la lista de todas las páginas creadas.

#### Resultado esperado

La aplicación **"Gestión de Empleados HR"** ha sido creada exitosamente. En el App Builder, la vista de páginas de la aplicación muestra al menos:
- Página 1: Home
- Página 2: Empleados (Interactive Report)
- Página 3: Empleados (Form)
- Página 9999: Login Page (creada automáticamente)

#### Verificación

- En la barra superior del App Builder, confirme que el nombre de la aplicación es **"Gestión de Empleados HR"**.
- Verifique que el Application ID ha sido asignado (un número de 3-6 dígitos visible en la URL o en las propiedades de la aplicación).
- Haga clic en la página **2 — Empleados** para abrirla en el diseñador de páginas. Confirme que existe una región de tipo **Interactive Report** con la fuente de datos `SELECT * FROM EMPLOYEES` (o similar).

---

### Paso 4: Ejecutar y verificar la aplicación

**Objetivo:** Ejecutar la aplicación generada, verificar que los datos del esquema HR se muestran correctamente en el reporte interactivo, y completar el ciclo CRUD editando y guardando un registro de empleado.

#### Instrucciones

1. Desde la vista de páginas de la aplicación en el App Builder, localice el botón **Run Application** (ícono de triángulo/play, generalmente en la barra de herramientas superior derecha, o accesible desde el menú de la aplicación).

   Alternativamente, haga clic en el ícono de **Run** (▶) visible en la tarjeta de la aplicación en la lista del App Builder.

2. Se abrirá una nueva pestaña o ventana del navegador con la **pantalla de login de la aplicación**. Ingrese las credenciales:

   | Campo | Valor |
   |---|---|
   | **Username** | `ADMIN` |
   | **Password** | `Oracle_APEX#2024` *(la misma del workspace)* |

   Haga clic en **Sign In**.

3. Será redirigido a la **página Home** de la aplicación. Observe que APEX ha generado automáticamente una interfaz con navegación lateral o superior.

4. Haga clic en el menú de navegación para ir a la página **Empleados**. Dependiendo del tema seleccionado, puede aparecer como un ítem en el menú lateral izquierdo o en la barra de navegación superior.

5. La página de **Empleados** cargará el **Interactive Report** mostrando los datos de la tabla `EMPLOYEES`. Verifique:
   - Se muestran múltiples columnas: `EMPLOYEE_ID`, `FIRST_NAME`, `LAST_NAME`, `EMAIL`, `PHONE_NUMBER`, `HIRE_DATE`, `JOB_ID`, `SALARY`, `COMMISSION_PCT`, `MANAGER_ID`, `DEPARTMENT_ID`.
   - Se muestran registros de empleados reales del esquema HR.
   - La barra de herramientas del reporte interactivo incluye opciones de búsqueda, filtrado y descarga.

6. **Explorar las capacidades del Interactive Report:**
   
   a. En el campo de búsqueda (Search Bar) del reporte, escriba `King` y observe cómo el reporte filtra automáticamente los resultados sin recargar la página.
   
   b. Haga clic en el encabezado de la columna **SALARY** para ordenar los resultados por salario.
   
   c. Haga clic en el botón **Actions** (o el ícono de menú del reporte) y explore las opciones disponibles: *Filter*, *Columns*, *Format*, *Download*. No es necesario aplicar cambios permanentes.

7. **Probar el formulario de edición (operación UPDATE):**
   
   a. Localice cualquier fila del reporte. Haga clic en el ícono de lápiz (✏) o en el valor del `EMPLOYEE_ID` de la primera fila para abrir el formulario de edición.
   
   b. Se cargará la **página de formulario** con todos los campos del empleado seleccionado pre-poblados con sus datos actuales.
   
   c. Localice el campo **PHONE_NUMBER** y modifique el valor agregando un dígito o cambiando el formato. Por ejemplo, si el valor actual es `515.123.4567`, cámbielo a `515.123.4568`.
   
   d. Haga clic en el botón **Save** (o **Apply Changes**) para guardar los cambios.

8. APEX ejecutará el proceso de actualización (`UPDATE` en la tabla `EMPLOYEES`) y lo redirigirá de vuelta al reporte interactivo. Verifique que el cambio se refleja en el reporte.

9. **Verificar el cambio en la base de datos** (regrese a la pestaña del App Builder):
   
   Abra una nueva pestaña del navegador, navegue a **SQL Workshop → SQL Commands** y ejecute:
   ```sql
   SELECT employee_id,
          first_name,
          last_name,
          phone_number
   FROM   employees
   WHERE  employee_id = <ID_DEL_EMPLEADO_EDITADO>;
   ```
   Reemplace `<ID_DEL_EMPLEADO_EDITADO>` con el ID del empleado que modificó en el paso anterior. Confirme que el número de teléfono en la base de datos refleja el cambio realizado desde la aplicación.

10. **Probar la operación INSERT (crear nuevo registro):**
    
    a. Regrese a la pestaña de la aplicación.
    
    b. En la página del reporte **Empleados**, busque el botón **Create** (generalmente en la esquina superior derecha del reporte).
    
    c. Haga clic en **Create**. Se abrirá el formulario vacío para ingresar un nuevo empleado.
    
    d. Complete los campos mínimos requeridos:
    
    | Campo | Valor de prueba |
    |---|---|
    | **EMPLOYEE_ID** | `300` |
    | **FIRST_NAME** | `Estudiante` |
    | **LAST_NAME** | `APEX` |
    | **EMAIL** | `EAPEX` |
    | **HIRE_DATE** | `01/01/2024` |
    | **JOB_ID** | `IT_PROG` |
    
    e. Haga clic en **Create** para guardar el nuevo registro.

11. Verifique que el nuevo empleado aparece en el reporte interactivo. Si no lo ve inmediatamente, use la barra de búsqueda para buscar `Estudiante`.

#### Resultado esperado

- El reporte interactivo muestra correctamente los 107 empleados del esquema HR (más el nuevo registro creado en el paso 10, totalizando 108).
- El cambio de número de teléfono realizado en el paso 7 se refleja tanto en la aplicación como en la consulta directa a la base de datos.
- El nuevo registro del empleado "Estudiante APEX" es visible en el reporte.
- No se producen errores de APEX (mensajes en rojo) durante ninguna operación.

#### Verificación

Ejecute en SQL Workshop:
```sql
-- Verificar el total de empleados (debe ser 108 si el INSERT fue exitoso)
SELECT COUNT(*) AS total_empleados FROM employees;

-- Verificar el nuevo registro
SELECT employee_id, first_name, last_name, email
FROM   employees
WHERE  last_name = 'APEX';
```

---

## 7. Validación y Pruebas Finales

Una vez completados los cuatro pasos del laboratorio, realice las siguientes verificaciones de validación integral:

### Lista de verificación de completitud

| # | Criterio de validación | Comando/Acción de verificación | Resultado esperado |
|---|---|---|---|
| 1 | Workspace `LAB_ESTUDIANTE` existe y está activo | APEX Admin → Manage Workspaces | Workspace visible, estado Active |
| 2 | Workspace asociado al esquema HR | Propiedades del workspace | Schema: HR |
| 3 | Aplicación "Gestión de Empleados HR" creada | App Builder → lista de aplicaciones | Aplicación visible con ID asignado |
| 4 | Página de reporte muestra datos de EMPLOYEES | Ejecutar aplicación → página Empleados | 107+ filas visibles |
| 5 | Formulario de edición funciona (UPDATE) | Editar un empleado y guardar | Cambio reflejado en BD |
| 6 | Formulario de creación funciona (INSERT) | Crear empleado "Estudiante APEX" | Registro visible en reporte y BD |
| 7 | SQL Workshop tiene acceso al esquema HR | SQL Workshop → `SELECT COUNT(*) FROM employees` | Resultado: 108 |

### Prueba de integridad de la aplicación

Ejecute esta consulta final en SQL Workshop para confirmar el estado completo:

```sql
-- Resumen del estado del esquema HR después del laboratorio
SELECT 'EMPLOYEES'   AS tabla, COUNT(*) AS registros FROM employees   UNION ALL
SELECT 'DEPARTMENTS' AS tabla, COUNT(*) AS registros FROM departments  UNION ALL
SELECT 'JOBS'        AS tabla, COUNT(*) AS registros FROM jobs         UNION ALL
SELECT 'LOCATIONS'   AS tabla, COUNT(*) AS registros FROM locations
ORDER BY tabla;
```

Los resultados deben mostrar datos en todas las tablas, confirmando que el esquema HR está íntegro y accesible desde el workspace.

---

## 8. Solución de Problemas

### Problema 1: El esquema HR no aparece al crear el workspace o no tiene tablas

**Síntomas:**
- Al seleccionar el esquema en el asistente de creación del workspace, `HR` no aparece en la lista desplegable.
- O bien, la lista de tablas en SQL Workshop está vacía después de crear el workspace.
- La consulta `SELECT COUNT(*) FROM employees` devuelve error `ORA-00942: table or view does not exist`.

**Causa:**
El esquema HR de Oracle viene bloqueado (`ACCOUNT STATUS: LOCKED`) y sin tablas instaladas en instalaciones nuevas de Oracle Database XE. Es necesario desbloquearlo y ejecutar los scripts de instalación del esquema de demostración. Adicionalmente, el usuario HR puede no tener los privilegios de `CONNECT` y `RESOURCE` necesarios.

**Solución:**

Conéctese a Oracle Database como `SYSDBA` (desde SQL*Plus o SQL Developer) y ejecute:

```sql
-- Paso 1: Desbloquear el usuario HR y establecer contraseña
ALTER USER hr IDENTIFIED BY hr ACCOUNT UNLOCK;

-- Paso 2: Verificar que HR tiene los privilegios necesarios
GRANT CONNECT, RESOURCE TO hr;
GRANT UNLIMITED TABLESPACE TO hr;

-- Paso 3: Verificar el estado del usuario
SELECT username, account_status FROM dba_users WHERE username = 'HR';
-- Resultado esperado: HR | OPEN
```

Si las tablas del esquema HR no existen (schema instalado pero sin objetos), ejecute el script de instalación:
```sql
-- Conectarse como HR y ejecutar el script de demostración
-- (Ubicación típica en Oracle Database XE 21c)
-- $ORACLE_HOME/demo/schema/human_resources/hr_main.sql
@?/demo/schema/human_resources/hr_main.sql
```

Después de ejecutar el script, regrese a APEX Administration Services y vuelva a intentar la creación del workspace. Si el workspace ya fue creado sin el esquema correcto, elimínelo y recréelo.

---

### Problema 2: Error al guardar el formulario — "ORA-00001: unique constraint violated" o "ORA-01400: cannot insert NULL"

**Síntomas:**
- Al hacer clic en **Create** o **Save** en el formulario de empleados, aparece un mensaje de error rojo en la parte superior de la página con texto como: `Error saving record. ORA-00001: unique constraint (HR.EMP_EMAIL_UK) violated` o `ORA-01400: cannot insert NULL into ("HR"."EMPLOYEES"."EMAIL")`.
- El registro no se guarda y el formulario permanece abierto con los datos ingresados.

**Causa:**
La tabla `EMPLOYEES` del esquema HR tiene restricciones de integridad que APEX respeta y propaga al usuario. Los errores más comunes son:
- **ORA-00001 (unique constraint):** Se está intentando insertar un `EMPLOYEE_ID` o `EMAIL` que ya existe en la tabla. El campo `EMAIL` tiene una restricción `UNIQUE`.
- **ORA-01400 (NOT NULL):** Se dejó vacío un campo obligatorio. Los campos `EMPLOYEE_ID`, `LAST_NAME`, `EMAIL`, `HIRE_DATE` y `JOB_ID` son `NOT NULL` en el esquema HR.

**Solución:**

Para el error de **EMPLOYEE_ID duplicado**, use un ID que no exista en la tabla:
```sql
-- Consultar el máximo ID actual para elegir uno disponible
SELECT MAX(employee_id) + 1 AS siguiente_id_disponible
FROM   employees;
```
Use el valor devuelto como `EMPLOYEE_ID` en el formulario.

Para el error de **EMAIL único**, asegúrese de que el valor del campo `EMAIL` en el formulario sea único. El esquema HR usa el formato de iniciales en mayúsculas (ej: `SKING`, `NKOCHHAR`). Use un valor como `EAPEX2024` para garantizar unicidad.

Para errores de **campos NULL**, verifique que todos los campos marcados con asterisco (*) en el formulario tengan un valor antes de guardar. Los campos obligatorios mínimos son: `EMPLOYEE_ID`, `LAST_NAME`, `EMAIL`, `HIRE_DATE` y `JOB_ID`.

> **Tip de buenas prácticas:** En aplicaciones de producción, se configurarían validaciones en APEX que muestren mensajes amigables antes de intentar el DML. Esto se cubrirá en laboratorios posteriores del curso.

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, realice los siguientes pasos de limpieza para mantener el entorno ordenado:

### Eliminar el registro de prueba creado durante el laboratorio

El empleado "Estudiante APEX" (ID 300) fue creado como dato de prueba y no debe permanecer en el esquema HR para los laboratorios siguientes. Elimínelo desde la aplicación o directamente en SQL Workshop:

```sql
-- Eliminar el empleado de prueba creado durante el laboratorio
DELETE FROM employees
WHERE  last_name = 'APEX'
AND    first_name = 'Estudiante';

-- Confirmar la eliminación
COMMIT;

-- Verificar que el total volvió a 107
SELECT COUNT(*) AS total_empleados FROM employees;
-- Resultado esperado: 107
```

### Revertir el cambio de número de teléfono (opcional)

Si desea restaurar el número de teléfono original del empleado que modificó en el Paso 4:

```sql
-- Restaurar número de teléfono original (ajuste el employee_id según corresponda)
-- Ejemplo para el empleado con ID 100 (Steven King):
UPDATE employees
SET    phone_number = '515.123.4567'
WHERE  employee_id = 100;

COMMIT;
```

> **Nota:** El cambio exacto a revertir depende del empleado que cada estudiante haya editado. Si no recuerda el valor original, puede consultar el script de instalación del esquema HR o simplemente dejar el valor modificado, ya que no afecta los laboratorios siguientes.

### Cerrar sesión de la aplicación

1. En la aplicación ejecutada, haga clic en el menú de usuario (esquina superior derecha) y seleccione **Sign Out**.
2. Cierre la pestaña del navegador con la aplicación.

### Estado final del entorno

| Elemento | Estado esperado al finalizar |
|---|---|
| Workspace `LAB_ESTUDIANTE` | Activo y disponible para laboratorios siguientes |
| Aplicación "Gestión de Empleados HR" | Creada y funcional (se usará en labs posteriores) |
| Tabla `EMPLOYEES` | 107 registros (sin el empleado de prueba) |
| Sesión de la aplicación | Cerrada |

> **Importante:** **No elimine el workspace ni la aplicación.** Ambos serán utilizados como base en los laboratorios 02 y siguientes de este curso.

---

## 10. Resumen

### Lo que se logró en este laboratorio

En este laboratorio completó exitosamente su primer ciclo completo de desarrollo con Oracle APEX:

1. **Creó un workspace** de APEX asociado al esquema HR, experimentando directamente cómo APEX gestiona los contenedores de desarrollo a través de su motor de administración.

2. **Exploró el entorno de desarrollo**, identificando App Builder, SQL Workshop y Administration como las tres áreas principales, y comprobó la conexión entre el workspace y el esquema de datos Oracle.

3. **Construyó una aplicación funcional** usando el asistente de APEX, observando cómo el motor de metadatos genera automáticamente páginas de reporte y formulario sin necesidad de escribir código HTML, CSS o JavaScript.

4. **Verificó el ciclo completo de datos**, confirmando que los cambios realizados desde la interfaz de la aplicación se persisten directamente en las tablas del esquema HR de Oracle Database, validando la arquitectura integrada de APEX estudiada en la lección 1.1.

### Conceptos clave reforzados

| Concepto de la lección 1.1 | Manifestación práctica en el laboratorio |
|---|---|
| APEX reside en el esquema `APEX_230100` | Administration Services administra objetos en ese esquema |
| ORDS como listener HTTP | La URL `http://localhost:8080/ords/` es el punto de entrada ORDS |
| Metadatos de aplicación en tablas APEX | El asistente creó páginas insertando filas en tablas de metadatos |
| Parsing schema como conexión a datos de negocio | El workspace usa `HR` como esquema de análisis para todas las consultas |
| Aplicaciones portables y exportables | La aplicación puede exportarse como archivo SQL/JSON desde App Builder |

### Próximos pasos

En el **Laboratorio 01-00-02** profundizará en la navegación del App Builder, explorará el Page Designer en detalle y comenzará a personalizar las páginas de la aplicación creada hoy. Los conceptos de arquitectura y el workspace configurado en este laboratorio serán la base de todo el trabajo siguiente.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Documentación oficial APEX 23.2 — Guía de inicio | https://docs.oracle.com/en/database/oracle/apex/23.2/htmgs/index.html |
| Oracle APEX — Entender la arquitectura | https://docs.oracle.com/en/database/oracle/apex/23.2/aeusg/understanding-apex-architecture.html |
| Instalación del esquema HR de Oracle | https://docs.oracle.com/en/database/oracle/oracle-database/21/comsc/installing-sample-schemas.html |
| Oracle APEX en apex.oracle.com (acceso gratuito) | https://apex.oracle.com/en/learn/getting-started/ |
| Foro oficial de la comunidad APEX | https://forums.oracle.com/ords/apexds/domain/dev-community/category/apex |

---

*Fin del Laboratorio 01-00-01*
