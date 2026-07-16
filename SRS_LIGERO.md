# 📝 ESPECIFICACIÓN DE REQUISITOS DE SOFTWARE (SRS)

## 📦 Sistema de Control de Existencias y Alertas (SCEA) - Edición Ágil
**ID del Documento:** `SRS-SCEA-v1.1`  
**Estado:** `Aprobado para Desarrollo`

---

## 🗺️ 1. MAPA DE ALCANCE Y LÍMITES DEL SISTEMA

### 1.1 Declaración de Propósito
Este documento consolida los acuerdos de ingeniería y requerimientos para el desarrollo de la plataforma web de control de inventarios de la tienda. El propósito de este artefacto es delimitar estrictamente las responsabilidades del sistema para evitar desviaciones en el alcance (*scope creep*) durante el sprint de desarrollo.

### 1.2 Límites Operacionales (En Alcance vs. Fuera de Alcance)

+-------------------------------------------------------------+
   |                  SISTEMA DE INVENTARIO SCEA                 |
   |                                                             |
   |  [EN ALCANCE]                                               |
   |   +-----------------------------------------------------+   |
   |   | - Validación de Códigos GS1 El Salvador (741)       |   |
   |   | - Registro estricto de Stock                        |   |
   |   | - Alertas visuales de stock crítico                 |   |
   |   | - Persistencia en PostgreSQL                        |   |
   |   +-----------------------------------------------------+   |
   |                                                             |
   |  [FUERA DE ALCANCE (Fronteras)]                             |
   |   +-----------------------------------------------------+   |
   |   | - Facturación y pasarelas de pago (POS)             |   |
   |   | - Base de datos de Clientes y CRM                   |   |
   |   | - Contabilidad automática e impuestos (IVA)         |   |
   |   | - App móvil nativa (Android / iOS)                  |   |
   |   +-----------------------------------------------------+   |
   +-------------------------------------------------------------+


---

## 👥 2. PERFILES DE USUARIO Y MATRIZ DE ACCESOS

### 2.1 Personas de Usuario (Roles)
* **🔑 Administrador del Sistema:** Responsable del control total de accesos, auditoría de catálogos de productos y parametrización global del software.
* **📦 Operador de Bodega:** Usuario en el terreno encargado del flujo diario (ingreso de mercadería, egreso por merma, despachos físicos y validaciones de código de barra).
* **📊 Auditor / Supervisor:** Rol de consulta táctica enfocado en la supervisión de alertas de desabastecimiento e informes analíticos rápidos.

### 2.2 Control de Accesos Basado en Roles (RBAC)

| Módulo de Negocio | Acción Específica | Administrador | Operador de Bodega | Auditor / Supervisor |
| :--- | :--- | :---: | :---: | :---: |
| **Catálogo de Productos** | Crear Ficha de Producto | 🟢 Sí | 🟢 Sí | 🔴 No |
| | Modificar Atributos Base | 🟢 Sí | 🟢 Sí | 🔴 No |
| | Remover Producto del Catálogo | 🟢 Sí | 🔴 No | 🔴 No |
| **Operaciones de Inventario**| Registrar Movimientos (Kárdex) | 🟢 Sí | 🟢 Sí | 🔴 No |
| | Visualizar Existencias en Tiempo Real| 🟢 Sí | 🟢 Sí | 🟢 Sí |
| **Métricas y Alertas** | Configurar Umbrales de Stock Bajo | 🟢 Sí | 🔴 No | 🔴 No |
| | Monitorear Alertas de Ruptura | 🟢 Sí | 🟢 Sí | 🟢 Sí |
| **Configuraciones** | Administración de Usuarios | 🟢 Sí | 🔴 No | 🔴 No |

---

## 🛠️ 3. ESPECIFICACIÓN DE HISTORIAS DE USUARIO (US)

### 📦 Módulo A: Gestión de Catálogo e Integridad

#### **Historia de Usuario ID: US-01**
> **COMO** Operador de Bodega  
> **QUIERO** dar de alta un producto nuevo ingresando su código de barras salvadoreño  
> **PARA** garantizar que solo se procesen productos autorizados y distribuidos localmente  

**Reglas de Negocio Clave:**
1. **Validación Geográfica:** El código de barras debe ser obligatoriamente tipo **GTIN-13** y debe iniciar con el prefijo **741** (perteneciente a El Salvador).
2. **Unicidad Estricta:** No pueden registrarse dos artículos con el mismo código de barras.

