# Finanzen — Mapa de lógica de negocio y reglas del sistema

## 1. Propósito del documento

Este documento sirve como referencia para el desarrollo y mantenimiento de **Finanzen**, una aplicación de escritorio de análisis financiero para micro y pequeños negocios. Se dirige a quienes modifican:

- modelos SQLAlchemy, migraciones y restricciones de MySQL;
- repositorios, servicios, controladores y autorización;
- importación y validación de Excel;
- usuarios, negocios, roles y categorías;
- consultas, análisis, recomendaciones y reportes;
- integración con Gemini, caché Redis y trabajadores en segundo plano;
- interfaces PySide6 y funciones auxiliares.

**Criterio de interpretación:** lo descrito como **Implementado** se sustenta en el código revisado. **Regla de diseño** expresa una condición que debe seguir respetándose. **Pendiente o por verificar** no debe presentarse como funcionalidad terminada. Si una decisión cambia, hay que modificar tanto el código como esta documentación.

## 2. Alcance general

Finanzen importa y conserva registros de **ventas** y **movimientos financieros**, permite consultarlos por negocio y período, analiza su comportamiento y genera indicadores, recomendaciones y reportes. También ofrece análisis en lenguaje natural mediante una API de IA.

No pretende sustituir un sistema contable, ERP o profesional de contabilidad. Quedan fuera de su alcance actual la contabilidad de partida doble, facturación, inventario integral, nómina, cuentas por cobrar o pagar, clientes, proveedores, préstamos y conciliación bancaria automática. Estas funciones requerirían decisiones de diseño, modelos y permisos adicionales.

## 3. Usuarios y negocios

**Implementado:** un usuario puede crear varios negocios y pertenecer a varios negocios. Se distinguen dos relaciones:

```text
users.user_id ──< businesses.creator_id
       │
       └──< user_business >── businesses.business_id
                  └── role
```

- `businesses.creator_id` registra quién creó el negocio. No equivale, por sí solo, a su único responsable actual.
- `user_business` representa la pertenencia de un usuario a un negocio y el rol asignado en esa relación.
- La restricción `uq_user_business` impide duplicar la combinación `user_id` + `business_id`.
- Los registros financieros y las importaciones pertenecen siempre a un negocio concreto.

Archivos: `app/models/users.py`, `business.py`, `user_business.py`.

## 4. Roles y autorización

Los roles de `app/models/user_business.py` son:

```text
ADMIN              → Administrador técnico
MANAGER            → Gerente
FINANCIAL_ANALYST  → Analista financiero
EMPLOYEE           → Empleado
```

**Regla de diseño:** el rol debe consultarse en la base de datos al ejecutar operaciones sensibles; no basta con ocultar botones ni confiar en un rol enviado desde la interfaz. `AuthorizationService` realiza esa comprobación desde `app/core/security/authorization.py`.

### 4.1. Matriz de permisos aplicada en los módulos principales

| Operación | Admin | Gerente | Analista | Empleado |
|---|:---:|:---:|:---:|:---:|
| Importar Excel | Sí | Sí | Sí | Sí |
| Crear categorías faltantes durante la importación | Sí | Sí | No | No |
| Análisis financiero | Sí | Sí | Sí | No |
| Generar reportes PDF | Sí | Sí | Sí | No |
| Acceder a recomendaciones desde el panel | Sí | Sí | Sí | No |
| Configurar negocio y categorías | Sí | Sí | No | No |
| Vincular/desvincular usuarios en Configuración | Sí | Sí | No | No |
| Eliminar una importación completa | Sí | Sí | No | No |
| Página de monitoreo | Sí | No | No | No |

La tabla recoge las validaciones y restricciones visibles en los servicios y en el panel revisados; **cada nuevo caso de uso debe verificar sus permisos en el servicio correspondiente**.

### 4.2. Administrador técnico

- `users.is_admin` identifica administradores del sistema.
- Cuando se crea un negocio, su creador se vincula como `MANAGER` y los administradores del sistema se vinculan como `ADMIN`.
- La creación del negocio actualmente **falla si no existe ningún administrador del sistema**.
- No se puede asignar manualmente `ADMIN` mediante la operación normal de vinculación de usuarios.
- No se puede desvincular a un administrador técnico mediante la operación ordinaria de Configuración.

### 4.3. Creador y eliminación del negocio

