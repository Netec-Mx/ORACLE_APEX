# Práctica 3: Configurar workspace, usuarios y políticas

## 1. Metadatos

| Atributo | Detalle |
|---|---|
| **Duración estimada** | 52 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear (Create) |
| **Módulo** | 3 — Administración de workspaces, usuarios y seguridad |
| **Versión APEX objetivo** | 23.2 |

---

## 2. Descripción General

En este laboratorio asumirás el rol de **administrador de instancia Oracle APEX** para gestionar el ciclo de vida completo de workspaces: crearás un workspace de desarrollo, lo asociarás a esquemas de base de datos, crearás usuarios con distintos niveles de acceso y verificarás las restricciones de cada rol. Posteriormente configurarás políticas de seguridad a nivel de workspace, explorarás los logs de actividad para auditar acciones y ejecutarás tareas de mantenimiento operativo como la purga de sesiones antiguas. Al finalizar, habrás construido un entorno de desarrollo correctamente aislado, seguro y monitoreable, aplicando directamente los conceptos de la Lección 3.1.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear y configurar workspaces desde la consola **APEX Administration Services** (Internal), asociando esquemas de base de datos existentes o nuevos.
- [ ] Crear usuarios de APEX con roles diferenciados (Desarrollador, Administrador de Workspace) y verificar las restricciones de acceso de cada rol.
- [ ] Asociar múltiples esquemas a un workspace y validar el acceso a objetos de base de datos desde el entorno de desarrollo.
- [ ] Configurar políticas de seguridad del workspace: política de contraseñas, expiración de sesión y número máximo de sesiones concurrentes.
- [ ] Utilizar los reportes de actividad y herramientas de mantenimiento de APEX Administration para auditar y mantener el entorno operativo.

---

## 4. Prerrequisitos

### Conocimientos previos

| Área | Nivel requerido |
|---|---|
| Conceptos de workspace en Oracle APEX (Lección 3.1) | Comprensión básica |
| Gestión de usuarios en Oracle Database (`CREATE USER`, `GRANT`) | Básico |
| Navegación en APEX App Builder | Completado Lab 01-00-01 |
| SQL básico (consultas `SELECT` sobre vistas del diccionario) | Básico |

### Acceso y recursos necesarios

| Recurso | Detalle |
|---|---|
| Cuenta **internal** de APEX Administration Services | URL: `https://<host>/apex/apex_admin` |
| Acceso DBA a Oracle Database (SQL Developer o SQL*Plus) | Para crear esquemas previos al laboratorio |
| Al menos dos esquemas disponibles: `LAB_SCHEMA_A` y `LAB_SCHEMA_B` | Creados en el paso de configuración del entorno |
| Navegador Chrome 110+ con JavaScript habilitado | Requerido |

> **⚠️ Nota para entornos apex.oracle.com:** La cuenta `internal` no está disponible en `apex.oracle.com`. Este laboratorio requiere una instancia con acceso administrativo completo. Usa la VM preconfigurada provista por el instructor o una instancia OCI con APEX 23.2 instalado.

---

## 5. Entorno del Laboratorio

### Hardware recomendado

| Componente | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| Almacenamiento libre | 50 GB | 100 GB SSD |
| Procesador | Intel i5 8ª gen / Ryzen 5 | Intel i7 / Ryzen 7 |
| Resolución de pantalla | 1280×768 | 1920×1080 |

### Software requerido

| Software | Versión | Uso en este lab |
|---|---|---|
| Oracle APEX | 23.2 | Consola de administración y workspace |
| Oracle Database | 19c / 21c XE | Creación de esquemas de prueba |
| Oracle SQL Developer | 23.1+ | Ejecución de scripts SQL de preparación |
| Google Chrome | 110+ | Acceso a APEX |
| Oracle REST Data Services (ORDS) | 23.2+ | Capa de acceso HTTP a APEX |

### Configuración inicial del entorno

Antes de iniciar los pasos del laboratorio, ejecuta los siguientes scripts en **SQL Developer** conectado como `SYS AS SYSDBA` (o usuario con privilegios DBA) para preparar los esquemas de base de datos que usarás durante la práctica.

