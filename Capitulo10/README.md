# Práctica 10: Despliegue, integración y checklist de producción

## Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 52 minutos                                 |
| **Complejidad**  | Difícil                                    |
| **Nivel Bloom**  | Crear (Create)                             |
| **Laboratorio**  | 10-00-01                                   |
| **Módulo**       | 10 — Despliegue, CI/CD e integración REST  |

---

## Descripción General

Este laboratorio final integra todos los conocimientos adquiridos en el curso en un proceso completo de despliegue profesional. Comenzarás exportando la aplicación APEX en sus diferentes modalidades (completa, componentes, esquema), configurarás un repositorio Git con la estructura recomendada para proyectos APEX y automatizarás el proceso de exportación con SQLcl. Luego implementarás un Web Source Module para consumir una API REST pública, crearás un servicio RESTful en ORDS que exponga datos de tu aplicación, y finalizarás ejecutando el checklist completo de preparación para producción: Security Advisor, revisión de rendimiento, accesibilidad con Lighthouse y generación de documentación técnica.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Exportar e importar una aplicación APEX completa usando el asistente nativo y SQLcl, distinguiendo los tipos de exportación disponibles.
- [ ] Configurar un repositorio Git con la estructura recomendada para proyectos APEX y automatizar la exportación periódica mediante un script shell.
- [ ] Crear un Web Source Module en APEX para consumir una API REST externa y mostrar los datos en la aplicación.
- [ ] Definir y probar un servicio RESTful en ORDS que exponga datos de Oracle como API con autenticación OAuth2 básica.
- [ ] Ejecutar el checklist completo de preparación para producción: Security Advisor, rendimiento, accesibilidad y documentación técnica.

---

## Prerrequisitos

### Conocimientos Requeridos

- Haber completado los laboratorios 01-00-01 al 09-00-01 (la aplicación base debe estar funcional).
- Conocimientos básicos de Git: `commit`, `push`, `pull`, `branches`.
- Familiaridad con conceptos REST: métodos HTTP (GET, POST), JSON, autenticación OAuth2.
- Comprensión del ciclo de vida de aplicaciones APEX (desarrollo → QA → producción).

### Acceso y Herramientas Requeridas

- Acceso al workspace APEX con la aplicación construida en laboratorios anteriores.
- SQLcl instalado y disponible en la terminal (`sql` o `sqlcl` en el PATH).
- Git 2.40+ instalado y configurado con usuario/correo.
- Postman 10.0+ instalado para pruebas de API.
- Cuenta en GitHub o GitLab (gratuita) para el repositorio remoto.
- Navegador Google Chrome 110+ con extensión Lighthouse disponible (incluida en DevTools).
- Conexión a Internet para consumir la API pública de tipos de cambio.

---

## Entorno de Laboratorio

### Hardware Mínimo Recomendado

| Componente     | Mínimo              | Recomendado          |
|----------------|---------------------|----------------------|
| RAM            | 8 GB                | 16 GB                |
| Almacenamiento | 50 GB libres (SSD)  | 100 GB SSD           |
| Procesador     | Intel i5 8ª gen     | Intel i7 / Ryzen 7   |
| Pantalla       | 1280×768            | 1920×1080            |

### Software del Entorno

| Componente          | Versión Mínima | Notas                                         |
|---------------------|----------------|-----------------------------------------------|
| Oracle APEX         | 23.2           | Workspace con aplicación de laboratorios previos |
| Oracle Database     | 19c / 21c XE   | Esquema con datos de muestra (≥1000 registros) |
| Oracle ORDS         | 23.2           | Activo y accesible en el entorno              |
| SQLcl               | 23.1+          | Instalado en la máquina local o en el servidor |
| Git                 | 2.40+          | Configurado con usuario y correo              |
| Postman             | 10.0+          | Para pruebas de endpoints REST                |
| Google Chrome       | 110+           | Con DevTools/Lighthouse habilitado            |
| Visual Studio Code  | 1.80+          | Editor para scripts shell y SQL               |

### Verificación del Entorno Antes de Comenzar

Ejecuta los siguientes comandos en tu terminal para confirmar que el entorno está listo:

```bash
# Verificar SQLcl
sql -v
# Salida esperada: SQLcl: Release 23.x.x ...

# Verificar Git
git --version
# Salida esperada: git version 2.40.x

# Verificar que la aplicación APEX es accesible
# Abre en el navegador:
# https://<tu-instancia>/ords/<workspace>/r/<app_id>/home
```

> **⚠️ Nota importante:** Si tu entorno es `apex.oracle.com`, algunas funcionalidades de ORDS RESTful Services pueden estar restringidas. Consulta con tu instructor para obtener acceso a un entorno alternativo con privilegios completos para los pasos 4 y 5 de este laboratorio.

---

## Pasos del Laboratorio

---

### Paso 1 — Exportación Completa de la Aplicación APEX (8 min)

**Objetivo:** Exportar la aplicación APEX en sus diferentes modalidades usando el asistente nativo del App Builder y comprender qué incluye cada tipo de exportación.

#### Instrucciones

1. Inicia sesión en tu workspace APEX y navega a **App Builder**.

2. Localiza tu aplicación de práctica (la construida en los laboratorios 04–09) y haz clic sobre ella para abrirla.

3. En el menú superior derecho de la aplicación, haz clic en **Export / Import** y selecciona **Export**.

4. En la pantalla de exportación, configura las siguientes opciones:

   | Opción                          | Valor                        |
   |---------------------------------|------------------------------|
   | Export Application              | ✅ Habilitado                |
   | Export Supporting Object Scripts| ✅ Habilitado                |
   | Export Public Reports           | ✅ Habilitado                |
   | Export Private Reports          | ❌ Deshabilitado             |
   | Export Interactive Report Subscriptions | ❌ Deshabilitado   |
   | Export Team Development        | ❌ Deshabilitado             |
   | As of                           | Current                      |
   | Encoding                        | Unicode UTF-8                |

