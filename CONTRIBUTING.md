# Guía de Contribución - Huellitas SF

Querido Intern de Talent Inc Colombia:

Para mantener el historial de cambios limpio, organizado y compatible con nuestros flujos de despliegue, es obligatorio seguir las siguientes convenciones al nombrar ramas y redactar mensajes de commit.

---

## 1. Convención de Commits

Todos los commits deben seguir una estructura estandarizada basada en Conventional Commits, pero adaptada a nuestro contexto de Salesforce y frentes de trabajo.

**Formato obligatorio:**
`tipo(scope/frente): descripción corta en español`

### Desglose del formato:
* **tipo**: Define la intención del commit. Los más comunes son:
  * `feat`: Una nueva funcionalidad.
  * `fix`: Corrección de un error o bug.
  * `docs`: Cambios en la documentación.
  * `refactor`: Refactorización de código que no añade funcionalidades ni corrige errores.
  * `repo`: Cambios en la estructura o workflows del repositorio (de esto se encargan principalmente Duván y Joan).
* **scope**: Corresponde al tipo de metadato o componente de Salesforce que se está modificando (ej. `permission-set`, `flow`, `apex`, `lwc`, `object`, etc.) Si tienes alguna duda del scope de tu commit, escríbele a Joan o a Duván.
* **frente**: Identifica el equipo o frente de trabajo responsable del cambio. Los valores permitidos son: `f1`, `f2`, `f3`, `f4`.
* **mensaje**: Una descripción breve, en minúsculas y **en español** sobre lo que hace el commit.

### Ejemplos de Commits Válidos:
* `feat(flow/f1): agregar validación para el estado de adopción`
* `fix(permission/f3): corregir acceso de lectura al campo email`
* `feat(apex/f2): crear controlador para la tabla de mascotas`
* `docs(readme/f4): actualizar instrucciones de configuración local`

---

## 2. Convención de Ramas (Branches)

El nombre de las ramas debe seguir una lógica muy similar a la de los commits para que sea fácil identificar qué se está trabajando y quién lo está haciendo.

**Formato obligatorio:**
`tipo/scope/frente-descripcion-corta`

### Ejemplos de Ramas Válidas:
* `feat/flow/f1-validacion-adopcion`
* `fix/permission/f3-acceso-email`
* `feat/apex/f2-controlador-mascotas`

---

## 3. Uso y Versionado del Manifest (`package.xml`)

En Salesforce, el archivo `manifest/package.xml` actúa como un mapa que indica qué metadatos se van a descargar (retrieve) o subir (deploy) a una organización.

### ¿Cómo lo usamos localmente?
Para el desarrollo diario, puedes usar herramientas como **Salesforce Inspector** o las extensiones de VS Code para generar un `package.xml` temporal. Esto te servirá exclusivamente para hacer *retrieve* de tus cambios específicos (clases, flujos, campos) desde tu entorno de desarrollo (org) hacia tu máquina local.

### ¿Cómo funciona en nuestro Pipeline?
No utilizamos tu `package.xml` manual para hacer los despliegues a los entornos superiores. Nuestro flujo automatizado (CI/CD) utiliza la herramienta **SFDX Git Delta**. Cuando creas un Pull Request, esta herramienta analiza tus commits, los compara con la rama principal y **genera automáticamente un manifest virtual** que contiene *únicamente* los metadatos que tú modificaste.

### Regla de Versionado 🚫
**NO debes incluir el archivo `manifest/package.xml` en tus commits.** 
Como el pipeline genera dinámicamente el manifest que realmente importa para el despliegue, versionar tu manifest local solo provocará conflictos de *merge* innecesarios con los Pull Requests de otros compañeros que estén trabajando en otros frentes. 

*Nota: Asegúrate de que el archivo `manifest/package.xml` (o la carpeta `manifest/` si es exclusiva para esto) esté incluido en tu `.gitignore`.*

---

## 4. Sincronización de Cambios (`git pull`)