```sql
-- ============================================================
-- SCRIPT DE PREPARACIÓN DEL ENTORNO - Lab 03-00-01
-- Ejecutar como SYS AS SYSDBA o usuario DBA equivalente
-- ============================================================

-- Crear esquema principal del workspace de desarrollo
CREATE USER lab_schema_a IDENTIFIED BY "Lab_Pass_2024#"
    DEFAULT TABLESPACE users
    TEMPORARY TABLESPACE temp
    QUOTA 100M ON users;

GRANT CONNECT, RESOURCE TO lab_schema_a;
GRANT CREATE SESSION TO lab_schema_a;
GRANT CREATE TABLE TO lab_schema_a;
GRANT CREATE VIEW TO lab_schema_a;
GRANT CREATE PROCEDURE TO lab_schema_a;
GRANT CREATE SEQUENCE TO lab_schema_a;

-- Crear esquema secundario (para demostrar asociación múltiple)
CREATE USER lab_schema_b IDENTIFIED BY "Lab_Pass_2024#"
    DEFAULT TABLESPACE users
    TEMPORARY TABLESPACE temp
    QUOTA 50M ON users;

GRANT CONNECT, RESOURCE TO lab_schema_b;
GRANT CREATE SESSION TO lab_schema_b;
GRANT CREATE TABLE TO lab_schema_b;

-- Crear tabla de prueba en esquema A
CONN lab_schema_a/"Lab_Pass_2024#"

CREATE TABLE empleados (
    emp_id     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre     VARCHAR2(100) NOT NULL,
    puesto     VARCHAR2(50),
    salario    NUMBER(10,2),
    fecha_alta DATE DEFAULT SYSDATE
);

INSERT INTO empleados (nombre, puesto, salario)
VALUES ('Ana García', 'Desarrolladora Senior', 85000);
INSERT INTO empleados (nombre, puesto, salario)
VALUES ('Carlos López', 'Analista de Datos', 72000);
INSERT INTO empleados (nombre, puesto, salario)
VALUES ('María Torres', 'Administradora de BD', 90000);

COMMIT;

-- Crear tabla de prueba en esquema B
CONN lab_schema_b/"Lab_Pass_2024#"

CREATE TABLE productos (
    prod_id    NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre     VARCHAR2(200) NOT NULL,
    categoria  VARCHAR2(50),
    precio     NUMBER(10,2)
);

INSERT INTO productos (nombre, categoria, precio)
VALUES ('Laptop Pro 15', 'Electrónica', 1299.99);
INSERT INTO productos (nombre, categoria, precio)
VALUES ('Monitor 4K', 'Electrónica', 449.99);

COMMIT;
```

> **✅ Verificación de preparación:** En SQL Developer, ejecuta `SELECT username, account_status FROM dba_users WHERE username IN ('LAB_SCHEMA_A', 'LAB_SCHEMA_B');` y confirma que ambos usuarios aparecen con estado `OPEN`.

---

## 6. Pasos del Laboratorio

---

### Paso 1 — Acceder a APEX Administration Services (Internal)

**Objetivo:** Iniciar sesión en la consola de administración global de APEX utilizando la cuenta `internal` y familiarizarse con el panel de administración.

#### Instrucciones

1. Abre Google Chrome y navega a la URL de administración de tu instancia APEX:
   ```
   https://<tu-host-apex>/apex/apex_admin
   ```
   *Ejemplo con ORDS local:* `http://localhost:8080/apex/apex_admin`

2. En la pantalla de inicio de sesión de **APEX Administration Services**, ingresa:
   - **Username:** `internal`
   - **Password:** *(contraseña establecida durante la instalación de APEX)*

3. Haz clic en **Sign In**.

4. Una vez dentro, observa el panel principal de **APEX Administration**. Identifica las siguientes secciones principales:
   - **Manage Workspaces** — gestión del ciclo de vida de workspaces.
   - **Manage Instance** — configuración global de la instancia APEX.
   - **Monitor Activity** — reportes de uso y logs de actividad.

5. Haz clic en **Manage Workspaces** y anota cuántos workspaces existen actualmente en la instancia.

#### Resultado esperado

Debes ver el panel de administración de APEX sin errores de autenticación. La sección **Manage Workspaces** debe mostrar al menos el workspace `INTERNAL` y cualquier workspace preexistente en la instancia.

#### Verificación

En la consola de administración, navega a **Monitor Activity → Workspace Activity** y confirma que tu sesión de inicio de sesión aparece registrada con el usuario `INTERNAL` y la marca de tiempo actual.

---

### Paso 2 — Crear un nuevo workspace de desarrollo

**Objetivo:** Crear el workspace `DEVTEAM_WS` desde la consola de administración, asociándolo al esquema `LAB_SCHEMA_A` y configurando el primer usuario administrador del workspace.

#### Instrucciones

1. En el panel de **APEX Administration**, haz clic en **Manage Workspaces**.

2. Haz clic en **Create Workspace**.

3. En el asistente de creación, selecciona la opción **Existing Schema** (ya que `LAB_SCHEMA_A` fue creado en el paso de preparación) y haz clic en **Next**.

4. Completa los campos del formulario con los siguientes valores:

   | Campo | Valor a ingresar |
   |---|---|
   | **Database Schema** | `LAB_SCHEMA_A` |
   | **Schema Password** | `Lab_Pass_2024#` |
   | **Space Quota (MB)** | `100` |

   Haz clic en **Next**.

5. En la sección de información del workspace, ingresa:

   | Campo | Valor a ingresar |
   |---|---|
   | **Workspace Name** | `DEVTEAM_WS` |
   | **Workspace Description** | `Workspace de desarrollo - Laboratorio 03` |

   Haz clic en **Next**.

