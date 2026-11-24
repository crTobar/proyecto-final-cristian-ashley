# PRD — Sistema Web de Citas para Veterinaria

# 1. Control del Documento (Document Control)

- **Propietario (Owner):** Cristian Martínez, Ashley Retana

- **Versión:** 1.1

- **Estado:** Borrador

- **Revisores:** Wilian Martínez (docente), administrador(a) de la veterinaria, secretarias de recepción

- **Registro de aprobación:** Pendiente

**Historial de Revisiones (Revision History)**

| Versión | Fecha | Autor | Descripción de los cambios |
|---|---|---|---|
| 1.0 | 10/11/2026 | Cristian Martínez, Ashley Retana | Creación inicial del documento. |
| 1.1 | 23/11/2026 | Cristian Martínez, Ashley Retana | Se resuelven las preguntas abiertas de la sección 8. |

# 2. Problema y Objetivos (Problem and Objectives)

- **Evidencia del problema:** Hoy las citas de la veterinaria se piden por llamada, por WhatsApp o llegando directamente al local, y las secretarias las anotan en una agenda (de papel o en una hoja compartida). Como cada canal se maneja por aparte, pasa seguido que dos clientes queden con la misma hora, que se le dé una cita a un veterinario que ese día no atiende o que el cliente llegue y su cita no aparezca. Además, la línea pasa ocupada en horas pico, el cliente no puede ver qué horarios hay libres sin preguntar, y muchas personas olvidan la cita y no avisan, dejando espacios vacíos que otro cliente sí habría usado.

- **Resultados deseados (Outcomes):** Que el cliente pueda agendar en pocos pasos desde el celular o la computadora, viendo solo los horarios realmente disponibles, y que la agenda de las secretarias sea la misma que usa el sistema web. Así cualquier cita, sea en línea, por teléfono o en recepción, bloquea el horario al instante y ya no se producen choques.

- **Métricas de éxito (Measures of success):**

  - Cero citas duplicadas o con choque de horario por mes.

  - Que al menos el 50% de las citas se agenden en línea en los primeros 3 meses.

  - Agendar una cita en línea en menos de 2 minutos.

  - Bajar las inasistencias (no-show) en un 30% gracias a los recordatorios.

  - Reducir las llamadas a recepción solo para pedir citas.

# 3. Partes Interesadas y Alcance (Stakeholders and Scope)

- **Roles afectados:**

  - **Cliente (dueño de la mascota):** se registra, registra a sus mascotas, agenda, reprograma o cancela citas y recibe recordatorios.

  - **Secretaria / recepción:** agenda citas por teléfono o en persona, confirma llegadas, reprograma y bloquea horarios.

  - **Veterinario:** consulta su agenda del día y ve los datos de la mascota y el motivo de la cita.

  - **Administrador:** configura servicios, duraciones, horarios de atención, veterinarios y días libres, y revisa reportes.

  - **Externos:** servicio de correo y/o mensajería para confirmaciones y recordatorios.

- **Límites del sistema (System boundary):** El sistema empieza cuando el cliente o la secretaria buscan un horario disponible y termina cuando la cita queda atendida, cancelada o marcada como inasistencia. No incluye el expediente clínico completo, la facturación ni el inventario de la veterinaria.

- **Inclusiones (In-scope):**

  - Registro e inicio de sesión de clientes y registro de sus mascotas.

  - Consulta de horarios disponibles por servicio y veterinario.

  - Agendar, reprogramar y cancelar citas en línea.

  - Agenda única para las secretarias, sincronizada en tiempo real con la web.

  - Configuración de servicios, duración de cada uno y horarios de los veterinarios.

  - Confirmaciones y recordatorios automáticos.

  - Registro de llegada, atención e inasistencia.

  - Reportes básicos de citas.

- **Exclusiones (Out-of-scope):**

  - Expediente clínico, diagnósticos y recetas.

  - Pagos en línea y facturación electrónica.

  - Inventario de medicamentos y productos.

  - App móvil nativa; la web será adaptada a celular.

  - Atención de emergencias (se siguen atendiendo por teléfono o en el local).

# 4. Contexto del Proceso (Process Context)

- **Procesos actuales:** El cliente llama o escribe por WhatsApp y pregunta si hay espacio. La secretaria revisa la agenda, le ofrece una hora y la anota a mano. Si otra secretaria está atendiendo a alguien en el mostrador al mismo tiempo, puede dar la misma hora sin saberlo. No se envían recordatorios y, si el cliente no llega, el espacio se pierde.

- **Procesos propuestos:**

  - El cliente entra a la web, inicia sesión, escoge su mascota, el servicio y ve solo los horarios libres según la duración del servicio y la agenda de los veterinarios.

  - Al escoger un horario, el sistema lo aparta unos minutos mientras el cliente confirma, para que nadie más lo tome.

  - Al confirmar, la cita aparece de inmediato en la agenda de las secretarias y del veterinario, y el cliente recibe una confirmación.

  - Si la cita se pide por teléfono o en recepción, la secretaria la registra en la misma agenda y el horario se bloquea igual para la web.

  - Un día antes el sistema envía un recordatorio; el cliente puede confirmar, reprogramar o cancelar.

  - El día de la cita, recepción marca la llegada y el veterinario la marca como atendida; si el cliente no llega, se registra como inasistencia.

