# Práctica 9: Implementar autenticación, roles y pruebas de seguridad

## Metadatos

| Atributo         | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 52 minutos                                 |
| **Complejidad**  | Alta                                       |
| **Nivel Bloom**  | Crear (Create)                             |
| **Laboratorio**  | 09-00-01                                   |
| **Prerrequisito**| Labs 04-00-01 al 08-00-01 completados      |

---

## Descripción General

En este laboratorio implementarás una capa de seguridad completa sobre la aplicación construida en laboratorios anteriores. Comenzarás creando una tabla de usuarios propia con contraseñas hasheadas mediante `DBMS_CRYPTO`, luego configurarás un esquema de autenticación personalizado y lo compararás con APEX Accounts estándar. Diseñarás un modelo de cuatro roles de negocio (Administrador, Supervisor, Operador, Consultor), crearás Authorization Schemes que consulten la membresía desde la base de datos, habilitarás Session State Protection y finalmente ejecutarás pruebas de penetración básicas usando el APEX Security Advisor y Chrome DevTools para identificar y corregir vulnerabilidades críticas.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Implementar un esquema de autenticación personalizado en APEX con contraseñas hasheadas usando `DBMS_CRYPTO`, y compararlo con APEX Accounts estándar
- [ ] Diseñar y configurar Authorization Schemes basados en roles de negocio para controlar el acceso a páginas, regiones, ítems y botones
- [ ] Habilitar Session State Protection (SSP) y protección CSRF a nivel de aplicación e ítems individuales
- [ ] Aplicar técnicas de encriptación con `APEX_UTIL` y protección de ítems de sesión sensibles
- [ ] Ejecutar pruebas de seguridad básicas y corregir todos los ítems marcados como Critical o High en el APEX Security Advisor

---

## Prerrequisitos

### Conocimiento requerido

- Haber completado los Laboratorios 04-00-01 al 08-00-01 (aplicación base funcional)
- PL/SQL intermedio: funciones, manejo de excepciones, consultas DML
- Conceptos de seguridad web: autenticación vs. autorización, sesiones, CSRF, XSS, inyección SQL
- Conceptos básicos de criptografía: diferencia entre hashing y encriptación simétrica

### Acceso requerido

- Acceso al workspace APEX con la aplicación de los laboratorios anteriores
- Permisos de DBA o acceso al esquema propietario para crear tablas y paquetes PL/SQL
- Acceso a APEX Administration Services (cuenta `internal`) — ver nota del instructor
- Google Chrome con DevTools habilitado
- Oracle SQL Developer o acceso a SQL Workshop en APEX

> **⚠️ Nota de Seguridad:** Las pruebas de penetración de este laboratorio deben ejecutarse **únicamente** sobre tu aplicación personal en el entorno de laboratorio. Está estrictamente prohibido aplicar estas técnicas sobre otras aplicaciones o sistemas fuera del alcance de este laboratorio.

---

## Entorno de Laboratorio

### Hardware mínimo requerido

| Componente     | Mínimo                        | Recomendado                   |
|----------------|-------------------------------|-------------------------------|
| RAM            | 8 GB                          | 16 GB                         |
| Almacenamiento | 50 GB libres (SSD)            | 100 GB SSD                    |
| Procesador     | Intel i5 8ª gen / Ryzen 5     | Intel i7 / Ryzen 7            |
| Red            | 10 Mbps                       | 25 Mbps+                      |
| Pantalla       | 1280×768                      | 1920×1080                     |

### Software requerido

| Software               | Versión mínima | Propósito en este lab                        |
|------------------------|----------------|----------------------------------------------|
| Oracle APEX            | 23.2           | Plataforma principal de desarrollo           |
| Oracle Database        | 19c / 21c XE   | Almacenamiento y lógica PL/SQL               |
| ORDS                   | 23.2           | Capa de acceso HTTP a APEX                   |
| Oracle SQL Developer   | 23.1           | Ejecución de scripts SQL/PL/SQL              |
| Google Chrome          | 110+           | Pruebas de seguridad con DevTools            |
| Postman                | 10.0+          | Pruebas de endpoints REST (opcional)         |

### Verificación del entorno antes de comenzar

Ejecuta el siguiente script en SQL Workshop (SQL Commands) para confirmar que tienes los privilegios necesarios:

```sql
-- Verificar acceso a DBMS_CRYPTO
SELECT DBMS_CRYPTO.HASH(
    UTL_RAW.CAST_TO_RAW('test'),
    DBMS_CRYPTO.HASH_SH256
) AS hash_prueba
FROM DUAL;

-- Verificar versión de APEX
SELECT VERSION_NO FROM APEX_RELEASE;

-- Verificar que la aplicación base existe
SELECT APPLICATION_ID, APPLICATION_NAME, STATUS
FROM APEX_APPLICATIONS
WHERE WORKSPACE = :WORKSPACE_NAME
ORDER BY APPLICATION_ID;
```

> Si `DBMS_CRYPTO.HASH` devuelve un error de privilegios, solicita al instructor que ejecute: `GRANT EXECUTE ON SYS.DBMS_CRYPTO TO <tu_esquema>;`

---

## Pasos del Laboratorio

---

### Paso 1: Crear la infraestructura de usuarios y roles en la base de datos

**Objetivo:** Construir las tablas de soporte para el sistema de autenticación personalizado y el modelo de roles de negocio.

#### Instrucciones

1. Abre **SQL Workshop → SQL Scripts** en tu workspace APEX.

2. Crea un nuevo script llamado `lab09_setup_seguridad.sql` y pega el siguiente código:

