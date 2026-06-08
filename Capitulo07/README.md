# Práctica 7: Implementar procesos y lógica de negocio con PL/SQL

## 1. Metadatos

| Atributo         | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 52 minutos                                   |
| **Complejidad**  | Alta                                         |
| **Nivel Bloom**  | Crear (Create)                               |
| **Laboratorio**  | 07-00-01                                     |
| **Módulo**       | 7 — Lógica de negocio con PL/SQL en APEX     |

---

## 2. Descripción General

En este laboratorio implementarás un flujo completo de aprobación de solicitudes dentro de tu aplicación APEX existente. Crearás un paquete PL/SQL (`PKG_SOLICITUDES`) que encapsula la lógica de negocio para aprobar, rechazar y escalar solicitudes; configurarás procesos de página en los puntos de ejecución correctos; implementarás auditoría automática con manejo explícito de transacciones; y aplicarás buenas prácticas de seguridad para prevenir SQL injection. Al finalizar, la aplicación contará con un backend robusto, trazable y seguro que refleja patrones de desarrollo profesional con Oracle APEX 23.2.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear y configurar procesos de página (Page Processes) en APEX asignando el punto de ejecución correcto (`On Submit - After Computations and Validations`) y condiciones de ejecución basadas en `REQUEST`.
- [ ] Desarrollar el paquete PL/SQL `PKG_SOLICITUDES` con procedimientos para aprobar, rechazar y escalar solicitudes, invocándolos desde procesos APEX con bind variables.
- [ ] Implementar manejo de transacciones con control explícito de `COMMIT`/`ROLLBACK`, bloques `EXCEPTION` y registro de errores en una tabla de auditoría (`AUDIT_LOG`).
- [ ] Configurar computaciones de página para calcular valores derivados (fecha de vencimiento, porcentaje de avance) y un proceso de aplicación (Application Process) para lógica global.
- [ ] Identificar y corregir código PL/SQL con SQL dinámico inseguro, reemplazándolo por bind variables y funciones de `APEX_UTIL`.

---

## 4. Prerrequisitos

### Conocimiento Previo

- Haber completado satisfactoriamente los Laboratorios **04-00-01** y **06-00-01**.
- Dominio de bloques PL/SQL anónimos, procedimientos, funciones y paquetes.
- Comprensión de transacciones Oracle: `COMMIT`, `ROLLBACK`, `SAVEPOINT`.
- Familiaridad con el Page Designer de Oracle APEX 23.2.
- Conceptos básicos de auditoría y logging en bases de datos relacionales.

### Acceso Requerido

- Workspace activo en Oracle APEX 23.2 (apex.oracle.com o instancia local/OCI).
- Acceso a SQL Developer o SQL Workshop en APEX para ejecutar scripts DDL/DML.
- Aplicación base creada en laboratorios anteriores (ID de aplicación disponible).
- Privilegios para crear objetos en el esquema de trabajo (`CREATE TABLE`, `CREATE PACKAGE`, `CREATE SEQUENCE`).

---

## 5. Entorno de Laboratorio

### Hardware Recomendado

| Componente      | Mínimo                        | Recomendado                    |
|-----------------|-------------------------------|--------------------------------|
| RAM             | 8 GB                          | 16 GB                          |
| Almacenamiento  | 50 GB libres (SSD)            | 100 GB SSD                     |
| Procesador      | Intel Core i5 8ª gen / Ryzen 5 | Intel Core i7 / Ryzen 7        |
| Pantalla        | 1280×768                      | 1920×1080                      |

### Software Requerido

| Software             | Versión Mínima | Uso en este Lab                          |
|----------------------|----------------|------------------------------------------|
| Oracle APEX          | 23.2           | Page Designer, SQL Workshop              |
| Oracle Database      | 19c / 21c XE   | Ejecución de PL/SQL y objetos de esquema |
| Oracle SQL Developer | 23.1           | Desarrollo y depuración de paquetes      |
| Google Chrome        | 110+           | Acceso a APEX App Builder                |
| Postman              | 10.0+          | Verificación opcional de procesos REST   |

### Preparación Inicial del Entorno

Antes de iniciar los pasos del laboratorio, ejecuta el siguiente script en **SQL Workshop → SQL Commands** (o en SQL Developer) para crear los objetos de base de datos necesarios. Este script es idempotente: puede ejecutarse varias veces sin error.

```sql
-- ============================================================
-- SCRIPT DE PREPARACIÓN - LAB 07-00-01
-- Ejecutar en SQL Workshop > SQL Commands o SQL Developer
-- ============================================================

-- 1. Tabla de solicitudes (si no existe de laboratorios anteriores)
CREATE TABLE solicitudes (
    solicitud_id     NUMBER          GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo           VARCHAR2(200)   NOT NULL,
    descripcion      VARCHAR2(4000),
    estado           VARCHAR2(30)    DEFAULT 'PENDIENTE'
                                     CHECK (estado IN ('PENDIENTE','EN_REVISION',
                                                       'APROBADA','RECHAZADA','ESCALADA')),
    prioridad        VARCHAR2(10)    DEFAULT 'MEDIA'
                                     CHECK (prioridad IN ('ALTA','MEDIA','BAJA')),
    solicitante      VARCHAR2(100)   NOT NULL,
    monto_solicitado NUMBER(12,2),
    fecha_creacion   DATE            DEFAULT SYSDATE,
    fecha_vencimiento DATE,
    fecha_resolucion DATE,
    comentario_resol VARCHAR2(1000),
    rowversion       NUMBER          DEFAULT 0,
    created_by       VARCHAR2(100),
    updated_by       VARCHAR2(100),
    updated_date     DATE
);

-- 2. Tabla de auditoría
CREATE TABLE audit_log (
    log_id       NUMBER          GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    accion       VARCHAR2(100)   NOT NULL,
    tabla_afect  VARCHAR2(100),
    registro_id  NUMBER,
    usuario_apex VARCHAR2(100),
    pagina_id    NUMBER,
    detalle      VARCHAR2(4000),
    fecha_accion TIMESTAMP       DEFAULT SYSTIMESTAMP,
    ip_address   VARCHAR2(50),
    resultado    VARCHAR2(20)    DEFAULT 'OK'
                                  CHECK (resultado IN ('OK','ERROR'))
);

-- 3. Tabla de notificaciones pendientes
CREATE TABLE notificaciones_pendientes (
    notif_id     NUMBER          GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    solicitud_id NUMBER          REFERENCES solicitudes(solicitud_id),
    tipo         VARCHAR2(50),
    destinatario VARCHAR2(200),
    asunto       VARCHAR2(500),
    cuerpo       CLOB,
    estado       VARCHAR2(20)    DEFAULT 'PENDIENTE',
    creado_en    TIMESTAMP       DEFAULT SYSTIMESTAMP
);

-- 4. Datos de muestra (mínimo para pruebas)
BEGIN
    FOR i IN 1..15 LOOP
        INSERT INTO solicitudes (titulo, descripcion, estado, prioridad,
                                  solicitante, monto_solicitado, fecha_creacion)
        VALUES (
            'Solicitud de prueba ' || i,
            'Descripción detallada de la solicitud número ' || i,
            CASE MOD(i,5)
                WHEN 0 THEN 'PENDIENTE'
                WHEN 1 THEN 'EN_REVISION'
                WHEN 2 THEN 'APROBADA'
                WHEN 3 THEN 'RECHAZADA'
                ELSE 'PENDIENTE'
            END,
            CASE MOD(i,3)
                WHEN 0 THEN 'ALTA'
                WHEN 1 THEN 'MEDIA'
                ELSE 'BAJA'
            END,
            'usuario' || i || '@empresa.com',
            ROUND(DBMS_RANDOM.VALUE(1000, 50000), 2),
            SYSDATE - MOD(i*3, 30)
        );
    END LOOP;
    COMMIT;
END;
/

-- 5. Verificación
SELECT 'solicitudes' tabla, COUNT(*) registros FROM solicitudes
UNION ALL
SELECT 'audit_log', COUNT(*) FROM audit_log
UNION ALL
SELECT 'notificaciones_pendientes', COUNT(*) FROM notificaciones_pendientes;
```