5. Haz clic en **Export** y guarda el archivo `.sql` resultante en la carpeta `~/apex-lab-exports/` (créala si no existe):

   ```bash
   mkdir -p ~/apex-lab-exports
   # Mueve el archivo descargado a esta carpeta
   mv ~/Downloads/f<app_id>.sql ~/apex-lab-exports/app_completa_$(date +%Y%m%d).sql
   ```

6. Ahora exporta **solo una página** individual. Regresa al App Builder, abre tu aplicación y navega a una página de reporte (por ejemplo, la página 2 o la que contenga el Interactive Grid principal). Haz clic en el botón **Page** en la barra de herramientas del Page Designer y selecciona **Export Page**.

7. Guarda este archivo con un nombre descriptivo:

   ```bash
   mv ~/Downloads/f<app_id>_page*.sql ~/apex-lab-exports/pagina_reporte_$(date +%Y%m%d).sql
   ```

8. Finalmente, exporta los **Componentes Compartidos**. Desde el App Builder, ve a **Shared Components** de tu aplicación. En la sección superior, busca el enlace **Export** (o ve a **App Builder → Export → Application Components**). Selecciona exportar:
   - LOVs (List of Values)
   - Templates (plantillas de página y región)
   - Haz clic en **Export Shared Components**.

   ```bash
   mv ~/Downloads/f<app_id>_components*.sql ~/apex-lab-exports/shared_components_$(date +%Y%m%d).sql
   ```

#### Salida Esperada

Debes tener tres archivos `.sql` en `~/apex-lab-exports/`:
- `app_completa_YYYYMMDD.sql` (~500 KB o más dependiendo de la complejidad)
- `pagina_reporte_YYYYMMDD.sql` (~50–150 KB)
- `shared_components_YYYYMMDD.sql` (~100–300 KB)

#### Verificación

```bash
ls -lh ~/apex-lab-exports/
# Debes ver los tres archivos con tamaños no nulos

# Verifica que el archivo de exportación completa contiene los metadatos de APEX
head -20 ~/apex-lab-exports/app_completa_*.sql
# Debes ver líneas similares a:
# -- Oracle APEX export file
# -- Version: 23.2.x
# -- Application: <ID> - <Nombre de tu aplicación>
```

---

### Paso 2 — Exportación con SQLcl y Configuración del Repositorio Git (12 min)

**Objetivo:** Usar SQLcl para exportar la aplicación desde la línea de comandos, configurar un repositorio Git con la estructura recomendada para proyectos APEX y crear un script de exportación automatizable.

#### Instrucciones

**Parte A: Exportar con SQLcl**

1. Abre una terminal y conéctate a la base de datos usando SQLcl:

   ```bash
   sql <usuario>/<password>@<host>:<puerto>/<servicio>
   # Ejemplo para base de datos local:
   sql apex_user/MiPassword123@localhost:1521/XEPDB1
   ```

2. Una vez conectado, usa el comando `apex export` de SQLcl para exportar la aplicación. Primero, identifica el ID de tu aplicación (visible en la URL del App Builder):

   ```sql
   -- Dentro de SQLcl, verifica el ID de tu aplicación
   SELECT application_id, application_name
   FROM   apex_applications
   WHERE  workspace = 'TU_WORKSPACE'
   ORDER  BY application_id;
   ```

3. Exporta la aplicación completa con SQLcl:

   ```bash
   # Desde la terminal (fuera de SQLcl), ejecuta:
   sql <usuario>/<password>@<host>:<puerto>/<servicio> <<EOF
   apex export -applicationid <APP_ID> -dir ~/apex-lab-exports/sqlcl/ -expSupportingObjects Y -expType APPLICATION_SOURCE
   exit
   EOF
   ```

   > **Nota:** El flag `-expType APPLICATION_SOURCE` exporta la aplicación en formato de archivos separados por componente (formato "split"), ideal para control de versiones con Git ya que cada página y componente queda en un archivo independiente.

4. Verifica la estructura generada:

   ```bash
   ls -la ~/apex-lab-exports/sqlcl/
   # Debes ver una carpeta con el nombre de la aplicación
   # y subcarpetas como: application/, pages/, shared_components/
   
   find ~/apex-lab-exports/sqlcl/ -name "*.sql" | head -20
   # Lista los primeros 20 archivos SQL generados
   ```

**Parte B: Configurar el Repositorio Git**

5. Crea la estructura de directorios recomendada para un proyecto APEX en Git:

   ```bash
   mkdir -p ~/apex-project/{apex,database/{ddl,dml,packages},scripts,docs}
   cd ~/apex-project
   
   # Estructura resultante:
   # apex-project/
   # ├── apex/          → Exportaciones de la aplicación APEX
   # ├── database/
   # │   ├── ddl/       → Scripts CREATE TABLE, CREATE INDEX, etc.
   # │   ├── dml/       → Scripts INSERT de datos de referencia
   # │   └── packages/  → Código PL/SQL (packages, procedures, functions)
   # ├── scripts/       → Scripts de automatización (shell, Python)
   # └── docs/          → Documentación técnica
   ```

6. Inicializa el repositorio Git y crea el `.gitignore`:

   ```bash
   cd ~/apex-project
   git init
   
   cat > .gitignore << 'EOF'
   # Archivos temporales
   *.tmp
   *.log
   *.bak
   
   # Archivos con credenciales (NUNCA versionar contraseñas)
   *credentials*
   *password*
   .env
   
   # Archivos de sistema
   .DS_Store
   Thumbs.db
   EOF
   ```