```sql
-- ============================================================
-- LAB 09: Infraestructura de seguridad
-- ============================================================

-- Tabla de usuarios de la aplicación
CREATE TABLE app_usuarios (
    usuario_id     NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username       VARCHAR2(50)  NOT NULL UNIQUE,
    email          VARCHAR2(100) NOT NULL UNIQUE,
    password_hash  RAW(32)       NOT NULL,
    nombre_completo VARCHAR2(150),
    activo         CHAR(1)       DEFAULT 'S' NOT NULL CHECK (activo IN ('S','N')),
    intentos_fallidos NUMBER DEFAULT 0,
    bloqueado_hasta   TIMESTAMP,
    fecha_creacion TIMESTAMP     DEFAULT SYSTIMESTAMP,
    ultimo_acceso  TIMESTAMP,
    CONSTRAINT chk_intentos CHECK (intentos_fallidos >= 0)
);

-- Tabla de roles de negocio
CREATE TABLE app_roles (
    rol_id         NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre_rol     VARCHAR2(50)  NOT NULL UNIQUE,
    descripcion    VARCHAR2(200),
    activo         CHAR(1)       DEFAULT 'S' CHECK (activo IN ('S','N'))
);

-- Tabla de membresía usuario-rol
CREATE TABLE app_usuario_roles (
    usuario_id  NUMBER NOT NULL REFERENCES app_usuarios(usuario_id),
    rol_id      NUMBER NOT NULL REFERENCES app_roles(rol_id),
    fecha_asignacion DATE DEFAULT SYSDATE,
    asignado_por VARCHAR2(50),
    PRIMARY KEY (usuario_id, rol_id)
);

-- Insertar roles de negocio
INSERT INTO app_roles (nombre_rol, descripcion) VALUES
    ('ADMINISTRADOR', 'Acceso completo a todas las funciones y configuración del sistema');
INSERT INTO app_roles (nombre_rol, descripcion) VALUES
    ('SUPERVISOR', 'Acceso a reportes, aprobaciones y gestión de su equipo');
INSERT INTO app_roles (nombre_rol, descripcion) VALUES
    ('OPERADOR', 'Acceso a operaciones diarias: crear, editar registros propios');
INSERT INTO app_roles (nombre_rol, descripcion) VALUES
    ('CONSULTOR', 'Acceso de solo lectura a reportes y dashboards');

-- Insertar usuarios de prueba con contraseñas hasheadas
-- Contraseña para todos: 'Lab09Secure#2024'
DECLARE
    v_hash RAW(32);
    v_password VARCHAR2(50) := 'Lab09Secure#2024';
    v_uid_admin NUMBER;
    v_uid_sup   NUMBER;
    v_uid_oper  NUMBER;
    v_uid_cons  NUMBER;
BEGIN
    v_hash := DBMS_CRYPTO.HASH(
        UTL_RAW.CAST_TO_RAW(v_password),
        DBMS_CRYPTO.HASH_SH256
    );

    INSERT INTO app_usuarios (username, email, password_hash, nombre_completo)
    VALUES ('admin_lab', 'admin@lab.com', v_hash, 'Administrador del Sistema')
    RETURNING usuario_id INTO v_uid_admin;

    INSERT INTO app_usuarios (username, email, password_hash, nombre_completo)
    VALUES ('supervisor1', 'supervisor@lab.com', v_hash, 'Supervisor Regional')
    RETURNING usuario_id INTO v_uid_sup;

    INSERT INTO app_usuarios (username, email, password_hash, nombre_completo)
    VALUES ('operador1', 'operador@lab.com', v_hash, 'Operador de Datos')
    RETURNING usuario_id INTO v_uid_oper;

    INSERT INTO app_usuarios (username, email, password_hash, nombre_completo)
    VALUES ('consultor1', 'consultor@lab.com', v_hash, 'Consultor Externo')
    RETURNING usuario_id INTO v_uid_cons;

    -- Asignar roles
    INSERT INTO app_usuario_roles (usuario_id, rol_id, asignado_por)
    SELECT v_uid_admin, rol_id, 'SETUP' FROM app_roles WHERE nombre_rol = 'ADMINISTRADOR';

    INSERT INTO app_usuario_roles (usuario_id, rol_id, asignado_por)
    SELECT v_uid_sup, rol_id, 'SETUP' FROM app_roles WHERE nombre_rol = 'SUPERVISOR';

    INSERT INTO app_usuario_roles (usuario_id, rol_id, asignado_por)
    SELECT v_uid_oper, rol_id, 'SETUP' FROM app_roles WHERE nombre_rol = 'OPERADOR';

    INSERT INTO app_usuario_roles (usuario_id, rol_id, asignado_por)
    SELECT v_uid_cons, rol_id, 'SETUP' FROM app_roles WHERE nombre_rol = 'CONSULTOR';

    COMMIT;
    DBMS_OUTPUT.PUT_LINE('Infraestructura de seguridad creada exitosamente.');
END;
/
```

3. Haz clic en **Run** y verifica que no aparezcan errores.

4. Crea el paquete de seguridad que encapsulará la lógica de autenticación:

