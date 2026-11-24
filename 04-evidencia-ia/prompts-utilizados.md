# Prompts utilizados

Registro de los prompts usados para generar las specs y el prototipo visual de **VetCitas – Sistema Web de Citas para Veterinaria**, en el orden en que se usaron.

| # | Etapa | Herramienta | Entrada |
|---|---|---|---|
| 1 | Generación de las specs (`03-specs/`) | Claude | Prompt 1 + `01-PRD.md` |
| 2 | Revisión de las specs | Claude | Los 4 archivos de `03-specs/` |
| 3 | Primer prototipo (descartado) | Google Stitch | Specs + prompts 3.1 y 3.2 |
| 4 | Prototipo final desde cero | Google Stitch | Prompt 4 + los 4 archivos de `03-specs/` |
| 5 | Iteraciones de corrección | Google Stitch | Prompts 5.1 a 5.5 en el mismo proyecto |

**Cómo se inyectaron las specs:** en cada prompt de Stitch se adjuntaron los archivos `architecture-spec.md`, `feature-core-01.md`, `feature-core-02.md` y `ui-ux-spec.md`. Cada corrección se hizo revisando las pantallas generadas contra las specs (reglas RN/RG, catálogo MSG, estados, datos semilla) y pidiendo solo los cambios necesarios.

---

## 1. Generación de las specs

```
Quiero que me ayudes a desarrollar los Specs de mi proyecto, que consiste en un sistema web. Necesito crear los cuatro archivos solicitados por el profesor:

architecture-spec.md: debe incluir el stack tecnológico sugerido, el modelo de datos, las entidades y las reglas globales del sistema.
feature-core-01.md: debe describir detalladamente la primera funcionalidad principal incluyendo su funcionamiento y todos los aspectos necesarios para desarrollarla.
feature-core-02.md: debe describir detalladamente la segunda funcionalidad principal siguiendo la misma estructura de especificación.
ui-ux-spec.md: debe definir los flujos de las pantallas, los estados de las interfaces y los componentes del sistema.

Las especificaciones deben incluir entidades y esquemas de datos, reglas de negocio explícitas, casos borde (Edge Cases), estados de interfaz (vacío, cargando, éxito y error) y criterios de aceptación verificables utilizando el formato Given / When / Then.
Quiero que los archivos estén redactados en Markdown, con precisión técnica, sin ambigüedades y con suficiente detalle para que una herramienta de inteligencia artificial pueda utilizarlos para generar el prototipo visual del sistema.
Mantén la coherencia entre los cuatro archivos y trabaja únicamente con las funcionalidades de mi proyecto, sin inventar características ni agregar apartados que no hayan sido solicitados por el profesor.
```

## 2. Revisión de las specs

### 2.1 Pedir la revisión

```
Revisa estos md del proyecto de la veterinaria que estamos haciendo.
```

### 2.2 Aplicar las correcciones

Se adjuntaron los 4 archivos de specs completos, las instrucciones del profesor y el PRD.

```
Estos son los documentos completos, ayúdame realizando los cambios que me diste anteriormente. También te adjunté lo que el profesor pide y nuestro documento para que puedas revisar todo.
```

Correcciones aplicadas después de la revisión:

1. `INASISTENCIA` ya no ocupa horario (tabla de estados, restricción de exclusión, RG-05).
2. Un cliente creado por recepción que se registra en línea se vincula por correo (RG-19), sin duplicados.
3. Reprogramar marca primero la original como `REPROGRAMADA` y luego crea la nueva, en una transacción (permite mover una cita 15 min).
4. RNF-05 (99 % de disponibilidad) documentado en architecture §1.
5. Se definió el modal M-02 (Detalle de bloqueo).
6. Se agregaron al catálogo los mensajes MSG-29 a MSG-44.
7. El asunto del recordatorio usa la fecha en vez de "mañana".
8. Queda fuera de alcance el correo al cancelar.
9. Se agregó `RECURSO_NO_ENCONTRADO` a `POST /api/citas/{id}/reprogramar`.
10. La validación V5 compara solo la fecha de `inicio`.
11. `PLAZO_VENCIDO` pasó a 422.

---

## 3. Primer prototipo (descartado)