7. Copia las exportaciones de SQLcl al directorio `apex/`:

   ```bash
   cp -r ~/apex-lab-exports/sqlcl/* ~/apex-project/apex/
   ```

8. Crea el script de exportación automatizada `scripts/export_apex.sh`:

   ```bash
   cat > ~/apex-project/scripts/export_apex.sh << 'EOF'
   #!/bin/bash
   # ============================================================
   # Script: export_apex.sh
   # Descripción: Exporta la aplicación APEX usando SQLcl
   #              y hace commit automático al repositorio Git.
   # Uso: ./scripts/export_apex.sh
   # ============================================================
   
   # --- CONFIGURACIÓN (ajusta estos valores a tu entorno) ---
   DB_USER="apex_user"
   DB_HOST="localhost"
   DB_PORT="1521"
   DB_SERVICE="XEPDB1"
   APP_ID="100"
   EXPORT_DIR="$(dirname "$0")/../apex"
   
   # Leer contraseña de variable de entorno (NO hardcodear en el script)
   if [ -z "$DB_PASSWORD" ]; then
     echo "ERROR: Define la variable de entorno DB_PASSWORD antes de ejecutar."
     exit 1
   fi
   
   echo "=== Iniciando exportación APEX - $(date) ==="
   
   # Exportar aplicación con SQLcl
   sql "${DB_USER}/${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_SERVICE}" <<SQLEOF
   apex export -applicationid ${APP_ID} -dir ${EXPORT_DIR} -expSupportingObjects Y -expType APPLICATION_SOURCE
   exit
   SQLEOF
   
   if [ $? -eq 0 ]; then
     echo "✅ Exportación completada exitosamente."
     
     # Hacer commit automático si hay cambios
     cd "$(dirname "$0")/.."
     git add apex/
     git diff --cached --quiet || git commit -m "chore: auto-export APEX app $(date '+%Y-%m-%d %H:%M')"
     echo "✅ Commit realizado en Git."
   else
     echo "❌ ERROR: La exportación falló. Revisa la conexión a la base de datos."
     exit 1
   fi
   
   echo "=== Exportación finalizada - $(date) ==="
   EOF
   
   chmod +x ~/apex-project/scripts/export_apex.sh
   ```

9. Realiza el primer commit del proyecto:

   ```bash
   cd ~/apex-project
   git add .
   git commit -m "feat: initial APEX project structure and first export"
   ```

10. Conecta el repositorio local a GitHub/GitLab (crea un repositorio vacío en tu cuenta primero):

    ```bash
    git remote add origin https://github.com/<tu-usuario>/apex-lab-proyecto.git
    git branch -M main
    git push -u origin main
    ```

#### Salida Esperada

- Repositorio Git inicializado con estructura de directorios correcta.
- Archivos de exportación APEX en formato split dentro de `apex/`.
- Script `export_apex.sh` funcional y ejecutable.
- Primer commit visible en el historial de Git.
- Repositorio remoto en GitHub/GitLab con los archivos subidos.

#### Verificación

```bash
cd ~/apex-project

# Verificar estructura del repositorio
find . -not -path './.git/*' -type f | sort
# Debes ver los archivos en apex/, scripts/, etc.

# Verificar historial de Git
git log --oneline
# Debes ver al menos: feat: initial APEX project structure and first export

# Verificar el script es ejecutable
ls -la scripts/export_apex.sh
# Debes ver: -rwxr-xr-x ...

# Probar la exportación automatizada
export DB_PASSWORD="MiPassword123"
./scripts/export_apex.sh
# Debes ver: ✅ Exportación completada exitosamente.
```

---

### Paso 3 — Consumir una API REST Externa con Web Source Module (10 min)

**Objetivo:** Crear un Web Source Module en APEX para consumir la API pública de tipos de cambio (ExchangeRate-API) y mostrar los datos en una página de la aplicación.

#### Instrucciones

> **API utilizada:** `https://open.er-api.com/v6/latest/USD` — API pública gratuita de tipos de cambio. No requiere clave API para uso básico.

**Parte A: Crear el Web Source Module**

1. En el App Builder, abre tu aplicación. Ve a **Shared Components** y busca la sección **Data Sources**. Haz clic en **Web Source Modules**.

2. Haz clic en **Create** y selecciona **From Scratch**.

3. Completa el asistente con los siguientes valores:

   **Paso 1 — General:**
   | Campo               | Valor                              |
   |---------------------|------------------------------------|
   | Name                | `API_TIPOS_CAMBIO`                 |
   | Type                | Simple HTTP                        |
   | Base URL            | `https://open.er-api.com`          |
   | Service URL Path    | `/v6/latest/USD`                   |

   **Paso 2 — Authentication:**
   | Campo               | Valor                              |
   |---------------------|------------------------------------|
   | Authentication Type | No Authentication                  |

4. Haz clic en **Discover** para que APEX consulte la API y detecte automáticamente la estructura del JSON. Espera a que aparezcan las columnas detectadas.

5. APEX mostrará las columnas inferidas del JSON. Verifica que estén presentes al menos:
   - `RESULT` (tipo VARCHAR2)
   - `BASE_CODE` (tipo VARCHAR2)
   - `TIME_LAST_UPDATE_UTC` (tipo VARCHAR2)
   - Columnas de tasas de cambio (pueden aparecer como un objeto anidado `RATES`)

6. Haz clic en **Create Web Source** para guardar.

**Parte B: Crear una Página que Muestre los Tipos de Cambio**

7. Crea una nueva página en la aplicación: **Create Page → Blank Page**.
   - Page Number: `50`
   - Name: `Tipos de Cambio`
   - Page Mode: Normal