**Implementado:** para eliminar un negocio, el solicitante debe tener permisos de gestión **y ser su creador**. La creación del negocio no concede al creador un permiso global sobre los demás negocios.

## 5. Creación de negocios

`BusinessService.create_business()` ejecuta dentro de una transacción:

```text
Crear Business
    ↓
Vincular al creador como MANAGER
    ↓
Vincular administradores técnicos como ADMIN
    ↓
Crear categorías predeterminadas
    ↓
COMMIT si todo tiene éxito / ROLLBACK si falla
```

El negocio almacena nombre, tipo de negocio, fecha de inicio de operaciones opcional, descripción opcional, logo opcional y creador. Los tipos están definidos en `app/models/business.py` (`TypeOfBusiness`): barbería, tienda de ropa, cafetería, farmacia, restaurante, ferretería, pulpería y otro.

No insertar manualmente las categorías iniciales en MySQL como sustituto de esta lógica.

## 6. Formato de importación Excel

La importación procesa archivos **`.xlsx`** mediante `ExcelReader`, `ExcelValidator` y `ExcelTransformer` (`app/excel_utils/`). Las hojas reconocidas son:

```text
Ventas
Movimientos
```

**Corrección frente al documento anterior:** el lector admite que una hoja esté ausente o vacía. Se rechaza el archivo cuando **ambas** carecen de datos procesables. La plantilla de referencia es `app/fixtures/Finanzen_Plantilla.xlsx`.

El lector utiliza la **tercera fila como encabezado** (`header=2`) en las hojas existentes. Por tanto, los archivos deben respetar el formato de la plantilla; no asumir encabezados en la primera fila.

### 6.1. Hoja `Movimientos`

```text
Fecha | Tipo | Categoría | Descripción | Monto | Método de Pago
```

### 6.2. Hoja `Ventas`

```text
Fecha | Producto/Servicio | Categoría | Cantidad | Precio Unitario | Total
```

Cada fila de ventas es un registro independiente de producto o servicio; no se utiliza un esquema de encabezado de factura con múltiples líneas vinculadas.

## 7. Validación del Excel

**Implementado:** se distingue entre errores de estructura y de datos. `ExcelValidator` valida las columnas requeridas, tipos, importes, fechas y consistencia entre campos.

Reglas principales:

- Se rechazan columnas faltantes o duplicadas en una hoja con datos.
- No se admiten fechas inválidas ni futuras.
- En `Ventas`, producto/servicio y categoría son obligatorios; cantidad, precio unitario y total deben ser mayores que cero.
- `Total` debe coincidir con `Cantidad × Precio Unitario` al comparar los valores redondeados a dos decimales.
- En `Movimientos`, el tipo y la categoría deben ser válidos y compatibles; la descripción debe estar presente; el monto debe ser mayor que cero.
- **En el Excel actual, `Método de Pago` es obligatorio y debe ser uno de los valores admitidos.**
- Los errores identifican, cuando corresponde, hoja, fila y columna.

**Regla de diseño:** no persistir parcialmente los registros válidos de un archivo que contiene errores. El procesamiento debe finalizar con éxito antes de guardar la importación.

> **Diferencia de capas:** en `app/models/movement.py`, `description` y `payment_method` permiten `NULL`, pero el validador del Excel exige ambos campos. No se debe deducir del esquema de MySQL que el importador admite valores vacíos.

## 8. Persistencia de importaciones

Una importación exitosa genera una fila en `imports` con:

```text
import_id | user_id | business_id | file_name | file_hash | imported_at
```

Los registros `movements` y `sales` creados se vinculan mediante `import_id`. `TransactionImportService.save_import_data()` agrupa en **una transacción** la creación de importación, categorías autorizadas que faltan y registros financieros.

```text
Archivo Excel
  ↓ lectura, normalización y validación
Datos correctos
  ↓ verificación de rol y categorías
Transacción MySQL
  ├── imports
  ├── sales
  └── movements
  ↓ COMMIT
Invalidación de caché Redis del negocio
```

Si falla una operación dentro de la transacción, no deben quedar importaciones o registros financieros a medio guardar.

## 9. Detección de archivos duplicados

**Implementado:** se calcula un hash SHA-256 del archivo (`app/utils/hash_utils.py`). La combinación `business_id` + `file_hash` es única en `imports`.