**Resultado esperado de verificación:**

```
TABLA                       REGISTROS
--------------------------- ---------
solicitudes                        15
audit_log                           0
notificaciones_pendientes           0
```

---

## 6. Pasos del Laboratorio

### Paso 1 — Crear el Paquete PL/SQL `PKG_SOLICITUDES`

**Objetivo:** Desarrollar el paquete de negocio central que encapsula toda la lógica de aprobación, rechazo y escalamiento de solicitudes, con manejo de transacciones y auditoría integrada.

#### Instrucciones

1. Abre **SQL Developer** (o navega a **SQL Workshop → SQL Commands** en APEX).

2. Ejecuta el siguiente script para crear la **especificación del paquete**:

```sql
CREATE OR REPLACE PACKAGE pkg_solicitudes AS

    -- Constantes de estado
    C_ESTADO_PENDIENTE   CONSTANT VARCHAR2(30) := 'PENDIENTE';
    C_ESTADO_EN_REVISION CONSTANT VARCHAR2(30) := 'EN_REVISION';
    C_ESTADO_APROBADA    CONSTANT VARCHAR2(30) := 'APROBADA';
    C_ESTADO_RECHAZADA   CONSTANT VARCHAR2(30) := 'RECHAZADA';
    C_ESTADO_ESCALADA    CONSTANT VARCHAR2(30) := 'ESCALADA';

    -- Tipo para resultado de operaciones
    TYPE t_resultado IS RECORD (
        exito    BOOLEAN,
        mensaje  VARCHAR2(500),
        log_id   NUMBER
    );

    -- Procedimientos principales
    PROCEDURE crear_solicitud (
        p_titulo           IN  solicitudes.titulo%TYPE,
        p_descripcion      IN  solicitudes.descripcion%TYPE,
        p_prioridad        IN  solicitudes.prioridad%TYPE,
        p_solicitante      IN  solicitudes.solicitante%TYPE,
        p_monto            IN  solicitudes.monto_solicitado%TYPE,
        p_usuario_apex     IN  VARCHAR2,
        p_solicitud_id     OUT solicitudes.solicitud_id%TYPE,
        p_resultado        OUT t_resultado
    );

    PROCEDURE aprobar_solicitud (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_resultado     OUT t_resultado
    );

    PROCEDURE rechazar_solicitud (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_resultado     OUT t_resultado
    );

    PROCEDURE escalar_solicitud (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_resultado     OUT t_resultado
    );

    -- Función auxiliar
    FUNCTION calcular_fecha_vencimiento (
        p_prioridad      IN solicitudes.prioridad%TYPE,
        p_fecha_inicio   IN DATE DEFAULT SYSDATE
    ) RETURN DATE;

    -- Procedimiento de auditoría (uso interno y externo)
    PROCEDURE registrar_auditoria (
        p_accion       IN audit_log.accion%TYPE,
        p_tabla        IN audit_log.tabla_afect%TYPE,
        p_registro_id  IN audit_log.registro_id%TYPE,
        p_usuario      IN audit_log.usuario_apex%TYPE,
        p_pagina_id    IN audit_log.pagina_id%TYPE,
        p_detalle      IN audit_log.detalle%TYPE,
        p_resultado    IN audit_log.resultado%TYPE DEFAULT 'OK'
    );

END pkg_solicitudes;
/
```

3. Ejecuta el siguiente script para crear el **cuerpo del paquete**:

```sql
CREATE OR REPLACE PACKAGE BODY pkg_solicitudes AS

    -- -------------------------------------------------------
    -- Procedimiento interno: registrar auditoría
    -- -------------------------------------------------------
    PROCEDURE registrar_auditoria (
        p_accion       IN audit_log.accion%TYPE,
        p_tabla        IN audit_log.tabla_afect%TYPE,
        p_registro_id  IN audit_log.registro_id%TYPE,
        p_usuario      IN audit_log.usuario_apex%TYPE,
        p_pagina_id    IN audit_log.pagina_id%TYPE,
        p_detalle      IN audit_log.detalle%TYPE,
        p_resultado    IN audit_log.resultado%TYPE DEFAULT 'OK'
    ) IS
        PRAGMA AUTONOMOUS_TRANSACTION;
    BEGIN
        INSERT INTO audit_log (
            accion, tabla_afect, registro_id,
            usuario_apex, pagina_id, detalle,
            fecha_accion, resultado
        ) VALUES (
            p_accion, p_tabla, p_registro_id,
            p_usuario, p_pagina_id, p_detalle,
            SYSTIMESTAMP, p_resultado
        );
        COMMIT;  -- Autónoma: no afecta la transacción principal
    EXCEPTION
        WHEN OTHERS THEN
            -- La auditoría nunca debe bloquear la transacción principal
            ROLLBACK;
    END registrar_auditoria;

    -- -------------------------------------------------------
    -- Función: calcular fecha de vencimiento por prioridad
    -- -------------------------------------------------------
    FUNCTION calcular_fecha_vencimiento (
        p_prioridad    IN solicitudes.prioridad%TYPE,
        p_fecha_inicio IN DATE DEFAULT SYSDATE
    ) RETURN DATE IS
        v_dias NUMBER;
    BEGIN
        v_dias := CASE UPPER(p_prioridad)
                      WHEN 'ALTA'  THEN 3
                      WHEN 'MEDIA' THEN 7
                      WHEN 'BAJA'  THEN 15
                      ELSE 7
                  END;
        RETURN p_fecha_inicio + v_dias;
    END calcular_fecha_vencimiento;

    -- -------------------------------------------------------
    -- Procedimiento: crear solicitud
    -- -------------------------------------------------------
    PROCEDURE crear_solicitud (
        p_titulo           IN  solicitudes.titulo%TYPE,
        p_descripcion      IN  solicitudes.descripcion%TYPE,
        p_prioridad        IN  solicitudes.prioridad%TYPE,
        p_solicitante      IN  solicitudes.solicitante%TYPE,
        p_monto            IN  solicitudes.monto_solicitado%TYPE,
        p_usuario_apex     IN  VARCHAR2,
        p_solicitud_id     OUT solicitudes.solicitud_id%TYPE,
        p_resultado        OUT t_resultado
    ) IS
        v_fecha_vcto DATE;
    BEGIN
        -- Calcular fecha de vencimiento según prioridad
        v_fecha_vcto := calcular_fecha_vencimiento(p_prioridad);

        -- Insertar usando bind variables (prevención SQL injection)
        INSERT INTO solicitudes (
            titulo, descripcion, estado, prioridad,
            solicitante, monto_solicitado,
            fecha_creacion, fecha_vencimiento,
            rowversion, created_by
        ) VALUES (
            p_titulo, p_descripcion, C_ESTADO_PENDIENTE, p_prioridad,
            p_solicitante, p_monto,
            SYSDATE, v_fecha_vcto,
            1, p_usuario_apex
        )
        RETURNING solicitud_id INTO p_solicitud_id;

        -- Registrar auditoría (transacción autónoma)
        registrar_auditoria(
            p_accion      => 'CREAR_SOLICITUD',
            p_tabla       => 'SOLICITUDES',
            p_registro_id => p_solicitud_id,
            p_usuario     => p_usuario_apex,
            p_pagina_id   => NULL,
            p_detalle     => 'Solicitud creada: ' || p_titulo ||
                             ' | Monto: ' || p_monto ||
                             ' | Prioridad: ' || p_prioridad,
            p_resultado   => 'OK'
        );

        p_resultado.exito   := TRUE;
        p_resultado.mensaje := 'Solicitud creada exitosamente con ID: ' || p_solicitud_id;

    EXCEPTION
        WHEN DUP_VAL_ON_INDEX THEN
            p_resultado.exito   := FALSE;
            p_resultado.mensaje := 'Error: Ya existe una solicitud con ese título.';
            registrar_auditoria('CREAR_SOLICITUD_ERROR','SOLICITUDES',NULL,
                                 p_usuario_apex,NULL,SQLERRM,'ERROR');
        WHEN OTHERS THEN
            p_resultado.exito   := FALSE;
            p_resultado.mensaje := 'Error inesperado: ' || SQLERRM;
            registrar_auditoria('CREAR_SOLICITUD_ERROR','SOLICITUDES',NULL,
                                 p_usuario_apex,NULL,SQLERRM,'ERROR');
            RAISE;  -- Re-lanzar para que APEX maneje la transacción
    END crear_solicitud;

    -- -------------------------------------------------------
    -- Procedimiento interno: cambiar estado con locking optimista
    -- -------------------------------------------------------
    PROCEDURE cambiar_estado (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_nuevo_estado  IN  solicitudes.estado%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_accion_audit  IN  VARCHAR2,
        p_resultado     OUT t_resultado
    ) IS
        v_rowversion_actual solicitudes.rowversion%TYPE;
        v_estado_actual     solicitudes.estado%TYPE;
        v_rows_updated      NUMBER;
    BEGIN
        -- Verificar que el registro existe y obtener estado actual
        BEGIN
            SELECT rowversion, estado
            INTO v_rowversion_actual, v_estado_actual
            FROM solicitudes
            WHERE solicitud_id = p_solicitud_id
            FOR UPDATE NOWAIT;  -- Bloqueo optimista
        EXCEPTION
            WHEN NO_DATA_FOUND THEN
                p_resultado.exito   := FALSE;
                p_resultado.mensaje := 'La solicitud con ID ' || p_solicitud_id || ' no existe.';
                RETURN;
            WHEN OTHERS THEN
                IF SQLCODE = -54 THEN  -- ORA-00054: resource busy
                    p_resultado.exito   := FALSE;
                    p_resultado.mensaje := 'La solicitud está siendo modificada por otro usuario. Intente nuevamente.';
                    RETURN;
                END IF;
                RAISE;
        END;

        -- Verificar rowversion (optimistic locking)
        IF v_rowversion_actual != p_rowversion THEN
            p_resultado.exito   := FALSE;
            p_resultado.mensaje := 'Conflicto de concurrencia: el registro fue modificado por otro usuario. ' ||
                                   'Por favor recargue la página y aplique sus cambios nuevamente.';
            registrar_auditoria(p_accion_audit || '_CONFLICTO','SOLICITUDES',
                                 p_solicitud_id, p_usuario_apex, NULL,
                                 'Rowversion esperada: ' || p_rowversion ||
                                 ' | Actual: ' || v_rowversion_actual, 'ERROR');
            RETURN;
        END IF;

        -- Actualizar registro
        UPDATE solicitudes
        SET    estado           = p_nuevo_estado,
               comentario_resol = p_comentario,
               fecha_resolucion = SYSDATE,
               updated_by       = p_usuario_apex,
               updated_date     = SYSDATE,
               rowversion       = rowversion + 1
        WHERE  solicitud_id     = p_solicitud_id;

        v_rows_updated := SQL%ROWCOUNT;

        IF v_rows_updated = 0 THEN
            p_resultado.exito   := FALSE;
            p_resultado.mensaje := 'No se pudo actualizar la solicitud.';
            RETURN;
        END IF;

        -- Auditoría
        registrar_auditoria(
            p_accion      => p_accion_audit,
            p_tabla       => 'SOLICITUDES',
            p_registro_id => p_solicitud_id,
            p_usuario     => p_usuario_apex,
            p_pagina_id   => NULL,
            p_detalle     => 'Estado anterior: ' || v_estado_actual ||
                             ' → Nuevo estado: ' || p_nuevo_estado ||
                             ' | Comentario: ' || SUBSTR(p_comentario, 1, 200),
            p_resultado   => 'OK'
        );

        p_resultado.exito   := TRUE;
        p_resultado.mensaje := 'Solicitud ' || LOWER(p_nuevo_estado) ||
                               ' exitosamente.';

    EXCEPTION
        WHEN OTHERS THEN
            p_resultado.exito   := FALSE;
            p_resultado.mensaje := 'Error al cambiar estado: ' || SQLERRM;
            registrar_auditoria(p_accion_audit || '_ERROR','SOLICITUDES',
                                 p_solicitud_id, p_usuario_apex, NULL,
                                 SQLERRM, 'ERROR');
            RAISE;
    END cambiar_estado;

    -- -------------------------------------------------------
    -- Procedimiento: aprobar solicitud
    -- -------------------------------------------------------
    PROCEDURE aprobar_solicitud (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_resultado     OUT t_resultado
    ) IS
    BEGIN
        cambiar_estado(
            p_solicitud_id => p_solicitud_id,
            p_nuevo_estado => C_ESTADO_APROBADA,
            p_comentario   => p_comentario,
            p_usuario_apex => p_usuario_apex,
            p_rowversion   => p_rowversion,
            p_accion_audit => 'APROBAR_SOLICITUD',
            p_resultado    => p_resultado
        );
    END aprobar_solicitud;

    -- -------------------------------------------------------
    -- Procedimiento: rechazar solicitud
    -- -------------------------------------------------------
    PROCEDURE rechazar_solicitud (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_resultado     OUT t_resultado
    ) IS
    BEGIN
        IF p_comentario IS NULL OR TRIM(p_comentario) IS NULL THEN
            p_resultado.exito   := FALSE;
            p_resultado.mensaje := 'El comentario de rechazo es obligatorio.';
            RETURN;
        END IF;

        cambiar_estado(
            p_solicitud_id => p_solicitud_id,
            p_nuevo_estado => C_ESTADO_RECHAZADA,
            p_comentario   => p_comentario,
            p_usuario_apex => p_usuario_apex,
            p_rowversion   => p_rowversion,
            p_accion_audit => 'RECHAZAR_SOLICITUD',
            p_resultado    => p_resultado
        );
    END rechazar_solicitud;

    -- -------------------------------------------------------
    -- Procedimiento: escalar solicitud
    -- -------------------------------------------------------
    PROCEDURE escalar_solicitud (
        p_solicitud_id  IN  solicitudes.solicitud_id%TYPE,
        p_comentario    IN  solicitudes.comentario_resol%TYPE,
        p_usuario_apex  IN  VARCHAR2,
        p_rowversion    IN  solicitudes.rowversion%TYPE,
        p_resultado     OUT t_resultado
    ) IS
    BEGIN
        cambiar_estado(
            p_solicitud_id => p_solicitud_id,
            p_nuevo_estado => C_ESTADO_ESCALADA,
            p_comentario   => p_comentario,
            p_usuario_apex => p_usuario_apex,
            p_rowversion   => p_rowversion,
            p_accion_audit => 'ESCALAR_SOLICITUD',
            p_resultado    => p_resultado
        );
    END escalar_solicitud;

END pkg_solicitudes;
/
```

