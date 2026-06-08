<img src="images/neteclogo (2).png" alt="logo" width="300"/>

# Desarrollo en APEX

## Plataforma de laboratorios

Te damos la bienvenida a la **plataforma de laboratorios** del curso **Desarrollo en APEX**. Aquí podrás explorar diferentes tecnologías a través de prácticas guiadas. ¡Desarrolla tus habilidades y lleva tus conocimientos al siguiente nivel!

Curso práctico para desarrollar aplicaciones web con Oracle APEX, desde configuración de workspace hasta seguridad, procesamiento y despliegue, aplicando técnicas para integrar datos Oracle en soluciones empresariales robustas.


## Lista de laboratorios

Cada uno de estos laboratorios está diseñado para ofrecerte una experiencia práctica. Haz clic en los enlaces para comenzar. 

------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 1

### [Práctica 1: Crear primer workspace y aplicación básica](Capitulo01/README.md#crear-primer-workspace-y-aplicación-básica)

  - Descripción: Esta práctica introduce al estudiante al ecosistema Oracle APEX partiendo desde cero conceptual hasta la primera aplicación funcional. Aplicando los conceptos de arquitectura estudiados en la lección 1.1 —motor de renderizado PL/SQL, ORDS como listener HTTP y esquema de datos de negocio— el estudiante creará un workspace, explorará las áreas principales del entorno de desarrollo y construirá una aplicación básica conectada al esquema HR de Oracle. Al finalizar, habrá completado el ciclo completo: acceder a datos reales, visualizarlos en un reporte y editar un registro a través de un formulario generado automáticamente.
    
  - ⏱️Duración estimada: 32 min
    
------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 2

### [Práctica 2: Diagnóstico de rendimiento básico](Capitulo02/README.md#diagnóstico-de-rendimiento-básico)

  - Descripción: En esta práctica  explorarás la arquitectura interna de Oracle APEX desde la perspectiva del rendimiento. Partirás de una aplicación deliberadamente construida con problemas comunes —consultas SQL sin índices, regiones que cargan datos innecesarios y páginas sin paginación— y utilizarás las herramientas de diagnóstico integradas de APEX (Debug Mode, Activity Log, APEX Monitor) junto con las DevTools del navegador para localizar y cuantificar cada cuello de botella. Al finalizar, habrás aplicado correcciones concretas y medido su impacto, desarrollando un flujo de trabajo sistemático de diagnóstico que podrás reutilizar en cualquier proyecto APEX.
    
  - ⏱️Duración estimada: 44 min
    
------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 3

### [Práctica 3: Configurar workspace, usuarios y políticas](Capitulo03/README.md#configurar-workspace-usuarios-y-políticas)

  - Descripción: En esta práctica asumirás el rol de administrador de instancia Oracle APEX para gestionar el ciclo de vida completo de workspaces: crearás un workspace de desarrollo, lo asociarás a esquemas de base de datos, crearás usuarios con distintos niveles de acceso y verificarás las restricciones de cada rol. Posteriormente configurarás políticas de seguridad a nivel de workspace, explorarás los logs de actividad para auditar acciones y ejecutarás tareas de mantenimiento operativo como la purga de sesiones antiguas. Al finalizar, habrás construido un entorno de desarrollo correctamente aislado, seguro y monitoreable, aplicando directamente los conceptos de la Lección 3.1.
    
  - ⏱️Duración estimada: 52 min
    
------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 4

### [Práctica 4: Construir una aplicación de ejemplo (formularios y reportes)](Capitulo04/README.md#construir-una-aplicación-de-ejemplo-formularios-y-reportes)

  - Descripción: En esta práctica construirás una aplicación funcional de Gestión de Proyectos en Oracle APEX utilizando el Create Application Wizard sobre un esquema de base de datos provisto por el instructor. Partirás de tablas existentes para generar automáticamente la estructura base de la aplicación y luego personalizarás cada página en el Page Designer: configurarás formularios con validaciones declarativas, crearás reportes clásicos con formato condicional, ajustarás la navegación global y aplicarás el Universal Theme con colores personalizados mediante el Theme Roller. Al finalizar, tendrás una aplicación multi-página completamente navegable y visualmente coherente.
    
  - ⏱️Duración estimada: 52 min
    
------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 5

### [Práctica 5: Implementar dashboards y reportes interactivos](Capitulo05/README.md#implementar-dashboards-y-reportes-interactivos)

  - Descripción: En esta práctica transformarás la aplicación construida en el Laboratorio 04 añadiendo un módulo completo de visualización y análisis de datos. Crearás un dashboard ejecutivo que combina un Interactive Report con funcionalidades avanzadas, un Interactive Grid para edición masiva en línea, y un conjunto de JET Charts con datos reales. Finalizarás auditando la accesibilidad de tus componentes con Google Lighthouse y corrigiendo al menos tres problemas identificados.
    
  - ⏱️Duración estimada: 52 min