- Un mismo archivo no puede importarse dos veces **en el mismo negocio**.
- El control se realiza en el servicio y existe una restricción de unicidad en la base de datos.
- La unicidad no es global para todos los negocios.
- Una modificación del contenido del archivo cambia su hash; no equivale a deduplicación fila por fila.

## 10. Tipos de movimientos financieros

La entidad `Movement` (`app/models/movement.py`) admite exactamente dos tipos:

```text
INCOME  → Ingreso
EXPENSE → Gasto
```

No añadir nuevos tipos sin revisar validadores, análisis, reportes, enums y migraciones correspondientes.

## 11. Categorías de movimientos

Las categorías de movimientos están definidas por enums, no por la tabla dinámica `categories`.

**Ingresos (`IncomeCategory`):** Ventas, Servicios, Otros ingresos.

**Gastos (`ExpenseCategory`):** Compras, Personal, Servicios básicos, Alquiler, Transporte, Mantenimiento, Publicidad, Impuestos, Otros.

Una categoría debe ser compatible con el tipo:

```text
Válido:    Ingreso + Servicios
Válido:    Gasto   + Alquiler
Inválido: Ingreso + Compras
Inválido: Gasto   + Ventas
```

**Regla de diseño:** esta comprobación no debe depender exclusivamente de un widget; ya se aplica en la validación de Excel y en la conversión a modelos durante la importación.

## 12. Métodos de pago

Los valores definidos en `PaymentMethod` son:

```text
CASH     → Efectivo
CARD     → Tarjeta
TRANSFER → Transferencia
OTHER    → Otro
```

No se distingue entre tarjeta de crédito y débito. El modelo de base de datos permite el campo `payment_method` nulo, pero **el importador actual no admite un campo vacío**: para un método desconocido debe registrarse el valor válido `Otro`. Si el equipo desea permitir omisiones reales, deberá cambiar la validación y acordar su representación.

## 13. Independencia entre ventas e ingresos

**Regla central de Finanzen:** `sales` y `movements` son entidades independientes y no existe una clave foránea directa entre ambas. Las ventas no se convierten automáticamente en movimientos de ingreso.

```text
Ventas registradas:  C$10,000
Ingresos registrados: C$8,500
```

Esta diferencia no implica por sí sola un error. Tampoco deben sumarse el total de ventas y los ingresos como si fueran dos entradas de dinero independientes. El **balance de movimientos** se calcula como `ingresos − gastos`, mientras que el total de ventas se presenta como indicador comercial separado.

Esta distinción también está recogida en las instrucciones de `app/prompts/financial_prompts.py`.

## 14. Ventas y consistencia monetaria

La entidad `Sale` incluye:

```text
sale_id, business_id, import_id, category_id,
date, product_service, quantity, unit_price, total,
created_by, updated_by, created_at, updated_at
```

La regla de validación de importación es:

```text
total ≈ quantity × unit_price  (comparación a 2 decimales)
```

`quantity`, `unit_price` y `total` se almacenan en columnas SQLAlchemy `Numeric(12, 2)`. Al modificar estas reglas debe conservarse un tratamiento consistente de decimales y moneda; no usar valores de coma flotante para cálculos contables sin analizar el impacto.

## 15. Categorías de ventas

A diferencia de las categorías de movimientos, las categorías de ventas se almacenan en `categories` y **pertenecen a cada negocio**.

```text
categories
├── category_id
├── business_id
├── name
└── is_system
```

La clave foránea `sales.category_id` apunta a la categoría asignada. La restricción `uq_category_business_name` garantiza unicidad de `business_id` + `name` en MySQL; además, la capa de servicio comprueba duplicados cuando se configuran categorías.

Se deben normalizar los nombres al comparar categorías importadas para reducir inconsistencias de espacios y mayúsculas/minúsculas.

## 16. Categorías iniciales vigentes

**Importante: el código actual difiere del documento original.** `app/services/business_service.py` define exactamente estas seis categorías predeterminadas:

```text
General
Productos
Servicios
Promociones
Combos
Otros
```

**Las seis se crean con `is_system=True`.** En la versión revisada ya **no** se crean por defecto `Novedades` ni `Misceláneos`, y la protección no se limita a `Otros`.

Si se decide modificar el conjunto o su protección, será necesario revisar el servicio de creación de negocios, la gestión de categorías existentes y las posibles migraciones de datos.

## 17. Protección de categorías del sistema