Cada vez que hagas un `git pull` para traer los últimos cambios de la rama principal (que refleja el estado de la Org Core), es fundamental recordar que **los cambios solo se descargan en tu repositorio local (tu computadora)**. Tu entorno de desarrollo de Salesforce (tu Org) no se actualiza automáticamente.

**👁️j👁️ (OJO) - Sincronización exclusiva en tu SDO:**
> Esta sincronización de metadatos se realiza **únicamente hacia la SDO de tu frente** (f1, f2, f3 o f4). Bajo ninguna circunstancia debes intentar desplegar desde tu entorno local hacia la Org Core (aunque por seguridad no tienes las credenciales de la Core, es vital mantener el orden de los entornos).

Para evitar confusiones o errores de despliegue, asegúrate de que tu proyecto local esté apuntando siempre a tu SDO antes de hacer un deploy:

* **Vía VS Code:** Revisa la barra inferior (footer) de tu editor. Ahí debe aparecer el alias de tu SDO. Si aparece otra organización o dice "No Default Org Set", haz clic ahí y selecciona la tuya.
* **Vía Terminal (CLI):** 
  * Para verificar a qué Org estás apuntando actualmente: `sf config get target-org`
  * Para establecer tu SDO como la predeterminada para todos los comandos: `sf config set target-org <alias-de-tu-sdo>`

**Pasos obligatorios después de un `git pull`:**

1. **Actualizar tu rama local:**
   Trae los cambios ejecutando `git pull` en tu terminal.
2. **Desplegar a tu Org de desarrollo:**
   Inmediatamente después de descargar los cambios, **debes** enviarlos a tu entorno de Salesforce. Esto garantiza que trabajes sobre la última versión y evita que, al hacer un nuevo desarrollo, sobrescribas o elimines accidentalmente el trabajo que tus compañeros acaban de subir.
   * **Vía VS Code:** Haz clic derecho sobre la carpeta `force-app` y selecciona **SFDX: Deploy Source to Org**.
   * **Vía Terminal (CLI):** Ejecuta el comando `sf project deploy start` (o `sfdx force:source:deploy -p force-app`).

> **Importante:** Si el despliegue falla después de un pull, notifícalo a tu frente (o a Duván/Joan). Podría faltar una configuración manual en tu Org (ej. habilitar una feature, crear un registro específico) necesaria para que el nuevo código funcione.

---

## 5. Flujo de Pull Requests (PRs) y Revisiones

La rama `main` está protegida. **El pipeline es la única vía a la organización Base: TODO cambio llega por PR**. Esto incluye metadatos de Salesforce, documentación, Runbooks y decisiones de arquitectura (ADRs). 

* **Revisiones:** Todos los PRs deben pasar por una revisión técnica a cargo de Joan y Duván. 
* **Aprobación y Merge:** En la mayoría de los casos, Edgar aprueba y fusiona los PRs en la rama principal (en dos ventanas diarias definidas).
* **Prohibición estricta:** Ningún desarrollador puede hacer cambios manuales directamente en el "Setup" de la Org Base. Todo se construye y prueba en las SDOs (Scratch/Developer Orgs) de cada pareja antes de enviar el PR.

---

## 6. Runbooks (Pasos Manuales)

Existen configuraciones o datos que no pueden viajar por metadatos a través del pipeline (por ejemplo, la creación de usuarios o asignación inicial de licencias). Todo lo que no viaje por metadata debe documentarse estrictamente como un paso manual en un **Runbook**.

### Reglas para los Runbooks:
1. **Viajan por PR:** Los Runbooks son archivos de texto (`.md`) que también deben someterse a revisión mediante un Pull Request.
2. **Ubicación:** Todos los pasos manuales deben guardarse en la carpeta `docs/runbook/` del repositorio.
3. **Nomenclatura del archivo:** Deben llamarse `RB-[frente]-[nn].md` (Ejemplo: `RB-f4-01.md`).
4. **Plantilla obligatoria:** Cada archivo de Runbook debe contener: Id de la HU relacionada, justificación de por qué no viaja por metadata (con enlace a la documentación oficial), prerrequisitos, pasos exactos, resultado esperado, validación y plan de rollback.
5. **Ejecución:** Estos pasos **NO** los ejecutas tú en la Org Base. Los ejecutarán Edgar, Joan y Duván en la Base justo después de que el release de tu HU haya sido desplegado.