4. Verifica que el paquete compiló sin errores ejecutando:

```sql
SELECT object_name, object_type, status
FROM user_objects
WHERE object_name = 'PKG_SOLICITUDES'
ORDER BY object_type;
```

**Resultado esperado:**

```
OBJECT_NAME        OBJECT_TYPE   STATUS
------------------ ------------- -------
PKG_SOLICITUDES    PACKAGE       VALID
PKG_SOLICITUDES    PACKAGE BODY  VALID
```

**Verificación:** Si el `STATUS` es `INVALID`, ejecuta `SHOW ERRORS PACKAGE BODY PKG_SOLICITUDES;` en SQL Developer para ver los errores de compilación y corrígelos antes de continuar.

---

### Paso 2 — Crear la Página de Gestión de Solicitudes en APEX

**Objetivo:** Construir la página de formulario en APEX que usará los procesos PL/SQL del paquete, con todos los ítems necesarios incluyendo el campo oculto `rowversion` para el locking optimista.

#### Instrucciones

1. En el **App Builder**, abre tu aplicación y haz clic en **Create Page**.

2. Selecciona **Blank Page**. Configura:
   - **Name:** `Gestión de Solicitudes`
   - **Page Mode:** `Normal`
   - **Page Number:** `20` (o el siguiente disponible en tu aplicación)

3. En el **Page Designer** de la página 20, crea una región de tipo **Form**:
   - Haz clic en el ícono **+** en la sección de Body.
   - Selecciona **Region → Form**.
   - **Title:** `Detalle de Solicitud`
   - **Table Name:** `SOLICITUDES`
   - **Primary Key Column:** `SOLICITUD_ID`

4. APEX generará automáticamente los ítems del formulario. Verifica que existan los siguientes ítems (ajusta si es necesario desde el panel de propiedades de cada ítem):

   | Ítem de Página       | Tipo              | Label                  | Notas                              |
   |----------------------|-------------------|------------------------|------------------------------------|
   | `P20_SOLICITUD_ID`   | Hidden            | —                      | Clave primaria, oculto             |
   | `P20_TITULO`         | Text Field        | Título                 | Required: Yes                      |
   | `P20_DESCRIPCION`    | Textarea          | Descripción            | Height: 4 rows                     |
   | `P20_PRIORIDAD`      | Select List       | Prioridad              | LOV: ALTA/MEDIA/BAJA               |
   | `P20_ESTADO`         | Display Only      | Estado                 | Solo lectura en edición            |
   | `P20_SOLICITANTE`    | Text Field        | Solicitante            | Required: Yes                      |
   | `P20_MONTO_SOLICITADO` | Number Field    | Monto Solicitado       | Format: 999,999,990.00             |
   | `P20_COMENTARIO_RESOL` | Textarea        | Comentario de Resolución | Height: 3 rows                  |
   | `P20_ROWVERSION`     | Hidden            | —                      | **Crítico para locking optimista** |
   | `P20_FECHA_VENCIMIENTO` | Display Only   | Fecha de Vencimiento   | Calculado por computación          |
   | `P20_RESULTADO_MSG`  | Display Only      | —                      | Muestra mensajes de resultado      |

5. Para el ítem `P20_PRIORIDAD`, configura la **Lista de Valores (LOV)** como estática:
   - Tipo: `Static Values`
   - Valores:
     ```
     ALTA;Alta
     MEDIA;Media
     BAJA;Baja
     ```

6. Crea tres botones en la región:

   | Botón           | Label     | Action          | Condition (Server-side)                   |
   |-----------------|-----------|-----------------|-------------------------------------------|
   | `BTN_CREAR`     | Crear     | Submit Page     | `P20_SOLICITUD_ID` is NULL                |
   | `BTN_APROBAR`   | Aprobar   | Submit Page     | `P20_SOLICITUD_ID` is NOT NULL            |
   | `BTN_RECHAZAR`  | Rechazar  | Submit Page     | `P20_SOLICITUD_ID` is NOT NULL            |
   | `BTN_ESCALAR`   | Escalar   | Submit Page     | `P20_SOLICITUD_ID` is NOT NULL            |

   Para cada botón, establece el **Button Name** igual al valor del Label en mayúsculas (ej. `CREAR`, `APROBAR`, `RECHAZAR`, `ESCALAR`). Esto define el valor de `:REQUEST` al hacer submit.

**Verificación:** Guarda la página y navega a ella en modo de ejecución. Debes ver el formulario vacío con los botones correspondientes. Aún no funcionarán porque los procesos PL/SQL se crearán en el siguiente paso.

---

### Paso 3 — Configurar los Procesos de Página (Page Processes)

**Objetivo:** Crear los procesos PL/SQL en el punto de ejecución correcto (`On Submit - After Computations and Validations`) con las condiciones adecuadas basadas en el valor de `REQUEST`.

#### Instrucciones

1. En el **Page Designer** de la página 20, navega a la sección **Processing** (panel izquierdo, pestaña de procesamiento).

2. Haz clic derecho en **Processes** y selecciona **Create Process**.

3. Crea el **Proceso 1: Crear Solicitud** con la siguiente configuración:

   - **Name:** `Proceso - Crear Solicitud`
   - **Type:** `Execute Code`
   - **Point:** `On Submit - After Computations and Validations`
   - **Sequence:** `10`
   - **Source (PL/SQL Code):**

```plsql
DECLARE
    v_resultado pkg_solicitudes.t_resultado;
    v_id        solicitudes.solicitud_id%TYPE;
BEGIN
    pkg_solicitudes.crear_solicitud(
        p_titulo       => :P20_TITULO,
        p_descripcion  => :P20_DESCRIPCION,
        p_prioridad    => :P20_PRIORIDAD,
        p_solicitante  => :P20_SOLICITANTE,
        p_monto        => TO_NUMBER(:P20_MONTO_SOLICITADO),
        p_usuario_apex => :APP_USER,
        p_solicitud_id => v_id,
        p_resultado    => v_resultado
    );

    IF v_resultado.exito THEN
        :P20_SOLICITUD_ID  := v_id;
        :P20_RESULTADO_MSG := v_resultado.mensaje;
        -- APEX maneja el COMMIT automáticamente al final del procesamiento
    ELSE
        -- Registrar el error para que APEX muestre el mensaje
        apex_error.add_error(
            p_message          => v_resultado.mensaje,
            p_display_location => apex_error.c_inline_in_notification
        );
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        apex_error.add_error(
            p_message          => 'Error al crear la solicitud: ' || SQLERRM,
            p_display_location => apex_error.c_inline_in_notification
        );
END;
```

   - **Condition:**
     - **Type:** `Request = Value`
     - **Value:** `CREAR`

4. Crea el **Proceso 2: Aprobar Solicitud** con:

   - **Name:** `Proceso - Aprobar Solicitud`
   - **Type:** `Execute Code`
   - **Point:** `On Submit - After Computations and Validations`
   - **Sequence:** `20`
   - **Source (PL/SQL Code):**

```plsql
DECLARE
    v_resultado pkg_solicitudes.t_resultado;
BEGIN
    pkg_solicitudes.aprobar_solicitud(
        p_solicitud_id => TO_NUMBER(:P20_SOLICITUD_ID),
        p_comentario   => :P20_COMENTARIO_RESOL,
        p_usuario_apex => :APP_USER,
        p_rowversion   => TO_NUMBER(:P20_ROWVERSION),
        p_resultado    => v_resultado
    );

    IF v_resultado.exito THEN
        :P20_RESULTADO_MSG := v_resultado.mensaje;
        -- Recargar el rowversion actualizado
        SELECT rowversion
        INTO   :P20_ROWVERSION
        FROM   solicitudes
        WHERE  solicitud_id = TO_NUMBER(:P20_SOLICITUD_ID);
    ELSE
        apex_error.add_error(
            p_message          => v_resultado.mensaje,
            p_display_location => apex_error.c_inline_in_notification
        );
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        apex_error.add_error(
            p_message          => 'Error al aprobar: ' || SQLERRM,
            p_display_location => apex_error.c_inline_in_notification
        );
END;
```

   - **Condition:**
     - **Type:** `Request = Value`
     - **Value:** `APROBAR`