**Implementado:** `ConfigurationRepository.delete_category()` impide eliminar toda categoría con `is_system=True`.

- Las categorías iniciales quedan protegidas frente a esa eliminación.
- `Otros` es esencial porque sirve como destino cuando se elimina una categoría personalizada utilizada por ventas.
- La restricción debe aplicarse en el servicio/repositorio, aunque la interfaz también muestre la prohibición.
- **El renombrado individual de categorías no aparece como una operación terminada en el flujo de Configuración revisado.** No afirmar que existe una protección de renombrado comprobada si aún no se ha implementado esa acción.

## 18. Categorías personalizadas

Un usuario autorizado para gestionar el negocio (actualmente `ADMIN` o `MANAGER`) puede crear categorías adicionales mediante Configuración. También pueden crearse categorías faltantes en la importación, pero **solo si el usuario posee uno de esos dos roles**.

Ejemplos:

```text
Farmacia:    Analgésicos, Vitaminas
Ferretería: Herramientas, Plomería
```

Las categorías creadas de este modo tienen `is_system=False`, pertenecen exclusivamente al negocio y pueden ser eliminadas conforme a las reglas de reasignación.

## 19. Prevención de categorías duplicadas

- El nombre de cada categoría debe ser no vacío y de hasta 100 caracteres en la configuración actual.
- No se permite repetir una categoría dentro del mismo negocio.
- La comprobación en la interfaz/servicio no sustituye la restricción `uq_category_business_name` de la base de datos.
- Dos negocios distintos sí pueden utilizar el mismo nombre de categoría.
- Para evitar diferencias entre normalización de aplicación y comparación de la base de datos, las modificaciones futuras deben considerar también la intercalación (`collation`) de MySQL.

## 20. Eliminación de categorías de ventas

**Implementado:** al eliminar una categoría personalizada, `ConfigurationRepository` reasigna primero sus ventas a la categoría del sistema **Otros** y después elimina la categoría.

```text
Categoría personalizada
       ↓
Ventas relacionadas → Otros
       ↓
Eliminar categoría original
       ↓
COMMIT de todos los cambios
```

Los registros históricos de ventas **no se eliminan** por el solo hecho de eliminar su categoría. Esta operación se ejecuta como parte de la transacción de guardado de Configuración. Si la categoría es del sistema, no existe o falta `Otros`, la eliminación no debe completarse.

## 21. Configuración del negocio y de usuarios

`ConfigurationService.save_business_changes()` permite coordinar cambios de nombre, tipo, descripción, adición de categorías y eliminación de categorías personalizadas. Se valida el permiso de gestión antes del guardado y se usa `commit`/`rollback`.

La administración de usuarios vinculados permite buscar cuentas, vincularlas con un rol válido y desvincularlas, respetando estas condiciones:

- No vincular dos veces al mismo usuario con el mismo negocio.
- No asignar `ADMIN` desde el formulario de vinculación habitual.
- No desvincular al creador del negocio desde la operación ordinaria.
- No desvincular a los administradores técnicos de la misma forma.

**Regla de diseño:** cuando el usuario modifica varias categorías antes de pulsar «Guardar», esas operaciones deben persistirse como un conjunto y no dejar cambios parciales.

## 22. Edición de datos financieros

El documento antiguo preveía que el Gerente pudiera editar filas individuales de ventas y movimientos después de importar. **No debe describirse esa capacidad como implementada en la versión revisada.**

Los modelos de tabla visibles, como `app/views/dashboard/table_models/movements_table_model.py`, exponen los datos para su consulta y actualización visual, pero no implementan actualmente un flujo integral de edición y guardado individual con `setData()` y un servicio correspondiente.

**Regla pendiente para una futura edición individual:**

- permitirla solo a roles autorizados según la decisión del equipo;
- validar que el registro pertenezca al negocio seleccionado;
- repetir las mismas reglas financieras de importación;
- registrar `updated_by` y `updated_at`;
- persistir mediante Service y Repository, no directamente desde Qt;
- invalidar la caché financiera correspondiente después del `commit`.

## 23. Tablas de consulta y presentación

La arquitectura actual separa los modelos de presentación Qt de la persistencia:

```text
QTableView
    ↓
QAbstractTableModel  (representación de filas)
    ↓
Vista / Controller
    ↓
Service
    ↓
Repository
    ↓
SQLAlchemy / MySQL
```