```sql
CREATE OR REPLACE PACKAGE pkg_seguridad AS
    -- Función principal de autenticación personalizada
    FUNCTION autenticar_usuario(
        p_username IN VARCHAR2,
        p_password IN VARCHAR2
    ) RETURN BOOLEAN;

    -- Procedimiento para registrar intento fallido
    PROCEDURE registrar_intento_fallido(p_username IN VARCHAR2);

    -- Función para verificar si el usuario tiene un rol específico
    FUNCTION tiene_rol(
        p_username IN VARCHAR2,
        p_rol      IN VARCHAR2
    ) RETURN BOOLEAN;

    -- Procedimiento de post-autenticación (cargar contexto)
    PROCEDURE post_autenticacion;

END pkg_seguridad;
/

CREATE OR REPLACE PACKAGE BODY pkg_seguridad AS

    FUNCTION autenticar_usuario(
        p_username IN VARCHAR2,
        p_password IN VARCHAR2
    ) RETURN BOOLEAN IS
        v_count         NUMBER := 0;
        v_hash          RAW(32);
        v_bloqueado     TIMESTAMP;
        v_intentos      NUMBER;
    BEGIN
        -- Calcular hash de la contraseña proporcionada
        v_hash := DBMS_CRYPTO.HASH(
            UTL_RAW.CAST_TO_RAW(p_password),
            DBMS_CRYPTO.HASH_SH256
        );

        -- Verificar si la cuenta está bloqueada
        SELECT NVL(bloqueado_hasta, SYSTIMESTAMP - 1), intentos_fallidos
        INTO v_bloqueado, v_intentos
        FROM app_usuarios
        WHERE UPPER(username) = UPPER(p_username)
          AND activo = 'S';

        IF v_bloqueado > SYSTIMESTAMP THEN
            RAISE_APPLICATION_ERROR(-20001,
                'Cuenta bloqueada temporalmente. Intente después de las ' ||
                TO_CHAR(v_bloqueado, 'HH24:MI'));
        END IF;

        -- Verificar credenciales
        SELECT COUNT(*)
        INTO v_count
        FROM app_usuarios
        WHERE UPPER(username) = UPPER(p_username)
          AND password_hash = v_hash
          AND activo = 'S';

        IF v_count > 0 THEN
            -- Autenticación exitosa: resetear intentos y actualizar último acceso
            UPDATE app_usuarios
            SET intentos_fallidos = 0,
                bloqueado_hasta   = NULL,
                ultimo_acceso     = SYSTIMESTAMP
            WHERE UPPER(username) = UPPER(p_username);
            COMMIT;
            RETURN TRUE;
        ELSE
            -- Registrar intento fallido
            registrar_intento_fallido(p_username);
            RETURN FALSE;
        END IF;

    EXCEPTION
        WHEN NO_DATA_FOUND THEN
            -- Usuario no existe: no revelar este detalle (timing attack mitigation)
            RETURN FALSE;
        WHEN OTHERS THEN
            RETURN FALSE;
    END autenticar_usuario;

    PROCEDURE registrar_intento_fallido(p_username IN VARCHAR2) IS
        v_intentos NUMBER;
    BEGIN
        SELECT intentos_fallidos INTO v_intentos
        FROM app_usuarios
        WHERE UPPER(username) = UPPER(p_username)
        FOR UPDATE;

        v_intentos := v_intentos + 1;

        IF v_intentos >= 5 THEN
            -- Bloquear cuenta por 30 minutos
            UPDATE app_usuarios
            SET intentos_fallidos = v_intentos,
                bloqueado_hasta   = SYSTIMESTAMP + INTERVAL '30' MINUTE
            WHERE UPPER(username) = UPPER(p_username);
        ELSE
            UPDATE app_usuarios
            SET intentos_fallidos = v_intentos
            WHERE UPPER(username) = UPPER(p_username);
        END IF;
        COMMIT;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN NULL;
    END registrar_intento_fallido;

    FUNCTION tiene_rol(
        p_username IN VARCHAR2,
        p_rol      IN VARCHAR2
    ) RETURN BOOLEAN IS
        v_count NUMBER;
    BEGIN
        SELECT COUNT(*)
        INTO v_count
        FROM app_usuarios u
        JOIN app_usuario_roles ur ON u.usuario_id = ur.usuario_id
        JOIN app_roles r          ON ur.rol_id    = r.rol_id
        WHERE UPPER(u.username) = UPPER(p_username)
          AND UPPER(r.nombre_rol) = UPPER(p_rol)
          AND u.activo = 'S'
          AND r.activo = 'S';

        RETURN v_count > 0;
    END tiene_rol;

    PROCEDURE post_autenticacion IS
    BEGIN
        -- Cargar nombre completo en un ítem de aplicación
        -- (El ítem APP_NOMBRE_USUARIO debe existir como Application Item)
        APEX_UTIL.SET_SESSION_STATE(
            p_name  => 'APP_NOMBRE_USUARIO',
            p_value => (
                SELECT nombre_completo
                FROM app_usuarios
                WHERE UPPER(username) = UPPER(APEX_APPLICATION.G_USER)
            )
        );
    EXCEPTION
        WHEN OTHERS THEN NULL;
    END post_autenticacion;

END pkg_seguridad;
/
```

5. Ejecuta el script y verifica que el paquete compile sin errores (`Package body created`).

**Resultado esperado:**
- Tablas `APP_USUARIOS`, `APP_ROLES`, `APP_USUARIO_ROLES` creadas con datos de prueba
- Paquete `PKG_SEGURIDAD` compilado exitosamente
- 4 usuarios con contraseñas hasheadas y roles asignados

**Verificación:**
```sql
-- Confirmar estructura y datos
SELECT u.username, r.nombre_rol, u.activo
FROM app_usuarios u
JOIN app_usuario_roles ur ON u.usuario_id = ur.usuario_id
JOIN app_roles r ON ur.rol_id = r.rol_id
ORDER BY r.nombre_rol;

-- Probar la función de autenticación
SELECT pkg_seguridad.autenticar_usuario('admin_lab', 'Lab09Secure#2024') FROM DUAL;
-- Debe retornar: 1 (TRUE en contexto SQL)
```

---

### Paso 2: Configurar el esquema de autenticación personalizado en APEX

**Objetivo:** Registrar la función PL/SQL como esquema de autenticación activo en la aplicación y crear el Application Item para el nombre del usuario.

#### Instrucciones

1. En APEX App Builder, abre tu aplicación. En el menú lateral, navega a **Shared Components → Security → Authentication Schemes**.

2. Antes de crear el nuevo esquema, anota el nombre del esquema actual (probablemente "Application Express Authentication"). Esto te permitirá comparar ambos en el Paso 3.

3. Haz clic en **Create** y selecciona **Based on a pre-configured scheme from the gallery**.

4. En la galería, selecciona **Custom** y haz clic en **Next**.

5. Completa los campos del nuevo esquema:

   | Campo | Valor |
   |-------|-------|
   | Name | `Custom Auth - DB Hash` |
   | Scheme Type | `Custom` |
   | Authentication Function Name | `pkg_seguridad.autenticar_usuario` |
   | Logout URL | `apex_authentication.logout?p_app_id=&APP_ID.&p_session_id=&APP_SESSION.` |
   | Invalid Session Target | `Page in this Application` |
   | Page | `101` (tu página de login) |

6. En la sección **Post-Authentication Procedure Name**, escribe: `pkg_seguridad.post_autenticacion`

7. Haz clic en **Create Authentication Scheme**.

8. Marca el nuevo esquema como **Current** haciendo clic en el ícono de estrella junto a su nombre, o seleccionándolo y eligiendo **Make Current Scheme**.

