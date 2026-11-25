# HU - F3 APP BUILDER

PM - Juan Florez
Equipo: @Juan Diego Velásquez - @Juan Florez

Alineada con el diccionario del modelo de datos (v2) y con el registro de decisiones de arquitectura (ADR-000 a ADR-049, 23 nov 2026).

---

## f3-s1 · Contestar el recordatorio de la cita

### Ficha

| Campo | Valor |
|---|---|
| Estado | Prototipada |
| Importancia | [POR DEFINIR] (el expediente no la define) |
| Esfuerzo | 5 SP / L (estimación propuesta por el PM, pendiente de validar con el equipo) |
| Prioridad de ejecución | 7 de 17 (propuesta; orden global de las 17 situaciones) |
| Depende de | f2-s1, f2-s2, f4-s4 |
| Habilita a | f3-s4 |

### HU funcional

Carolina Méndez. La recepción ve la cita confirmada o cambiada.

"Como dueña de una mascota con cita en Huellitas, quiero que me recuerden la cita el día antes por WhatsApp y poder confirmarla o cambiarla ahí mismo, para no tener que llamar ni perder la cita cuando no puedo ir"

**Como** dueña de una mascota con una cita programada,
**Quiero** recibir un recordatorio automático por WhatsApp el día anterior y poder confirmar o cambiar la fecha desde el mismo chat,
**Para** no perder mi turno ni tener que llamar a la sede.