5. Crea el **Proceso 3: Rechazar Solicitud** (secuencia 30, condición `REQUEST = 'RECHAZAR'`):

```plsql
DECLARE
    v_resultado pkg_solicitudes.t_resultado;
BEGIN
    pkg_solicitudes.rechazar_solicitud(
        p_solicitud_id => TO_NUMBER(:P20_SOLICITUD_ID),
        p_comentario   => :P20_COMENTARIO_RESOL,
        p_usuario_apex => :APP_USER,
        p_rowversion   => TO_NUMBER(:P20_ROWVERSION),
        p_resultado    => v_resultado
    );

    IF v_resultado.exito THEN
        :P20_RESULTADO_MSG := v_resultado.mensaje;
        SELECT rowversion
        INTO   :P20_ROWVERSION
        FROM   solicitudes
        WHERE  solicitud_id = TO_NUMBER(:P20_SOLICITUD_ID);
    ELSE
        apex_error.add_error(
            p_message          => v_resultado.mensaje,
            p_display_location => apex_error.c_inline_in_notification
        );
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        apex_error.add_error(
            p_message          => 'Error al rechazar: ' || SQLERRM,
            p_display_location => apex_error.c_inline_in_notification
        );
END;
```

6. Crea el **Proceso 4: Escalar Solicitud** (secuencia 40, condición `REQUEST = 'ESCALAR'`):

```plsql
DECLARE
    v_resultado pkg_solicitudes.t_resultado;
BEGIN
    pkg_solicitudes.escalar_solicitud(
        p_solicitud_id => TO_NUMBER(:P20_SOLICITUD_ID),
        p_comentario   => :P20_COMENTARIO_RESOL,
        p_usuario_apex => :APP_USER,
        p_rowversion   => TO_NUMBER(:P20_ROWVERSION),
        p_resultado    => v_resultado
    );

    IF v_resultado.exito THEN
        :P20_RESULTADO_MSG := v_resultado.mensaje;
        SELECT rowversion
        INTO   :P20_ROWVERSION
        FROM   solicitudes
        WHERE  solicitud_id = TO_NUMBER(:P20_SOLICITUD_ID);
    ELSE
        apex_error.add_error(
            p_message          => v_resultado.mensaje,
            p_display_location => apex_error.c_inline_in_notification
        );
    END IF;
EXCEPTION
    WHEN OTHERS THEN
        apex_error.add_error(
            p_message          => 'Error al escalar: ' || SQLERRM,
            p_display_location => apex_error.c_inline_in_notification
        );
END;
```

7. Crea el **Proceso 5: Auditoría de Carga de Página** para registrar accesos:

   - **Name:** `Proceso - Auditoría Carga Página`
   - **Type:** `Execute Code`
   - **Point:** `Before Header`
   - **Sequence:** `5`
   - **Source:**

```plsql
BEGIN
    -- Solo auditar cuando se carga una solicitud existente
    IF :P20_SOLICITUD_ID IS NOT NULL THEN
        pkg_solicitudes.registrar_auditoria(
            p_accion      => 'VER_SOLICITUD',
            p_tabla       => 'SOLICITUDES',
            p_registro_id => TO_NUMBER(:P20_SOLICITUD_ID),
            p_usuario     => :APP_USER,
            p_pagina_id   => TO_NUMBER(:APP_PAGE_ID),
            p_detalle     => 'Visualización de solicitud desde IP: ' ||
                             OWA_UTIL.GET_CGI_ENV('REMOTE_ADDR'),
            p_resultado   => 'OK'
        );
    END IF;
END;
```

   - **Condition:** No condition (siempre ejecutar, la condición interna lo controla).

**Resultado esperado:** En el panel **Processing** del Page Designer debes ver 5 procesos listados. Los procesos 1-4 están en el punto `On Submit - After Computations and Validations` con sus respectivas condiciones. El proceso 5 está en `Before Header`.

**Verificación:** Guarda la página. Navega a ella en modo ejecución, llena el formulario con datos válidos y haz clic en **Crear**. Verifica que aparezca el mensaje de éxito y que el registro exista en la tabla `SOLICITUDES`.

---

### Paso 4 — Implementar Computaciones de Página

**Objetivo:** Configurar computaciones (Computations) para calcular automáticamente la fecha de vencimiento al cargar la página y para mantener sincronizado el `rowversion` del registro.

#### Instrucciones

1. En el **Page Designer**, sección **Pre-Rendering**, haz clic derecho en **Before Regions** y selecciona **Create Computation**.

2. Crea la **Computación 1: Fecha de Vencimiento**:

   - **Item Name:** `P20_FECHA_VENCIMIENTO`
   - **Type:** `PL/SQL Function Body`
   - **Sequence:** `10`
   - **Point:** `Before Regions`
   - **PL/SQL Function Body:**

```plsql
BEGIN
    IF :P20_SOLICITUD_ID IS NOT NULL THEN
        -- Cargar fecha desde base de datos para registro existente
        RETURN TO_CHAR(
            (SELECT fecha_vencimiento
             FROM   solicitudes
             WHERE  solicitud_id = TO_NUMBER(:P20_SOLICITUD_ID)),
            'DD/MM/YYYY'
        );
    ELSIF :P20_PRIORIDAD IS NOT NULL THEN
        -- Calcular para nueva solicitud según prioridad seleccionada
        RETURN TO_CHAR(
            pkg_solicitudes.calcular_fecha_vencimiento(:P20_PRIORIDAD),
            'DD/MM/YYYY'
        );
    ELSE
        RETURN TO_CHAR(SYSDATE + 7, 'DD/MM/YYYY');  -- Default: 7 días
    END IF;
END;
```

3. Crea la **Computación 2: Cargar Rowversion**:

   - **Item Name:** `P20_ROWVERSION`
   - **Type:** `SQL Query (return single value)`
   - **Sequence:** `20`
   - **Point:** `Before Regions`
   - **SQL Query:**

```sql
SELECT NVL(rowversion, 0)
FROM   solicitudes
WHERE  solicitud_id = TO_NUMBER(:P20_SOLICITUD_ID)
```

   - **Condition:**
     - **Type:** `Item is NOT NULL`
     - **Item:** `P20_SOLICITUD_ID`

4. Crea la **Computación 3: Porcentaje de Avance** (ejemplo de cálculo derivado):

   - **Item Name:** Primero crea el ítem `P20_PORCENTAJE_AVANCE` como **Display Only** con label `Avance del Proceso`.
   - **Type:** `PL/SQL Function Body`
   - **Sequence:** `30`
   - **Point:** `Before Regions`
   - **PL/SQL Function Body:**

```plsql
DECLARE
    v_estado VARCHAR2(30);
BEGIN
    IF :P20_SOLICITUD_ID IS NULL THEN
        RETURN '0%';
    END IF;

    SELECT estado INTO v_estado
    FROM   solicitudes
    WHERE  solicitud_id = TO_NUMBER(:P20_SOLICITUD_ID);

    RETURN CASE v_estado
               WHEN 'PENDIENTE'   THEN '10%'
               WHEN 'EN_REVISION' THEN '50%'
               WHEN 'APROBADA'    THEN '100%'
               WHEN 'RECHAZADA'   THEN '100%'
               WHEN 'ESCALADA'    THEN '75%'
               ELSE '0%'
           END;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN '0%';
END;
```