8. En el Page Designer de la página 50, agrega una región de tipo **Classic Report**:
   - Haz clic derecho en el árbol de componentes → **Add Region**.
   - Title: `Tipos de Cambio (Fuente: ExchangeRate-API)`
   - Type: **Classic Report**
   - Source Type: **Web Source**
   - Web Source Module: `API_TIPOS_CAMBIO`

9. En las propiedades de la región, ve a la sección **Columns** y configura las columnas que deseas mostrar. Si la API devuelve un objeto `rates` con pares clave-valor, puede ser necesario procesar el JSON con `APEX_JSON`. Agrega el siguiente código en el **Source → SQL Query** de la región si el descubrimiento automático no generó columnas de tasas individuales:

   ```sql
   SELECT key   AS moneda,
          value AS tasa_usd
   FROM   JSON_TABLE(
            apex_web_service.make_rest_request(
              p_url         => 'https://open.er-api.com/v6/latest/USD',
              p_http_method => 'GET'
            ),
            '$.rates.*' COLUMNS (
              key   VARCHAR2(10)  PATH '$'  -- nombre de la moneda
            )
          )
   -- Nota: Si el entorno lo soporta, usar directamente el Web Source Module
   -- Esta alternativa con APEX_WEB_SERVICE sirve como fallback
   ```

   > **Alternativa recomendada:** Si el Web Source Module generó columnas correctamente en el paso 5, usa directamente el módulo como fuente y omite el SQL manual.

10. Guarda y ejecuta la página. Debes ver una tabla con monedas y sus tasas de cambio respecto al USD.

11. Agrega un elemento de texto estático en la parte superior de la página para indicar la fuente y la fecha de actualización:
    - Tipo: **Static Content**
    - HTML: `<p class="u-color-5-text"><strong>Fuente:</strong> ExchangeRate-API (open.er-api.com) | <strong>Base:</strong> USD | Los datos se actualizan cada 24 horas.</p>`

#### Salida Esperada

- Página 50 visible en la aplicación mostrando una tabla con al menos 10 monedas y sus tasas de cambio frente al USD.
- El Web Source Module `API_TIPOS_CAMBIO` visible en Shared Components → Web Source Modules.
- Los datos se cargan dinámicamente desde la API externa en cada carga de página.

#### Verificación

```bash
# Desde Postman, verifica que la API externa responde correctamente:
# GET https://open.er-api.com/v6/latest/USD
# Respuesta esperada: JSON con campo "result": "success" y objeto "rates"
```

En APEX:
- Navega a la página 50 de tu aplicación en ejecución.
- Confirma que la tabla muestra datos de monedas (EUR, GBP, MXN, JPY, etc.).
- Abre las **Developer Tools** del navegador → **Network** → recarga la página → verifica que hay una llamada a `open.er-api.com` con respuesta 200.

---

### Paso 4 — Crear un Servicio RESTful en ORDS (10 min)

**Objetivo:** Definir un endpoint REST en ORDS que exponga datos de la aplicación (tabla de productos o pedidos) como una API JSON, con autenticación OAuth2 básica.

#### Instrucciones

**Parte A: Crear el Módulo REST en ORDS**

1. Desde el workspace APEX, ve a **SQL Workshop → RESTful Services**.

2. Haz clic en **Create Module** y completa:

   | Campo                  | Valor                          |
   |------------------------|--------------------------------|
   | Module Name            | `api.ventas`                   |
   | Base Path              | `/api/ventas/`                 |
   | Is Published           | Yes                            |
   | Pagination Size        | 25                             |

3. Haz clic en **Add Template** para agregar una plantilla de recurso:

   | Campo                  | Valor                          |
   |------------------------|--------------------------------|
   | URI Template           | `productos`                    |

4. Dentro de la plantilla `productos`, haz clic en **Add Handler** y configura:

   | Campo                  | Valor                          |
   |------------------------|--------------------------------|
   | Method                 | GET                            |
   | Source Type            | Collection Query               |
   | Source (SQL Query)     | Ver abajo                      |

   ```sql
   SELECT p.producto_id,
          p.nombre,
          p.precio,
          p.stock,
          TO_CHAR(p.fecha_alta, 'YYYY-MM-DD') AS fecha_alta
   FROM   ventas_productos p
   WHERE  p.stock > 0
   ORDER  BY p.nombre
   ```

5. Guarda el handler. Ahora agrega un segundo template para obtener un producto por ID:

   | Campo                  | Valor                          |
   |------------------------|--------------------------------|
   | URI Template           | `productos/:id`                |

6. Agrega un handler GET a este template:

   ```sql
   SELECT p.producto_id,
          p.nombre,
          p.precio,
          p.stock,
          TO_CHAR(p.fecha_alta, 'YYYY-MM-DD') AS fecha_alta
   FROM   ventas_productos p
   WHERE  p.producto_id = :id
   ```

**Parte B: Configurar Autenticación OAuth2**

7. En la sección **RESTful Services**, ve a **OAuth Clients** y haz clic en **Register Client**:

   | Campo                  | Valor                                    |
   |------------------------|------------------------------------------|
   | Name                   | `cliente_lab10`                          |
   | Description            | Cliente de prueba para Lab 10            |
   | Allowed Origins        | `http://localhost` (o tu dominio)        |
   | Support Email          | `lab@ejemplo.com`                        |
   | Roles                  | (dejar vacío para acceso básico)         |
   | Privilege              | (dejar vacío por ahora)                  |

8. Después de registrar, APEX/ORDS mostrará el **Client ID** y el **Client Secret**. **Anótalos en un lugar seguro** — los necesitarás en el siguiente paso.