El primer intento en Stitch inventó datos y funciones fuera del alcance (precios, urgencias, WhatsApp, reseñas, especialidades, IDs de reglas visibles). Se corrigió dos veces y luego se decidió rehacerlo desde cero con un prompt completo (sección 4).

### 3.1 Primera corrección

```
Corrige el prototipo para que siga EXACTAMENTE el ui-ux-spec.md adjunto. No inventes nada que no esté en los specs.

1. Datos: usa solo los datos semilla: Dra. María Solís (L-V 08:00-16:00), Dr. Andrés Vargas (L-V 10:00-17:00, Sáb 08:00-12:00), clienta Ana Rodríguez (8888-1234), mascotas Luna (Perro, Labrador, 3 años) y Miso (Gato, 2 años). Servicios: Consulta 30 min, Vacunación 15 min, Baño 45 min, Cirugía programada 120 min. Horario: L-V 08:00-17:00, Sáb 08:00-12:00.
2. Quita: precios, urgencias 24/7, WhatsApp, QR, microchip, peso, vacunas, expediente clínico, reseñas, especialidades y consultorios. Quita la pantalla de búsqueda de Google y la página de inicio; "/" lleva a Iniciar sesión.
3. No muestres códigos RN/RF/RG en la interfaz.
4. El nombre y rol del usuario arriba deben coincidir con quien inició sesión (Ana Rodríguez – Cliente; Secretaria – Recepción).
5. P-04 Agendar: 4 pasos: 1 Mascota y servicio, 2 Veterinario y fecha, 3 Horario (chips Mañana/Tarde), 4 Confirmar. El temporizador de 5:00 aparece solo en el paso 4.
6. Agrega: P-05 Mis citas (pestañas Próximas/Historial), P-06 Detalle de cita con los modales de confirmar y cancelar, aviso MSG-03 cuando faltan menos de 4 h, y P-09 Mi agenda del veterinario con "Marcar atendida" y sin datos del cliente.
7. Muestra el mensaje "Ese horario acaba de ser reservado" en el paso 3 y al menos un estado vacío, cargando y error con los textos del catálogo §8.
8. Menú por rol como en §2 (Cliente: Agendar cita · Mis citas · Mis mascotas · Cerrar sesión).
```

### 3.2 Segunda corrección

```
Ajusta el prototipo según ui-ux-spec.md, sin agregar nada que no esté en los specs:

1. P-05: el texto debe decir que se puede cancelar o reprogramar hasta 4 horas antes (no 2). Quita "historial clínico" y "cartilla digital".
2. P-06: quita código de cita, consultorio, especialidad, mapa de ubicación e instrucciones de preparación. Agrega el motivo de la cita. El aviso MSG-03 solo aparece si faltan menos de 4 h; en ese caso los botones Reprogramar y Cancelar cita quedan deshabilitados.
3. Usa solo estas etiquetas de estado (§4.2): Agendada, Confirmada, Atendida, Cancelada, Reprogramada, Inasistencia, y la insignia adicional "En sala".
4. P-09 (veterinario): menú solo "Mi agenda" y "Cerrar sesión"; solo vista por día; "Marcar atendida" habilitado solo en citas con "En sala"; sin consultorios ni salas.
5. Usa fechas de 2026 en todas las pantallas.
6. Crea P-08 Agenda de recepción: columnas por veterinario, filas de 15 min, citas con color por estado, un bloqueo gris, botones "Nueva cita" y "Bloquear horario", "Actualizado hace N s".
7. Crea los pasos 2, 3 y 4 de Agendar: calendario, chips Mañana/Tarde, temporizador 5:00 solo en el paso 4, y el estado de error "Ese horario acaba de ser reservado".
```

---

## 4. Prototipo final desde cero (Google Stitch)

Se adjuntaron los 4 archivos de `03-specs/`.

