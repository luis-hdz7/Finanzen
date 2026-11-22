# Indicaciones del Proyecto Integrador 2.0 — Finanzen

> **Documento de trabajo para el equipo de desarrollo.** Actualizado a partir de la rama `main` del repositorio [luis-hdz7/Finanzen](https://github.com/luis-hdz7/Finanzen) (consulta: 22 de noviembre de 2026).
>
> Este archivo explica cómo preparar el entorno, ejecutar la aplicación, respetar la organización del código y colaborar sin introducir errores. No sustituye al `README.md` ni a `estructura_proyecto.md`: los complementa.

---

## 1. Estructura del proyecto

La aplicación de escritorio **Finanzen** está desarrollada principalmente en Python con PySide6. Su código está organizado por responsabilidades: presentación, controladores, lógica de negocio, persistencia, procesamiento de datos e integraciones externas.

### 1.1. Estructura general

La siguiente vista es **resumida**: muestra las carpetas principales y algunos archivos representativos, no todos los recursos ni todos los módulos del repositorio.

```text
Finanzen/
├── app/
│   ├── ai/
│   │   └── ai_provider.py
│   ├── config/
│   │   └── settings.py
│   ├── controllers/
│   │   ├── user_controller.py
│   │   ├── analysis_controller.py
│   │   ├── report_controller.py
│   │   └── ...
│   ├── core/
│   │   ├── app_container.py
│   │   ├── app_controller.py
│   │   ├── screen_utils.py
│   │   └── security/
│   ├── database/
│   │   ├── base.py
│   │   ├── connection.py
│   │   └── init_db.py
│   ├── errors/
│   ├── excel_utils/
│   ├── fixtures/
│   ├── generated/
│   │   └── resources_rc.py
│   ├── models/
│   │   ├── users.py
│   │   ├── business.py
│   │   ├── user_business.py
│   │   ├── recovery.py
│   │   ├── category.py
│   │   ├── imports.py
│   │   ├── movement.py
│   │   └── sale.py
│   ├── pdf_templates/
│   │   ├── pdf_generator.py
│   │   ├── secciones/
│   │   └── styles/
│   ├── prompts/
│   │   └── financial_prompts.py
│   ├── repositories/
│   ├── resources/
│   │   ├── resources.qrc
│   │   ├── images/
│   │   ├── icons/
│   │   └── fonts/
│   ├── services/
│   ├── ui/
│   │   ├── dashboard/
│   │   ├── login/
│   │   └── ...
│   ├── utils/
│   │   ├── analysis/
│   │   └── visualizations/
│   ├── views/
│   │   ├── login/
│   │   ├── createAccount/
│   │   ├── createBusiness/
│   │   ├── chooseABusiness/
│   │   ├── dashboard/
│   │   └── profile/
│   ├── workers/
│   └── main.py
├── migrations/
│   ├── env.py
│   └── versions/
├── .env.example
├── .gitignore
├── alembic.ini
├── consultas.sql
├── estructura_proyecto.md
├── Indicaciones.md
├── Lógica de Negocio y Reglas del Sistema.md
├── pyproject.toml
├── requirements.txt
├── README.md
└── test.py
```

**Importante:** el repositorio **no incluye** actualmente una carpeta `.vscode/` versionada. La configuración de VS Code puede crearse de forma local; `.gitignore` la excluye. Tampoco deben aparecer en Git los directorios locales `venv/`, `__pycache__/` ni el archivo `.env`.

### 1.2. Responsabilidades de las carpetas

| Ruta | Responsabilidad |
|---|---|
| `app/views/` | Ventanas, páginas y comportamiento de la interfaz en Python. |
| `app/ui/` | Diseños `.ui` creados con Qt Designer. |
| `app/controllers/` | Coordinación entre acciones de la interfaz y servicios. |
| `app/services/` | Casos de uso, reglas y operaciones del negocio. |
| `app/repositories/` | Consultas y operaciones de persistencia mediante SQLAlchemy. |
| `app/models/` | Entidades y relaciones de la base de datos. |
| `app/database/` | Base declarativa, motor de conexión y sesiones. |
| `app/core/` | Inicio, navegación entre pantallas, dependencias y seguridad. |
| `app/config/` | Lectura centralizada de variables de entorno. |
| `app/excel_utils/` | Lectura, transformación y validación de archivos Excel. |
| `app/errors/` | Tipos y estructuras de errores especializados. |
| `app/ai/` | Comunicación con el proveedor externo de IA (Gemini). |
| `app/prompts/` | Construcción de instrucciones para análisis financiero con IA. |
| `app/workers/` | Tareas en segundo plano para no bloquear la interfaz. |
| `app/pdf_templates/` | Plantillas HTML/CSS y generación de informes PDF. |
| `app/resources/` | Imágenes, iconos, fuentes y descriptor `.qrc`. |
| `app/generated/` | Código generado automáticamente desde recursos Qt. |
| `app/utils/` | Validaciones, formatos, fechas, estadísticas y gráficos reutilizables. |
| `app/fixtures/` | Plantillas y archivos de ejemplo utilizados para pruebas de importación. |
| `migrations/` | Historial de cambios del esquema de base de datos con Alembic. |

**Regla de organización:** una vista no debería contener consultas SQL complejas ni decisiones financieras que pertenezcan a servicios. Los controladores coordinan; los servicios aplican la lógica; los repositorios consultan la base de datos.

---

## 2. Preparación del entorno

Estas instrucciones utilizan **Windows PowerShell** y asumen que todos los comandos se ejecutan desde la **raíz del repositorio**.

### 2.1. Obtener el proyecto

```powershell
git clone https://github.com/luis-hdz7/Finanzen.git
cd Finanzen
```

Si ya se tiene una copia local, utilizarla y comprobar primero el estado de Git con `git status`.

### 2.2. Verificar Python

El archivo `pyproject.toml` del repositorio declara **Python >= 3.14**. Se recomienda utilizar Python 3.14 para trabajar con la configuración actual.

```powershell
python --version
```

Si se tienen varias versiones instaladas en Windows, comprobar la disponibilidad con `py -0p` y seleccionar la versión 3.14 cuando corresponda.

### 2.3. Crear y activar el entorno virtual

Crear el entorno virtual una sola vez:

```powershell
py -3.14 -m venv venv
```

Si el comando `py` no está disponible, usar `python -m venv venv` **solo después de comprobar que `python` apunta a la versión requerida**.

Activarlo en PowerShell:

```powershell
.\venv\Scripts\Activate.ps1
```

Cuando está activo suele aparecer `(venv)` en la terminal. Para salir:

```powershell
deactivate
```

**Nota:** si PowerShell bloquea la activación por su política de ejecución, se puede utilizar directamente `.\venv\Scripts\python.exe` para ejecutar Python y pip, sin cambiar la política del sistema.

### 2.4. Instalar las dependencias

```powershell
python -m pip install -r requirements.txt
```

El proyecto utiliza, entre otras bibliotecas, **PySide6, SQLAlchemy, Alembic, pandas, openpyxl, Matplotlib, redis, google-genai, Jinja2 y WeasyPrint**. Algunas plataformas pueden requerir bibliotecas nativas adicionales para generar PDF con WeasyPrint; consultar su documentación si la instalación o la generación de reportes falla.

Comprobar el entorno:

```powershell
python -m pip check
```

**No subir `venv/` al repositorio.** Cada integrante instala sus propias dependencias.

---

## 3. Configuración de servicios y variables de entorno

Finanzen **no funciona únicamente instalando paquetes de Python**. También necesita configurar los servicios externos que utilizan los diferentes módulos.

### 3.1. Crear el archivo `.env`

El repositorio contiene `.env.example`, pero actualmente esa plantilla solo muestra `EMAIL`, `PASSWORD` y `DATABASE_URL`. El código de `app/config/settings.py` utiliza además variables de **Gemini y Redis**. Por ello, al configurar el entorno local, se deben incluir las variables necesarias para las funciones que se vayan a utilizar.

```powershell
Copy-Item .env.example .env
```

Ejemplo orientativo de estructura del archivo `.env` (sin credenciales reales):

```dotenv
# Correo utilizado para recuperación de cuentas
EMAIL=
PASSWORD=

# Base de datos MySQL
DATABASE_URL=

# Proveedor de IA (Gemini)
GEMINI_API_KEY=
GEMINI_MODEL=
GEMINI_TEMPERATURE=0.3
GEMINI_MAX_OUTPUT_TOKENS=2048
GEMINI_TIMEOUT=60000

# Caché Redis
REDIS_URL=
```

**No escribir valores de ejemplo ficticios que parezcan credenciales reales.** Completar cada variable con los datos autorizados para el entorno del equipo. Algunos parámetros de Gemini tienen valores por defecto en `settings.py`; aun así, conviene documentar los valores seleccionados para el entorno.

### 3.2. MySQL

La conexión se configura con `DATABASE_URL`. La aplicación utiliza SQLAlchemy y el conector de MySQL instalado en las dependencias.

1. Verificar que el servidor MySQL esté disponible.
2. Crear o identificar la base de datos de desarrollo autorizada.
3. Configurar `DATABASE_URL` con el dialecto/controlador compatible y las credenciales correspondientes.
4. Aplicar las migraciones (sección 4) antes de trabajar con tablas.

No compartir usuarios, contraseñas ni bases de datos de producción en commits, capturas o ejemplos públicos.

### 3.3. Redis

`REDIS_URL` define el acceso al servidor Redis. En el código revisado, Redis se utiliza como **caché de análisis predefinidos de IA**. Si no está configurado, el servicio de caché permanece deshabilitado; no sustituye la base de datos principal.

La caché utiliza resultados separados por negocio, tipo de análisis y período, con una duración de **24 horas**. Las operaciones que modifican datos financieros deben invalidar la caché correspondiente; no omitir esta integración al modificar procesos de importación o eliminación.

### 3.4. Gemini

La comunicación con la IA se concentra en `app/ai/ai_provider.py`; los mensajes e instrucciones financieras están en `app/prompts/financial_prompts.py`. Se requiere una `GEMINI_API_KEY` válida para las funciones de IA.

No colocar claves en el código, en las vistas ni en prompts. **Nunca incluir el archivo `.env` en un commit.**

### 3.5. Correo electrónico

Las variables `EMAIL` y `PASSWORD` corresponden al envío de códigos para recuperación de cuenta. Deben configurarse con una cuenta de servicio autorizada por el equipo; no incluir claves de correo en el repositorio.

### 3.6. Mantener `.env.example`

Cuando se añada una nueva variable obligatoria en `settings.py`, **actualizar `.env.example` con el nombre y una descripción, dejando el valor vacío**. La plantilla debe ser suficiente para que otra persona identifique qué necesita configurar, sin revelar secretos.

---

## 4. Base de datos y migraciones con Alembic

Finanzen utiliza **SQLAlchemy** para sus modelos y repositorios y **Alembic** para controlar la evolución del esquema.

Archivos relevantes:

```text
app/database/base.py
app/database/connection.py
app/models/
alembic.ini
migrations/env.py
migrations/versions/
```

### 4.1. Aplicar migraciones existentes

Con MySQL configurado y el entorno virtual activo:

```powershell
python -m alembic upgrade head
```

Para consultar el estado:

```powershell
python -m alembic current
python -m alembic heads
python -m alembic history
```

Estos comandos trabajan contra la base de datos definida por `DATABASE_URL`; **verificar el entorno antes de ejecutarlos**.

### 4.2. Si se modifica un modelo

1. Realizar los cambios en el archivo correspondiente dentro de `app/models/`.
2. Verificar sus relaciones y restricciones de integridad.
3. Crear una revisión de migración, si corresponde:

```powershell
python -m alembic revision --autogenerate -m "describir cambio del esquema"
```

4. **Revisar manualmente la migración generada**: Alembic puede necesitar ajustes para renombres, datos existentes o restricciones.
5. Probar la migración sobre una base de datos de desarrollo o de pruebas.
6. Incluir el cambio de modelo y la migración de `migrations/versions/` en el mismo conjunto de cambios.

**No usar `Base.metadata.create_all()` como sustituto de las migraciones del proyecto** ni modificar manualmente la base de datos compartida sin coordinarlo.

---

## 5. Gestión de dependencias

El repositorio contiene **`requirements.txt`** y **`pyproject.toml`**. Ambos deben revisarse cuando se modifican las dependencias, aunque actualmente **no contienen listas idénticas**: `requirements.txt` incluye paquetes que no están declarados en la lista de `pyproject.toml`.

### 5.1. Agregar o actualizar un paquete

Instalarlo en el entorno virtual:

```powershell
python -m pip install nombre-del-paquete
```

Luego:

1. Confirmar que se usa desde el proyecto y verificar su compatibilidad con Python 3.14.
2. Actualizar la declaración de dependencias correspondiente en `requirements.txt`.
3. Revisar también `pyproject.toml` y actualizarlo si el paquete forma parte de las dependencias del proyecto.
4. Probar los módulos afectados.
5. Revisar las diferencias antes del commit.

**Sobre `pip freeze > requirements.txt`:** este comando captura *todo* lo instalado en el entorno, incluidos paquetes ajenos al proyecto. No debe ejecutarse indiscriminadamente sobre un entorno usado para otros trabajos. Si se usa para reconstruir el archivo desde un entorno limpio, revisar cuidadosamente el resultado:

```powershell
python -m pip freeze
git diff -- requirements.txt pyproject.toml
```

El criterio principal es que una instalación nueva a partir de los archivos del repositorio funcione de forma reproducible.

### 5.2. Antes de confirmar cambios

```powershell
python -m pip check
git status
git diff -- requirements.txt pyproject.toml
```

Incluir los archivos modificados de dependencias en el mismo commit:

```powershell
git add requirements.txt pyproject.toml
git commit -m "chore: actualizar dependencias"
```

**Regla:** no agregar una dependencia nueva que sea necesaria para ejecutar Finanzen sin documentarla en los archivos adecuados.

---

## 6. Ejecución del proyecto

### 6.1. Desde la terminal

Con el entorno activo y las conexiones configuradas:

```powershell
python -m app.main
```

El punto de entrada `app/main.py` crea la aplicación PySide6, carga recursos Qt, inicializa `AppContainer` y `AppController`, abre el flujo de inicio de sesión y ejecuta el bucle de eventos.

Si no se activa el entorno, también puede usarse su intérprete directamente desde la raíz del proyecto:

```powershell
.\venv\Scripts\python.exe -m app.main
```

### 6.2. Desde Visual Studio Code (opcional)

El repositorio **ignora `.vscode/`**, por lo que el atajo `Ctrl + Shift + B` **no está garantizado en una clonación limpia**. Quien quiera utilizarlo puede crear localmente `.vscode/tasks.json` con el siguiente contenido:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Ejecutar Finanzen",
      "type": "process",
      "command": "${workspaceFolder}\\venv\\Scripts\\python.exe",
      "args": ["-m", "app.main"],
      "options": { "cwd": "${workspaceFolder}" },
      "group": { "kind": "build", "isDefault": true },
      "problemMatcher": []
    }
  ]
}
```

Después, utilizar **Ctrl + Shift + B**. Esta configuración asume Windows y un entorno virtual llamado `venv` dentro del proyecto. No es necesario subir `tasks.json` a Git si `.vscode/` continúa ignorado.

### 6.3. Solución básica de problemas

- **No encuentra un módulo:** revisar que el intérprete sea el de `venv` y ejecutar `python -m pip install -r requirements.txt`.
- **Error de base de datos:** revisar `DATABASE_URL`, disponibilidad del servidor y migraciones.
- **La IA no responde:** revisar `GEMINI_API_KEY`, conectividad y mensajes del proveedor.
- **Redis no está disponible:** revisar `REDIS_URL`; la aplicación puede operar sin caché, aunque las consultas de IA no reutilicen esos resultados.
- **Faltan imágenes o iconos:** revisar el `.qrc` y regenerar `resources_rc.py`.
- **Falla la generación de PDF:** revisar dependencias de WeasyPrint y los archivos de `app/pdf_templates/`.

---

## 7. Gestión de recursos de Qt y diseño de interfaz

### 7.1. Archivos relevantes

```text
app/resources/resources.qrc
app/resources/images/
app/resources/icons/
app/resources/fonts/
app/generated/resources_rc.py
app/ui/
app/views/
```

Los archivos `.ui` describen la estructura visual de las pantallas. Las clases Python de `app/views/` cargan esas interfaces y conectan eventos con los controladores.

### 7.2. Agregar una imagen, icono o fuente

1. Guardar el archivo en su subcarpeta de `app/resources/`.
2. Agregar o actualizar la referencia en `app/resources/resources.qrc`.
3. Regenerar los recursos desde la raíz del proyecto:

```powershell
pyside6-rcc app/resources/resources.qrc -o app/generated/resources_rc.py
```

4. Ejecutar Finanzen y comprobar que el recurso aparece correctamente.
5. Incluir en el commit el recurso original, `resources.qrc` y `resources_rc.py`:

```powershell
git add app/resources/ app/generated/resources_rc.py
git commit -m "feat: agregar recursos de interfaz"
```

### 7.3. Regla sobre archivos generados

**Nunca editar manualmente `app/generated/resources_rc.py`.** Se produce automáticamente a partir del archivo `.qrc`. Los cambios deben hacerse sobre el recurso original o el `.qrc` y luego recompilarse.

### 7.4. Modificar pantallas

- Cambios puramente visuales: revisar primero el archivo correspondiente de `app/ui/`.
- Comportamiento de botones, navegación o presentación: revisar `app/views/` y sus controladores.
- Reglas de acceso y operaciones financieras: implementarlas en `app/core/security/` y/o `app/services/`, no solo ocultando controles de la interfaz.
- Después de modificar una `.ui`, verificar los nombres de controles utilizados desde Python para evitar errores de carga o de búsqueda de widgets.

---

## 8. Reglas de desarrollo por módulo

### 8.1. Usuarios, negocios y seguridad

Los modelos de `app/models/` incluyen usuarios, negocios, roles asociados a negocios, recuperación de cuentas, importaciones, ventas, movimientos y categorías.

- Centralizar las verificaciones de acceso en las funciones y servicios de seguridad existentes, especialmente `app/core/security/authorization.py`.
- Las contraseñas deben tratarse mediante los mecanismos de hash existentes, como `app/core/security/password.py` (Argon2), nunca como texto plano.
- Comprobar los permisos **en la operación de negocio**, incluso si la interfaz ya restringe una pantalla.
- Evitar consultas que mezclen datos de diferentes negocios por no filtrar `business_id`.

### 8.2. Importación de Excel

El flujo de importación utiliza `app/excel_utils/`, servicios, repositorios y workers.

- Reutilizar los validadores y transformadores existentes en vez de duplicarlos en las vistas.
- Comprobar columnas, categorías, fechas, valores y errores antes de persistir.
- Mantener la asociación de ventas y movimientos con su negocio e importación.
- Conservar las verificaciones de autorización, la atomicidad de las operaciones y la invalidación de caché que corresponda.
- Utilizar los archivos de `app/fixtures/` como datos de ejemplo, **no** subir información financiera real de terceros sin autorización.

### 8.3. Análisis financiero y recomendaciones

- Mantener separados los cálculos deterministas (`analysis_service.py`, utilidades estadísticas y repositorios) de los análisis generados por IA.
- No confundir **ventas** con **ingresos registrados como movimientos**, ni interpretar automáticamente ambos como ganancias.
- Documentar cálculos, períodos comparativos y criterios usados al cambiar reglas financieras.
- Evitar añadir consultas SQL a las vistas o al archivo `main.py`.

### 8.4. Inteligencia artificial y Redis

- Mantener la integración con Gemini en `app/ai/ai_provider.py`.
- Modificar prompts en `app/prompts/financial_prompts.py` y comprobar que no promuevan conclusiones sin respaldo en los datos.
- Utilizar `app/workers/ai_worker.py` para solicitudes que no deben bloquear la interfaz.
- Respetar el comportamiento de caché de `app/services/redisCacheService.py`: 24 horas de duración y separación por negocio y período.
- No guardar en Redis conversaciones libres del asistente como si fueran análisis predefinidos; el flujo actual distingue ambos tipos de consulta.

### 8.5. Reportes PDF

- Los datos y cálculos del reporte se preparan en los servicios correspondientes.
- `app/services/report_document_service.py` adapta datos, formatos y gráficos.
- `app/pdf_templates/` contiene la estructura HTML, CSS y el generador que usa Jinja2 y WeasyPrint.
- Al cambiar un reporte, verificar los cuatro tipos: **general, ventas, ingresos y gastos**, según corresponda.
- Probar contenido, tablas, gráficos, caracteres especiales y generación del PDF final.

### 8.6. Operaciones largas

Para importaciones, IA, análisis y generación de reportes se usan workers bajo `app/workers/`. No ejecutar operaciones lentas directamente en el hilo principal de la interfaz. Respetar las señales de éxito, error, progreso y cancelación que correspondan.

---

## 9. Flujo de trabajo con Git y GitHub

Antes de comenzar a modificar archivos:

```powershell
git status
git fetch origin
git pull --ff-only origin main
```

El comando `git pull --ff-only` supone que se está trabajando en `main` y que no hay cambios locales que impidan actualizar. **Si existen cambios sin guardar o una rama de trabajo activa, revisar la situación antes de ejecutar un pull.**

### 9.1. Usar ramas para cambios nuevos

Se recomienda trabajar en una rama para cada funcionalidad o corrección:

```powershell
git switch -c feat/nueva-funcionalidad
```

Después de desarrollar y probar:

```powershell
git status
git diff
git add app/
git commit -m "feat: describir funcionalidad"
git push -u origin feat/nueva-funcionalidad
```

Abrir un Pull Request para revisión cuando el equipo utilice ese flujo. Revisar siempre los archivos antes de hacer `git add`; **no usar `git add app/` si hay datos, credenciales u otros cambios no relacionados dentro de esa carpeta**.

### 9.2. Tipos de commit sugeridos

| Prefijo | Cuándo usarlo |
|---|---|
| `feat:` | Nueva funcionalidad. |
| `fix:` | Corrección de errores. |
| `refactor:` | Reorganización interna sin cambios funcionales esperados. |
| `docs:` | Actualización de documentación. |
| `chore:` | Mantenimiento y dependencias. |
| `test:` | Cambios en pruebas automatizadas. |

Ejemplos:

```text
feat: agregar filtro de movimientos por fecha
fix: corregir validación de archivos Excel
refactor: separar consultas en repositorios
docs: actualizar indicaciones de instalación
chore: actualizar dependencias
```

### 9.3. Revisiones antes de un commit

1. Ejecutar el módulo o flujo modificado y probar los escenarios relevantes.
2. Revisar el resultado de `git status` y `git diff`.
3. Verificar que no se incluyen contraseñas, tokens, `.env`, `venv/`, cachés ni datos privados.
4. Si cambió la base de datos, incluir y probar la migración correspondiente.
5. Si cambiaron dependencias, revisar `requirements.txt` y `pyproject.toml`.
6. Si cambiaron recursos Qt, regenerar y agregar `resources_rc.py`.
7. Confirmar únicamente archivos relacionados con el cambio.

**Nota:** el repositorio tiene un archivo `test.py`, pero eso no implica que exista una suite automatizada completa. Se deben realizar pruebas manuales de los flujos afectados y ampliar pruebas automatizadas cuando sea posible.

---

## 10. Archivos ignorados y seguridad

La configuración actual de `.gitignore` excluye, entre otros:

```text
venv/
__pycache__/
*.pyc
.vscode/
.env
*.db
*.sqlite
*.sqlite3
```

**Nunca subir a GitHub:**

- Credenciales de MySQL, Redis, correo o Gemini.
- El archivo local `.env`.
- Entornos virtuales, archivos temporales o cachés de Python.
- Archivos con datos financieros privados de usuarios o negocios, salvo que exista autorización para compartirlos.

Tener un archivo en `.gitignore` no elimina información que **ya haya sido versionada**. Si accidentalmente se comparte una clave, debe revocarse o rotarse cuanto antes y corregirse el historial según la política del equipo.

---

## 11. Guía rápida según el tipo de cambio

| Tipo de trabajo | Qué revisar o actualizar |
|---|---|
| Cambiar una pantalla | `app/ui/`, `app/views/`, controlador asociado. |
| Agregar lógica de negocio | `app/services/`, controlador y pruebas pertinentes. |
| Modificar consultas | `app/repositories/` y el servicio que las utiliza. |
| Modificar tablas | `app/models/` y `migrations/versions/`. |
| Agregar recursos | `app/resources/`, `.qrc` y `app/generated/resources_rc.py`. |
| Añadir un paquete | `requirements.txt` y, según corresponda, `pyproject.toml`. |
| Cambiar variables de entorno | `app/config/settings.py` y `.env.example` (sin secretos). |
| Cambiar importación de Excel | `app/excel_utils/`, servicios, repositorios, workers y fixtures de prueba. |
| Cambiar análisis o recomendaciones | `app/services/`, `app/utils/analysis/`, repositorios y prompts si aplica. |
| Cambiar integración con IA | `app/ai/`, `app/prompts/`, worker de IA y uso de caché. |
| Cambiar informes PDF | `app/services/report_*`, `app/pdf_templates/` y gráficos. |

---

## 12. Comandos de referencia

Todos los comandos siguientes se ejecutan desde la raíz del repositorio en PowerShell, salvo indicación distinta.

| Tarea | Comando |
|---|---|
| Verificar Python | `python --version` |
| Crear entorno con Python 3.14 | `py -3.14 -m venv venv` |
| Activar entorno | `.\venv\Scripts\Activate.ps1` |
| Instalar dependencias | `python -m pip install -r requirements.txt` |
| Revisar dependencias | `python -m pip check` |
| Crear `.env` local | `Copy-Item .env.example .env` |
| Aplicar migraciones | `python -m alembic upgrade head` |
| Consultar migración actual | `python -m alembic current` |
| Ejecutar Finanzen | `python -m app.main` |
| Regenerar recursos Qt | `pyside6-rcc app/resources/resources.qrc -o app/generated/resources_rc.py` |
| Ver archivos modificados | `git status` |
| Revisar cambios | `git diff` |

---

## 13. Criterio final para el equipo

Antes de publicar un cambio en Finanzen, comprobar lo siguiente:

- [ ] El código respeta las responsabilidades de vistas, controladores, servicios y repositorios.
- [ ] La aplicación se inicia y los flujos afectados se probaron.
- [ ] Las dependencias y variables nuevas quedaron documentadas.
- [ ] Las migraciones se generaron y verificaron cuando hubo cambios de modelos.
- [ ] Los recursos Qt se regeneraron cuando fue necesario.
- [ ] No se incluyeron secretos, archivos locales ni datos privados.
- [ ] El commit explica un cambio concreto y no mezcla tareas inconexas.

**Fuente de referencia:** [Repositorio Finanzen — rama `main`](https://github.com/luis-hdz7/Finanzen). Como el proyecto sigue evolucionando, contrastar estas indicaciones con los archivos reales cuando cambie la arquitectura.