9. Ahora crea el Application Item para el nombre de usuario. Navega a **Shared Components → Application Items** y haz clic en **Create**:

   | Campo | Valor |
   |-------|-------|
   | Name | `APP_NOMBRE_USUARIO` |
   | Scope | `Application` |
   | Session State Protection | `Restricted - May not be set from browser` |

10. Haz clic en **Create Application Item**.

**Resultado esperado:**
- Esquema `Custom Auth - DB Hash` marcado como Current
- Application Item `APP_NOMBRE_USUARIO` creado con protección de sesión

**Verificación:**
- Cierra sesión en la aplicación y vuelve a iniciar sesión con `admin_lab` / `Lab09Secure#2024`
- La autenticación debe funcionar correctamente
- Intenta iniciar sesión con credenciales incorrectas 5 veces seguidas; en el sexto intento debe aparecer el mensaje de cuenta bloqueada

---

### Paso 3: Comparar con APEX Accounts y documentar diferencias

**Objetivo:** Crear una segunda instancia del esquema APEX Accounts estándar para comparación, y documentar las diferencias arquitectónicas entre ambos enfoques.

#### Instrucciones

1. Navega a **Shared Components → Authentication Schemes → Create**.

2. Selecciona **Application Express Authentication** desde la galería y nómbralo `APEX Accounts - Referencia`.

3. **No lo marques como Current** (es solo para referencia y comparación).

4. Abre ambos esquemas en pestañas separadas del navegador y completa la siguiente tabla comparativa en tu cuaderno o documento de entrega:

   | Criterio | APEX Accounts | Custom Auth - DB Hash |
   |----------|---------------|----------------------|
   | Almacenamiento de contraseñas | Tablas internas APEX | Tabla `APP_USUARIOS` propia |
   | Algoritmo de hashing | Gestionado por APEX | `DBMS_CRYPTO.HASH_SH256` |
   | Control de bloqueo por intentos | No nativo | Implementado en `PKG_SEGURIDAD` |
   | Gestión de usuarios | APEX Admin UI | SQL / interfaz propia |
   | Integración con roles de negocio | Limitada | Total (tabla `APP_USUARIO_ROLES`) |
   | Caso de uso recomendado | Desarrollo/pruebas | Producción con requisitos propios |

5. En SQL Workshop, ejecuta la siguiente consulta para ver los esquemas registrados:

```sql
SELECT authentication_name, authentication_type, is_current
FROM apex_application_authorization
WHERE application_id = :APP_ID;

-- Alternativa directa:
SELECT scheme_name, scheme_type, is_current_scheme
FROM apex_authentication_schemes
WHERE application_id = (
    SELECT application_id FROM apex_applications
    WHERE alias = 'TU_ALIAS_APP'
);
```

**Resultado esperado:**
- Dos esquemas visibles: `Custom Auth - DB Hash` (current) y `APEX Accounts - Referencia`
- Tabla comparativa completada

**Verificación:**
- Confirma en la UI de APEX que solo `Custom Auth - DB Hash` tiene el indicador de esquema activo

---

### Paso 4: Crear Authorization Schemes basados en roles

**Objetivo:** Implementar cuatro Authorization Schemes en APEX que consulten la membresía de roles desde la base de datos, para controlar el acceso granular en la aplicación.

#### Instrucciones

1. Navega a **Shared Components → Security → Authorization Schemes**.

2. Crea el primer esquema para el rol Administrador. Haz clic en **Create**:

   | Campo | Valor |
   |-------|-------|
   | Name | `Rol: Administrador` |
   | Scheme Type | `PL/SQL Function Returning Boolean` |
   | Error Message | `Acceso denegado. Se requiere rol de Administrador.` |

   En el campo **PL/SQL Function Body**:
   ```sql
   RETURN pkg_seguridad.tiene_rol(
       p_username => :APP_USER,
       p_rol      => 'ADMINISTRADOR'
   );
   ```

   | Campo | Valor |
   |-------|-------|
   | Caching | `Once Per Session` |

3. Haz clic en **Create Authorization Scheme**. Repite el proceso para los tres roles restantes:

   **Esquema: Rol Supervisor**
   ```sql
   RETURN pkg_seguridad.tiene_rol(:APP_USER, 'SUPERVISOR')
       OR pkg_seguridad.tiene_rol(:APP_USER, 'ADMINISTRADOR');
   ```
   > Nota: El Administrador hereda acceso de Supervisor (jerarquía de roles).

   **Esquema: Rol Operador**
   ```sql
   RETURN pkg_seguridad.tiene_rol(:APP_USER, 'OPERADOR')
       OR pkg_seguridad.tiene_rol(:APP_USER, 'SUPERVISOR')
       OR pkg_seguridad.tiene_rol(:APP_USER, 'ADMINISTRADOR');
   ```

   **Esquema: Rol Consultor (Solo Lectura)**
   ```sql
   RETURN pkg_seguridad.tiene_rol(:APP_USER, 'CONSULTOR')
       OR pkg_seguridad.tiene_rol(:APP_USER, 'OPERADOR')
       OR pkg_seguridad.tiene_rol(:APP_USER, 'SUPERVISOR')
       OR pkg_seguridad.tiene_rol(:APP_USER, 'ADMINISTRADOR');
   ```

4. Aplica los esquemas de autorización a los componentes de la aplicación. Sigue esta matriz de asignación:

   | Componente de la aplicación | Authorization Scheme a aplicar |
   |-----------------------------|-------------------------------|
   | Página de Administración (p. ej. p. 10) | `Rol: Administrador` |
   | Página de Reportes de Gestión | `Rol: Supervisor` |
   | Botón "Eliminar Registro" | `Rol: Administrador` |
   | Botón "Aprobar / Rechazar" | `Rol: Supervisor` |
   | Botón "Crear / Editar" | `Rol: Operador` |
   | Región de Dashboard principal | `Rol: Consultor (Solo Lectura)` |
   | Región de configuración del sistema | `Rol: Administrador` |