No introducir lógica SQL en `QAbstractTableModel`, ni dejar que un `QMessageBox` sea la única comprobación de permisos o integridad.

## 24. Auditoría básica

Los modelos `Movement` y `Sale` tienen:

```text
created_by
updated_by
created_at
updated_at
```

Esto permite identificar al creador, la última edición y sus fechas, cuando la aplicación actualiza correctamente los campos. **No existe, como parte del modelo revisado, un historial versionado completo de cada modificación.** Para auditoría exhaustiva sería necesario diseñar otra solución.

## 25. Eliminación o reversión de importaciones

**Implementado:** los usuarios con rol `ADMIN` o `MANAGER` pueden eliminar una importación completa del negocio.

```text
Verificar rol y pertenencia al negocio
         ↓
Eliminar sales vinculadas al import_id
         ↓
Eliminar movements vinculados al import_id
         ↓
Eliminar imports
         ↓
COMMIT
         ↓
Invalidar caché Redis del negocio
```

El servicio utiliza una transacción y el repositorio comprueba que el `import_id` pertenece al `business_id` solicitado. No deben quedar filas financieras huérfanas o parcialmente eliminadas. Esta operación es destructiva: en la interfaz deben comunicarla claramente y solicitar una confirmación adecuada.

## 26. Eliminación de negocios

**Implementado:** la eliminación de un negocio se restringe al creador con un rol autorizado para gestionar ese negocio. El repositorio elimina sus ventas, movimientos, importaciones, categorías, vínculos de usuarios y finalmente el negocio, dentro de la transacción que coordina el servicio.

**Consideración pendiente:** revisar la limpieza de claves Redis asociadas a negocios eliminados. No confundir eliminar datos de MySQL con eliminar automáticamente la caché externa.

## 27. Períodos e historial financiero

Los datos importados se conservan como historial del negocio hasta que una operación autorizada los elimine. Importar un nuevo Excel no reemplaza automáticamente las importaciones anteriores.

Las utilidades de fecha (`app/utils/date_utils.py`) incluyen, entre otras:

```text
Esta semana
Este mes
Mes anterior
Últimos 15 días
Últimos 30 días
Últimos 3 meses
Último año
```

Los módulos de consulta, análisis, IA y reportes exponen **subconjuntos** de estos períodos y, donde corresponde, un rango `Personalizado`. No todos los módulos tienen necesariamente las mismas opciones del documento antiguo. Los períodos deben calcularse con un inicio y final definidos; si hay comparación, el período de referencia debe estar explícito.

## 28. Análisis financiero determinista

`app/services/analysis_service.py` consulta `AnalysisRepository` y combina estadísticas, categorías, comparaciones y hallazgos mediante utilidades de `app/utils/analysis/`.

Los análisis incluyen, según disponibilidad de datos:

- total de ingresos y gastos de movimientos;
- balance de movimientos (`ingresos − gastos`);
- evolución temporal y análisis por categorías;
- indicadores comerciales basados en `sales`;
- comparación con el período inmediatamente anterior de igual duración;
- variaciones y hallazgos calculados a partir de reglas;
- visualizaciones para presentar resultados.

**Implementado:** el acceso al servicio de análisis está restringido a `ADMIN`, `MANAGER` y `FINANCIAL_ANALYST`. No deben fabricarse hallazgos ni porcentajes cuando faltan datos de referencia o la comparación no es válida.

## 29. Indicadores y análisis de ventas

El análisis comercial puede incluir total vendido, cantidad de registros, promedio por venta, evolución, categorías y comportamiento de productos o servicios. Estos indicadores se calculan a partir de `sales` y son distintos de los ingresos registrados en `movements`.

**Regla de diseño:** evitar indicadores ambiguos como «porcentaje de ingresos provenientes de ventas» si no existe una relación contable verificable entre ambas entidades. Cuando se presenta una comparación entre ventas e ingresos, identificarla como **comparación de dos conjuntos independientes**, no como conciliación automática ni participación contable probada.

## 30. Recomendaciones basadas en reglas

**Implementado:** `app/services/recommendation_service.py` y `app/repositories/recommendation_repository.py` generan recomendaciones deterministas a partir de datos y reglas del sistema. La aplicación dispone de una presentación separada para recomendaciones por áreas (ventas, gastos, ingresos).

El módulo debe:

- utilizar información real del negocio y períodos definidos;
- diferenciar hechos observados de recomendaciones;
- no atribuir causas sin evidencia;
- explicar condiciones relevantes de la regla cuando sea útil;
- evitar duplicar conclusiones sin aportar información nueva.

**Corrección:** el documento original mencionaba `scikit-learn` como tecnología prevista. En el código y dependencias revisados no se identificó una implementación de aprendizaje automático basada en esa librería. Debe tratarse como **posible mejora futura**, no como parte confirmada del sistema.

## 31. Análisis y asistente de inteligencia artificial

**Implementado:** la integración de IA utiliza el proveedor Gemini a través de `app/ai/ai_provider.py`. `app/prompts/financial_prompts.py` prepara instrucciones y `app/services/aiRecommendation_service.py` obtiene el contexto financiero desde los repositorios.

Existen cuatro análisis predefinidos:

```text
General
Ventas
Ingresos
Gastos
```

También existe una modalidad de **consulta libre del asistente financiero**. El procesamiento se realiza mediante `app/workers/ai_worker.py` para no bloquear la interfaz.

**Reglas:** las respuestas deben basarse en los datos disponibles, no inventar cifras ni interpretar ventas como ganancias. La IA es apoyo para la lectura financiera, no una fuente contable ni una decisión automática vinculante.

La configuración se lee de variables como `GEMINI_API_KEY`, `GEMINI_MODEL`, `GEMINI_TEMPERATURE`, `GEMINI_MAX_OUTPUT_TOKENS` y `GEMINI_TIMEOUT`, declaradas en `app/config/settings.py`.

## 32. Caché de análisis con Redis

**Implementado:** `app/services/redisCacheService.py` almacena resultados de análisis de IA predefinidos con un tiempo de expiración de **86,400 segundos (24 horas)**.

- Las claves distinguen negocio, versión de los datos, tipo de análisis y fechas.
- Antes de solicitar un análisis predefinido, se puede recuperar su respuesta de caché si corresponde a una versión vigente.
- Al guardar datos mediante importación o eliminar una importación, se invalida la versión de caché de ese negocio.
- Se usan operaciones de Redis para comprobar que la versión no cambió al leer o guardar un resultado.
- **Las consultas libres del asistente no se almacenan en esta caché.**
- Redis es caché: la información financiera persistente sigue en MySQL.

Si Redis no está configurado o no está disponible, el servicio puede quedar deshabilitado; esto no convierte a Redis en una base de datos transaccional alternativa.

**Revisión futura:** asegurarse de invalidar resultados si se agregan nuevos caminos para modificar o borrar datos financieros, editar categorías relevantes para el análisis o eliminar negocios.

## 33. Reportes financieros PDF

**Implementado:** se admiten cuatro tipos de informe: `general`, `ventas`, `ingresos` y `gastos`, con acceso de `ADMIN`, `MANAGER` y `FINANCIAL_ANALYST`.

La generación combina:

```text
ReportService (datos y reglas)
      ↓
ReportDocumentService (formato y gráficos)
      ↓
PdfGenerator (Jinja2 + WeasyPrint)
      ↓
Plantillas HTML/CSS en app/pdf_templates/
      ↓
PDF
```

Los reportes pueden incorporar comparaciones con un período anterior, hallazgos, tablas y gráficos generados con Matplotlib. `app/workers/report_worker.py` prepara y guarda el documento sin bloquear el flujo principal de la interfaz. No deben exponer datos de un negocio a usuarios que no tengan el rol autorizado.

## 34. Seguridad de cuentas y recuperación

Las credenciales se administran en servicios de usuarios y recuperación. Entre las reglas implementadas se encuentran:

- Validación de nombre, correo electrónico y fecha de nacimiento al registrarse.
- Edad mínima de **18 años** en el proceso de registro actual.
- Contraseñas de al menos ocho caracteres con mayúscula, minúscula, número y carácter especial.
- Contraseñas almacenadas mediante **hash Argon2**, no en texto plano.
- Correos normalizados antes de las búsquedas de autenticación.
- Recuperación mediante código/token con vencimiento definido (generación actual de **15 minutos**).
- Soporte de cuentas recordadas con `QSettings` para la lista de correos y almacenamiento de credenciales a través de un backend seguro en Windows cuando está disponible; no guardar contraseñas en texto plano en `QSettings`.