**Prototipo:** [HUELLITAS-PANTALLAS en Figma](https://www.figma.com/design/PcTqknolBVhIw2h0QnbFap/HUELLITAS-PANTALLAS?node-id=5-1385) (8 WhatsApp + 4 consolas + vacío/cargando/error, terminado en la semana 1).

**Se cumple cuando**
1. El día antes, Carolina recibe el mensaje de Huellitas sin haber escrito ella primero.
2. Si contesta que no puede ir, le ofrecen otros horarios.
3. Escoge uno y la cita cambia en la agenda.
4. Si contesta que sí va, la cita queda confirmada para la recepción.

**Condiciones**
- Huellitas escribe primero, así que el mensaje sale con plantilla de WhatsApp aprobada por Meta. El chat web no sirve de reemplazo.
- Sin Salesforce Scheduler: las citas funcionan con lo ya contratado.
- Las sedes abren de lunes a sábado, de 7 de la mañana a 7 de la noche.
- La mueve la meta de bajar la inasistencia del 18 % a menos del 10 %.

**Preguntas abiertas**
- ¿A qué hora sale el recordatorio? Supuesto: dentro del horario de la sede.
- ¿Qué pasa si no contesta? Supuesto: la cita sigue sin confirmar.
- ¿Entra también el recordatorio de vacuna? Supuesto: solo la cita.
- ¿Quien pidió no recibir mensajes deja de recibir el recordatorio? Ver f4-s4.

**Criterios de aceptación (medibles y alcanzables)**
- **CA1 (Envío proactivo):** Dado que la cita está en estado "Programada" para el día siguiente, cuando inicie el horario laboral de la sede, el sistema debe enviar un mensaje de plantilla a través de WhatsApp automáticamente.
- **CA2 (Confirmación exitosa):** Dado que la clienta recibe el mensaje, cuando responda afirmativamente, el estado de la cita debe cambiar a "Confirmada" y la recepción lo verá reflejado al refrescar la vista o la lista de la agenda.
- **CA3 (Reprogramación inteligente):** Dado que la clienta responde que no puede asistir, cuando solicite cambiarla, el agente debe ofrecer primero los horarios libres de ese mismo día; si no hay disponibilidad, ofrecerá las tres opciones más próximas disponibles.

### HU técnica

**Componentes**

*Objetos y campos*
- `Cita__c` (de f2): `Estado__c` (Programada, Confirmada, Atendida, Cancelada, No asistió), `Tipo__c`, `Inicio__c`, `Fin__c`, `Cita_Origen__c` y los Lookups `Mascota__c`, `Contacto__c` y `Sede__c`, más el Lookup al veterinario (`VeterinarioId` en el diccionario; nombre de API por confirmar con f2).
- `Sede__c`: `Hora_Apertura__c`, `Hora_Cierre__c` y, de f2, `Dias_Abiertos__c` y `Capacidad_Simultanea__c`.
- `Tipo_Cita__mdt` (de f2): duración de la cita por tipo.
- `MessagingEndUser` y `MessagingSession`.
- `ContactPointTypeConsent` del propósito Servicio Clínico (lo crea f4, RB-f4-05).

*Flows y acciones*
- `FL_Recordatorio_Cita` (programado, al inicio del horario de la sede): citas del día siguiente en Programada, envía la plantilla `recordatorio_cita`.
- Acciones: Confirmar cita, Ofrecer horarios, Mover cita. Las dos últimas reutilizan el subflow de disponibilidad y `Cita_Mover` de f2.

*Canal:* WhatsApp.

**Configuración**
- Huecos = `Hora_Apertura__c` – `Hora_Cierre__c` de la sede en los `Dias_Abiertos__c`, con su `Capacidad_Simultanea__c`, menos las citas Programadas o Confirmadas. La duración sale del tipo de la cita (`Tipo_Cita__mdt`).
- Reprogramación: primero los huecos del mismo día; si no hay, las tres opciones más próximas.
- Mover crea una cita nueva con `Cita_Origen__c` y deja la anterior Cancelada con motivo Movida. El mensaje de confirmación da el número de la cita nueva.
- Envío al inicio del horario de la sede. Sin respuesta, la cita sigue. Solo citas, no vacunas.
- Quien no quiere promociones sí recibe este mensaje (es de utilidad): el Flow lee el consentimiento de Servicio Clínico, no el de Promociones.

**Seguridad**
- Cambia la cita quien escribe desde el número de un `Contact` del hogar de la mascota.
- Agente: lee `Contact`, `Mascota__c` y `Cita__c`; solo edita `Estado__c`, `Inicio__c` y `Fin__c`; no borra.
- Recepción de la sede ve el resultado al refrescar. Veterinario: lectura de su agenda. Cliente: sin acceso a Salesforce.

**Datos de prueba**
- QA-Hogar Carolina, Carolina Méndez, Tomate, cita mañana en Chapinero.
- Casos: confirma, reprograma, no contesta, cita ya movida.
- Carolina por WhatsApp y Yulieth en consola. SDO de la pareja (S2–S3) y base (S4).
- Testing Center: la respuesta sí; el envío no.

**Dependencias**
- f2 (dueño de `Cita__c`, del subflow de disponibilidad y de `Cita_Mover`).
- Plantilla `recordatorio_cita` aprobada por Meta (s1-plantillas).
- Enrutamiento por el Agente de Entrada (Opción B, Omni-Channel) con f1.
- f4-s4: el propósito Servicio Clínico en `DataUsePurpose`.

**Despliegue**
- Por metadata (PR): Flow, acciones y los campos de `Cita__c` que acepte f2.
- A mano: plantilla de WhatsApp. El Flow va en RB-base-flows.

**Autor:** Juan Diego Velásquez con Juan Florez.

**ADR que la respaldan:** 005, 028, 037, 040, 044.

---

## f3-s2 · Registrar una mascota nueva

### Ficha

| Campo | Valor |
|---|---|
| Estado | Prototipada |
| Importancia | [POR DEFINIR] (el expediente no la define) |
| Esfuerzo | 3 SP / M (estimación propuesta por el PM, pendiente de validar con el equipo) |
| Prioridad de ejecución | 5 de 17 (propuesta; orden global de las 17 situaciones) |
| Depende de | s2-base |
| Habilita a | f3-s3, f4-s1 |

### HU funcional

El cliente (referencia: Carolina). Sebastián Ortiz la ve en cualquier sede.

"Como cliente de Huellitas que acaba de adoptar una mascota, quiero registrarla por WhatsApp, para que cualquier sede la tenga cuando la lleve, sin volver a dar sus datos"

**Como** cliente de Huellitas que acaba de adoptar una mascota,
**Quiero** registrarla por WhatsApp dando sus datos básicos,
**Para** que la clínica la tenga en su sistema unificado sin importar a qué sede vaya.

**Prototipo:** pantalla pendiente (Tatiana la entrega en la semana 2).

**Se cumple cuando**
1. Queda registrada con nombre, especie, raza y fecha de nacimiento aproximada.
2. Queda a nombre del cliente que escribió.
3. El veterinario la ve en cualquier sede.

**Condiciones**
- Hoy la mascota queda dos veces si la familia cambia de sede. La HU pide un solo registro.
- El plan lo paga la mamá, a la cita viene el hijo y escribe la hija. Cómo se modela el hogar lo decide la pareja con un ADR.
- Todo dato es ficticio: no se migran las 9.800 mascotas.

**Preguntas abiertas**
- ¿Y si quien escribe aún no es cliente? Supuesto: se registra la persona con su celular y luego la mascota.
- ¿Se pide la sede? Supuesto: no.
- ¿Raza desconocida? Supuesto: un valor como Mestizo es válido.
- ¿Se registran las otras personas del hogar aquí? Supuesto: no.

**Criterios de aceptación**
- **CA1 (Registro único):** Dado que la dueña escribe por WhatsApp, cuando ingresa especie, raza y fecha de nacimiento, el sistema crea un solo registro en `Mascota__c`, validando que no exista otra con el mismo nombre y especie en ese hogar.
- **CA2 (Creación completa del cliente):** Dado que el número de WhatsApp no existe en la base, cuando la persona escribe para registrar la mascota, el sistema crea primero su `Account` (tipo Hogar), luego su `Contact` con el celular asociado, y finalmente vincula la mascota a ambos.

### HU técnica

**Componentes**

*Objetos y campos*
- `Account` (record type Hogar), `Contact` (`MobilePhone`, `AccountId`), `AccountContactRelation` (el rol en el hogar va en `Roles`: Titular, Pagador, Cuidador…), `MessagingEndUser`.
- `Mascota__c` (se crea): `Name`, `Hogar__c` (Lookup a `Account`, obligatorio, borrado restringido), `Titular__c` (Lookup a `Contact`, informativo), `Especie__c` (Perro, Gato, Otra), `Raza__c`, `Fecha_Nacimiento__c`, `Sexo__c`, `Estado__c`.

*Flows y acciones*
- `FL_Mascota_Registrar` (autolaunched): busca a la persona por el celular normalizado (+57 y diez dígitos). Si el número no existe, crea el `Account` Hogar, el `Contact` y la mascota; si existe, la crea en su hogar. Mismo nombre y especie en el hogar: avisa y no duplica.
- Acción: Registrar mascota.

*Canal:* WhatsApp.

**Configuración**
- No se pide sede. Raza vacía o «Mestizo» es válida.
- No registra a las otras personas del hogar.

**Seguridad**
- `Mascota__c`: Public Read Only. Edición por permission set.
- Agente: crea, no borra. Cliente: sin acceso a Salesforce.
- Sebastián lee la mascota en cualquier sede.

**Datos de prueba**
- QA-Hogar Carolina. Gatita nueva.
- Casos: sin raza, Mestizo, duplicada en el hogar, persona que no es cliente.
- Carolina por WhatsApp y Sebastián en otra sede. SDO (S2–S3) y base (S4).
- Testing Center: sí.

**Dependencias**
- Modelo compartido de la S2 (`Account` Hogar y `Contact`).
- Frente 1, si quien escribe aún no es cliente (`Lead`): falta acordar qué pasa con un Lead abierto del mismo celular.
- Cada `Contact` debe tener su `Individual` para Data 360 y el consentimiento: falta decidir si este Flow lo crea y si pide el aviso de datos.

**Despliegue**
- Por metadata (PR): objeto `Mascota__c`, campos, Flow, permission set y acción.
- A mano: ninguno.

**Autor:** Juan Diego Velásquez con Juan Florez.

**ADR que la respaldan:** 001, 003, 019, 026, 041.

---

## f3-s3 · ¿Qué vacunas le faltan?

### Ficha

| Campo | Valor |
|---|---|
| Estado | Prototipada |
| Importancia | [POR DEFINIR] (el expediente no la define) |
| Esfuerzo | 5 SP / L (estimación propuesta por el PM, pendiente de validar con el equipo) |
| Prioridad de ejecución | 11 de 17 (propuesta; orden global de las 17 situaciones) |
| Depende de | f3-s2 |
| Habilita a | — |

### HU funcional

La dueña de Lola. Sebastián quiere ver las vacunas de otra sede.

"Como dueña de una mascota, quiero preguntar por WhatsApp qué vacunas le faltan, para saberlo en el momento, sin que la recepción busque el Excel de cada sede"

**Como** dueña de una mascota,
**Quiero** preguntar por WhatsApp qué vacunas le faltan y cuáles tiene aplicadas,
**Para** saber las fechas exactas sin que la recepción deba buscar en los Excel de cada sede.

**Prototipo:** pantalla pendiente (Tatiana la entrega en la semana 2).

**Se cumple cuando**
1. Recibe las vacunas aplicadas, con fecha y sede.
2. Recibe las que faltan, con la fecha en que le tocan.
3. Funciona aunque se haya vacunado en dos sedes.
4. Si quien pregunta no es la dueña, no recibe esa información.

**Condiciones**
- Hoy cualquiera que pregunte por Lola recibe su información. Eso no puede seguir, y tampoco se pide un formulario cada vez.
- Testing Center no simula un código de verificación ni pasar a una persona: esa parte se prueba a mano en el QA cruzado.

**Preguntas abiertas**
- ¿Cuál es el calendario de vacunas por especie y edad? Sin eso no hay fecha. Supuesto: un esquema de ejemplo que valide Edgar y que la administradora pueda cambiar.
- ¿Quién cuenta como dueña cuando escribe la hija y paga la mamá?
- ¿Cómo se reconoce a quien escribe sin un formulario?

**Criterios de aceptación**
- **CA1 (Seguridad por hogar):** Dado que alguien consulta el esquema de una mascota, el sistema validará la identidad y solo responderá si el número de WhatsApp pertenece a cualquier `Contact` vinculado al `Account` del hogar de esa mascota.
- **CA2 (Cálculo dinámico):** Dado que se consulta el estado de vacunación, el sistema debe calcular la próxima dosis pendiente en tiempo real usando el calendario del producto (`Product2`: nacimiento + primera dosis, o última aplicación + meses de refuerzo), dejando el campo de fecha en `Vacuna_Aplicada__c` solo como un dato histórico.

### HU técnica

**Componentes**

*Objetos y campos*
- `Vacuna_Aplicada__c` (se crea): Master-Detail a `Mascota__c`, Lookup a `Product2` (la vacuna) y Lookup a `Sede__c`. `Fecha_Aplicacion__c`, `Proxima_Dosis__c` (solo histórico), `Lote__c`.
- `Product2` (familia Vacuna): se agregan `Especie__c`, `Edad_Primera_Dosis_Semanas__c` y `Refuerzo_Cada_Meses__c`.

*Flows y acciones*
- `FL_Vacunas_Consultar`: cruza el número del `MessagingEndUser` / `MessagingSession`, normalizado a +57, con los `Contact` del hogar. Devuelve las aplicadas y calcula las pendientes.
- Acción: Consultar vacunas.

*Canal:* WhatsApp, sin formulario.

**Configuración**
- Aplicadas: `Fecha_Aplicacion__c` y sede, de todas las sedes.
- Pendiente sin aplicación: `Fecha_Nacimiento__c` + `Edad_Primera_Dosis_Semanas__c`.
- Refuerzo: última `Fecha_Aplicacion__c` + `Refuerzo_Cada_Meses__c`.
- Natalia edita especie, primera dosis y refuerzo en el producto.
- Varias mascotas: el agente pregunta cuál.
- Si el número corresponde a más de un perfil, el agente pide otro dato y no responde.

**Seguridad**
- `Vacuna_Aplicada__c` hereda la compartición de la mascota por Master-Detail.
- Responde si el número es de cualquier `Contact` del hogar de la mascota. Otro número: sin datos.
- Veterinario: crea. Recepción: lectura. Agente: lectura.

**Datos de prueba**
- QA-Hogar Lola. Carolina y la hija como contactos del hogar. Un número fuera del hogar.
- Lola con una vacuna en Chía y otra en Chapinero.
- Casos: contacto del hogar recibe fechas y sedes; el otro número no; mascota sin vacunas; dos mascotas.
- Testing Center: la respuesta sí. La verificación del número va a mano en el QA cruzado.

**Dependencias**
- f3-s2.
- Esquema de vacunas de ejemplo (Edgar).
- `Vacuna_Aplicada__c` Master-Detail a `Mascota__c` en el modelo oficial.

**Despliegue**
- Por metadata (PR): `Vacuna_Aplicada__c`, campos de `Product2`, Flow y acción.
- A mano: carga del esquema de vacunas (dato), en un paso RB del frente 3. Las vacunas se buscan por `ProductCode`, no por Id.

**Autor:** Juan Diego Velásquez con Juan Florez.

**ADR que la respaldan:** 008, 011, 017, 023, 026.

---

## f3-s4 · Contar lo que pasa antes de la consulta

### Ficha

| Campo | Valor |
|---|---|
| Estado | Prototipada |
| Importancia | [POR DEFINIR] (el expediente no la define) |
| Esfuerzo | 3 SP / M (estimación propuesta por el PM, pendiente de validar con el equipo) |
| Prioridad de ejecución | 12 de 17 (propuesta; orden global de las 17 situaciones) |
| Depende de | f3-s1 |
| Habilita a | — |

### HU funcional

El cliente. Sebastián lee las respuestas; Natalia cambia las preguntas.

"Como dueño de un perro con consulta, quiero contar antes de la cita qué le pasa, para que la consulta rinda y saber con tiempo si debe ir en ayunas"

**Como** dueño de un paciente con una consulta o cirugía programada,
**Quiero** responder un cuestionario previo por WhatsApp sobre los síntomas y recibir instrucciones precisas,
**Para** agilizar la atención del veterinario y saber con tiempo si debo llevarlo en ayunas.

**Prototipo:** [HUELLITAS-PANTALLAS en Figma](https://www.figma.com/design/PcTqknolBVhIw2h0QnbFap/HUELLITAS-PANTALLAS?node-id=5-1385) (4 WhatsApp + consola del veterinario, terminado en la semana 1).

**Se cumple cuando**
1. Dos días antes recibe las preguntas: qué le pasa, desde cuándo, qué ha comido y qué medicamentos le dieron.
2. El veterinario ve las respuestas al abrir la cita.
3. El cliente sabe con tiempo si debe ir en ayunas.

**Condiciones**
- Huellitas escribe primero, con la plantilla del cuestionario previo.
- Natalia o Daniel tienen que poder agregar una pregunta sin programador.
- Sin Feedback Management ni sus encuestas.

**Preguntas abiertas**
- ¿Qué consultas piden ayuno y cuántas horas? Supuesto: la regla la da Edgar y la cambia la administradora.
- ¿Va a toda cita o solo a consultas? Supuesto: consultas y cirugías.
- ¿Qué pasa si la cita se agenda con menos de dos días, o si no contesta?

**Criterios de aceptación**
- **CA1 (Envío dinámico):** Dado que la cita (Consulta o Cirugía) está programada, cuando falten exactamente 2 días, **o de inmediato si la cita se creó con menos de 48 horas de anticipación**, el sistema envía la plantilla de pre-consulta automáticamente. Esto exige que el flujo se dispare tanto de forma programada diaria como al momento de crear el registro.
- **CA2 (Instrucciones de ayuno):** Dado que la cita tiene marcado `Ayuno__c`, cuando el cliente responda las preguntas, el agente le indica que debe ir en ayunas y cuántas horas, con el texto estándar que valida Edgar.

### HU técnica

**Componentes**

*Objetos y campos*
- `Pregunta_Preconsulta__c` (se crea): `Name`, `Orden__c`, `Vigente__c`, `Dispara_Ayuno__c`.
- `Cita__c` (de f2): se agregan `Respuestas_Preconsulta__c` (Long Text) y `Estado_Preconsulta__c` (Pendiente, Respondida, Sin respuesta). Usa `Tipo__c` y el checkbox `Ayuno__c`.

*Flows y acciones*
- `FL_Preconsulta_Enviar`: programado (dos días antes) y al crear la cita (de inmediato si faltan menos de 48 h). Filtra por `Tipo__c` (Consulta o Cirugía).
- El agente recorre las preguntas con `Vigente__c`, guarda el texto en `Respuestas_Preconsulta__c` y marca `Estado_Preconsulta__c`.
- Lightning page de `Cita__c`: respuestas arriba.

*Canal:* WhatsApp, plantilla. Sin Feedback Management.

**Configuración**
- Pregunta nueva = registro con `Vigente__c`.
- El ayuno se indica en el mensaje cuando `Cita__c.Ayuno__c` está marcado; las horas van en la plantilla estándar.
- Los dos disparadores no pueden enviar dos veces la misma preconsulta.
- Quien no quiere promociones sí recibe este mensaje: el Flow lee el consentimiento de Servicio Clínico.

**Seguridad**
- Sebastián: lee respuestas de las citas de su sede.
- Natalia y Daniel: editan `Pregunta_Preconsulta__c`.
- Agente: escribe respuestas y estado. No borra la cita.

**Datos de prueba**
- Consulta en dos días y una cirugía, Chapinero, QA-Hogar Carolina.
- Casos: llega la plantilla; cita creada con menos de 48 h; Sebastián ve las respuestas; pregunta nueva sin programador.
- Cliente por WhatsApp, Sebastián y Natalia en consola. SDO (S2–S3) y base (S4).
- Testing Center: preguntas y respuestas sí. El envío no.

**Dependencias**
- f3-s1 (`Tipo__c` y plantillas).
- f2, dueño de `Cita__c`.
- Edgar: valores de ayuno y esquema.

**Despliegue**
- Por metadata (PR): objeto `Pregunta_Preconsulta__c`, campos de `Cita__c`, Flow, página y acción.
- A mano: plantilla `preconsulta` y carga de preguntas. El Flow va en RB-base-flows.

**Autor:** Juan Diego Velásquez con Juan Florez.

**ADR que la respaldan:** 005, 017, 028, 040, 044.

---

## Decisiones seguras, según el registro de decisiones

- `Account` es el hogar o la empresa. `Contact` es la persona. `Mascota__c` tiene Lookup al hogar y es Public Read Only: el veterinario la ve en cualquier sede (ADR-001, ADR-022).
- `Cita__c` reemplaza a Field Service y a Salesforce Scheduler. `Vacuna_Aplicada__c` es Master-Detail de `Mascota__c`, con Lookup a `Product2` (familia Vacuna) y a `Sede__c` (ADR-005, ADR-011).
- La sede es `Sede__c` (el modelo oficial no usa `Location`). Los huecos libres salen de `Hora_Apertura__c`, `Hora_Cierre__c`, `Dias_Abiertos__c` y `Capacidad_Simultanea__c` (ADR-005, ADR-010).
- `Pregunta_Preconsulta__c` es el catálogo de preconsulta. Las respuestas van en `Cita__c.Respuestas_Preconsulta__c` (ADR-005).
- WhatsApp usa `MessagingEndUser` y `MessagingSession`. El enrutamiento lo hace el Agente de Entrada (Opción B) (ADR-013, ADR-025).
- El opt-out de promociones usa `Individual`, `DataUsePurpose` y `ContactPointTypeConsent`; no apaga el recordatorio ni la preconsulta (ADR-009, ADR-044).

## Implementaciones nuevas propuestas

- Objetos que creamos: `Mascota__c`, `Vacuna_Aplicada__c` y `Pregunta_Preconsulta__c`.
- Campos que agregamos al modelo oficial: en `Product2`, `Especie__c`, `Edad_Primera_Dosis_Semanas__c` y `Refuerzo_Cada_Meses__c`; en `Cita__c`, `Respuestas_Preconsulta__c` y `Estado_Preconsulta__c`.
- Flows: `FL_Recordatorio_Cita`, `FL_Mascota_Registrar`, `FL_Vacunas_Consultar` y `FL_Preconsulta_Enviar`.
- Huecos calculados al momento desde `Sede__c`, reutilizando los campos y el subflow de disponibilidad de f2.
- La próxima dosis se calcula desde el calendario de `Product2`; `Proxima_Dosis__c` queda solo como histórico.
- Acceso a vacunas: cualquier `Contact` del hogar de la mascota, por el teléfono registrado. `Titular__c` queda solo como dato informativo.

## Posibles conflictos

- `Cita__c` es de f2. El registro de decisiones propone los estados Programada, Confirmada, Atendida, Cancelada y No asistió, el campo `Tipo__c` y los campos de preconsulta (ADR-005, conflicto 7). Falta que f2 lo confirme.
- Huecos libres: f2-s1, f2-s2 y f3-s1 usan un solo subflow de disponibilidad (ADR-005). No se duplica la lógica.
- Consentimiento: resuelto en ADR-044. La baja solo apaga Promociones y no se marca el opt-out de `MessagingEndUser`, así que el recordatorio y la preconsulta siguen.
- Camino de WhatsApp: canal y plantillas pendientes (paquete BYOC o EDO), por definir con Juan Florez.
- Quien no es cliente: esta HU crea `Contact` y `Account` Hogar; el frente 1 usa `Lead`. ADR-041 propone Lead para quien cotiza y hogar para quien registra una mascota. Falta acordar con f1 qué pasa con un Lead abierto del mismo celular (conflicto 5).
- `Individual` del `Contact` que crea f3-s2: ADR-019 pide uno por cada `Contact`, y ADR-009 trata a quien no tiene consentimiento como sin autorización, lo que también afecta al recordatorio y a la preconsulta.

## Notas de diseño del frente 3

**Horarios libres (f3-s1).** Un Flow los calcula al momento con `Hora_Apertura__c`, `Hora_Cierre__c` y `Dias_Abiertos__c` de `Sede__c`, menos las citas activas. Reutiliza el subflow de disponibilidad de f2, así cada cita puede durar distinto.

**Quién ve los datos de la mascota (f3-s3).** Accede cualquier `Contact` del hogar ligado a la mascota, identificado por su teléfono registrado. El campo `Titular__c` se conserva solo como dato informativo, no como filtro de seguridad.

---

## Tareas y pipeline

### Tareas semana 1

**Instrucciones para Operación Clínica (Frente 3 - Juan Diego)**

Su objetivo es dejar la SDO lista y los permisos de Meta en trámite para no tener bloqueos cuando comience el desarrollo.

- **Checklist SDO F3:** Debe completar y documentar la habilitación de su SDO con capturas de pantalla, confirmando Agentforce, Digital Engagement con WhatsApp y Testing Center.
- **Aprobación de plantillas (crítico):** Debe crear y enviar a aprobación de Meta las plantillas de WhatsApp para el recordatorio automático (f3-s1) y el cuestionario pre-consulta (f3-s4). Esto puede tardar y debe quedar en trámite ya mismo.
- **Canal activo:** Debe registrar el número de WhatsApp de la SDO de su frente y dejarlo activo o en trámite en Setup.

### Pipeline: tareas frente 3 (Operación Clínica - Juan Diego y equipo)

- **Construcción del modelo y seguridad:** Juan Diego debe iniciar la creación de los objetos personalizados `Mascota__c`, `Vacuna_Aplicada__c` y `Cita__c` en su SDO. Debe configurar los niveles de acceso (OWD) para que el veterinario (Sebastián) pueda leer el historial clínico multisede sin restricciones.
- **Agente de Entrada (enrutamiento A2A):** Al haberse decidido la Opción B, Juan Diego debe coordinarse con Juan Enrique (F1) para construir el esqueleto de este Agente de Entrada en una de sus SDOs. Este agente inicial recibirá la intención y usará Omni-Channel para derivarla a los agentes especializados de cada frente.
- **Validación de conexión:** Abrir un primer PR de prueba hacia el pipeline para asegurar que la integración de su SDO funciona correctamente.