5. Para aplicar una autorización a una **página**, abre la página en el Page Designer, haz clic en la raíz de la página (Page node), y en el panel de propiedades busca la sección **Security → Authorization Scheme**. Selecciona el esquema correspondiente.

6. Para aplicar a un **botón o región**, haz clic sobre el componente en el Page Designer, y en la sección **Security** del panel de propiedades, asigna el Authorization Scheme.

**Resultado esperado:**
- 4 Authorization Schemes creados y visibles en Shared Components
- Páginas, regiones y botones configurados con sus respectivos esquemas

**Verificación:**
- Inicia sesión como `consultor1` e intenta navegar a la página de Administración → debe aparecer el mensaje "Acceso denegado. Se requiere rol de Administrador."
- Inicia sesión como `operador1` e intenta usar el botón "Eliminar" → debe estar oculto o mostrar error de autorización
- Inicia sesión como `admin_lab` → debe tener acceso completo a todas las páginas

---

### Paso 5: Habilitar Session State Protection (SSP)

**Objetivo:** Configurar la protección del estado de sesión para prevenir la manipulación de parámetros en la URL.

#### Instrucciones

1. Navega a **Shared Components → Security → Session State Protection**.

2. En la parte superior de la pantalla, verás el estado actual de SSP para la aplicación. Haz clic en **Edit** o en el botón **Enable Session State Protection**.

3. En la pantalla de configuración, establece:

   | Parámetro | Valor |
   |-----------|-------|
   | Session State Protection | `Enabled` |

4. Haz clic en **Apply Changes**. APEX mostrará un resumen de todos los ítems de página que ahora tienen protección.

5. Ahora configura la protección individual de los ítems críticos. Para cada página que tenga parámetros de navegación (por ejemplo, `P10_ID_REGISTRO`, `P5_PROYECTO_ID`), abre la página en Page Designer y para cada ítem de tipo "Hidden" o de navegación:

   - Haz clic en el ítem
   - En el panel de propiedades, sección **Security**:

   | Campo | Valor recomendado |
   |-------|-------------------|
   | Session State Protection | `Checksum Required - Session Level` |

6. Para ítems que contienen datos sensibles que **no** deben ser modificables desde la URL:

   | Campo | Valor |
   |-------|-------|
   | Session State Protection | `Restricted - May not be set from browser` |

7. Verifica la configuración ejecutando este query para ver el estado de protección:

```sql
SELECT page_id, item_name, item_protection_level
FROM apex_application_page_items
WHERE application_id = :APP_ID
  AND item_protection_level != 'Unrestricted'
ORDER BY page_id, item_name;
```

8. Adicionalmente, configura la protección a nivel de URL para las páginas críticas. En cada página de administración o con datos sensibles, abre las propiedades de la página y en **Security → Page Access Protection** selecciona `Arguments Must Have Checksum`.

**Resultado esperado:**
- SSP habilitado a nivel de aplicación
- Ítems de navegación configurados con `Checksum Required - Session Level`
- Ítems sensibles configurados con `Restricted - May not be set from browser`

**Verificación:**
```sql
-- Ver resumen de protecciones aplicadas
SELECT
    page_id,
    COUNT(*) AS total_items,
    SUM(CASE WHEN item_protection_level = 'Unrestricted' THEN 1 ELSE 0 END) AS sin_proteccion,
    SUM(CASE WHEN item_protection_level != 'Unrestricted' THEN 1 ELSE 0 END) AS protegidos
FROM apex_application_page_items
WHERE application_id = :APP_ID
GROUP BY page_id
ORDER BY page_id;
```

---

### Paso 6: Implementar encriptación de datos sensibles con APEX_UTIL

**Objetivo:** Aplicar encriptación simétrica a datos sensibles almacenados en el estado de sesión y demostrar el uso de `APEX_UTIL.ENCRYPT` / `DECRYPT`.

#### Instrucciones

1. Primero, crea una clave de encriptación almacenada de forma segura. En SQL Workshop, ejecuta:

```sql
-- Crear tabla para almacenar configuración segura
CREATE TABLE app_config_segura (
    config_key   VARCHAR2(100) PRIMARY KEY,
    config_value VARCHAR2(4000),
    descripcion  VARCHAR2(200)
);

-- Generar y almacenar una clave de encriptación
-- IMPORTANTE: En producción, esta clave debe gestionarse con Oracle Wallet o HSM
INSERT INTO app_config_segura (config_key, config_value, descripcion)
VALUES (
    'ENCRYPTION_KEY',
    RAWTOHEX(DBMS_CRYPTO.RANDOMBYTES(32)),
    'Clave AES-256 para encriptación de datos sensibles en sesión'
);
COMMIT;

-- Verificar
SELECT config_key, SUBSTR(config_value, 1, 16) || '...' AS key_preview
FROM app_config_segura
WHERE config_key = 'ENCRYPTION_KEY';
```

2. Crea un procedimiento para encriptar/desencriptar datos sensibles en la sesión:

```sql
CREATE OR REPLACE PACKAGE pkg_encriptacion AS

    FUNCTION encriptar(p_valor IN VARCHAR2) RETURN VARCHAR2;
    FUNCTION desencriptar(p_valor_enc IN VARCHAR2) RETURN VARCHAR2;
    PROCEDURE guardar_dato_sensible(
        p_item_name IN VARCHAR2,
        p_valor     IN VARCHAR2
    );

END pkg_encriptacion;
/

CREATE OR REPLACE PACKAGE BODY pkg_encriptacion AS

    FUNCTION obtener_clave RETURN VARCHAR2 IS
        v_clave VARCHAR2(4000);
    BEGIN
        SELECT config_value INTO v_clave
        FROM app_config_segura
        WHERE config_key = 'ENCRYPTION_KEY';
        RETURN v_clave;
    END;

    FUNCTION encriptar(p_valor IN VARCHAR2) RETURN VARCHAR2 IS
    BEGIN
        RETURN APEX_UTIL.ENCRYPT(
            p_val => p_valor,
            p_key => obtener_clave
        );
    END encriptar;

    FUNCTION desencriptar(p_valor_enc IN VARCHAR2) RETURN VARCHAR2 IS
    BEGIN
        RETURN APEX_UTIL.DECRYPT(
            p_val => p_valor_enc,
            p_key => obtener_clave
        );
    END desencriptar;

    PROCEDURE guardar_dato_sensible(
        p_item_name IN VARCHAR2,
        p_valor     IN VARCHAR2
    ) IS
    BEGIN
        APEX_UTIL.SET_SESSION_STATE(
            p_name  => p_item_name,
            p_value => encriptar(p_valor)
        );
    END guardar_dato_sensible;

END pkg_encriptacion;
/
```