```
Crea un prototipo web navegable y responsive (mobile-first) de "VetCitas", un sistema de citas para una veterinaria en Costa Rica. Sigue EXACTAMENTE los specs adjuntos (sobre todo ui-ux-spec.md). No inventes pantallas, datos, servicios ni funciones que no estén en los specs. Toda la interfaz en español.

REGLAS PARA EL PROTOTIPO
- No muestres códigos internos (RN-, RF-, RG-, MSG-) en la interfaz.
- No incluyas: precios ni pagos, urgencias, WhatsApp, códigos QR, expediente clínico, microchip, peso, vacunas al día, reseñas, especialidades de los veterinarios, consultorios o salas, mapas, ni página de inicio publicitaria.
- El nombre y rol que aparece arriba siempre es el de la sesión activa.
- Fechas en formato DD/MM/YYYY, año 2026, horas en formato 12 h (9:00 a. m.).
- Estilo sencillo y limpio, sin exceso de colores.

DATOS SEMILLA (usar solo estos)
- Horario de atención: lunes a viernes 8:00 a. m.–5:00 p. m., sábado 8:00 a. m.–12:00 m. d., domingo cerrado.
- Servicios: Consulta 30 min, Vacunación 15 min, Baño 45 min, Cirugía programada 120 min.
- Veterinarios: Dra. María Solís (L-V 8:00–4:00 p. m.), Dr. Andrés Vargas (L-V 10:00 a. m.–5:00 p. m. y sábado 8:00 a. m.–12:00 m. d.). Ninguna cita puede quedar fuera del turno del veterinario.
- Cliente: Ana Rodríguez, 8888-1234, ana@correo.com.
- Mascotas de Ana: Luna (Perro, Labrador, 3 años) y Miso (Gato, 2 años).
- Cuentas demo: Ana Rodríguez (Cliente), Secretaria (Recepción), Dra. María Solís (Veterinario), Administrador.

ESTADOS DE CITA (etiqueta siempre visible, no solo color)
Apartada (gris), Agendada (azul), Confirmada (verde), Atendida (verde azulado), Cancelada (rojo), Reprogramada (violeta), Inasistencia (naranja). Insignia adicional "En sala" (cian) cuando la secretaria registró la llegada. No uses otras etiquetas como "Pendiente", "En espera" o "A tiempo".

MENÚ POR ROL
- Cliente: Agendar cita · Mis citas · Mis mascotas · Cerrar sesión. Inicio: Mis citas.
- Secretaria: Agenda · Cerrar sesión.
- Veterinario: Mi agenda · Cerrar sesión (solo vista por día).
- Administrador: Configuración · Reportes · Cerrar sesión.

PANTALLAS
1. Iniciar sesión: correo y contraseña (mostrar/ocultar), botón "Iniciar sesión", enlace "Crear cuenta", y accesos rápidos a las 4 cuentas demo. Error: "Correo o contraseña incorrectos."
2. Registro: nombre completo, correo, teléfono (XXXX-XXXX), contraseña, confirmar contraseña y casilla obligatoria "Acepto el tratamiento de mis datos personales conforme a la Ley 8968". Botón "Crear cuenta".
3. Mis mascotas: tarjetas con nombre, especie, raza, edad y "N citas activas"; botón "Agregar mascota" y "Editar" (modal con nombre, especie, raza, edad). Vacío: "Aún no tienes mascotas registradas" + "Registrar mascota".
4. Agendar cita (4 pasos con barra "Paso N de 4"):
   - Paso 1 Mascota y servicio: tarjetas de mascota (deshabilitada si tiene 2 citas activas, con texto "2 citas activas") y servicios con duración.
   - Paso 2 Veterinario y fecha: "Cualquiera disponible" o un veterinario; calendario con días sin disponibilidad, domingos y fechas pasadas deshabilitados.
   - Paso 3 Horario: chips de hora cada 15 min agrupados en "Mañana" y "Tarde". Mostrar el error "Ese horario acaba de ser reservado. Te mostramos otros horarios disponibles." Vacío: "No hay horarios disponibles para este día" + "Probar con otra fecha".
   - Paso 4 Confirmar: resumen (mascota, servicio, veterinario asignado, fecha, hora, duración), campo "Motivo" opcional (máx. 500) y temporizador de apartado que empieza en 5:00 (solo en este paso). En 0:00 abre un modal "Tu reserva temporal venció. Elige un horario de nuevo." con botón "Elegir otro horario". Botón "Confirmar cita".
   - Éxito: "¡Cita agendada! Te enviamos la confirmación a ana@correo.com." con botones "Ver mis citas" y "Agendar otra".
5. Mis citas: pestañas "Próximas" e "Historial", filtro por mascota, tarjetas con fecha y hora, servicio, mascota, veterinario e insignia de estado. Vacío Próximas: "No tienes citas próximas" + "Agendar cita". Vacío Historial: "Aún no hay citas en tu historial".
6. Detalle de cita: estado, fecha y hora, duración, servicio, veterinario, mascota y motivo. Botones "Confirmar asistencia" (solo si está Agendada), "Reprogramar" y "Cancelar cita". Mostrar dos casos:
   - Cita a más de 4 horas: botones habilitados.
   - Cita a menos de 4 horas: botones Reprogramar y Cancelar deshabilitados y el aviso "Faltan menos de 4 horas para tu cita. Llama a recepción para cancelar o reprogramar."
   Modal cancelar: "¿Cancelar esta cita?" / "Se liberará el horario del {fecha} a las {hora}." / "Sí, cancelar" y "No, mantener". Éxito: "La cita fue cancelada."
7. Reprogramar cita: tarjeta "Cita actual" (mascota y servicio fijos) y 3 pasos: veterinario y fecha, horario, confirmar con comparación "Cita actual → Nueva cita" y temporizador de 5:00. Éxito: "La cita fue reprogramada para el {fecha} a las {hora}."
8. Agenda de recepción (secretaria): navegación de fecha ("‹", "Hoy", "›"), cambio "Día"/"Semana", filtro de veterinarios, botones "Nueva cita" y "Bloquear horario", indicador "Actualizado hace N s". Cuadrícula con una columna por veterinario y filas de 15 min; citas con color por estado y un bloqueo en gris. Clic en una cita abre un panel con datos del cliente (nombre, teléfono, correo), mascota, servicio, canal, motivo y botones "Registrar llegada", "Reprogramar", "Cancelar cita" y "Ver historial de la mascota". Clic en un bloqueo abre un panel con "Quitar bloqueo".
   - Modal Nueva cita: buscar cliente (o "Registrar nuevo cliente"), mascota, servicio, veterinario obligatorio, fecha, horario, canal (Teléfono / Presencial) y motivo. Éxito: "Cita agendada."
   - Modal Bloquear horario: veterinario, fecha, hora de inicio y fin (cada 15 min), motivo. Error si hay citas: "Hay citas en ese horario. Cancélalas o reprográmalas primero."
   - Modal Historial de la mascota: lista de citas anteriores con fecha, servicio, veterinario, estado y motivo.
9. Mi agenda (veterinario): solo vista por día; lista de citas con hora, duración, servicio, mascota (nombre, especie, raza, edad), motivo e insignia de estado. NO mostrar datos del cliente. Botón "Marcar atendida" habilitado solo en citas con insignia "En sala"; en las demás, deshabilitado. Éxito: "Cita marcada como atendida."
10. Configuración (administrador): 4 pestañas: Servicios (nombre, duración en múltiplos de 15 min, activo), Horario de atención (lunes a domingo), Veterinarios y turnos, Días libres y feriados. Botón "Guardar cambios".
11. Reportes (administrador): filtros desde/hasta y agrupar por (Día, Servicio, Veterinario, Canal); tarjetas "Total de citas", "Atendidas", "Inasistencias", "Tasa de inasistencia", "Citas en línea" y una tabla.

ESTADOS DE INTERFAZ
En cada lista o pantalla de datos muestra los estados vacío, cargando (esqueletos grises) y error (mensaje "Algo salió mal. Intenta de nuevo." con botón "Reintentar"). Agrega un control para ver cada estado en el prototipo.

REGLAS A SIMULAR
- Apartado de 5 minutos al elegir horario en línea.
- No se puede elegir un horario ocupado ni bloqueado.
- El cliente agenda con al menos 2 horas de anticipación y hasta 60 días.
- El cliente cancela o reprograma hasta 4 horas antes; después, solo recepción.
- Máximo 2 citas activas por mascota.
- Si el cliente no llega 15 minutos después de la hora, la cita pasa a Inasistencia.
- Los recordatorios y confirmaciones son solo por correo.
```