##### 🧪 Criterios de Aceptación (Gherkin)

```gherkin
Escenario: Registro exitoso de un producto nacional salvadoreño
  Dado que el Operador de Bodega se encuentra autenticado en el panel de "Alta de Productos"
  Y el código de barras ingresado es "7411234567890" (Prefijo SV válido)
  Y este identificador no existe actualmente en el sistema
  Cuando completa los campos obligatorios:
    | Campo       | Valor                  |
    | Nombre      | Café Molido Local 500g |
    | Stock Inicial| 50                     |
  Y confirma la acción haciendo clic en "Dar de Alta"
  Entonces la base de datos PostgreSQL almacena el nuevo registro de manera persistente
  Y el sistema muestra el mensaje de confirmación: "Producto registrado bajo catálogo nacional."

Escenario: Intento de registro con un código internacional o inválido
  Dado que el Operador de Bodega intenta dar de alta un producto
  Cuando ingresa el código de barras "7501234567890" (Prefijo de otro país)
  Entonces el sistema bloquea el registro inmediatamente
  Y despliega una alerta visual: "Error: El código de barras no cumple con el estándar de El Salvador (Prefijo 741)."


Historia de Usuario ID: US-02
COMO Operador de Bodega

QUIERO actualizar las propiedades no identificadoras de un artículo

PARA corregir discrepancias en precios o descripciones sin alterar su identidad en inventario

Reglas de Negocio Clave:

Inmutabilidad de la Clave Primaria: El campo Código de Barras debe quedar bloqueado (sólo lectura) durante el proceso de edición.

🧪 Criterios de Aceptación (Gherkin)

Escenario: Modificación exitosa de propiedades básicas
  Dado que el Operador de Bodega visualiza la ficha técnica de un producto existente
  Cuando modifica el precio unitario de "$3.50" a "$3.75"
  Y guarda los cambios
  Entonces el sistema actualiza únicamente el campo de precio en PostgreSQL
  Y despliega un banner de confirmación: "Catálogo actualizado de manera exitosa."

Escenario: Intento de alteración de la identidad del producto (Código de Barras)
  Dado que el Operador de Bodega se encuentra en el formulario de "Edición de Producto"
  Cuando intenta interactuar con el campo "Código de Barras"
  Entonces el sistema mantiene el campo en estado Deshabilitado (ReadOnly)
  Y no permite ningún tipo de entrada por teclado en dicho input.

  🔒 4. RESTRICCIONES TÉCNICAS INNEGOCIABLESIDRestricciónEspecificación de ArquitecturaTEC-01Motor de Base de DatosPostgreSQL 15+ (Exclusivo, no se admiten bases de datos NoSQL ni otros RDBMS).TEC-02Algoritmo de ValidaciónValidación del algoritmo de módulo 10 para el dígito verificador del código GTIN-13, asegurando que inicie con los dígitos 741.TEC-03Seguridad de ComunicacionesTodo el tráfico de datos debe ser forzado a través de HTTPS mediante TLS 1.3.TEC-04Compatibilidad de ClientesEl Front-End Web debe ser 100% responsivo y compatible con Safari Mobile, Chrome (Mobile/Desktop) y Firefox.

  🎯 5. REQUISITOS DE CALIDAD NO FUNCIONALES (ISO/IEC 25010)
⚡ Rendimiento y Eficiencia (NFR-01)
Descripción: El tiempo de respuesta de las consultas al kárdex de inventario no debe superar los 1.5 segundos bajo cargas normales.

Validación: Pruebas de estrés automatizadas simulando la lectura masiva de una tabla con 10,000 registros de productos.

🛡️ Seguridad en Reposo (NFR-02)
Descripción: Las credenciales de acceso deben ser encriptadas con la función de derivación de claves robustas bcrypt (factor de trabajo de 12).

Validación: Auditoría del esquema del script de base de datos para verificar que no existan contraseñas almacenadas en texto plano.

🛡️ Robustez de Validación (NFR-03)
Descripción: La validación del código salvadoreño (741) debe ejecutarse tanto en el Frontend (usando una expresión regular rápida) como en el Backend (capa lógica) para evitar inserciones maliciosas mediante API de terceros.

Validación: Pruebas unitarias enviando payloads manipulados vía Postman al endpoint de creación.

🧬 6. MATRIZ DE TRAZABILIDAD CRUZADA (RTM)
Esta matriz asocia los requerimientos de negocio con los componentes de software y casos de prueba técnicos para garantizar cobertura total.

[Requerimiento] ───► [Historia de Usuario] ───► [Componente Backend] ───► [Caso de Prueba (QA)]
    REQ-INV-01            US-01 (Crear)          productController.js         TC-REG-01 (Exitoso)
    REQ-INV-02            US-01 (Validar SV)     barcodeValidator.js          TC-REG-02 (Rechazo 741)
    REQ-INV-03            US-02 (Editar)         productController.js         TC-ED-01 (ReadOnly)


ID RequerimientoID de Historia de UsuarioArchivo / Componente Back-EndID del Caso de Prueba QAREQ-INV-01US-01 (Crear)controllers/productController.jsTC-REG-001-OKREQ-INV-02US-01 (Validar SV)utils/barcodeValidator.jsTC-REG-002-FAILREQ-INV-03US-02 (Editar)controllers/productController.jsTC-ED-001-OKREQ-INV-04US-03 (Eliminar)controllers/productController.jsTC-DEL-001-OKREQ-STK-01US-04 (Alertas)services/stockAlertService.jsTC-ALT-001-WARN


🚫 7. CRITERIOS DE RECHAZO DE REQUERIMIENTOS (DUMP-LIST)
Para proteger al equipo de desarrollo de expectativas poco realistas, se acordó descartar las siguientes solicitudes tras un análisis de viabilidad técnica:

+-----------------------------------------------------------------------------------------------+
|                                REQUERIMIENTOS DECLARADOS INVÁLIDOS                             |
+------------------------------------+-------------------------+--------------------------------+
| Solicitud Original del Cliente     | Motivo de la Exclusión  | Justificación Técnica          |
+------------------------------------+-------------------------+--------------------------------+
| "El sistema debe estimar las       | Falta de Viabilidad /   | El desarrollo de modelos       |
| compras ideales del próximo mes    | Complejidad Excesiva    | predictivos (ML/IA) requiere   |
| de forma predictiva"               |                         | una cantidad masiva de datos   |
|                                    |                         | históricos que la tienda aún   |
|                                    |                         | no posee.                      |
+------------------------------------+-------------------------+--------------------------------+
| "La aplicación debe cargar al      | No Medible (Subjetivo)  | El concepto "instantáneo" no   |
| instante sin importar el internet" |                         | es cuantificable. Se cambió a  |
|                                    |                         | un SLA técnico de latencia     |
|                                    |                         | menor a 1.5s bajo conexión 4G. |
+------------------------------------+-------------------------+--------------------------------+

💬 8. NOTAS TÉCNICAS Y ACUERDOS PREVIOS
[!IMPORTANT]
Preguntas de Ingeniería abiertas para revisión con Stakeholders:

¿Cómo se aprovisionará el servidor de PostgreSQL actual? ¿Contamos con accesos de superusuario para configurar esquemas y triggers de auditoría?

Respecto a la lectura de códigos de barra: ¿los operarios usarán pistolas lectoras de hardware (las cuales emulan entradas de teclado estándar) o se requiere acceso a la cámara del dispositivo móvil para decodificación óptica?

¿El sistema debe integrarse con algún Active Directory local para el inicio de sesión único (SSO), o implementamos un servicio de autenticación autónomo basado en JWT?

[!NOTE]
Suposiciones Críticas del Proyecto:

La base de datos PostgreSQL ya se encuentra instalada en un servidor físico dentro de la intranet de la tienda o en la nube privada del cliente.

Las alertas por stock mínimo se reflejarán únicamente en un panel del Dashboard Web dinámico y no requerirán, en esta primera etapa, integraciones de envío de correos (SMTP) o mensajería de texto (Twilio/SMS).

🗂️ 9. METADATOS DEL PROYECTO
Identificación del Producto: Sistema de Control de Existencias y Alertas (SCEA)

Líder Técnico de QA y Desarrollo: Oscar Eduardo Guerrero Menéndez

Repositorio de Código Fuente: GitHub - Eduarpotato

Fecha Límite de Liberación: 15 de Julio de 2026

Versión Base del Documento: v1.1-Agile-SRS

"El software de calidad se define por su capacidad de solucionar el problema correcto, de la manera más sencilla posible."

$ git tag -a v1.1-srs-rewrite -m "SRS completamente rediseñado y blindado contra copias"