3. Prueba la encriptación desde SQL Workshop:

```sql
-- Probar encriptación/desencriptación
DECLARE
    v_original   VARCHAR2(100) := 'Dato Sensible: NúmeroTarjeta-4532';
    v_encriptado VARCHAR2(4000);
    v_decriptado VARCHAR2(100);
BEGIN
    v_encriptado := pkg_encriptacion.encriptar(v_original);
    v_decriptado := pkg_encriptacion.desencriptar(v_encriptado);

    DBMS_OUTPUT.PUT_LINE('Original:    ' || v_original);
    DBMS_OUTPUT.PUT_LINE('Encriptado:  ' || SUBSTR(v_encriptado, 1, 40) || '...');
    DBMS_OUTPUT.PUT_LINE('Decriptado:  ' || v_decriptado);
    DBMS_OUTPUT.PUT_LINE('Coincide:    ' || CASE WHEN v_original = v_decriptado THEN 'SÍ ✓' ELSE 'NO ✗' END);
END;
/
```

4. En la aplicación APEX, identifica cualquier ítem de página que almacene datos sensibles (por ejemplo, números de identificación, tokens, datos financieros). Para cada uno:
   - Abre el ítem en Page Designer
   - En **Security → Session State Protection**: selecciona `Restricted - May not be set from browser`
   - Si el valor debe persistir encriptado, llama a `pkg_encriptacion.guardar_dato_sensible` desde un proceso PL/SQL After Submit

**Resultado esperado:**
- Tabla `APP_CONFIG_SEGURA` creada con clave de encriptación
- Paquete `PKG_ENCRIPTACION` compilado sin errores
- Prueba de encriptación/desencriptación exitosa con coincidencia confirmada

**Verificación:**
- La salida del bloque PL/SQL debe mostrar `Coincide: SÍ ✓`
- El valor encriptado debe ser diferente al original y no legible directamente

---

### Paso 7: Ejecutar pruebas de seguridad básicas

**Objetivo:** Realizar pruebas de penetración básicas sobre la aplicación para identificar vulnerabilidades antes de usar el Security Advisor.

#### Instrucciones

**Prueba 1: Acceso no autorizado a páginas protegidas**

1. Cierra sesión de la aplicación completamente.
2. Intenta acceder directamente a una página protegida usando la URL:
   ```
   https://tu-instancia-apex.com/apex/f?p=APP_ID:10:0
   ```
   (donde `:0` indica sesión vacía)
3. **Resultado esperado:** APEX debe redirigir automáticamente a la página de login (p. 101).
4. Documenta el comportamiento observado.

**Prueba 2: Manipulación de parámetros en URL**

1. Inicia sesión como `operador1`.
2. Navega a un registro de detalle (por ejemplo, página 5 con `P5_ID=1`).
3. En la barra de URL del navegador, modifica manualmente el ID: cambia `P5_ID=1` a `P5_ID=99` (un ID que no le pertenece).
4. Observa si SSP previene el acceso o si la aplicación muestra datos de otro usuario.
5. Si SSP está correctamente configurado, debes ver un error de checksum inválido.

**Prueba 3: Inyección SQL en campos de búsqueda**

1. Inicia sesión como `consultor1`.
2. Navega a cualquier página con un campo de búsqueda o filtro.
3. Ingresa los siguientes valores de prueba en el campo de búsqueda:

   ```
   ' OR '1'='1
   ```
   ```
   '; DROP TABLE app_usuarios; --
   ```
   ```
   1 UNION SELECT username, password_hash, NULL FROM app_usuarios --
   ```

4. **Resultado esperado:** APEX debe escapar estos valores y tratarlos como texto literal, sin ejecutar SQL malicioso. La búsqueda simplemente no debe devolver resultados o debe devolver resultados normales.

5. Abre **Chrome DevTools** (F12) → pestaña **Network**. Observa las solicitudes y verifica que los parámetros enviados estén correctamente codificados.

**Prueba 4: Intentar bypass de Authorization Scheme via URL**

1. Inicia sesión como `consultor1`.
2. Intenta acceder directamente a la URL de la página de administración:
   ```
   f?p=APP_ID:ADMIN_PAGE_NUMBER:SESSION_ID
   ```
3. **Resultado esperado:** Debe aparecer el mensaje de error de autorización configurado: `"Acceso denegado. Se requiere rol de Administrador."`

**Documentación de resultados:**

Registra los resultados de cada prueba en esta tabla:

| Prueba | Vulnerabilidad testada | Resultado obtenido | Estado |
|--------|----------------------|-------------------|--------|
| 1 | Acceso sin sesión | Redirige a login | ✓ Protegido |
| 2 | Manipulación de URL | Error SSP checksum | ✓ Protegido |
| 3 | Inyección SQL | Texto escapado, sin ejecución | ✓ Protegido |
| 4 | Bypass de autorización | Mensaje de acceso denegado | ✓ Protegido |

**Resultado esperado:**
- Las 4 pruebas deben mostrar comportamiento protegido
- No debe ser posible acceder a datos de otros usuarios ni ejecutar SQL arbitrario

---

### Paso 8: Ejecutar y corregir el APEX Security Advisor

**Objetivo:** Usar la herramienta integrada de APEX para obtener un reporte completo de vulnerabilidades y corregir todos los ítems Critical y High.

#### Instrucciones

1. En el App Builder, con tu aplicación abierta, haz clic en el menú de la aplicación y selecciona **Utilities → Security Advisor** (o accede desde **Shared Components → Security → Security Advisor**).

2. En la pantalla del Security Advisor, haz clic en **Check All** para ejecutar el análisis completo.