- **Responsabilidades:** El sistema valida la disponibilidad y bloquea los horarios; la secretaria registra las citas que llegan por teléfono o en persona y marca llegadas; el veterinario mantiene su agenda al día; el administrador configura servicios y horarios; y el cliente es responsable de mantener sus datos y avisar si no puede asistir.

- **Reglas de negocio:**

  - **RN-01:** Un mismo horario de un veterinario no puede tener dos citas; la validación se hace al momento de guardar para evitar choques.

  - **RN-02:** Cada servicio tiene una duración (ej. consulta 30 min, vacunación 15 min, cirugía programada 2 h) y la cita ocupa todo ese tiempo.

  - **RN-03:** Solo se ofrecen horarios dentro del horario de atención y de los turnos del veterinario; los días libres y feriados no aparecen.

  - **RN-04:** Al escoger un horario en línea, se aparta por 5 minutos; si el cliente no confirma, se libera.

  - **RN-05:** Las citas en línea se agendan con al menos 2 horas de anticipación.

  - **RN-06:** El cliente puede cancelar o reprogramar en línea hasta 4 horas antes; después solo por medio de recepción.

  - **RN-07:** Cada mascota puede tener como máximo 2 citas activas a la vez.

  - **RN-08:** Las secretarias pueden bloquear horarios (reuniones, cirugías, imprevistos) y esos espacios dejan de mostrarse en la web.

  - **RN-09:** Si el cliente no llega 15 minutos después de la hora, la cita se marca como inasistencia.

  - **RN-10:** Solo el administrador puede cambiar servicios, duraciones y horarios de atención.

# 5. Requisitos (Requirements)

- **Capacidades identificadas y priorizadas (Funcionales):**

  - **\[P0 - Crítico\] RF-01 Registro de clientes y mascotas:** El sistema debe permitir que el cliente cree su cuenta e inicie sesión, y registre una o varias mascotas (nombre, especie, raza, edad).

  - **\[P0 - Crítico\] RF-02 Consultar disponibilidad:** El sistema debe mostrar los horarios libres según el servicio escogido, su duración, el veterinario y la agenda actual.

  - **\[P0 - Crítico\] RF-03 Agendar cita en línea:** El cliente debe poder escoger mascota, servicio, veterinario (opcional), fecha y hora, y confirmar la cita. El sistema valida RN-01 a RN-07 antes de guardarla.

  - **\[P0 - Crítico\] RF-04 Agenda sincronizada de secretarias:** Las secretarias deben poder ver la agenda del día y de la semana por veterinario, y crear citas por teléfono o en recepción en la misma agenda; cualquier cambio se refleja de inmediato en la web.

  - **\[P0 - Crítico\] RF-05 Bloqueo y validación de choques:** El sistema debe impedir que dos citas ocupen el mismo horario del mismo veterinario, sin importar si se crearon en línea o en recepción.

  - **\[P1 - Alto\] RF-06 Reprogramar y cancelar:** El cliente debe poder reprogramar o cancelar su cita dentro del plazo permitido, y la secretaria en cualquier momento; el horario liberado vuelve a quedar disponible.

  - **\[P1 - Alto\] RF-07 Confirmaciones y recordatorios:** El sistema debe enviar una confirmación al agendar y un recordatorio 24 horas antes, con opción de confirmar o cancelar.

  - **\[P1 - Alto\] RF-08 Configuración de servicios y horarios:** El administrador debe poder definir servicios, su duración, los veterinarios, sus turnos, días libres y feriados.

  - **\[P1 - Alto\] RF-09 Bloquear horarios:** Las secretarias deben poder bloquear espacios de la agenda para que no se puedan reservar.

  - **\[P1 - Alto\] RF-10 Registro de llegada y atención:** Recepción debe poder marcar la llegada del cliente, el veterinario marcar la cita como atendida y el sistema marcar las inasistencias según RN-09.

  - **\[P2 - Medio\] RF-11 Historial de citas:** Sería deseable que el cliente y la secretaria puedan ver el historial de citas de cada mascota.

  - **\[P2 - Medio\] RF-12 Reportes:** Sería deseable que el administrador vea reportes de citas por día, por servicio, por veterinario y de inasistencias.