6. En la sección del administrador del workspace, completa:

   | Campo | Valor a ingresar |
   |---|---|
   | **Administrator Username** | `ws_admin` |
   | **Administrator Password** | `Admin_Dev_2024#` |
   | **Confirm Password** | `Admin_Dev_2024#` |
   | **Administrator Email** | `ws_admin@labapex.local` |
   | **First Name** | `Workspace` |
   | **Last Name** | `Admin` |

   Haz clic en **Next**.

7. Revisa el resumen de configuración. Confirma que todos los datos son correctos y haz clic en **Create Workspace**.

8. Espera a que aparezca el mensaje de confirmación: *"Workspace DEVTEAM_WS created successfully."*

#### Resultado esperado

El workspace `DEVTEAM_WS` aparece en la lista de workspaces de la consola de administración con estado **Active**. El esquema `LAB_SCHEMA_A` aparece como esquema asociado.

#### Verificación

Ejecuta la siguiente consulta en SQL Developer para confirmar el registro del workspace en el repositorio de metadatos de APEX:

```sql
-- Verificar workspace creado en el repositorio APEX
-- Ejecutar como DBA o usuario con acceso a vistas APEX_*
SELECT workspace_id,
       workspace,
       schemas_on_workspace,
       num_apex_users,
       created_on,
       allow_app_building_yn
FROM   apex_workspaces
WHERE  workspace = 'DEVTEAM_WS';
```

Debes obtener exactamente **1 fila** con `WORKSPACE = 'DEVTEAM_WS'` y `SCHEMAS_ON_WORKSPACE` incluyendo `LAB_SCHEMA_A`.

---

### Paso 3 — Crear usuarios con diferentes roles en el workspace

**Objetivo:** Crear tres usuarios dentro de `DEVTEAM_WS` con roles diferenciados: un segundo administrador de workspace, un desarrollador estándar y un usuario de solo lectura. Verificar las diferencias de acceso entre cada rol.

#### Instrucciones

**Parte A — Iniciar sesión en el workspace como administrador**

1. Abre una **nueva pestaña** en Chrome y navega a la URL de tu instancia APEX (sin `/apex_admin`):
   ```
   https://<tu-host-apex>/apex
   ```

2. En la pantalla de login, ingresa:
   - **Workspace:** `DEVTEAM_WS`
   - **Username:** `ws_admin`
   - **Password:** `Admin_Dev_2024#`

3. Haz clic en **Sign In**. Debes acceder al **APEX App Builder** del workspace `DEVTEAM_WS`.

**Parte B — Crear usuario Desarrollador**

4. En el menú superior derecho, haz clic en el ícono de administración (engranaje) y selecciona **Manage Users and Groups**.

5. Haz clic en **Create User**.

6. Completa el formulario con los siguientes valores:

   | Campo | Valor |
   |---|---|
   | **Username** | `dev_juan` |
   | **Email Address** | `juan.developer@labapex.local` |
   | **First Name** | `Juan` |
   | **Last Name** | `Pérez` |
   | **Password** | `Dev_2024#Pass` |
   | **Confirm Password** | `Dev_2024#Pass` |
   | **User is a workspace administrator** | **No** (desactivado) |
   | **User is a developer** | **Yes** (activado) |
   | **Account Availability** | **Unlocked** |

7. Haz clic en **Create User**. Confirma el mensaje de éxito.

**Parte C — Crear segundo Administrador de Workspace**

8. Repite el proceso (clic en **Create User**) con los siguientes datos:

   | Campo | Valor |
   |---|---|
   | **Username** | `ws_admin2` |
   | **Email Address** | `admin2@labapex.local` |
   | **First Name** | `Laura` |
   | **Last Name** | `Martínez` |
   | **Password** | `Admin2_2024#` |
   | **Confirm Password** | `Admin2_2024#` |
   | **User is a workspace administrator** | **Yes** (activado) |
   | **User is a developer** | **Yes** (activado) |
   | **Account Availability** | **Unlocked** |

9. Haz clic en **Create User**.

**Parte D — Verificar restricciones del rol Desarrollador**

10. Abre una ventana de **incógnito** (Ctrl+Shift+N) e inicia sesión en el workspace `DEVTEAM_WS` con las credenciales de `dev_juan`.

11. Dentro del App Builder, intenta navegar a **Manage Users and Groups** (menú de engranaje). Observa que esta opción **no está disponible** para el rol Developer.

12. Confirma que `dev_juan` **sí puede** acceder a **SQL Workshop** y **App Builder** (funcionalidades disponibles para desarrolladores).

#### Resultado esperado

- `dev_juan` puede acceder a SQL Workshop y App Builder pero **no** puede gestionar usuarios ni configurar el workspace.
- `ws_admin2` tiene acceso completo a todas las funcionalidades de administración del workspace.
- La lista de usuarios en **Manage Users and Groups** muestra 3 usuarios: `ws_admin`, `ws_admin2` y `dev_juan`.

#### Verificación

Ejecuta la siguiente consulta para confirmar los usuarios creados:

```sql
-- Verificar usuarios del workspace DEVTEAM_WS
SELECT workspace,
       user_name,
       email,
       is_admin,
       is_developer,
       account_locked,
       date_created
FROM   apex_workspace_apex_users
WHERE  workspace = 'DEVTEAM_WS'
ORDER  BY user_name;
```

Debes obtener **3 filas**: `dev_juan` (is_admin=N, is_developer=Y), `ws_admin` (is_admin=Y) y `ws_admin2` (is_admin=Y, is_developer=Y).

---

### Paso 4 — Asociar un esquema adicional al workspace

**Objetivo:** Agregar `LAB_SCHEMA_B` como esquema secundario del workspace `DEVTEAM_WS` y verificar que los desarrolladores pueden consultar objetos de ambos esquemas desde SQL Workshop.

#### Instrucciones

1. Regresa a la pestaña con la sesión de `ws_admin` en el workspace `DEVTEAM_WS`.

2. Haz clic en el ícono de engranaje (administración) y selecciona **Manage Workspaces** → **Manage Workspace to Schema Assignments**.

   > *Alternativa:* Si estás en la consola de administración `internal`, ve a **Manage Workspaces → Manage Workspace to Schema Assignments**.

3. Haz clic en **Add Schema to Workspace**.

4. Completa el formulario:

   | Campo | Valor |
   |---|---|
   | **Workspace** | `DEVTEAM_WS` |
   | **Schema Name** | `LAB_SCHEMA_B` |

5. Haz clic en **Add Schema**.

6. Confirma que `LAB_SCHEMA_B` aparece ahora en la lista de esquemas asociados a `DEVTEAM_WS`.

7. Para verificar el acceso, ve a **SQL Workshop → SQL Commands** (como `ws_admin` o `dev_juan`).

8. En el selector de esquema (parte superior del SQL Workshop), confirma que puedes cambiar entre `LAB_SCHEMA_A` y `LAB_SCHEMA_B`.

9. Con el esquema `LAB_SCHEMA_B` seleccionado, ejecuta:

   ```sql
   SELECT * FROM productos;
   ```

10. Con el esquema `LAB_SCHEMA_A` seleccionado, ejecuta:

    ```sql
    SELECT * FROM empleados;
    ```

#### Resultado esperado

- Ambas consultas retornan los registros insertados durante la preparación del entorno.
- El selector de esquema en SQL Workshop muestra tanto `LAB_SCHEMA_A` como `LAB_SCHEMA_B`.
- Los 3 registros de `empleados` y los 2 registros de `productos` son visibles correctamente.

#### Verificación

Desde la consola `internal`, navega a **Manage Workspaces → Existing Workspaces** y haz clic en `DEVTEAM_WS`. En la sección de detalles, confirma que el campo **Schemas** lista ambos: `LAB_SCHEMA_A, LAB_SCHEMA_B`.

---

### Paso 5 — Configurar políticas de seguridad del workspace

**Objetivo:** Configurar políticas de seguridad a nivel de instancia y de workspace: política de contraseñas, tiempo de expiración de sesión y número máximo de sesiones concurrentes.

#### Instrucciones

**Parte A — Configurar políticas a nivel de instancia (desde Internal)**

1. Regresa a la consola de **APEX Administration Services** (pestaña con sesión `internal`).

2. Haz clic en **Manage Instance → Instance Settings**.

3. Navega a la sección **Security**. Configura los siguientes parámetros:

   | Parámetro | Valor a configurar | Justificación |
   |---|---|---|
   | **Require HTTPS** | `Yes` (si el entorno lo soporta) | Cifrado en tránsito |
   | **Maximum Login Failures Allowed** | `5` | Protección contra fuerza bruta |
   | **Account Locking Duration (minutes)** | `15` | Bloqueo temporal tras intentos fallidos |
   | **Session Timeout (seconds)** | `3600` | 1 hora de inactividad máxima |

4. Haz clic en **Apply Changes**.

**Parte B — Configurar política de contraseñas del workspace**

5. En la consola `internal`, navega a **Manage Workspaces → Existing Workspaces**.

6. Haz clic en el nombre `DEVTEAM_WS` para abrir sus detalles.

7. Haz clic en **Edit Workspace** y navega a la sección de configuración de seguridad del workspace.

8. Configura los siguientes parámetros de contraseña:

   | Parámetro | Valor |
   |---|---|
   | **Minimum Password Length** | `10` |
   | **Password Must Contain At Least One Uppercase Character** | `Yes` |
   | **Password Must Contain At Least One Lowercase Character** | `Yes` |
   | **Password Must Contain At Least One Numeric Character** | `Yes` |
   | **Password Must Contain At Least One Punctuation Character** | `Yes` |
   | **Password Cannot Contain Username** | `Yes` |

9. Haz clic en **Apply Changes**.

**Parte C — Verificar la política de contraseñas**

10. En el workspace `DEVTEAM_WS` (sesión de `ws_admin`), intenta cambiar la contraseña de `dev_juan` a una contraseña débil: `password`.

11. Navega a **Manage Users and Groups**, edita el usuario `dev_juan` e intenta establecer la contraseña `password`.