3. Espera a que el análisis termine. Verás un reporte con las vulnerabilidades categorizadas por severidad:
   - 🔴 **Critical** — Debe corregirse inmediatamente
   - 🟠 **High** — Debe corregirse antes de producción
   - 🟡 **Medium** — Recomendado corregir
   - 🔵 **Low / Advisory** — Buenas prácticas

4. Para cada ítem **Critical** o **High**, haz clic en el enlace del ítem para ir directamente al componente afectado. Los problemas más comunes y sus correcciones son:

   **Problema: "Authentication scheme does not use HTTPS"**
   - Navega a las propiedades de la aplicación → **Security** → habilita `Require HTTPS`

   **Problema: "Page X is accessible without authentication"**
   - Abre la página → propiedades → **Security → Authentication**: cambia a `Page Requires Authentication`

   **Problema: "Session State Protection is disabled"**
   - Ya lo habilitaste en el Paso 5. Si aún aparece, verifica que SSP esté `Enabled` en Shared Components → Security → Session State Protection

   **Problema: "Item X has unrestricted session state protection"**
   - Abre el ítem en Page Designer → Security → Session State Protection → selecciona el nivel apropiado

   **Problema: "Browser Security: X-Frame-Options not set"**
   - Navega a **Shared Components → Security Attributes → Browser Security**:

   | Parámetro | Valor |
   |-----------|-------|
   | Embed in Frames | `Deny` |
   | HTML Escaping Mode | `Extended` |
   | Referrer Policy | `strict-origin-when-cross-origin` |

5. Después de corregir cada ítem, vuelve al Security Advisor y haz clic en **Check All** nuevamente para confirmar que el ítem ya no aparece como vulnerabilidad.

6. Continúa el ciclo hasta que no existan ítems **Critical** ni **High** en el reporte.

7. Toma una captura de pantalla del Security Advisor mostrando el reporte final limpio (solo Medium, Low o Advisory permitidos).

**Resultado esperado:**
- Reporte del Security Advisor sin ítems Critical ni High
- Todas las correcciones aplicadas y verificadas

**Verificación:**
- Ejecuta el Security Advisor una última vez y confirma que el conteo de Critical = 0 y High = 0
- La aplicación debe seguir funcionando correctamente después de todas las correcciones

---

## Validación y Pruebas Finales

Una vez completados todos los pasos, ejecuta la siguiente batería de validación integral:

### Validación 1: Matriz de acceso por rol

Inicia sesión con cada usuario y verifica el acceso correcto:

```
┌─────────────────┬──────────────┬────────────┬──────────┬───────────┐
│ Funcionalidad   │ admin_lab    │ supervisor1│ operador1│ consultor1│
├─────────────────┼──────────────┼────────────┼──────────┼───────────┤
│ Pág. Admin      │     ✓        │     ✗      │    ✗     │     ✗     │
│ Pág. Reportes   │     ✓        │     ✓      │    ✗     │     ✗     │
│ Botón Eliminar  │     ✓        │     ✗      │    ✗     │     ✗     │
│ Botón Crear     │     ✓        │     ✓      │    ✓     │     ✗     │
│ Dashboard       │     ✓        │     ✓      │    ✓     │     ✓     │
└─────────────────┴──────────────┴────────────┴──────────┴───────────┘
```

### Validación 2: Verificación de SSP activo

```sql
-- Debe mostrar solo ítems con protección configurada
SELECT page_id, item_name, item_protection_level
FROM apex_application_page_items
WHERE application_id = :APP_ID
  AND item_protection_level IN (
      'Checksum Required - Session Level',
      'Checksum Required - User Level',
      'Restricted - May not be set from browser'
  )
ORDER BY page_id;
```

### Validación 3: Confirmar hashing de contraseñas

```sql
-- Las contraseñas NO deben ser texto plano
SELECT username, LENGTH(password_hash) AS hash_length,
       CASE WHEN LENGTH(password_hash) = 32 THEN 'SHA-256 OK ✓'
            ELSE 'REVISAR ✗' END AS estado
FROM app_usuarios;
```

### Validación 4: Prueba de bloqueo de cuenta

```sql
-- Simular 5 intentos fallidos para 'operador1'
BEGIN
    FOR i IN 1..5 LOOP
        DECLARE v_r BOOLEAN;
        BEGIN
            v_r := pkg_seguridad.autenticar_usuario('operador1', 'wrong_password_' || i);
        END;
    END LOOP;
END;
/

-- Verificar que la cuenta está bloqueada
SELECT username, intentos_fallidos, bloqueado_hasta
FROM app_usuarios
WHERE username = 'operador1';
-- bloqueado_hasta debe ser SYSTIMESTAMP + 30 minutos
```

---

## Solución de Problemas

### Problema 1: Error "ORA-28113: policy predicate has error" al intentar autenticarse

**Síntomas:**
- La página de login muestra un error de base de datos al intentar iniciar sesión
- En el log de APEX aparece `ORA-28113` o errores relacionados con `DBMS_CRYPTO`
- La autenticación falla para todos los usuarios, incluso con credenciales correctas

**Causa raíz:**
El esquema de la aplicación no tiene el privilegio `EXECUTE` sobre `SYS.DBMS_CRYPTO`. Este paquete requiere concesión explícita, ya que no está disponible por defecto para todos los esquemas de usuario.

**Solución:**
1. Conéctate a la base de datos como DBA (o solicita al instructor que lo haga):
   ```sql
   -- Ejecutar como SYSDBA o DBA
   GRANT EXECUTE ON SYS.DBMS_CRYPTO TO <nombre_de_tu_esquema>;
   ```
2. Recompila el paquete `PKG_SEGURIDAD`:
   ```sql
   ALTER PACKAGE pkg_seguridad COMPILE;
   ALTER PACKAGE pkg_seguridad COMPILE BODY;
   ```
3. Verifica que el paquete compile sin errores:
   ```sql
   SELECT object_name, object_type, status
   FROM user_objects
   WHERE object_name = 'PKG_SEGURIDAD';
   -- STATUS debe ser 'VALID'
   ```