9. Protege el módulo REST: regresa al módulo `api.ventas`, edítalo y en **Security** asigna un **Privilege** o activa la autenticación (dependiendo de la versión de ORDS, el proceso puede variar; consulta con tu instructor si la opción no es visible en `apex.oracle.com`).

**Parte C: Probar el Endpoint con Postman**

10. Abre Postman y crea una nueva colección llamada `Lab 10 - APEX REST`.

11. Crea una petición para obtener el token OAuth2:
    - Method: **POST**
    - URL: `https://<tu-instancia>/ords/<workspace>/oauth/token`
    - Body → x-www-form-urlencoded:
      | Key             | Value                    |
      |-----------------|--------------------------|
      | `grant_type`    | `client_credentials`     |
      | `client_id`     | `<tu Client ID>`         |
      | `client_secret` | `<tu Client Secret>`     |

    Envía la petición. Debes recibir un JSON con `access_token`.

12. Crea una segunda petición para consumir el endpoint de productos:
    - Method: **GET**
    - URL: `https://<tu-instancia>/ords/<workspace>/api/ventas/productos`
    - Headers:
      | Key             | Value                              |
      |-----------------|------------------------------------|
      | `Authorization` | `Bearer <access_token del paso 11>`|

    Envía la petición. Debes recibir un JSON con el array de productos.

#### Salida Esperada

- Módulo REST `api.ventas` publicado y accesible.
- Endpoint `GET /api/ventas/productos` devuelve JSON con array de productos.
- Endpoint `GET /api/ventas/productos/:id` devuelve un producto específico.
- Token OAuth2 generado correctamente desde Postman.
- Respuesta 200 con datos JSON en la petición autenticada.

#### Verificación

```bash
# Desde la terminal, prueba el endpoint sin autenticación (debe dar 401):
curl -i "https://<instancia>/ords/<workspace>/api/ventas/productos"
# Esperado: HTTP/1.1 401 Unauthorized

# Obtén el token y prueba con autenticación:
TOKEN=$(curl -s -X POST \
  "https://<instancia>/ords/<workspace>/oauth/token" \
  -d "grant_type=client_credentials&client_id=<ID>&client_secret=<SECRET>" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -H "Authorization: Bearer $TOKEN" \
  "https://<instancia>/ords/<workspace>/api/ventas/productos"
# Esperado: JSON con campo "items" conteniendo array de productos
```

---

### Paso 5 — Checklist de Producción: Security Advisor y Rendimiento (7 min)

**Objetivo:** Ejecutar el Security Advisor de APEX, revisar el rendimiento con Debug Mode y configurar los parámetros de producción de la aplicación.

#### Instrucciones

**Parte A: Security Advisor**

1. Desde el App Builder, abre tu aplicación. En el menú superior, haz clic en **Utilities** y selecciona **Security Advisor**.

2. Haz clic en **Check All** para ejecutar todas las verificaciones de seguridad disponibles.

3. Revisa los resultados. El Security Advisor clasifica los hallazgos en tres niveles:
   - 🔴 **High Risk** — Debe corregirse antes de producción.
   - 🟡 **Medium Risk** — Recomendado corregir.
   - 🟢 **Low Risk / Informational** — Buenas prácticas.

4. Para cada hallazgo de **High Risk**, haz clic en el enlace para navegar directamente al componente problemático y aplica la corrección sugerida. Los problemas más comunes son:
   - Páginas sin autenticación requerida (cambiar Authentication a "Page Requires Authentication").
   - Elementos con Source Type `Database Column` sin escape HTML (activar "Escape Special Characters").
   - Ausencia de Content Security Policy en la aplicación.

5. Documenta en el archivo `docs/security_advisor_report.md` los hallazgos encontrados y las correcciones aplicadas:

   ```bash
   cat > ~/apex-project/docs/security_advisor_report.md << 'EOF'
   # Reporte Security Advisor - $(date +%Y-%m-%d)
   
   ## Resumen
   - Verificaciones ejecutadas: [N]
   - High Risk encontrados: [N] → Corregidos: [N]
   - Medium Risk encontrados: [N] → Corregidos: [N]
   
   ## Hallazgos y Correcciones
   
   | Severidad | Componente | Descripción | Acción Tomada |
   |-----------|------------|-------------|---------------|
   | High      | Página X   | ...         | ...           |
   
   EOF
   ```

**Parte B: Revisión de Rendimiento con Debug Mode**

6. Ejecuta tu aplicación en el navegador y navega a la página de reporte principal (Interactive Grid o Interactive Report con más datos).

7. Activa el Debug Mode agregando `&p_debug=YES` al final de la URL:
   ```
   https://<instancia>/ords/<workspace>/r/<app_id>/<page_alias>?p_debug=YES
   ```

8. Recarga la página. Regresa al App Builder y ve a **Workspace Utilities → Debug Messages**.

9. Encuentra la sesión de debug más reciente y haz clic en ella. Identifica:
   - El componente o query con mayor tiempo de ejecución (columna `Elapsed`).
   - Si algún query supera los **500ms**, documenta el problema.

10. Si encuentras un query lento, ve a **SQL Workshop → SQL Commands** y analiza su plan de ejecución:

    ```sql
    -- Reemplaza con el query identificado en Debug Messages
    EXPLAIN PLAN FOR
    SELECT *
    FROM   ventas_pedidos vp
    JOIN   ventas_productos pr ON vp.producto_id = pr.producto_id
    WHERE  vp.fecha_pedido >= ADD_MONTHS(SYSDATE, -3);
    
    SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
    ```

**Parte C: Configurar Parámetros de Producción**

11. En el App Builder, ve a **Edit Application Properties** (ícono de llave inglesa junto al nombre de la aplicación).