12. Observa el **mensaje de error** que indica que la contraseña no cumple con la política configurada.

13. Establece en cambio la contraseña: `Dev_NewPass_2024#` (que sí cumple la política).

14. Haz clic en **Apply Changes** y confirma el éxito.

#### Resultado esperado

- Los parámetros de seguridad de la instancia se guardan correctamente.
- Al intentar establecer la contraseña `password` para `dev_juan`, APEX muestra un error de validación indicando que no cumple la política de complejidad.
- La contraseña `Dev_NewPass_2024#` se acepta correctamente.

#### Verificación

Cierra la sesión de `dev_juan` (en la ventana de incógnito) e intenta iniciar sesión con la contraseña antigua `Dev_2024#Pass`. Debe fallar. Inicia sesión con `Dev_NewPass_2024#`. Debe funcionar correctamente.

---

### Paso 6 — Explorar logs de actividad y reportes de uso

**Objetivo:** Utilizar las herramientas de monitoreo de APEX Administration para auditar las acciones realizadas durante el laboratorio y generar un reporte de uso del workspace.

#### Instrucciones

**Parte A — Revisar el log de actividad del workspace**

1. En la consola de **APEX Administration Services** (sesión `internal`), navega a **Monitor Activity**.

2. Haz clic en **Workspace Activity**.

3. En el filtro de workspace, selecciona `DEVTEAM_WS`.

4. Observa las entradas del log. Deberías ver registradas las siguientes acciones:
   - Inicio de sesión de `ws_admin`
   - Inicio de sesión de `dev_juan`
   - Inicio de sesión de `ws_admin2`
   - Intentos de inicio de sesión fallidos (si los hubo)

5. Haz clic en cualquier entrada para ver el detalle completo: usuario, dirección IP, timestamp, tipo de acción.

**Parte B — Revisar el reporte de uso de la instancia**

6. Navega a **Monitor Activity → Page Views by Workspace**.

7. Observa el reporte que muestra el número de vistas de página por workspace. Confirma que `DEVTEAM_WS` aparece con actividad reciente.

8. Navega a **Monitor Activity → Login Attempts**.

9. Filtra por workspace `DEVTEAM_WS`. Confirma que aparecen los intentos de inicio de sesión exitosos y (si aplica) los fallidos con la contraseña débil del paso anterior.

**Parte C — Consultar logs desde SQL**

10. En SQL Developer (como DBA), ejecuta las siguientes consultas para acceder a los logs desde la base de datos:

```sql
-- Ver actividad reciente en el workspace DEVTEAM_WS
SELECT workspace,
       apex_user,
       ip_address,
       time_stamp,
       page_id,
       application_id
FROM   apex_workspace_activity_log
WHERE  workspace = 'DEVTEAM_WS'
ORDER  BY time_stamp DESC
FETCH FIRST 20 ROWS ONLY;
```

```sql
-- Ver intentos de inicio de sesión (exitosos y fallidos)
SELECT workspace,
       login_name,
       authentication_result,
       ip_address,
       time_stamp
FROM   apex_workspace_access_log
WHERE  workspace = 'DEVTEAM_WS'
ORDER  BY time_stamp DESC
FETCH FIRST 20 ROWS ONLY;
```

```sql
-- Resumen de usuarios activos por workspace
SELECT workspace,
       user_name,
       email,
       last_login,
       is_admin,
       account_locked
FROM   apex_workspace_apex_users
WHERE  workspace = 'DEVTEAM_WS'
ORDER  BY last_login DESC NULLS LAST;
```

#### Resultado esperado

- El reporte **Workspace Activity** muestra al menos 5-8 entradas correspondientes a las acciones realizadas durante el laboratorio.
- El reporte **Login Attempts** muestra tanto los intentos exitosos como el intento fallido con la contraseña débil (si se realizó).
- Las consultas SQL retornan datos coherentes con las actividades del laboratorio.

#### Verificación

Confirma que la columna `AUTHENTICATION_RESULT` en `apex_workspace_access_log` muestra `Authentication Succeeded` para los inicios de sesión exitosos y `Authentication Failed` para el intento con contraseña incorrecta.

---

### Paso 7 — Tareas de mantenimiento operativo

**Objetivo:** Ejecutar tareas de mantenimiento del workspace: purgar sesiones antiguas, limpiar logs de actividad y exportar la configuración del workspace como respaldo.

#### Instrucciones

**Parte A — Gestión y purga de sesiones**

1. En la consola `internal`, navega a **Monitor Activity → Active Sessions**.

2. Observa las sesiones activas actuales en la instancia. Identifica las sesiones del workspace `DEVTEAM_WS`.

3. Para simular la purga de sesiones antiguas, navega a **Manage Instance → Session Management**.

4. Revisa la configuración de **Purge Sessions Older Than (days)**. Establece el valor en `1` para pruebas de laboratorio.

5. Haz clic en **Purge Sessions** y confirma la acción en el diálogo de confirmación.