**Resultado esperado:** Al cargar la página con un `P20_SOLICITUD_ID` válido en la URL (ej. `...&P20_SOLICITUD_ID=1`), los campos `P20_FECHA_VENCIMIENTO`, `P20_ROWVERSION` y `P20_PORCENTAJE_AVANCE` deben poblarse automáticamente con los valores correctos.

**Verificación:** Ejecuta la siguiente consulta para confirmar que las computaciones devuelven datos coherentes:

```sql
SELECT solicitud_id, estado, rowversion,
       TO_CHAR(fecha_vencimiento,'DD/MM/YYYY') fecha_vcto,
       CASE estado
           WHEN 'PENDIENTE'   THEN '10%'
           WHEN 'EN_REVISION' THEN '50%'
           WHEN 'APROBADA'    THEN '100%'
           WHEN 'RECHAZADA'   THEN '100%'
           WHEN 'ESCALADA'    THEN '75%'
       END porcentaje
FROM   solicitudes
WHERE  ROWNUM <= 5;
```

---

### Paso 5 — Crear un Proceso de Aplicación (Application Process)

**Objetivo:** Configurar un Application Process que se ejecute globalmente en todas las páginas para verificar si existen solicitudes vencidas y actualizar su estado, demostrando la diferencia entre procesos de página y de aplicación.

#### Instrucciones

1. En el **App Builder**, navega a tu aplicación y selecciona **Shared Components**.

2. En la sección **Logic**, haz clic en **Application Processes**.

3. Haz clic en **Create** y configura:

   - **Name:** `APP_PROC_VERIFICAR_VENCIMIENTOS`
   - **Point:** `On New Instance (new session)`
   - **Sequence:** `10`
   - **Source (PL/SQL):**

```plsql
DECLARE
    v_count NUMBER := 0;
BEGIN
    -- Actualizar solicitudes vencidas que aún están pendientes
    UPDATE solicitudes
    SET    estado       = 'ESCALADA',
           updated_by   = 'SISTEMA_AUTO',
           updated_date = SYSDATE,
           rowversion   = rowversion + 1,
           comentario_resol = 'Escalada automáticamente por vencimiento de plazo'
    WHERE  estado           = 'PENDIENTE'
    AND    fecha_vencimiento < SYSDATE
    AND    fecha_vencimiento IS NOT NULL;

    v_count := SQL%ROWCOUNT;

    IF v_count > 0 THEN
        -- Registrar en auditoría cuántas se escalaron
        INSERT INTO audit_log (
            accion, tabla_afect, usuario_apex,
            pagina_id, detalle, fecha_accion, resultado
        ) VALUES (
            'AUTO_ESCALAR_VENCIDAS', 'SOLICITUDES', 'SISTEMA',
            0,
            v_count || ' solicitud(es) escalada(s) automáticamente por vencimiento',
            SYSTIMESTAMP, 'OK'
        );
        COMMIT;

        -- Guardar en variable de aplicación para notificar al usuario
        :APP_SOLICITUDES_VENCIDAS := v_count;
    END IF;

EXCEPTION
    WHEN OTHERS THEN
        -- Nunca bloquear el inicio de sesión por este proceso
        INSERT INTO audit_log (
            accion, tabla_afect, usuario_apex,
            pagina_id, detalle, fecha_accion, resultado
        ) VALUES (
            'AUTO_ESCALAR_ERROR', 'SOLICITUDES', 'SISTEMA',
            0, SQLERRM, SYSTIMESTAMP, 'ERROR'
        );
        COMMIT;
END;
```

4. Haz clic en **Create Process**.

5. Para que la variable `:APP_SOLICITUDES_VENCIDAS` funcione, crea un **Application Item**:
   - Navega a **Shared Components → Application Items**.
   - Crea: **Name:** `APP_SOLICITUDES_VENCIDAS`, **Scope:** `Application`.

**Resultado esperado:** El proceso se ejecutará una vez por sesión de usuario. Al iniciar sesión, APEX verificará solicitudes vencidas y actualizará su estado automáticamente.

**Verificación:** Inserta manualmente una solicitud con fecha de vencimiento en el pasado y luego inicia una nueva sesión:

```sql
-- Crear solicitud vencida para prueba
INSERT INTO solicitudes (titulo, descripcion, estado, prioridad,
                          solicitante, fecha_vencimiento)
VALUES ('Solicitud vencida TEST', 'Para probar escalado automático',
        'PENDIENTE', 'ALTA', 'test@test.com', SYSDATE - 1);
COMMIT;
```

Después de iniciar sesión en APEX, ejecuta:

```sql
SELECT solicitud_id, titulo, estado
FROM   solicitudes
WHERE  titulo = 'Solicitud vencida TEST';
-- Debe mostrar estado = 'ESCALADA'
```

---

### Paso 6 — Implementar Seguridad: Identificar y Corregir SQL Dinámico Inseguro

**Objetivo:** Revisar ejemplos de código PL/SQL con vulnerabilidades de SQL injection y aplicar las correcciones usando bind variables y `APEX_UTIL`.

#### Instrucciones

1. En **SQL Workshop → SQL Commands**, analiza el siguiente ejemplo de código **INSEGURO** (NO ejecutes este código en producción):

```sql
-- ⚠️ CÓDIGO INSEGURO - SOLO PARA ANÁLISIS EDUCATIVO
-- Este código es vulnerable a SQL Injection
CREATE OR REPLACE PROCEDURE buscar_solicitudes_inseguro (
    p_estado IN VARCHAR2
) AS
    v_sql    VARCHAR2(4000);
    v_cursor SYS_REFCURSOR;
BEGIN
    -- ❌ VULNERABLE: concatenación directa de parámetros en SQL dinámico
    v_sql := 'SELECT * FROM solicitudes WHERE estado = ''' || p_estado || '''';
    OPEN v_cursor FOR v_sql;
    -- ...
END;
/
-- Un atacante podría pasar: ' OR '1'='1
-- Resultando en: SELECT * FROM solicitudes WHERE estado = '' OR '1'='1'
-- Que devuelve TODOS los registros sin importar el estado
```

2. Ejecuta el siguiente código **SEGURO** que corrige la vulnerabilidad:

```sql
-- ✅ CÓDIGO SEGURO - Usando bind variables
CREATE OR REPLACE PROCEDURE buscar_solicitudes_seguro (
    p_estado IN VARCHAR2
) AS
    v_cursor SYS_REFCURSOR;
    -- Sanitizar con APEX_UTIL antes de usar (si viene de input de usuario)
    v_estado_sanitizado VARCHAR2(30);
BEGIN
    -- Sanitización: solo permitir valores válidos del dominio
    v_estado_sanitizado := CASE UPPER(TRIM(p_estado))
                               WHEN 'PENDIENTE'   THEN 'PENDIENTE'
                               WHEN 'EN_REVISION' THEN 'EN_REVISION'
                               WHEN 'APROBADA'    THEN 'APROBADA'
                               WHEN 'RECHAZADA'   THEN 'RECHAZADA'
                               WHEN 'ESCALADA'    THEN 'ESCALADA'
                               ELSE NULL
                           END;

    IF v_estado_sanitizado IS NULL THEN
        RAISE_APPLICATION_ERROR(-20001, 'Estado inválido: ' ||
              SUBSTR(REPLACE(p_estado, CHR(39), ''), 1, 30));
    END IF;

    -- ✅ SEGURO: bind variable en SQL dinámico
    OPEN v_cursor FOR
        'SELECT * FROM solicitudes WHERE estado = :1'
        USING v_estado_sanitizado;

    -- Alternativa aún mejor: SQL estático con validación previa
    -- OPEN v_cursor FOR
    --     SELECT * FROM solicitudes WHERE estado = v_estado_sanitizado;

END buscar_solicitudes_seguro;
/
```