Archivos relevantes: `app/core/security/account_validation.py`, `password.py`, `app/services/user_services.py`, `recovery_services.py`, `remembered_accounts_service.py` y `app/utils/recoveryCode.py`.

## 35. Arquitectura y separación de responsabilidades

La estructura principal de ejecución es:

```text
main.py
   ↓
AppContainer + AppController
   ↓
views/ + ui/ (PySide6)
   ↓
controllers/
   ↓
services/
   ↓
repositories/
   ↓
models/ + database/ (SQLAlchemy)
   ↓
MySQL
```

Módulos complementarios:

```text
excel_utils/    → lectura, transformación y validación de Excel
utils/          → fechas, formatos, estadísticas y gráficos
ai/ + prompts/ → proveedor de IA e instrucciones
workers/        → operaciones de larga duración fuera del hilo UI
pdf_templates/  → plantillas de reportes
core/security/ → validación de cuentas y autorización
```

**Responsabilidades:**

- **UI y Views:** recibir acciones, representar información y comunicar errores.
- **Controllers:** traducir solicitudes de la interfaz a casos de uso y coordinar respuestas.
- **Services:** aplicar reglas, permisos, transacciones y coordinación de operaciones.
- **Repositories:** ejecutar consultas y persistencia orientadas a entidades del modelo.
- **Models:** definir estructura, relaciones, enums y restricciones ORM.
- **Database:** administrar engine, sesiones y base declarativa.
- **Workers:** ejecutar operaciones costosas sin bloquear la interfaz gráfica.

No ubicar toda la lógica en `main.py`, en una vista o en un modelo Qt.

## 36. Transacciones e integridad

Las operaciones que afectan varias entidades relacionadas deben ser atómicas:

```text
COMMIT   → se completaron todas las operaciones
ROLLBACK → ocurrió un error y no debe persistir un estado parcial
```

Ejemplos: creación de negocio y categorías, importación de Excel, eliminación de categorías con reasignación de ventas, eliminación de importación y eliminación de negocio. La invalidación de Redis, cuando corresponda, debe ocurrir **después de confirmar** la modificación persistente para evitar exponer análisis obsoletos.

La base de datos utiliza claves foráneas, restricciones de unicidad e índices; estas protecciones complementan, pero no reemplazan, las validaciones de los servicios.

## 37. Migraciones del esquema

**Implementado:** el repositorio contiene `alembic.ini`, `migrations/env.py` y versiones de migración en `migrations/versions/`.

**Reglas de trabajo:**

1. No cambiar un modelo que altere el esquema sin revisar si requiere una migración Alembic.
2. Evaluar efectos sobre datos existentes, índices, claves foráneas, roles y valores de enums.
3. Revisar cualquier migración autogenerada antes de aplicarla.
4. Probar las migraciones en una base de desarrollo y conservar una estrategia de recuperación.
5. Mantener `app/models/` y el esquema real sincronizados.

Las migraciones deben ser revisadas por el equipo, especialmente cuando afectan datos financieros históricos.

## 38. Modelo de datos actual

Las ocho entidades principales definidas en `app/models/` son:

| Clase | Tabla MySQL | Función |
|---|---|---|
| `Users` | `users` | Cuentas y administradores del sistema |
| `Business` | `businesses` | Negocios y su creador |
| `UserBusiness` | `user_business` | Pertenencia y rol por negocio |
| `Recovery` | `recoveries` | Códigos, tokens y expiración de recuperación |
| `Import` | `imports` | Procedencia de archivos importados |
| `Movement` | `movements` | Ingresos y gastos |
| `Sale` | `sales` | Registros de ventas |
| `Category` | `categories` | Categorías de ventas por negocio |

Relaciones principales:

```text
users        1 ── N businesses       (creator_id)
users        N ── M businesses       (user_business)
users        1 ── N recoveries
users        1 ── N imports
businesses   1 ── N imports
businesses   1 ── N movements
businesses   1 ── N sales
businesses   1 ── N categories
imports      1 ── N movements
imports      1 ── N sales
categories   1 ── N sales
users        1 ── N movements/sales   (created_by, updated_by)
```

No existe relación directa `movements ↔ sales`. Las relaciones y opciones `ondelete` deben revisarse en los modelos y en las migraciones antes de cambiar reglas de eliminación.

## 39. Diferencias identificadas respecto al documento anterior

