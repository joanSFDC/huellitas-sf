# Paso manual RB-base-02: verificar o activar Data Protection and Privacy

## Id

RB-base-02

## HU

ADR-009. Situaciones que lo piden: f1-s2, f1-s3, f4-s2 y f4-s4. Es parte de `s2-base` (modelo de datos compartido).

## ¿Por qué no viaja por metadata?

El ajuste "Make data protection details available in records" no tiene un campo en la Metadata API: `PrivacySettings` (archivo `Privacy.settings`) solo expone otros parámetros de privacidad y consentimiento (captura de autorización, bloqueo y versionado, auditoría, archivado), y ninguno activa los objetos `Individual` y `ContactPointTypeConsent`. Se revisó la documentación el 2026-11-27.
- [PrivacySettings (Metadata API)](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_privacysettings.htm)
- [Set Up Tracking and Storage of Certain Data Privacy Preferences](https://help.salesforce.com/s/articleView?id=sf.individuals_setup_data_privacy.htm&type=5): "Data protection details are available in all your org’s records by default". Por eso lo más probable es que en la Org Core solo haya que verificarlo.
- [Individual: "disponible si Data Protection and Privacy está activo"](https://developer.salesforce.com/docs/atlas.en-us.object_reference.meta/object_reference/sforce_api_objects_individual.htm)
- [Objetos de consentimiento](https://help.salesforce.com/s/articleView?id=sf.consent_mgmt_fields.htm&language=en_US&type=5)

## Prerrequisitos

- Quien ejecuta tiene el permiso "Customize Application".
- Se ejecuta después de RB-base-01 (ADR-018).
- Va **antes de abrir el primer PR de la base (PR-A)**: el modelo usa `Contact.IndividualId`, `Lead.IndividualId` y los objetos `Individual`, `ContactPointTypeConsent` y `DataUsePurpose`.

## Org destino

Org Core (la base).

## Quién lo ejecuta

Edgar, con Joan o Duván registrando el paso.

## Pasos exactos

1. Setup > Quick Find "Data Protection and Privacy" > Data Protection and Privacy.
2. Si "Make data protection details available in records" ya está marcada (es el valor por defecto), no se cambia nada: anotar "ya estaba activa" en el estado.
3. Si no está marcada: Edit > marcar "Make data protection details available in records" > Save.

## Resultado esperado y evidencia

Existen los objetos `Individual`, `ContactPointTypeConsent` y `DataUsePurpose`, y en Contact aparece el campo `IndividualId`. Captura del Object Manager en `docs/runbook/img/`.

## Cómo se verifica

Joan o Duván, con el usuario administrador: buscan los tres objetos en Object Manager y confirman el campo `IndividualId` en Contact.

## Cómo se deshace

Se desmarca la misma casilla: la documentación oficial dice que se puede activar o desactivar desde Setup. Lo que no documenta es qué pasa con los registros de `Individual` y `ContactPointTypeConsent` existentes; una fuente no oficial indica que quedan guardados y vuelven a verse al reactivar. No desactivar después de cargar consentimientos: son la prueba ante la Ley 1581 (ADR-009) y nadie los borra.

## Estado y fecha

- [ ] Pendiente
- [ ] Ya estaba activa: AAAA-MM-DD, [nombre]
- [ ] Ejecutado: AAAA-MM-DD, [nombre]
- [ ] Verificado: AAAA-MM-DD, [nombre]