6. Observa el mensaje de confirmación indicando cuántas sesiones fueron purgadas.

**Parte B — Limpieza de logs de actividad**

7. Navega a **Manage Instance → Log Settings** (o **Purge Activity Log**).

8. Revisa la configuración de retención de logs. Observa el parámetro **Days to Retain Activity Log**.

9. Para propósitos del laboratorio, **no ejecutes** la purga de logs completa (esto eliminaría la evidencia de las actividades del lab). En su lugar, documenta los valores actuales de retención:

   ```
   Activity Log Retention: _____ días
   Click Count Log Retention: _____ días
   ```

> **📝 Nota:** En un entorno de producción, se recomienda mantener los logs de actividad entre 30 y 90 días dependiendo de los requisitos de auditoría de la organización.

**Parte C — Exportar configuración del workspace**

10. En la consola `internal`, navega a **Manage Workspaces → Export Workspace**.

11. Selecciona el workspace `DEVTEAM_WS` en el selector.

12. Configura las opciones de exportación:

    | Opción | Valor |
    |---|---|
    | **Export Workspace** | `DEVTEAM_WS` |
    | **Include Team Development** | `No` |
    | **Export Public Themes** | `No` |

13. Haz clic en **Export Workspace** y guarda el archivo `.sql` generado en tu equipo local.

14. Abre el archivo exportado en un editor de texto (VS Code) y observa su estructura. Identifica:
    - El nombre del workspace en el script.
    - El esquema asociado.
    - Los metadatos de configuración del workspace.

#### Resultado esperado

- La purga de sesiones muestra un mensaje indicando el número de sesiones eliminadas.
- El archivo de exportación del workspace se descarga correctamente como un archivo `.sql`.
- El archivo exportado contiene el script de re-creación del workspace `DEVTEAM_WS` con todos sus metadatos de configuración.

#### Verificación

Abre el archivo `.sql` exportado y busca la cadena `DEVTEAM_WS`. Confirma que aparece en el script de creación del workspace. Verifica también que `LAB_SCHEMA_A` aparece referenciado como esquema asociado.

---

## 7. Validación y Pruebas Finales

Ejecuta las siguientes verificaciones para confirmar que todos los objetivos del laboratorio han sido completados exitosamente.

### Lista de verificación final

```sql
-- ============================================================
-- SCRIPT DE VALIDACIÓN FINAL - Lab 03-00-01
-- Ejecutar como DBA en SQL Developer
-- ============================================================

-- Verificación 1: Workspace DEVTEAM_WS existe y está activo
SELECT 'WORKSPACE_EXISTS' AS verificacion,
       CASE WHEN COUNT(*) = 1 THEN 'PASS' ELSE 'FAIL' END AS resultado
FROM   apex_workspaces
WHERE  workspace = 'DEVTEAM_WS';

-- Verificación 2: Ambos esquemas asociados al workspace
SELECT 'SCHEMAS_ASSOCIATED' AS verificacion,
       schemas_on_workspace AS detalle
FROM   apex_workspaces
WHERE  workspace = 'DEVTEAM_WS';

-- Verificación 3: Tres usuarios creados en el workspace
SELECT 'USER_COUNT' AS verificacion,
       COUNT(*) AS total_usuarios,
       CASE WHEN COUNT(*) = 3 THEN 'PASS' ELSE 'FAIL' END AS resultado
FROM   apex_workspace_apex_users
WHERE  workspace = 'DEVTEAM_WS';

-- Verificación 4: Roles correctamente asignados
SELECT user_name,
       is_admin,
       is_developer,
       account_locked
FROM   apex_workspace_apex_users
WHERE  workspace = 'DEVTEAM_WS'
ORDER  BY user_name;

-- Verificación 5: Actividad registrada en logs
SELECT 'ACTIVITY_LOGGED' AS verificacion,
       COUNT(*) AS total_eventos,
       CASE WHEN COUNT(*) > 0 THEN 'PASS' ELSE 'FAIL' END AS resultado
FROM   apex_workspace_access_log
WHERE  workspace = 'DEVTEAM_WS';
```

### Resultados esperados de validación

| Verificación | Resultado esperado |
|---|---|
| `WORKSPACE_EXISTS` | `PASS` — 1 fila retornada |
| `SCHEMAS_ASSOCIATED` | Muestra `LAB_SCHEMA_A, LAB_SCHEMA_B` |
| `USER_COUNT` | `PASS` — total_usuarios = 3 |
| Roles de usuarios | `ws_admin`: is_admin=Y; `ws_admin2`: is_admin=Y, is_developer=Y; `dev_juan`: is_admin=N, is_developer=Y |
| `ACTIVITY_LOGGED` | `PASS` — al menos 1 evento registrado |

### Prueba funcional de acceso por roles