| Tema | Documento anterior | Código actual revisado |
|---|---|---|
| Roles | Gerente, analista, empleado | Incluye `ADMIN` técnico |
| Categorías iniciales | Ocho categorías | Seis categorías |
| `is_system` | Solo «Otros» protegida | Las seis categorías predeterminadas están protegidas |
| Excel | Dos hojas obligatorias | Puede procesar una sola hoja con datos |
| Método de pago | Opcional / «Otro sin especificar» | En importación es requerido; «Otro» es valor válido |
| Edición de registros | El gerente podrá editar individualmente | No verificada como función terminada en el flujo de tablas revisado |
| Recomendaciones | Previsión de usar `scikit-learn` | Reglas deterministas y análisis con Gemini; sin implementación comprobada de `scikit-learn` |
| Caché | No documentada | Redis por negocio, versión, tipo y período; TTL 24 horas |
| Informes | Mención general | Generación PDF con Jinja2, Matplotlib y WeasyPrint |
| Eliminación de negocio | No especificada | Solo el creador con permiso de gestión |
| Migraciones | Decisiones futuras | Alembic ya integrado |

## 40. Aspectos que el equipo debe revisar explícitamente

Los siguientes puntos se proponen como **temas de revisión**, no como fallos confirmados ni tareas ya completadas:

1. **Edición individual:** decidir si se implementará, quién podrá usarla y qué validaciones y auditoría tendrá.
2. **Categorías del sistema:** confirmar si todas las predeterminadas deben mantenerse protegidas o si alguna debería permitir edición/eliminación.
3. **Excel y nulabilidad:** alinear la obligatoriedad de `Descripción` y `Método de Pago` entre importador y modelo cuando sea necesario.
4. **Redis:** verificar invalidación para cualquier nuevo camino de modificación de información utilizada en análisis, y limpieza de claves tras eliminar negocios.
5. **Migraciones:** comprobar que la base de datos de cada integrante haya aplicado las versiones requeridas.
6. **Dependencias:** mantener coherentes `requirements.txt` y `pyproject.toml` al incorporar paquetes usados por IA, Redis o reportes.
7. **Seguridad:** probar permisos en el nivel de servicio, incluso si se intenta llamar operaciones sin pasar por la vista.
8. **Pruebas:** cubrir importaciones fallidas, duplicados, categorías protegidas, reasignación a «Otros», permisos, rol técnico y regresiones de análisis.

## 41. Principio para nuevas decisiones

Antes de implementar una funcionalidad, responder:

1. ¿Qué necesidad de negocio resuelve?
2. ¿Qué entidad y qué negocio afecta?
3. ¿Qué roles pueden ejecutarla y quién comprueba el permiso?
4. ¿Qué validaciones y restricciones de base de datos requiere?
5. ¿Debe ejecutarse en una transacción o un worker?
6. ¿Invalida resultados de caché o modifica cálculos existentes?
7. ¿Requiere migración, actualización de reportes y pruebas?
8. ¿Debe actualizarse este documento?

**Regla final:** las vistas comunican; los servicios aplican las reglas; los repositorios persisten; la base de datos protege la integridad. Una funcionalidad no se considera terminada solo porque aparezca en la interfaz o en este mapa de decisiones.

---

## Referencias dentro del repositorio

- [`README.md`](https://github.com/luis-hdz7/Finanzen/blob/main/README.md) — descripción y alcance.
- [`app/models/`](https://github.com/luis-hdz7/Finanzen/tree/main/app/models) — entidades, relaciones y enums.
- [`app/core/security/authorization.py`](https://github.com/luis-hdz7/Finanzen/blob/main/app/core/security/authorization.py) — autorización.
- [`app/services/`](https://github.com/luis-hdz7/Finanzen/tree/main/app/services) — reglas y casos de uso.
- [`app/excel_utils/`](https://github.com/luis-hdz7/Finanzen/tree/main/app/excel_utils) — lectura y validación de Excel.
- [`app/repositories/`](https://github.com/luis-hdz7/Finanzen/tree/main/app/repositories) — acceso a datos.
- [`app/workers/`](https://github.com/luis-hdz7/Finanzen/tree/main/app/workers) — operaciones en segundo plano.
- [`migrations/`](https://github.com/luis-hdz7/Finanzen/tree/main/migrations) — migraciones Alembic.