---

## 7. Decisiones de Arquitectura (ADR) y Código Personalizado

Se premia y prioriza el uso de la funcionalidad estándar de Salesforce. Si tu historia de usuario requiere crear código personalizado (Apex o LWC), **NO puedes empezar a programar de inmediato**. 

Debes crear primero un registro de decisión (ADR) documentando por qué el estándar no es suficiente, enviarlo por PR y **esperar la aprobación de Edgar** antes de construir la solución técnica.

---

## 8. Recuperar Cambios Declarativos (Retrieve desde tu SDO)

Cada frente (f1, f2, f3, f4) trabaja de forma aislada en su propia SDO (Scratch/Developer Org). La rama `main` refleja el estado de la **Org Base (Core)**. 

Si creas o modificas algo directamente en el *Setup* de tu SDO usando la interfaz de Salesforce (por ejemplo: un Custom Object, un campo nuevo, un Flow o un Permission Set), Git no se enterará automáticamente. **Es tu responsabilidad extraer (retrieve) esa metadata a tu computadora** antes de hacer commit.

### Pasos para hacer Retrieve de tus cambios:

1. **Haz el cambio en tu SDO:** Construye y prueba la funcionalidad en la interfaz de Salesforce.
2. **Genera un Manifest temporal (Dos opciones):** 
   Un manifest (`package.xml`) le dice a VS Code exactamente qué descargar. Dado que creaste cosas en la SDO, necesitas armar este archivo indicando qué quieres recuperar. *(Recuerda la Regla 3: Este archivo es solo para tu uso local, ¡asegúrate de que esté en tu `.gitignore`!)*

   * **Opción A: Usando Salesforce Inspector (Extensión de navegador)**
     1. Ve a tu SDO en el navegador y abre **Salesforce Inspector** (o Inspector Reloaded).
     2. Haz clic en la opción **Download Metadata**.
     3. En la lista, busca y selecciona los tipos de componentes que modificaste (por ejemplo, `CustomObject`, `Flow`, `ApexClass`).
     4. La herramienta generará en pantalla un bloque de código XML con tus selecciones.
     5. Copia ese código XML.
     6. Ve a VS Code, crea (o abre) el archivo `manifest/package.xml` temporal y pega el código allí.

   * **Opción B: Usando SFDX Package.xml Generator (Extensión de VS Code)**
     1. Asegúrate de tener instalada la extensión **Salesforce Package.xml Generator** en VS Code.
     2. Abre la paleta de comandos (`Ctrl+Shift+P` en Windows o `Cmd+Shift+P` en Mac).
     3. Escribe y selecciona: **SFDX Package.xml Generator: Choose Metadata Components**.
     4. Se abrirá una interfaz visual dentro de VS Code que leerá tu SDO.
     5. Navega por las carpetas (ej. *CustomObject*, *Flow*) y marca únicamente las casillas de los metadatos exactos que tú creaste o modificaste.
     6. Haz clic en el botón **Update Package.xml**. La herramienta armará el archivo automáticamente por ti en tu carpeta `manifest/`.
3. **Descarga la Metadata (Retrieve):**
   * **Vía Manifest:** Haz clic derecho sobre tu `package.xml` temporal en VS Code y selecciona **SFDX: Retrieve Source in Manifest from Org**.
   * **Vía Org Browser (VS Code):** Ve al ícono de la nube en la barra lateral, busca el componente que creaste y haz clic en el ícono de descarga (Retrieve Source from Org).
   * **Vía Terminal (CLI):** Puedes pedir un componente específico directamente por consola:
     `sf project retrieve start -m CustomObject:Mascota__c`
4. **Verifica los archivos:** Asegúrate de que los archivos `.xml` (o las carpetas correspondientes) hayan aparecido dentro de tu carpeta `force-app/main/default/...`.
5. **Haz Commit y Push:** Ahora esos archivos XML forman parte de tu repositorio local y están listos para viajar en tu Pull Request hacia la Org Base.