| Acción | Usuario `dev_juan` | Usuario `ws_admin` |
|---|---|---|
| Acceder a App Builder | ✅ Permitido | ✅ Permitido |
| Acceder a SQL Workshop | ✅ Permitido | ✅ Permitido |
| Gestionar usuarios del workspace | ❌ No disponible | ✅ Permitido |
| Configurar políticas del workspace | ❌ No disponible | ✅ Permitido |
| Ver reportes de actividad del workspace | ❌ No disponible | ✅ Permitido |

---

## 8. Solución de Problemas

### Problema 1: Error al crear el workspace — "Schema does not exist or insufficient privileges"

**Síntoma:** Al intentar crear el workspace `DEVTEAM_WS` con el esquema `LAB_SCHEMA_A`, el asistente muestra el error: *"ORA-01435: user does not exist"* o *"Schema LAB_SCHEMA_A does not exist"*, y el proceso de creación falla en el paso de asignación de esquema.

**Causa:** El esquema `LAB_SCHEMA_A` no fue creado correctamente durante la preparación del entorno, o fue creado con un nombre diferente (mayúsculas/minúsculas). Oracle Database almacena los nombres de usuarios/esquemas en mayúsculas por defecto, pero el script de preparación puede haber fallado si el usuario DBA no tenía los privilegios necesarios o si ya existía un usuario con ese nombre en estado `LOCKED` o `EXPIRED`.

**Solución:**

```sql
-- Paso 1: Verificar si el esquema existe y su estado actual
SELECT username, account_status, created
FROM   dba_users
WHERE  username = 'LAB_SCHEMA_A';

-- Paso 2a: Si no existe, crearlo (ejecutar como SYS AS SYSDBA)
CREATE USER lab_schema_a IDENTIFIED BY "Lab_Pass_2024#"
    DEFAULT TABLESPACE users
    TEMPORARY TABLESPACE temp
    QUOTA 100M ON users;

GRANT CONNECT, RESOURCE TO lab_schema_a;

-- Paso 2b: Si existe pero está LOCKED o EXPIRED, desbloquearlo
ALTER USER lab_schema_a ACCOUNT UNLOCK;
ALTER USER lab_schema_a IDENTIFIED BY "Lab_Pass_2024#";

-- Paso 3: Verificar que APEX puede ver el esquema
-- (El usuario de APEX, típicamente APEX_230200, necesita visibilidad)
-- En Oracle 19c+, verificar que no hay restricciones de contenedor
SELECT username FROM dba_users WHERE username = 'LAB_SCHEMA_A';
```

Después de corregir el esquema, regresa al asistente de creación de workspace en la consola `internal` y repite el proceso desde el Paso 2.

---

### Problema 2: La política de contraseñas no rechaza contraseñas débiles

**Síntoma:** Después de configurar la política de contraseñas en el Paso 5 (longitud mínima 10, mayúsculas, minúsculas, números y caracteres especiales requeridos), al intentar establecer la contraseña `password` para el usuario `dev_juan`, APEX la acepta sin mostrar ningún error de validación.

**Causa:** Las políticas de contraseña configuradas en **Workspace Settings** aplican únicamente a los usuarios creados **después** de que la política fue configurada, o la política fue guardada en la configuración del workspace pero no se activó correctamente porque el cambio se realizó desde el workspace y no desde la consola `internal`. Adicionalmente, en APEX 23.2, algunas configuraciones de políticas de contraseña requieren ser establecidas a nivel de **instancia** (desde `internal`) para tener efecto global, no solo a nivel de workspace.

**Solución:**

1. Regresa a la consola **APEX Administration Services** (sesión `internal`).

2. Navega a **Manage Instance → Instance Settings → Security**.

3. En la sección **Password Policy**, configura los mismos parámetros de complejidad (longitud mínima 10, mayúsculas, minúsculas, números, caracteres especiales).

4. Haz clic en **Apply Changes**.

5. Para forzar la re-evaluación del usuario existente, ve a **Manage Workspaces → Existing Workspaces → DEVTEAM_WS → Manage Users** y edita el usuario `dev_juan`.

6. Intenta nuevamente establecer la contraseña `password`. Esta vez debe aparecer el error de validación.

> **📝 Nota técnica:** En APEX 23.2, la jerarquía de políticas de contraseña es: la política de instancia (configurada desde `internal`) tiene precedencia sobre la política de workspace. Si ambas están configuradas, se aplica la más restrictiva. Siempre configura la política de contraseñas a nivel de instancia para garantizar su aplicación uniforme en todos los workspaces.

---

## 9. Limpieza del Entorno

> **⚠️ Importante:** Ejecuta la limpieza **solo si** este es un entorno de práctica temporal y los datos creados no son necesarios para laboratorios posteriores. Si estás continuando con los laboratorios 04 al 10, **conserva** el workspace `DEVTEAM_WS` y los esquemas asociados.

### Opción A — Limpieza parcial (recomendada si continúas con el curso)

```sql
-- Mantener el workspace pero limpiar usuarios de prueba adicionales
-- Ejecutar desde SQL Workshop en DEVTEAM_WS como ws_admin
-- O desde APEX Administration como internal

-- Eliminar solo el usuario de prueba ws_admin2 si no se necesita
-- (Realizar desde Manage Users and Groups en el workspace)
-- No ejecutar SQL directamente; usar la interfaz de APEX
```