---

## 5. Iteraciones de corrección (mismo proyecto de Stitch)

### 5.1 Correcciones de la primera generación

Problemas encontrados: inventó la mascota "Rocky", marcó la raza como obligatoria, el correo de Ana no coincidía, el texto del aviso de horario ocupado no era el del catálogo y faltaban los pasos 1, 2 y 4.

```
Corrige: quita a "Rocky" (solo existen Luna y Miso). En Editar mascota la raza es opcional. El correo de Ana es ana@correo.com. El aviso del paso 3 debe decir exactamente: "Ese horario acaba de ser reservado. Te mostramos otros horarios disponibles." Agrega también los pasos 1, 2 y 4 de Agendar cita (el paso 4 con temporizador de 5:00 y el modal "Tu reserva temporal venció. Elige un horario de nuevo.").
```

### 5.2 Pantallas del cliente

```
Crea las pantallas 5, 6 y 7 del prompt original: Mis citas (pestañas Próximas e Historial), Detalle de cita (un caso a más de 4 horas con botones habilitados y otro a menos de 4 horas con botones deshabilitados y el aviso "Faltan menos de 4 horas para tu cita. Llama a recepción para cancelar o reprogramar.") y Reprogramar cita.
```

### 5.3 Pantallas del personal

```
Crea las pantallas 8, 9, 10 y 11 del prompt original: Agenda de recepción con sus modales (Nueva cita, Bloquear horario, Historial de la mascota, panel de cita y panel de bloqueo), Mi agenda del veterinario (sin datos del cliente; "Marcar atendida" solo con insignia "En sala"), Configuración y Reportes.
```