12. Configura los siguientes parámetros para producción:

    | Sección              | Parámetro                        | Valor Producción        |
    |----------------------|----------------------------------|-------------------------|
    | Security             | Browser Cache                    | Disabled                |
    | Security             | Embed in Frames                  | Deny                    |
    | Globalization        | Application Date Format          | DD/MM/YYYY              |
    | Error Handling       | Error Handling Function          | `app_error_handler`     |
    | Substitution Strings | APP_ENV                          | `PRODUCTION`            |

13. Desactiva el Debug Mode para usuarios finales: en **Security → Runtime API Usage**, asegúrate de que **Allow Runtime Developer Toolbar** esté en `Only For Developers`.

14. Guarda los cambios con **Apply Changes**.

#### Salida Esperada

- Security Advisor ejecutado sin hallazgos de High Risk pendientes.
- Archivo `docs/security_advisor_report.md` creado con los hallazgos documentados.
- Debug Messages revisados e identificado el componente más lento.
- Propiedades de la aplicación configuradas para producción.

#### Verificación

- Ejecuta nuevamente el Security Advisor: debe mostrar 0 hallazgos de High Risk.
- Accede a la aplicación como usuario final (sin sesión de desarrollador): el Developer Toolbar NO debe ser visible.
- Verifica en Application Properties que `Browser Cache = Disabled` y `Embed in Frames = Deny`.

---

### Paso 6 — Auditoría de Accesibilidad con Lighthouse y Documentación Final (5 min)

**Objetivo:** Ejecutar una auditoría de accesibilidad con Google Lighthouse sobre la aplicación en ejecución y generar la documentación técnica final del proyecto.

#### Instrucciones

**Parte A: Auditoría con Lighthouse**

1. Abre tu aplicación en Google Chrome y navega a la página principal (dashboard o página de inicio).

2. Abre las **DevTools** con `F12` o `Ctrl+Shift+I`.

3. Ve a la pestaña **Lighthouse** (si no es visible, haz clic en `>>` para mostrar más pestañas).

4. Configura la auditoría:
   - Mode: **Navigation**
   - Device: **Desktop**
   - Categories: selecciona **Performance**, **Accessibility**, **Best Practices** y **SEO**.

5. Haz clic en **Analyze page load** y espera a que termine el análisis (30–60 segundos).

6. Revisa los resultados. Para una aplicación APEX en producción, los objetivos mínimos son:

   | Categoría       | Puntuación Mínima Aceptable |
   |-----------------|-----------------------------|
   | Performance     | ≥ 70                        |
   | Accessibility   | ≥ 85                        |
   | Best Practices  | ≥ 80                        |
   | SEO             | ≥ 70                        |

7. Para cada problema de **Accessibility** con impacto **Critical** o **Serious**, documenta la corrección necesaria. Los problemas más comunes en APEX son:
   - Elementos de imagen sin atributo `alt`.
   - Contraste de color insuficiente en textos.
   - Formularios sin etiquetas `<label>` asociadas.

8. Exporta el reporte de Lighthouse: haz clic en el ícono de descarga (⬇) en la esquina superior derecha del reporte y guarda como `lighthouse_report.html`:

   ```bash
   mv ~/Downloads/localhost*.html ~/apex-project/docs/lighthouse_report_$(date +%Y%m%d).html
   ```

**Parte B: Documentación Técnica del Proyecto**

9. Crea el archivo de documentación técnica principal del proyecto:

   ```bash
   cat > ~/apex-project/docs/README.md << 'EOF'
   # Documentación Técnica — Aplicación APEX de Ventas
   
   ## Información General
   - **Nombre de la Aplicación:** [Nombre de tu aplicación]
   - **ID de Aplicación APEX:** [APP_ID]
   - **Workspace:** [Nombre del workspace]
   - **Versión APEX:** 23.2
   - **Fecha de última actualización:** $(date +%Y-%m-%d)
   - **Desarrollado por:** [Tu nombre]
   
   ## Arquitectura
   - **Base de Datos:** Oracle Database 21c XE
   - **Esquema:** [nombre del esquema]
   - **Tablas principales:** ventas_productos, ventas_pedidos
   - **Paquetes PL/SQL:** [lista de paquetes]
   
   ## Páginas de la Aplicación
   
   | Página | Nombre                | Tipo               | Descripción                        |
   |--------|-----------------------|--------------------|------------------------------------|
   | 1      | Inicio / Dashboard    | Dashboard          | KPIs y gráficas principales        |
   | 2      | Productos             | Interactive Grid   | CRUD de productos                  |
   | 3      | Pedidos               | Interactive Report | Reporte de pedidos con filtros      |
   | 50     | Tipos de Cambio       | Classic Report     | Integración con API externa        |
   | 101    | Login                 | Login              | Autenticación de usuarios          |
   
   ## APIs y Servicios REST
   
   ### Servicios Consumidos (Web Sources)
   | Módulo              | URL Base                    | Descripción                  |
   |---------------------|-----------------------------|------------------------------|
   | API_TIPOS_CAMBIO    | https://open.er-api.com     | Tipos de cambio USD          |
   
   ### Servicios Expuestos (ORDS)
   | Endpoint                          | Método | Autenticación | Descripción             |
   |-----------------------------------|--------|---------------|-------------------------|
   | /api/ventas/productos             | GET    | OAuth2        | Lista de productos       |
   | /api/ventas/productos/:id         | GET    | OAuth2        | Producto por ID          |
   
   ## Seguridad
   - Autenticación: APEX Application Authentication (custom)
   - Autorización: Roles definidos en Authorization Schemes
   - Security Advisor: Ejecutado el $(date +%Y-%m-%d) — 0 hallazgos High Risk
   
   ## Despliegue
   - Exportación: Ver `scripts/export_apex.sh`
   - Repositorio: [URL del repositorio Git]
   - Proceso de despliegue: Exportar con SQLcl → Push a Git → Importar en destino
   
   ## Checklist de Producción
   - [x] Security Advisor ejecutado — 0 High Risk
   - [x] Debug Mode desactivado para usuarios finales
   - [x] Browser Cache deshabilitado
   - [x] Error Handling configurado
   - [x] Lighthouse Accessibility ≥ 85
   - [x] Endpoints REST probados con Postman
   - [x] Exportación automatizada configurada en Git
   EOF
   ```