3. Agrega un proceso de validación de seguridad en la página 20. En el **Page Designer**, en la sección **Validations**, crea una nueva validación:

   - **Name:** `VAL_TITULO_SEGURO`
   - **Type:** `PL/SQL Function (returning Error Text)`
   - **Sequence:** `10`
   - **PL/SQL Function:**

```plsql
DECLARE
    v_titulo_limpio VARCHAR2(200);
BEGIN
    -- Verificar que el título no contiene caracteres de SQL injection
    -- APEX_UTIL.PREPARE_URL y APEX_ESCAPE.HTML son tus aliados
    IF :P20_TITULO IS NOT NULL THEN
        -- Verificar longitud razonable
        IF LENGTH(:P20_TITULO) > 200 THEN
            RETURN 'El título no puede exceder 200 caracteres.';
        END IF;

        -- Detectar patrones sospechosos (lista básica, no exhaustiva)
        IF REGEXP_LIKE(:P20_TITULO,
           '(--|;|/\*|\*/|xp_|EXEC\s|EXECUTE\s|DROP\s|ALTER\s)',
           'i') THEN
            -- Registrar intento sospechoso
            pkg_solicitudes.registrar_auditoria(
                p_accion      => 'INTENTO_SQLI',
                p_tabla       => 'SOLICITUDES',
                p_registro_id => NULL,
                p_usuario     => :APP_USER,
                p_pagina_id   => TO_NUMBER(:APP_PAGE_ID),
                p_detalle     => 'Posible SQL injection en P20_TITULO: ' ||
                                 SUBSTR(:P20_TITULO, 1, 100),
                p_resultado   => 'ERROR'
            );
            RETURN 'El título contiene caracteres no permitidos.';
        END IF;
    END IF;

    RETURN NULL;  -- NULL = validación exitosa
END;
```

   - **Associated Item:** `P20_TITULO`
   - **When Button Pressed:** `CREAR` (aplicar solo al crear)

4. Verifica que el paquete de seguridad `buscar_solicitudes_seguro` compiló correctamente:

```sql
SELECT object_name, object_type, status
FROM user_objects
WHERE object_name = 'BUSCAR_SOLICITUDES_SEGURO';
```

**Resultado esperado:**

```
OBJECT_NAME                  OBJECT_TYPE   STATUS
---------------------------- ------------- -------
BUSCAR_SOLICITUDES_SEGURO    PROCEDURE     VALID
```

**Verificación:** Intenta crear una solicitud con un título que contenga `--` (comentario SQL). El sistema debe rechazar el formulario con el mensaje de validación y registrar el intento en `AUDIT_LOG`.

---

## 7. Validación y Pruebas

Ejecuta las siguientes pruebas para validar que todos los componentes funcionan correctamente.

### Prueba 1 — Flujo Completo de Creación y Aprobación

```sql
-- Verificar que el flujo completo funciona
-- 1. Contar solicitudes antes
SELECT COUNT(*) solicitudes_antes FROM solicitudes;

-- 2. Verificar que audit_log registra acciones
SELECT accion, tabla_afect, usuario_apex, resultado,
       TO_CHAR(fecha_accion,'DD/MM/YYYY HH24:MI:SS') fecha
FROM   audit_log
ORDER  BY log_id DESC
FETCH  FIRST 10 ROWS ONLY;
```

Desde la interfaz APEX:
1. Navega a la página 20 sin parámetros (modo creación).
2. Llena: Título = `Solicitud de Prueba Final`, Prioridad = `ALTA`, Solicitante = `test@empresa.com`, Monto = `5000`.
3. Haz clic en **Crear**. Verifica el mensaje de éxito.
4. Anota el ID generado. Navega a `...&P20_SOLICITUD_ID=[ID]`.
5. Llena el campo Comentario con `Aprobado en prueba`.
6. Haz clic en **Aprobar**. Verifica que el estado cambia a `APROBADA`.

### Prueba 2 — Locking Optimista (Concurrencia)

```sql
-- Simular modificación concurrente
-- Abrir dos pestañas del navegador con la misma solicitud
-- En pestaña 1: cargar solicitud ID=1
-- En pestaña 2: aprobar la misma solicitud (esto incrementa rowversion)
-- En pestaña 1: intentar rechazar (rowversion desactualizado)
-- Resultado esperado: mensaje de conflicto de concurrencia

-- Verificar en BD que el rowversion se incrementó correctamente
SELECT solicitud_id, estado, rowversion, updated_by, updated_date
FROM   solicitudes
WHERE  solicitud_id = 1;
```

### Prueba 3 — Validación de Seguridad

Desde la interfaz APEX, intenta crear una solicitud con:
- **Título:** `Test -- DROP TABLE solicitudes`
- **Resultado esperado:** El formulario debe rechazarse con mensaje de error y el intento debe aparecer en `AUDIT_LOG` con `accion = 'INTENTO_SQLI'`.

```sql
-- Verificar registro del intento
SELECT accion, detalle, usuario_apex, resultado
FROM   audit_log
WHERE  accion = 'INTENTO_SQLI'
ORDER  BY log_id DESC;
```

### Prueba 4 — Proceso de Aplicación

```sql
-- Verificar que el proceso de aplicación escaló solicitudes vencidas
SELECT accion, detalle, fecha_accion
FROM   audit_log
WHERE  accion = 'AUTO_ESCALAR_VENCIDAS'
ORDER  BY log_id DESC;

-- Verificar que no hay solicitudes PENDIENTES vencidas
SELECT COUNT(*)
FROM   solicitudes
WHERE  estado           = 'PENDIENTE'
AND    fecha_vencimiento < SYSDATE;
-- Resultado esperado: 0
```

### Checklist de Validación Final

| Componente                                   | Resultado Esperado              | ✓/✗ |
|----------------------------------------------|---------------------------------|-----|
| Paquete `PKG_SOLICITUDES` compilado          | STATUS = VALID (spec + body)    |     |
| Proceso Crear ejecuta con REQUEST='CREAR'    | Registro insertado + auditoría  |     |
| Proceso Aprobar con rowversion correcto      | Estado = APROBADA               |     |
| Proceso Aprobar con rowversion incorrecto    | Mensaje de conflicto            |     |
| Computación fecha_vencimiento                | Calculada según prioridad       |     |
| Application Process escala vencidas          | Solicitudes vencidas escaladas  |     |
| Validación SQL injection                     | Rechaza títulos con `--`        |     |
| Tabla `AUDIT_LOG` registra todas las acciones| Filas visibles en consulta      |     |

---

## 8. Solución de Problemas

### Problema 1 — El proceso PL/SQL falla con `ORA-06502: PL/SQL: numeric or value error`

**Síntomas:** Al hacer clic en Aprobar/Rechazar/Escalar, aparece el error `ORA-06502` en la notificación de APEX. El estado de la solicitud no cambia.

**Causa:** El ítem `P20_ROWVERSION` está vacío o contiene un valor no numérico cuando se llama a `TO_NUMBER(:P20_ROWVERSION)`. Esto ocurre cuando la computación del `rowversion` no se ejecutó correctamente (por ejemplo, si el ítem `P20_SOLICITUD_ID` no estaba disponible en el momento de la computación) o cuando se navega a la página sin pasar el parámetro de clave primaria en la URL.

