# Configuración del pipeline

Repositorio: [joanSFDC/huellitas-sf](https://github.com/joanSFDC/huellitas-sf).

Este documento registra cómo quedó armado el motor de despliegue. El destino actual es la SDO del frente 4, no la org base. Cuando Edgar entregue la base, se repite la External Client App allá y se reemplazan los valores de los secretos. Los workflows no cambian.

## Qué hace el pipeline

Hay dos workflows.

| Archivo | Cuándo corre | Qué hace en la org |
|---|---|---|
| `.github/workflows/pr-main-branch.yml` | Al abrir, actualizar o reabrir un pull request hacia `main` | No escribe. Revisa el código y ejecuta `sf project deploy validate`. |
| `.github/workflows/deploy-main-branch.yml` | Al fusionar en `main`, solo si cambió algo dentro de `force-app/` | Escribe. Ejecuta `sf project deploy start` con el delta. |

Los dos usan el grupo de concurrencia `huellitas-org` con `cancel-in-progress: false`. Si llega otro pull request mientras uno está corriendo, espera. Una org de Salesforce solo admite una validación o un despliegue a la vez.

El despliegue no corre por cambios en `.github/`, documentación o `sfdx-project.json`. El pull request #1 solo trajo esos archivos, por eso `Validar` corrió y `Desplegar` no apareció en Actions. Eso es lo esperado.

## 1. Proyecto Salesforce

El repositorio se convirtió en un proyecto de Salesforce DX. Los archivos que entraron son:

- `sfdx-project.json`, con `sourceApiVersion` `68.0`, la versión de API de la SDO.
- `force-app/` como carpeta de metadata. En el arranque no tenía archivos, así que Git no la publicó.
- `config/project-scratch-def.json`.
- `.forceignore`.
- `.sgdignore`, que excluye `.github/` del cálculo del delta.

`.gitignore` ignora `server.key`, `server.csr` y `*.pem`. La llave privada no está en Git.

## 2. Org y usuario

La SDO se creó en Solutions Workspace y se conectó a la CLI con:

```bash
sf org login web \
  --alias sdo-f4 \
  --instance-url https://trailsignup-a9c03d8d4335ec.my.salesforce.com
```

La URL de My Domain es la que usa el pipeline. La URL de Lightning (`lightning.force.com`) la rechaza la CLI.

El usuario con el que el pipeline entra es `jorduz@idevops.com`. Joan lo creó como usuario de integración. La licencia prevista es Salesforce Integration y el perfil preautorizado en la App es Salesforce API Only System Integrations. Ese perfil se confirmó en la pantalla de la App. La licencia no se pudo leer por consulta.

## 3. Certificado

La llave y el certificado se generaron fuera del repositorio, en `~/huellitas-jwt/`:

```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=huellitas-github-actions"
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt
chmod 600 server.key
```

`server.crt` se sube a la External Client App. `server.key` se queda en la máquina y en el secreto `ORG_JWT_KEY`. No se cifró ni se guardó una copia dentro del repositorio.

## 4. External Client App

En la SDO hay una sola External Client App.

| Campo | Valor |
|---|---|
| Nombre | DevOps External App |
| Estado | Local, habilitada |
| Contacto | jorduz@salesforce.com |
| Descripción | Huellitas DevOps External App for GitHub |
| Usuarios permitidos | Admin approved users are pre-authorized |
| Perfil preautorizado | Salesforce API Only System Integrations |
| Permission sets preautorizados | Ninguno |
| IP | Bypass. No hay rangos cargados |
| Página de inicio | Ninguna |
| Refresh token | Vence por inactividad a los 720 días |

Scopes activos: API, web, full, refresh token, OpenID, SSO, Salesforce APIs, Interaction, Data Cloud y Einstein GPT. Offline access está apagado.

El Consumer Key se copia en Settings, OAuth Settings, Consumer Key and Secret. Salesforce pide un código de verificación por correo. Ese valor es el secreto `ORG_CLIENT_ID`. No se guarda el Consumer Secret: el JWT no lo usa.

La prueba local, antes de crear los secretos, fue:

```bash
sf org login jwt \
  --client-id "CONSUMER_KEY" \
  --jwt-key-file ~/huellitas-jwt/server.key \
  --username "jorduz@idevops.com" \
  --instance-url "https://trailsignup-a9c03d8d4335ec.my.salesforce.com" \
  --alias sdo-f4-jwt
```

El resultado fue `Successfully authorized jorduz@idevops.com` en la org `00Dg800000U88TlEAJ`, la misma de `sdo-f4`.

## 5. Secretos de GitHub

Están en el repositorio, en Settings, Secrets and variables, Actions. No están en un Environment. Los cuatro nombres son los que leen los workflows:

| Secreto | Contenido |
|---|---|
| `ORG_CLIENT_ID` | Consumer Key de DevOps External App |
| `ORG_USERNAME` | `jorduz@idevops.com` |
| `ORG_INSTANCE_URL` | `https://trailsignup-a9c03d8d4335ec.my.salesforce.com` |
| `ORG_JWT_KEY` | Contenido completo de `server.key`, incluidas las líneas BEGIN y END |

Comandos usados:

```bash
gh secret set ORG_JWT_KEY --repo joanSFDC/huellitas-sf < ~/huellitas-jwt/server.key
gh secret set ORG_CLIENT_ID --repo joanSFDC/huellitas-sf --body "CONSUMER_KEY"
gh secret set ORG_USERNAME --repo joanSFDC/huellitas-sf --body "jorduz@idevops.com"
gh secret set ORG_INSTANCE_URL --repo joanSFDC/huellitas-sf --body "https://trailsignup-a9c03d8d4335ec.my.salesforce.com"
gh secret delete SERVER_KEY --repo joanSFDC/huellitas-sf
```

`SERVER_KEY` se había creado con otro nombre y ningún workflow lo leía. Se borró.

En el workflow la llave se escribe en un archivo temporal del runner, con permiso `600`. No se imprime en el log. El runner entra con:

```bash
sf org login jwt \
  --client-id "$ORG_CLIENT_ID" \
  --jwt-key-file "$RUNNER_TEMP/jwt/server.key" \
  --username "$ORG_USERNAME" \
  --instance-url "$ORG_INSTANCE_URL" \
  --alias org \
  --set-default
```

El alias `org` es el de la organización dentro del runner, no el nombre de una rama.

## 6. Workflow de validación

Archivo: `.github/workflows/pr-main-branch.yml`. El job se llama `Validar`. Ese es el nombre del check en el pull request.

Pasos:

1. Baja el repositorio con historial completo (`fetch-depth: 0`).
2. Instala Node.js 24.
3. Instala la Salesforce CLI, `sfdx-git-delta` y Code Analyzer.
4. Entra a la org con JWT.
5. Corre `sf code-analyzer run` sobre `force-app`. Si la carpeta no tiene archivos, el paso avisa y sigue.
6. Calcula el delta entre `HEAD` y `origin/main`, ignorando `.github/` por `.sgdignore`.
7. Si el `package.xml` tiene `<types>`, ejecuta `sf project deploy validate` con `--test-level NoTestRun`.
8. Si además hay `destructiveChanges.xml` con `<types>`, la validación incluye `--post-destructive-changes`.
9. Si el delta no trae metadata, termina sin llamar a Salesforce.

`deploy validate` hace el chequeo de un despliegue y lo revierte. La org queda igual.

## 7. Workflow de despliegue

Archivo: `.github/workflows/deploy-main-branch.yml`. El job se llama `Desplegar`.

Se dispara con un push a `main` que cambie `force-app/**`. Un pull request cerrado sin fusionar no lo dispara.

Los pasos de Node, CLI, JWT y delta son los mismos. La diferencia es el comando final: `sf project deploy start`. Compara contra el commit anterior de `main` (`github.event.before`), no contra `HEAD~1`, para no perder archivos cuando el merge trae varios commits.

No activa Flows. Si un Flow viaja en metadata, tiene que ir en estado Draft. La activación en la base queda para el paso `RB-base-flows`, que ejecuta Edgar después del release.

## 8. Prueba hecha

El esqueleto se subió en la rama `repo/pipeline-pr-deploy` y se fusionó en el [pull request #1](https://github.com/joanSFDC/huellitas-sf/pull/1) el 20 de noviembre de 2026.

`Validar` terminó en verde. Comprobó Node 24, la lectura de los cuatro secretos y el login JWT. No había metadata en `force-app`, así que no validó un despliegue real. `Desplegar` no se ejecutó, porque el merge no tocó `force-app`.

## 9. Lo que falta

- Proteger `main`: exigir pull request y el check `Validar` antes de fusionar. El check ya corrió una vez, así que el nombre ya existe.
- Un pull request que agregue metadata, por ejemplo la etiqueta `Pipeline_Prueba`, para ver `Validar` con un delta real y, después del merge, `Desplegar` en verde.
- Confirmar en Setup que `jorduz@idevops.com` tiene licencia Salesforce Integration y los permisos de usuario de Flow y de Service Cloud.
- Cuando exista la org base, crear allí la misma External Client App con el mismo `server.crt` y actualizar `ORG_CLIENT_ID`, `ORG_USERNAME` y `ORG_INSTANCE_URL`.
- Cambiar `NoTestRun` a `RunSpecifiedTests` cuando entre la primera clase Apex. Cada clase nueva necesita al menos 75 % de cobertura.
