# Paso manual RB-base-01: activar Contacts to Multiple Accounts

## Id

RB-base-01

## HU

ADR-003. Situaciones que lo piden: f1-s3, f2-s1 y f3-s3. Es parte de `s2-base` (modelo de datos compartido).

## ¿Por qué no viaja por metadata?

El campo `enableRelateContactToMultipleAccounts` existe en `AccountSettings`, pero Salesforce indica no usar la Metadata API para activar esta función: puede quedar en un estado inconsistente.
- [AccountSettings (Metadata API)](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_accountsettings.htm)
- [Problema conocido al activarla por despliegue](https://help.salesforce.com/s/issue?id=a028c00000gAxBWAA0&language=en_US)

## Prerrequisitos

- Quien ejecuta tiene el permiso "Customize Application".
- Ningún otro paso previo. Este va primero (ADR-018).
- Va **antes de abrir el primer PR de la base (PR-A)**: "Validar PR" valida contra la Org Core y necesita el objeto `AccountContactRelation`.
- Orders, Entitlement Management y Messaging no tienen runbook: viajan por PR como ajustes en `force-app/main/default/settings/` (`Order.settings-meta.xml`, `Entitlement.settings-meta.xml` y `LiveMessage.settings-meta.xml`).

## Org destino

Org Core (la base).

## Quién lo ejecuta

Edgar, con Joan o Duván registrando el paso.

## Pasos exactos

1. Setup > Quick Find "Account Settings" > Account Settings > Edit.
2. En la sección "Contacts to Multiple Accounts", marcar "Allow users to relate a contact to multiple accounts".
3. Save.
4. Si la opción no aparece: abrir un caso con Salesforce Support con el texto "Feature requested: Enable Contacts with Multiple Accounts" y el Id de la org. [Guía oficial](https://help.salesforce.com/s/articleView?id=000381518&language=en_US&type=1).

## Resultado esperado y evidencia

En Setup > Object Manager existe el objeto `AccountContactRelation`, con el campo `Roles`. Captura de pantalla del Object Manager en `docs/runbook/img/`.

## Cómo se verifica

Joan o Duván, con el usuario administrador (todavía no existen usuarios de prueba): buscan `AccountContactRelation` en Object Manager y confirman el campo `Roles`.

## Cómo se deshace

Se desmarca la misma casilla. Salesforce borra las relaciones indirectas que existan en ese momento, así que solo se desactiva antes de cargar datos.

## Estado y fecha

- [ ] Pendiente
- [ ] Ejecutado: AAAA-MM-DD, [nombre]
- [ ] Verificado: AAAA-MM-DD, [nombre]