10. Realiza el commit final del proyecto:

    ```bash
    cd ~/apex-project
    git add .
    git commit -m "docs: add production checklist, security report and Lighthouse audit"
    git push origin main
    ```

#### Salida Esperada

- Reporte Lighthouse exportado en `docs/`.
- Puntuación de Accessibility ≥ 85 (o lista de correcciones identificadas si está por debajo).
- Archivo `docs/README.md` con documentación técnica completa.
- Commit final en el repositorio Git con todos los artefactos del laboratorio.

#### Verificación

```bash
cd ~/apex-project

# Verificar que todos los documentos están en el repositorio
ls -la docs/
# Debe mostrar: README.md, security_advisor_report.md, lighthouse_report_*.html

# Verificar el historial de Git completo del laboratorio
git log --oneline
# Debe mostrar al menos 3 commits del laboratorio de hoy

# Verificar sincronización con el remoto
git status
# Debe mostrar: nothing to commit, working tree clean
```

---

## Validación y Pruebas Finales

Ejecuta las siguientes verificaciones para confirmar que todos los objetivos del laboratorio se cumplieron satisfactoriamente:

### Lista de Verificación Final

| # | Verificación | Comando / Acción | Resultado Esperado |
|---|--------------|------------------|--------------------|
| 1 | Exportación completa existe | `ls -lh ~/apex-lab-exports/app_completa_*.sql` | Archivo ≥ 100 KB |
| 2 | Repositorio Git configurado | `git -C ~/apex-project log --oneline` | ≥ 3 commits visibles |
| 3 | Script de exportación funciona | `export DB_PASSWORD=xxx && ~/apex-project/scripts/export_apex.sh` | `✅ Exportación completada` |
| 4 | Web Source Module activo | Navegar a página 50 de la app | Tabla de monedas visible con datos |
| 5 | Endpoint REST responde | `curl -H "Authorization: Bearer $TOKEN" .../api/ventas/productos` | JSON con array de productos |
| 6 | Security Advisor limpio | App Builder → Utilities → Security Advisor | 0 High Risk |
| 7 | Lighthouse Accessibility | Chrome DevTools → Lighthouse | Score ≥ 85 |
| 8 | Documentación completa | `ls ~/apex-project/docs/` | 3 archivos presentes |

### Prueba de Importación (Opcional — si el tiempo lo permite)

Para verificar que la exportación es válida, importa la aplicación con un ID diferente:

1. En el App Builder, ve a **Import**.
2. Carga el archivo `app_completa_YYYYMMDD.sql`.
3. En las opciones de instalación, selecciona **Install as New Application** con ID `200`.
4. Verifica que la aplicación 200 se crea correctamente y es funcional.
5. **Elimina la aplicación 200** después de verificar (para no dejar aplicaciones huérfanas).

---

## Solución de Problemas

### Problema 1: SQLcl no reconoce el comando `apex export`

**Síntoma:** Al ejecutar `apex export -applicationid ...` dentro de SQLcl, aparece el error:
```
Unknown Command: apex
```
o bien el comando no produce ningún archivo de salida.

**Causa:** La versión de SQLcl instalada es anterior a la 22.x, o SQLcl no tiene el módulo APEX habilitado. El comando `apex export` fue introducido como parte de los comandos nativos de SQLcl a partir de la versión 22.x y requiere que SQLcl esté conectado a una base de datos con APEX instalado.

**Solución:**

```bash
# Paso 1: Verifica la versión de SQLcl
sql -v
# Si es anterior a 22.x, actualiza SQLcl desde:
# https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/

# Paso 2: Verifica que estás conectado antes de ejecutar el comando
# Dentro de SQLcl, primero conéctate y luego ejecuta:
sql usuario/password@host:puerto/servicio
SQL> apex export -applicationid 100 -dir /ruta/salida

# Paso 3: Alternativa con la herramienta APEXExport.class (Java)
# Si SQLcl no está disponible, usa el exportador Java incluido en APEX:
java oracle.apex.APEXExport \
  -db <host>:<puerto>:<servicio> \
  -user <usuario> \
  -password <password> \
  -applicationid <APP_ID>
# El archivo APEXExport.class está en $ORACLE_HOME/apex/utilities/
```

---

### Problema 2: El Web Source Module devuelve error 403 o datos vacíos al ejecutar la página 50

**Síntoma:** La página 50 (Tipos de Cambio) carga pero la región del reporte aparece vacía, o en el navegador se ve un error `ORA-29273: HTTP request failed` o un código de estado 403/SSL error en los logs de debug.

**Causa:** El servidor de Oracle Database no tiene acceso de red saliente al dominio `open.er-api.com`, ya sea porque el servidor está en una red privada sin salida a Internet, porque hay una ACL (Access Control List) de red en la base de datos que bloquea las conexiones HTTP salientes, o porque el certificado SSL del sitio externo no está en el wallet de Oracle.

**Solución:**