------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 6

### [Práctica 6: Añadir comportamientos dinámicos y validaciones](Capitulo06/README.md#añadir-comportamientos-dinámicos-y-validaciones)

  - Descripción: Este práctica añade interactividad avanzada a la aplicación construida en los laboratorios anteriores mediante el sistema de Dynamic Actions de Oracle APEX y la integración de JavaScript personalizado. Se trabajará progresivamente desde acciones declarativas puras (mostrar/ocultar campos, habilitar/deshabilitar botones, calcular valores) hasta la integración de la API JavaScript de APEX para realizar llamadas AJAX al servidor y validaciones híbridas cliente-servidor. Al finalizar, la aplicación contará con una experiencia de usuario reactiva, con validaciones robustas tanto en el navegador como en la base de datos.
    
  - ⏱️Duración estimada: 52 min

------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 7

### [Práctica 7: Implementar procesos y lógica de negocio con PL/SQL](Capitulo07/README.md#implementar-procesos-y-lógica-de-negocio-con-plsql)

  - Descripción: En esta práctica implementarás un flujo completo de aprobación de solicitudes dentro de tu aplicación APEX existente. Crearás un paquete PL/SQL (PKG_SOLICITUDES) que encapsula la lógica de negocio para aprobar, rechazar y escalar solicitudes; configurarás procesos de página en los puntos de ejecución correctos; implementarás auditoría automática con manejo explícito de transacciones; y aplicarás buenas prácticas de seguridad para prevenir SQL injection. Al finalizar, la aplicación contará con un backend robusto, trazable y seguro que refleja patrones de desarrollo profesional con Oracle APEX 23.2.
    
  - ⏱️Duración estimada: 52 min

------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 8

### [Práctica 8: Crear y aplicar componentes compartidos en varios módulos](Capitulo08/README.md#crear-y-aplicar-componentes-compartidos-en-varios-módulos)

  - Descripción: En esta práctica construirás una base sólida de componentes reutilizables dentro de la aplicación APEX desarrollada en los laboratorios anteriores. Centralizarás listas de valores duplicadas, crearás plantillas personalizadas de región e ítem, implementarás procesos globales de aplicación para auditoría y mantenimiento, y diseñarás un plugin APEX básico que encapsule un componente de UI personalizado. Al finalizar, la aplicación seguirá el principio de "definir una vez, usar en todas partes", reduciendo la duplicación de código y estandarizando el desarrollo del equipo.
    
  - ⏱️Duración estimada: 52 min

------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 9

### [Práctica 9: Implementar autenticación, roles y pruebas de seguridad](Capitulo09/README.md#implementar-autenticación-roles-y-pruebas-de-seguridad)

  - Descripción: En esta práctica implementarás una capa de seguridad completa sobre la aplicación construida en laboratorios anteriores. Comenzarás creando una tabla de usuarios propia con contraseñas hasheadas mediante DBMS_CRYPTO, luego configurarás un esquema de autenticación personalizado y lo compararás con APEX Accounts estándar. Diseñarás un modelo de cuatro roles de negocio (Administrador, Supervisor, Operador, Consultor), crearás Authorization Schemes que consulten la membresía desde la base de datos, habilitarás Session State Protection y finalmente ejecutarás pruebas de penetración básicas usando el APEX Security Advisor y Chrome DevTools para identificar y corregir vulnerabilidades críticas.
    
  - ⏱️Duración estimada: 52 min
    
------------------------------------------------------------------------------------------------------------------------------------------------------  

### Capítulo 10

### [Práctica 10: Despliegue, integración y checklist de producción](Capitulo10/README.md#despliegue-integración-y-checklist-de-producción)

  - Descripción: Esta práctica final integra todos los conocimientos adquiridos en el curso en un proceso completo de despliegue profesional. Comenzarás exportando la aplicación APEX en sus diferentes modalidades (completa, componentes, esquema), configurarás un repositorio Git con la estructura recomendada para proyectos APEX y automatizarás el proceso de exportación con SQLcl. Luego implementarás un Web Source Module para consumir una API REST pública, crearás un servicio RESTful en ORDS que exponga datos de tu aplicación, y finalizarás ejecutando el checklist completo de preparación para producción: Security Advisor, revisión de rendimiento, accesibilidad con Lighthouse y generación de documentación técnica.
    
  - ⏱️Duración estimada: 52 min

------------------------------------------------------------------------------------------------------------------------------------------------------    

## 📬 **Contacto y más información**


Si tienes alguna pregunta o necesitas más detalles, no dudes en [contactarnos](mailto:soporte@netec.com). También puedes encontrar más recursos en nuestra [página](https://netec.com).



---

¡Gracias por visitar nuestra plataforma! No olvides revisar todos los laboratorios y comenzar tu viaje de aprendizaje hoy mismo.