Desde la interfaz de APEX (workspace `DEVTEAM_WS`, sesión `ws_admin`):
1. Ve a **Manage Users and Groups**.
2. Selecciona `ws_admin2` y haz clic en **Delete**.
3. Confirma la eliminación.

### Opción B — Limpieza completa (solo si NO continúas con laboratorios posteriores)

```sql
-- ============================================================
-- SCRIPT DE LIMPIEZA COMPLETA - Lab 03-00-01
-- ADVERTENCIA: Ejecutar solo si no se continúa con Labs 04-10
-- Ejecutar como SYS AS SYSDBA
-- ============================================================

-- Paso 1: Eliminar el workspace desde la consola internal
-- (Realizar desde APEX Administration Services → Manage Workspaces
--  → Remove Workspace → seleccionar "Remove Workspace Only" o
--  "Remove Workspace and Schema" según necesidad)

-- Paso 2: Eliminar esquemas de base de datos (solo si se eligió
-- "Remove Workspace Only" en el paso anterior)
DROP USER lab_schema_b CASCADE;
DROP USER lab_schema_a CASCADE;

-- Verificar limpieza
SELECT username FROM dba_users
WHERE  username IN ('LAB_SCHEMA_A', 'LAB_SCHEMA_B');
-- Debe retornar 0 filas

SELECT workspace FROM apex_workspaces
WHERE  workspace = 'DEVTEAM_WS';
-- Debe retornar 0 filas
```

> **🔴 Recordatorio crítico:** Al eliminar un workspace desde la consola `internal`, selecciona siempre **"Remove Workspace Only"** si deseas preservar los datos del esquema. La opción **"Remove Workspace and Schema"** elimina permanentemente el esquema y todos sus objetos sin posibilidad de recuperación.

---

## 10. Resumen

### Lo que construiste en este laboratorio

En este laboratorio completaste el ciclo completo de administración operativa de Oracle APEX desde la perspectiva del administrador de instancia:

| Tarea completada | Concepto aplicado |
|---|---|
| Acceso a APEX Administration Services (internal) | Arquitectura de instancia APEX |
| Creación del workspace `DEVTEAM_WS` con esquema `LAB_SCHEMA_A` | Relación workspace-esquema |
| Creación de usuarios con roles diferenciados (Admin, Developer) | Jerarquía de usuarios APEX |
| Asociación del esquema secundario `LAB_SCHEMA_B` | Workspace multi-esquema |
| Configuración de política de contraseñas y seguridad de sesión | Políticas de seguridad de instancia |
| Exploración de logs de actividad y reportes de uso | Monitoreo operativo |
| Purga de sesiones y exportación de configuración | Mantenimiento del workspace |

### Conceptos clave reforzados

- La **jerarquía de administración** en APEX tiene tres niveles: administrador de instancia (`internal`), administrador de workspace y desarrollador. Cada nivel tiene responsabilidades y accesos claramente delimitados.
- Un **workspace** puede asociarse a múltiples esquemas, pero siempre tiene un esquema principal. Esta flexibilidad permite que una aplicación acceda a datos distribuidos en diferentes esquemas sin comprometer el aislamiento entre workspaces.
- Las **políticas de seguridad** configuradas a nivel de instancia tienen precedencia sobre las configuradas a nivel de workspace, garantizando un piso mínimo de seguridad en toda la instalación.
- Los **logs de actividad** de APEX son accesibles tanto desde la interfaz gráfica de administración como mediante consultas SQL sobre las vistas `apex_workspace_activity_log` y `apex_workspace_access_log`, lo que permite integrarlos con herramientas de monitoreo externas.

### Próximos pasos

En el **Laboratorio 04**, utilizarás el workspace `DEVTEAM_WS` creado en este laboratorio como base para comenzar a construir la aplicación web progresiva del curso. Asegúrate de que el workspace esté activo, que `dev_juan` tenga acceso funcional y que ambos esquemas (`LAB_SCHEMA_A` y `LAB_SCHEMA_B`) estén correctamente asociados antes de proceder.

### Recursos adicionales

| Recurso | URL |
|---|---|
| Oracle APEX Administration Guide — Managing Workspaces | https://docs.oracle.com/en/database/oracle/apex/23.2/aeadm/managing-workspaces.html |
| Oracle APEX Administration Guide — Security | https://docs.oracle.com/en/database/oracle/apex/23.2/aeadm/security-settings.html |
| Oracle APEX Administration Guide — Managing Users | https://docs.oracle.com/en/database/oracle/apex/23.2/aeadm/managing-users-across-workspaces.html |
| Ask TOM — APEX Workspace and Schema Management | https://asktom.oracle.com |
| Oracle APEX Community Forums | https://forums.oracle.com/ords/apexds/domain/dev-community/category/apex |

---
*Lab 03-00-01 — Oracle APEX 23.2 — Versión 1.0*