```sql
-- Paso 1: Verificar si existe una ACL que permita conexiones al dominio
SELECT host, lower_port, upper_port, ace_order, grant_option, inverted
FROM   dba_network_acl_privileges
WHERE  host LIKE '%er-api%';

-- Paso 2: Si no existe ACL, crearla (requiere privilegio DBA)
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host       => 'open.er-api.com',
    ace        => xs$ace_type(
                    privilege_list => xs$name_list('connect', 'resolve'),
                    principal_name => 'APEX_230200',  -- ajusta al esquema APEX de tu versión
                    principal_type => xs_acl.ptype_db
                  )
  );
  COMMIT;
END;
/

-- Paso 3: Si el problema es SSL, importar el certificado al wallet de Oracle
-- Consulta con tu DBA para ejecutar:
-- orapki wallet add -wallet $ORACLE_BASE/admin/wallet -trusted_cert -cert <cert.pem>
```

Si el entorno no permite modificar ACLs (por ejemplo, en `apex.oracle.com`), usa una API alternativa que ya esté en la lista de permitidos, o simula los datos con una tabla local para el propósito del laboratorio:

```sql
-- Alternativa: Crear tabla local con datos de ejemplo para la demostración
CREATE TABLE lab_tipos_cambio AS
SELECT 'EUR' AS moneda, 0.92 AS tasa_usd, SYSDATE AS fecha_actualizacion FROM DUAL UNION ALL
SELECT 'GBP', 0.79, SYSDATE FROM DUAL UNION ALL
SELECT 'MXN', 17.15, SYSDATE FROM DUAL UNION ALL
SELECT 'JPY', 149.50, SYSDATE FROM DUAL UNION ALL
SELECT 'CAD', 1.36, SYSDATE FROM DUAL;
-- Usa esta tabla como fuente del Classic Report en la página 50
```

---

## Limpieza del Entorno

Después de completar y validar el laboratorio, realiza las siguientes acciones de limpieza para mantener el entorno ordenado:

```bash
# 1. Eliminar archivos de exportación temporales (conserva los del repositorio Git)
rm -rf ~/apex-lab-exports/

# 2. Revocar el cliente OAuth2 de prueba si ya no se necesita
# En APEX → SQL Workshop → RESTful Services → OAuth Clients
# Selecciona "cliente_lab10" → Delete

# 3. Eliminar la aplicación de prueba (ID 200) si se creó durante la validación
# En App Builder → selecciona la aplicación 200 → Delete Application

# 4. Desactivar el Debug Mode en la aplicación principal
# App Builder → Edit Application Properties → Security
# → Allow Runtime Developer Toolbar → Only For Developers

# 5. Verificar que el repositorio Git está limpio y sincronizado
cd ~/apex-project
git status
# Debe mostrar: nothing to commit, working tree clean
git push origin main
# Debe mostrar: Everything up-to-date
```

> **⚠️ No eliminar:** El repositorio Git `~/apex-project/` y la aplicación principal de práctica. Estos son los artefactos finales del curso y pueden ser requeridos para evaluación.

---

## Resumen

En este laboratorio final completaste el ciclo de vida completo de una aplicación APEX a nivel profesional:

| Actividad                          | Herramienta Utilizada                    | Resultado                                    |
|------------------------------------|------------------------------------------|----------------------------------------------|
| Exportación de aplicación          | APEX Export Wizard + SQLcl               | 3 tipos de exportación generados             |
| Control de versiones               | Git + GitHub/GitLab                      | Repositorio con estructura profesional       |
| Automatización de exportación      | Script Shell + SQLcl                     | Pipeline básico de CI/CD funcional           |
| Consumo de API externa             | APEX Web Source Module                   | Página 50 con tipos de cambio en tiempo real |
| Exposición de datos como API       | ORDS RESTful Services + OAuth2           | Endpoint `/api/ventas/productos` operativo   |
| Revisión de seguridad              | APEX Security Advisor                    | 0 hallazgos de High Risk                     |
| Auditoría de accesibilidad         | Google Lighthouse                        | Score de Accessibility documentado           |
| Documentación técnica              | Markdown en repositorio Git              | README completo con checklist de producción  |

### Conceptos Clave del Laboratorio

- **Exportación en formato split** (APPLICATION_SOURCE en SQLcl): genera archivos individuales por componente, lo que facilita el diff en Git y el trabajo colaborativo en equipos.
- **Web Source Modules**: abstraen la complejidad de consumir APIs externas, permitiendo usar datos REST directamente como fuente en regiones APEX sin escribir código de integración manual.
- **ORDS RESTful Services**: permiten exponer datos de Oracle como APIs estándar REST, habilitando la integración con aplicaciones móviles, sistemas externos y microservicios.
- **Security Advisor + Lighthouse**: son las dos herramientas de auditoría complementarias — Security Advisor cubre la capa de aplicación APEX y Lighthouse cubre la experiencia del usuario final en el navegador.

### Recursos Adicionales

- [Oracle APEX Export/Import Documentation](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/exporting-an-application.html)
- [SQLcl APEX Export Command Reference](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/23.1/sqcug/apex.html)
- [ORDS RESTful Services Developer Guide](https://docs.oracle.com/en/database/oracle/oracle-rest-data-services/23.2/orddg/)
- [APEX Web Source Modules](https://docs.oracle.com/en/database/oracle/apex/23.2/htmdb/managing-web-source-modules.html)
- [Google Lighthouse Documentation](https://developer.chrome.com/docs/lighthouse/overview/)
- [APEX Security Best Practices](https://apex.oracle.com/en/platform/security/)

---

> **🎓 Felicitaciones:** Has completado el curso de Oracle APEX de nivel intermedio. Desde la configuración inicial del workspace en el Laboratorio 01 hasta el despliegue profesional con CI/CD, APIs REST y checklist de producción en este Laboratorio 10, has construido las competencias necesarias para desarrollar, asegurar y desplegar aplicaciones APEX de nivel empresarial.