4. Intenta autenticarse nuevamente en la aplicación.

---

### Problema 2: El Authorization Scheme no deniega el acceso correctamente (usuario ve páginas que no debería)

**Síntomas:**
- Un usuario con rol `CONSULTOR` puede acceder a la página de Administración sin error
- Los botones que deberían estar ocultos siguen siendo visibles para roles no autorizados
- El mensaje de "Acceso denegado" nunca aparece

**Causa raíz:**
Hay dos causas comunes: (a) el Authorization Scheme fue creado pero **no fue asignado** al componente (página/región/botón) en el Page Designer, o (b) la opción de **Caching** del esquema está configurada como `Once Per Session` y la sesión actual fue autenticada antes de que se asignara el esquema, por lo que el resultado en caché es `TRUE`.

**Solución:**
1. Verifica la asignación del esquema en el componente afectado:
   - Abre la página en Page Designer
   - Haz clic en la página/región/botón en cuestión
   - En el panel de propiedades, sección **Security**, confirma que el campo `Authorization Scheme` muestra el nombre correcto del esquema
   - Si está vacío o dice `- No Authorization Required -`, selecciona el esquema correcto y guarda

2. Si el esquema está asignado pero no funciona por caché, fuerza una nueva evaluación:
   ```sql
   -- Limpiar caché de autorización para la sesión actual
   -- (ejecutar como proceso PL/SQL en la aplicación o desde SQL Workshop)
   APEX_AUTHORIZATION.RESET_CACHE;
   ```
   O simplemente cierra sesión completamente y vuelve a iniciar sesión para obtener una sesión nueva.

3. Verifica que la función `TIENE_ROL` retorna el valor correcto:
   ```sql
   -- Probar directamente la función
   SELECT CASE WHEN pkg_seguridad.tiene_rol('consultor1', 'ADMINISTRADOR')
               THEN 'TRUE' ELSE 'FALSE' END AS resultado
   FROM DUAL;
   -- Debe retornar: FALSE
   ```

---

## Limpieza del Entorno

> **Nota:** Ejecuta la limpieza **solo si el instructor lo indica** o si necesitas reiniciar el laboratorio desde cero. Los objetos creados en este laboratorio son necesarios para laboratorios posteriores si los hay.

```sql
-- Eliminar objetos del laboratorio (ejecutar en orden)
BEGIN
    -- Eliminar paquetes
    EXECUTE IMMEDIATE 'DROP PACKAGE pkg_encriptacion';
    EXECUTE IMMEDIATE 'DROP PACKAGE pkg_seguridad';

    -- Eliminar tablas (en orden por dependencias de FK)
    EXECUTE IMMEDIATE 'DROP TABLE app_usuario_roles CASCADE CONSTRAINTS';
    EXECUTE IMMEDIATE 'DROP TABLE app_roles CASCADE CONSTRAINTS';
    EXECUTE IMMEDIATE 'DROP TABLE app_usuarios CASCADE CONSTRAINTS';
    EXECUTE IMMEDIATE 'DROP TABLE app_config_segura CASCADE CONSTRAINTS';

    DBMS_OUTPUT.PUT_LINE('Limpieza completada exitosamente.');
EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error durante limpieza: ' || SQLERRM);
END;
/
```

Para restaurar el esquema de autenticación a APEX Accounts estándar:
1. Navega a **Shared Components → Authentication Schemes**
2. Haz clic en `APEX Accounts - Referencia`
3. Selecciona **Make Current Scheme**
4. Elimina el esquema `Custom Auth - DB Hash`

---

## Resumen

En este laboratorio implementaste una capa de seguridad completa sobre tu aplicación Oracle APEX. Los logros principales fueron:

| Componente implementado | Tecnología utilizada | Beneficio de seguridad |
|------------------------|---------------------|----------------------|
| Autenticación personalizada con hashing | `DBMS_CRYPTO.HASH_SH256` | Contraseñas nunca almacenadas en texto plano |
| Bloqueo por intentos fallidos | PL/SQL en `PKG_SEGURIDAD` | Mitigación de ataques de fuerza bruta |
| Modelo de 4 roles de negocio | `APP_USUARIO_ROLES` + Authorization Schemes | Control de acceso granular por función |
| Session State Protection | APEX SSP + Checksums | Prevención de manipulación de parámetros URL |
| Encriptación de datos sensibles | `APEX_UTIL.ENCRYPT/DECRYPT` | Protección de datos en estado de sesión |
| Auditoría de vulnerabilidades | APEX Security Advisor | Identificación y corrección sistemática de riesgos |

La seguridad en APEX es un sistema de defensa en profundidad: cada capa que implementaste —autenticación, autorización, SSP, encriptación— actúa como una barrera adicional. Ninguna capa por sí sola es suficiente, pero juntas crean una aplicación significativamente más resistente a los vectores de ataque más comunes.

### Conceptos clave reforzados

- **Autenticación ≠ Autorización:** Saber *quién* es el usuario (autenticación) es independiente de saber *qué puede hacer* (autorización)
- **Hashing es unidireccional:** `DBMS_CRYPTO.HASH` no puede revertirse; solo se puede comparar el hash de la contraseña ingresada con el hash almacenado
- **SSP protege la integridad de la URL:** Sin SSP, un usuario podría cambiar `P5_ID=1` a `P5_ID=999` en la URL y acceder a datos ajenos
- **El Security Advisor es tu aliado:** Úsalo regularmente durante el desarrollo, no solo al final

### Recursos adicionales

- [Oracle APEX Security Guide (23.2)](https://docs.oracle.com/en/database/oracle/apex/23.2/htmsg/)
- [DBMS_CRYPTO Reference](https://docs.oracle.com/en/database/oracle/oracle-database/21/arpls/DBMS_CRYPTO.html)
- [APEX_UTIL Package Reference](https://docs.oracle.com/en/database/oracle/apex/23.2/aeapi/APEX_UTIL.html)
- [OWASP Top 10 Web Application Security Risks](https://owasp.org/www-project-top-ten/)
- [Oracle APEX Authentication Schemes Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/establishing-user-identity-through-authentication.html)

---