**Solución:**
1. Verifica que la computación `P20_ROWVERSION` tiene el **Point** configurado como `Before Regions` (no `After Footer`).
2. Confirma que la condición de la computación es `Item is NOT NULL` sobre `P20_SOLICITUD_ID`.
3. Agrega un valor por defecto al ítem `P20_ROWVERSION` en sus propiedades: **Default Value** = `0`.
4. Modifica los procesos para manejar el caso nulo:

```plsql
-- Al inicio de cada proceso de aprobación/rechazo/escalamiento
-- Reemplazar TO_NUMBER(:P20_ROWVERSION) por:
NVL(TO_NUMBER(:P20_ROWVERSION), 0)
```

5. Para confirmar que el rowversion se está cargando, ejecuta en SQL Workshop:
```sql
SELECT solicitud_id, rowversion FROM solicitudes WHERE ROWNUM = 1;
```
Compara el valor con lo que muestra la sesión APEX en **Session State** (App Builder → Session).

---

### Problema 2 — El paquete `PKG_SOLICITUDES` compila como `INVALID` después de cambios en la tabla

**Síntomas:** Al ejecutar `SELECT status FROM user_objects WHERE object_name = 'PKG_SOLICITUDES'`, el resultado muestra `INVALID`. Los procesos de página fallan con `ORA-04068: existing state of packages has been discarded` o `ORA-04061`.

**Causa:** Si se modificó la estructura de la tabla `SOLICITUDES` (agregar/eliminar columnas) o si se recompiló algún objeto del que depende el paquete, Oracle invalida automáticamente los objetos dependientes. Esto también ocurre si el cuerpo del paquete referencia columnas con `%TYPE` y la tabla cambió.

**Solución:**
1. Identifica los errores de compilación específicos:
```sql
SELECT line, position, text
FROM   user_errors
WHERE  name = 'PKG_SOLICITUDES'
ORDER  BY sequence;
```
2. Si el error es por cambio de tabla, recompila el paquete:
```sql
ALTER PACKAGE pkg_solicitudes COMPILE;
ALTER PACKAGE pkg_solicitudes COMPILE BODY;
```
3. Si persiste, verifica que la tabla tiene todas las columnas referenciadas:
```sql
SELECT column_name, data_type, data_length
FROM   user_tab_columns
WHERE  table_name = 'SOLICITUDES'
ORDER  BY column_id;
```
4. Si se agregaron columnas nuevas que el paquete necesita, re-ejecuta el script de creación del cuerpo del paquete completo (Paso 1, instrucción 3).
5. Para invalidaciones frecuentes en desarrollo, usa el comando de recompilación en cascada:
```sql
EXEC DBMS_UTILITY.COMPILE_SCHEMA(schema => USER, compile_all => FALSE);
```

---

## 9. Limpieza del Entorno

Si necesitas reiniciar el laboratorio o liberar recursos, ejecuta los siguientes scripts **en el orden indicado**:

```sql
-- ============================================================
-- SCRIPT DE LIMPIEZA - LAB 07-00-01
-- ⚠️ ADVERTENCIA: Elimina todos los objetos creados en este lab
-- Ejecutar SOLO si deseas reiniciar desde cero
-- ============================================================

-- 1. Eliminar procesos de página y computaciones
-- (Se eliminan manualmente desde el Page Designer de APEX
--  o eliminando y recreando la página 20)

-- 2. Eliminar Application Process (desde Shared Components > Application Processes)

-- 3. Eliminar Application Item
-- (Shared Components > Application Items > APP_SOLICITUDES_VENCIDAS)

-- 4. Eliminar objetos de base de datos
DROP PROCEDURE buscar_solicitudes_seguro;
DROP PROCEDURE buscar_solicitudes_inseguro;

DROP PACKAGE pkg_solicitudes;

-- 5. Limpiar datos de prueba (mantener estructura para otros labs)
DELETE FROM audit_log
WHERE  accion IN ('INTENTO_SQLI','AUTO_ESCALAR_VENCIDAS',
                   'CREAR_SOLICITUD','APROBAR_SOLICITUD',
                   'RECHAZAR_SOLICITUD','ESCALAR_SOLICITUD',
                   'AUTO_ESCALAR_ERROR','VER_SOLICITUD');

DELETE FROM solicitudes WHERE titulo LIKE '%TEST%' OR titulo LIKE '%Prueba%';

COMMIT;

-- 6. Verificar limpieza
SELECT 'PKG_SOLICITUDES' objeto,
       COUNT(*) existencias
FROM   user_objects
WHERE  object_name = 'PKG_SOLICITUDES'
UNION ALL
SELECT 'audit_log registros', COUNT(*) FROM audit_log
UNION ALL
SELECT 'solicitudes registros', COUNT(*) FROM solicitudes;
```

> **Nota:** Las tablas `SOLICITUDES`, `AUDIT_LOG` y `NOTIFICACIONES_PENDIENTES` se conservan intencionalmente ya que serán utilizadas en los Laboratorios 08, 09 y 10.

---

## 10. Resumen

En este laboratorio implementaste un sistema completo de lógica de negocio backend para una aplicación Oracle APEX 23.2. Los conceptos y habilidades clave que desarrollaste incluyen:

| Concepto                                  | Implementación Realizada                                                        |
|-------------------------------------------|---------------------------------------------------------------------------------|
| **Puntos de ejecución de procesos**       | Procesos en `Before Header` (auditoría) y `On Submit - After Computations and Validations` (lógica de negocio) |
| **Paquetes PL/SQL**                       | `PKG_SOLICITUDES` con procedimientos encapsulados, tipos de registro y constantes |
| **Manejo de transacciones**               | `PRAGMA AUTONOMOUS_TRANSACTION` para auditoría; `COMMIT`/`ROLLBACK` explícitos  |
| **Locking optimista (concurrencia)**      | Campo `rowversion` verificado antes de cada actualización                        |
| **Computaciones de página**               | Cálculo de fecha de vencimiento, porcentaje de avance y carga de rowversion     |
| **Application Process**                   | Proceso global de escalado automático ejecutado al inicio de sesión             |
| **Seguridad PL/SQL**                      | Bind variables, validación de dominio, detección de patrones de SQL injection   |
| **Manejo de errores con APEX**            | `APEX_ERROR.ADD_ERROR` para mensajes de usuario; `EXCEPTION` con re-lanzamiento |

### Patrones Arquitectónicos Aplicados

- **Separación de responsabilidades:** La lógica de negocio reside en el paquete PL/SQL; APEX solo orquesta la invocación.
- **Auditoría trazable:** Cada operación queda registrada en `AUDIT_LOG` con usuario, timestamp y detalle.
- **Defensa en profundidad:** Validaciones en múltiples capas (APEX Validations + PL/SQL).
- **Encadenamiento de procesos:** Múltiples procesos con secuencias controladas para flujos complejos.

### Recursos Adicionales

- [Oracle APEX 23.2 — Understanding Page Processing](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/understanding-page-processing-and-page-rendering.html)
- [Oracle APEX 23.2 — Creating Page Processes](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/creating-page-processes.html)
- [Oracle PL/SQL — PRAGMA AUTONOMOUS_TRANSACTION](https://docs.oracle.com/en/database/oracle/oracle-database/21/lnpls/AUTONOMOUS_TRANSACTION-pragma.html)
- [Oracle APEX — APEX_ERROR Package](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_ERROR.html)
- [Oracle APEX — APEX_UTIL Package](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_UTIL.html)
- [Oracle LiveSQL — PL/SQL and APEX Integration](https://livesql.oracle.com)

---
