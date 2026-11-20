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