### 5.4 Últimas correcciones

Problemas encontrados: el veterinario tenía "Marcar inasistencia" (contradice RN-09), los menús de secretaria y veterinario tenían opciones de otro rol, los reportes no tenían "Citas en línea" y calculaban mal la tasa, la cita original aparecía "Reprogramada" antes de confirmar y había datos incoherentes entre pantallas.

```
Últimas correcciones según los specs:
1. Mi agenda (veterinario): quita el botón "Marcar inasistencia"; la inasistencia la marca el sistema automáticamente 15 min después de la hora. Menú solo "Mi agenda" y "Cerrar sesión". La cita de 9:00–9:45 es "Baño", no "Consulta de revisión".
2. Agenda de recepción: menú solo "Agenda" y "Cerrar sesión". El filtro dice "Todos (2)".
3. Reportes: usa las tarjetas Total de citas, Atendidas, Inasistencias, Tasa de inasistencia y Citas en línea (%). Tasa de inasistencia = inasistencias / (atendidas + inasistencias). El total no incluye citas reprogramadas. Quita "Meta > 85%". Filtros: desde, hasta y agrupar por (Día, Servicio, Veterinario, Canal).
4. Reprogramar: la cita original se muestra con su estado actual (Confirmada) hasta confirmar; usa la misma cita de Luna del 23/03/2026 a las 9:00 a. m.
5. Haz coherentes los datos entre pantallas: las citas de Miso de la agenda de recepción también aparecen en Mis citas, y "En sala" solo aparece en citas de hoy cuya hora ya llegó.
6. Quita especialidades ("Médico General", "Especialista") y la palabra "recargo". Agrega el motivo en Detalle de cita. El botón del modal de tiempo vencido dice "Elegir otro horario". En el paso 2 el veterinario es opcional ("Cualquiera disponible" por defecto).
```

### 5.5 Reportes y Configuración

Problemas encontrados: Reportes tenía secciones que no están en la spec P-11 ("Productividad médica", "Adherencia a reglas de negocio", "Demanda de servicios", "Distribución por estados" y un botón de descarga PDF/Excel). Configuración mostraba como editables las reglas fijas de architecture §3.4 (anticipación mínima, citas activas por mascota, paso de intervalo).

```
Ajusta Reportes y Configuración según ui-ux-spec.md:
1. Reportes: quita "Productividad médica", "Adherencia a reglas de negocio", "Demanda de servicios", "Distribución por estados" y el botón "Descargar reporte". Deja solo los filtros (desde, hasta, agrupar por Día/Servicio/Veterinario/Canal), las 5 tarjetas (Total de citas, Atendidas, Inasistencias, Tasa de inasistencia, Citas en línea) y una tabla con columnas Etiqueta, Total, Atendidas, Inasistencias y Canceladas.
2. Configuración: quita las tarjetas de anticipación mínima, citas activas por mascota y paso de intervalo (son reglas fijas, no se configuran). Deja solo 4 pestañas: Servicios, Horario de atención, Veterinarios y turnos, Días libres y feriados.
```

Resultado: las pantallas finales están en `capturas/` y el enlace en `enlace-prototipo.md`.