- **Cualidades (No funcionales):**

  - **RNF-01 (Rendimiento):** la consulta de horarios disponibles debe cargar en menos de 2 segundos.

  - **RNF-02 (Consistencia):** los cambios en la agenda deben verse en la web y en recepción en menos de 5 segundos, y la reserva de un horario debe ser atómica para que no se duplique.

  - **RNF-03 (Usabilidad):** agendar una cita no debe tomar más de 4 pasos y la web debe funcionar bien en celular.

  - **RNF-04 (Seguridad):** contraseñas cifradas, conexión HTTPS y permisos por rol (cliente, secretaria, veterinario, administrador).

  - **RNF-05 (Disponibilidad):** 99% del tiempo durante el horario de atención.

  - **RNF-06 (Privacidad):** los datos de clientes y mascotas solo los ve el personal de la veterinaria.

- **Restricciones (Constraints):**

  - Debe cumplir con la Ley 8968 de Protección de Datos Personales de Costa Rica.

  - Las secretarias deben usar el sistema como agenda principal; no se puede mantener la agenda en papel al mismo tiempo.

  - Presupuesto limitado: debe funcionar en las computadoras que ya tiene la veterinaria y en un hosting de bajo costo.

  - Primera versión lista en un máximo de 3 meses.

# 6. Descripciones de Apoyo (Supporting Descriptions)

- **Diagramas de flujo de datos:** Según lo indicado para esta etapa, el documento no incluye diagramas. El flujo de información, descrito en texto, es el siguiente:

  - El cliente o la secretaria envían la solicitud (mascota, servicio, fecha y hora).

  - El sistema consulta los horarios del veterinario, la duración del servicio y las citas ya agendadas.

  - Si el horario está libre, guarda la cita y la publica en la agenda compartida.

  - El servicio de correo o mensajería envía la confirmación y luego el recordatorio.

  - **Estados de una cita:** Apartada (5 min) → Agendada → Confirmada → Atendida. También puede pasar a Cancelada, Reprogramada o Inasistencia.

- **Definiciones de terminología (Glosario):**

  - **Agenda compartida:** la misma agenda que usan la web, las secretarias y los veterinarios.

  - **Choque de horario:** cuando dos citas ocupan el mismo tiempo de un mismo veterinario.

  - **Bloqueo de horario:** espacio de la agenda que la secretaria marca como no disponible.

  - **Apartado temporal:** reserva de 5 minutos de un horario mientras el cliente confirma.

  - **No-show / inasistencia:** cuando el cliente no llega a la cita.

  - **Servicio:** tipo de atención (consulta, vacunación, baño, cirugía programada) con una duración definida.

# 7. Aceptación y Trazabilidad (Acceptance and Traceability)

- **Criterios de aceptación:**

  - **RF-05 (choque de horario):** Dado que la secretaria registró una cita por teléfono con la Dra. Solís el martes a las 10:00, cuando un cliente busca horarios en la web para ese martes, entonces las 10:00 con la Dra. Solís ya no aparece disponible.

  - **RF-03 (apartado):** Dado que dos clientes escogen el mismo horario casi al mismo tiempo, cuando el primero lo aparta, entonces el segundo ve el mensaje "Ese horario acaba de ser reservado" y se le muestran otras opciones.

  - **RF-06 (plazo):** Dado que la cita es hoy a las 3:00 p.m., cuando el cliente intenta cancelarla en línea a las 12:00 m.d., entonces el sistema no lo permite y le indica que llame a recepción.

  - **RF-07 (recordatorio):** Dado que hay una cita mañana a las 9:00 a.m., cuando se cumplen 24 horas antes, entonces el cliente recibe el recordatorio con opción de confirmar o cancelar.

- **Trazabilidad:**

  - **Cero choques de horario →** RF-02, RF-03, RF-04, RF-05 → QA-01 Validación de choques

  - **50% de citas en línea →** RF-01, RF-02, RF-03 → Reporte de citas por canal (RF-12)

  - **Agendar en menos de 2 min →** RF-02, RF-03 → QA-02 Prueba de usabilidad

  - **Menos inasistencias →** RF-06, RF-07, RF-10 → Reporte de inasistencias (RF-12)

# 8. Evaluación y Asuntos Pendientes (Evaluation and Open Matters)

- **Viabilidad:** Es viable porque el problema principal es organizar una sola agenda; no requiere integrarse con sistemas complejos y puede funcionar con un hosting sencillo.

- **Riesgos:** Que las secretarias sigan usando la agenda en papel por costumbre (mitigación: capacitación y retirar la agenda física) y que algunos clientes mayores prefieran seguir llamando (mitigación: la secretaria agenda por ellos en el mismo sistema).

- **Preguntas abiertas (resueltas en la versión 1.1):**

    - ¿Los recordatorios se envían por correo, por WhatsApp o por ambos? Solo por correo electrónico en la primera versión. WhatsApp queda para una versión futura.

    - ¿El cliente puede escoger veterinario o se asigna automáticamente? El cliente puede escogerlo o dejar "Cualquiera disponible"; en ese caso el sistema asigna al veterinario con menos citas ese día. La secretaria siempre escoge el veterinario.

    - ¿Se cobra algún depósito para cirugías programadas? No. No se cobran depósitos en ninguna cita, ya que los pagos están fuera del alcance.
