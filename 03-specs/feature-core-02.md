# feature-core-02 — Gestión del ciclo de vida de la cita

> Depende de `architecture-spec.md` (entidades, estados T-01…T-09, `RN-xx`, `RG-xx`, errores, `CitaDetalle`) y de `feature-core-01.md` (cómo nacen las citas). No redefine nada de esos archivos.

## 1. Alcance y trazabilidad

| Elemento | Detalle |
|---|---|
| Funcionalidad | Confirmar, cancelar y reprogramar citas; confirmaciones y recordatorios por correo; registrar llegada; marcar atendida; marcar inasistencia automática |
| Requerimientos PRD | RF-06, RF-07, RF-10 |
| Reglas | RN-05, RN-06, RN-07, RN-09 (+ RN-01, RN-03 para el nuevo horario al reprogramar) · RG-01, RG-04, RG-05, RG-07, RG-12, RG-13, RG-17, RG-18 |
| No funcionales | RNF-02 (atomicidad y sincronía), RNF-04 (permisos por rol), RNF-06 (privacidad) |
| Métricas del PRD | Bajar las inasistencias un 30 % · reducir llamadas a recepción para gestionar citas |
| Diagramas relacionados | Estados de `Cita` (T-05…T-09) · Secuencia "cancelar o reprogramar una cita" (con camino alt de plazo vencido) · DFD nivel 1 (proceso de notificaciones) |
| Pantallas | P-05, P-06, P-07 (cliente) · P-08 con panel C-14 y modal M-04 (secretaria) · P-09 (veterinario) |

## 2. Actores y precondiciones

| Actor | Puede |
|---|---|
| CLIENTE | Ver sus citas; confirmar, cancelar y reprogramar las suyas dentro del plazo (RN-06) |
| SECRETARIA | Cancelar y reprogramar en cualquier momento (mientras no haya llegada registrada); registrar llegada |
| VETERINARIO | Marcar como atendida una cita **suya** con llegada registrada |
| SISTEMA | Enviar confirmaciones y recordatorios (J-02, J-04); marcar inasistencias (J-03) |

Precondición común: la cita existe y el solicitante tiene permiso sobre ella (si no: `RECURSO_NO_ENCONTRADO` para clientes, `SIN_PERMISO` para otro rol sin acceso).

## 3. Reglas de esta funcionalidad

| ID | Regla |
|---|---|
| R-C2-01 | **Plazo del cliente (RN-06):** el cliente puede cancelar o reprogramar solo si `inicio − ahora ≥ 4 h` (el límite exacto de 4 h 00 min 00 s **sí** se permite). Después debe llamar a recepción. |
| R-C2-02 | Nadie puede cancelar ni reprogramar una cita con `llegada_en` registrada, ni una cita que no esté `AGENDADA` o `CONFIRMADA`. |
| R-C2-03 | Cancelar libera el horario de inmediato (`CANCELADA` no ocupa horario, RG-05). |
| R-C2-04 | Reprogramar conserva mascota y servicio; se puede cambiar fecha, hora y veterinario. El nuevo horario cumple las mismas validaciones que una cita nueva: cliente → `feature-core-01.md` §5.2 (V2…V8, incluyendo RN-05 de 2 h para el **nuevo** horario); secretaria → §6.2 (V2…V7). La regla RN-06 se evalúa sobre la cita **original**. |
| R-C2-05 | La cita nueva de una reprogramación nace `AGENDADA` aunque la original estuviera `CONFIRMADA`, con `cita_origen_id` y el `canal` de la original (RG-12). Original y nueva cambian en **una sola transacción y en este orden**: (1) el original pasa a `REPROGRAMADA`; (2) la nueva se inserta `AGENDADA` (recepción) o pasa de `APARTADA` a `AGENDADA` (web). Así se puede mover una cita a un horario que solapa con el suyo (ej. 10:00 → 10:15). Si algún paso falla se revierte todo y el original no cambia. |
| R-C2-06 | Solo el rol `CLIENTE` dueño confirma asistencia, y solo desde `AGENDADA` mientras `ahora < inicio`. |
| R-C2-07 | **Llegada:** solo `SECRETARIA`, en el mismo día calendario de la cita (RG-01), con estado `AGENDADA`/`CONFIRMADA` y `llegada_en` NULL. No cambia el estado (RG-13). |
| R-C2-08 | **Atendida:** solo el `VETERINARIO` asignado a la cita, con estado `AGENDADA`/`CONFIRMADA` y `llegada_en` NO NULL (T-08). |
| R-C2-09 | **Inasistencia (RN-09):** J-03 marca `INASISTENCIA` cuando `llegada_en` es NULL y `ahora ≥ inicio + 15 min`. `INASISTENCIA` es terminal: no hay deshacer. El resto del bloque queda libre (RG-05). |
| R-C2-10 | **Recordatorio (RF-07, RG-07):** se envía una sola vez por cita, solo si: estado `AGENDADA`; `ahora ≥ inicio − 24 h`; `agendada_en ≤ inicio − 24 h` (la cita existía en ese momento); `inicio > ahora`; el cliente tiene correo; y no existe `Notificacion` `RECORDATORIO` para esa cita. |
| R-C2-11 | **Confirmación (RF-07):** al pasar a `AGENDADA` (T-02 o T-03) se crea una `Notificacion` `CONFIRMACION` si el cliente tiene correo. Una cita nacida de una reprogramación también la genera. |
| R-C2-12 | Fallar un envío de correo nunca cambia el estado de la cita (J-04, máx. 3 intentos). |
| R-C2-13 | **Fuera de alcance:** no se envía correo al cancelar (ni por el cliente ni por recepción) ni al reprogramar, salvo la `CONFIRMACION` de la cita nueva (R-C2-11). RF-07 solo cubre confirmación y recordatorio. |

### 3.1 Cálculo de `acciones_permitidas` (lo hace el servidor)

Todas son `false` si `estado = APARTADA`.

| Acción | CLIENTE (cita propia) | SECRETARIA | VETERINARIO (cita propia) |
|---|---|---|---|
| `confirmar` | `estado = AGENDADA` ∧ `ahora < inicio` | `false` | `false` |
| `cancelar` y `reprogramar` | `estado ∈ {AGENDADA, CONFIRMADA}` ∧ `llegada_en = NULL` ∧ `inicio − ahora ≥ 4 h` | `estado ∈ {AGENDADA, CONFIRMADA}` ∧ `llegada_en = NULL` | `false` |
| `registrar_llegada` | `false` | `estado ∈ {AGENDADA, CONFIRMADA}` ∧ `llegada_en = NULL` ∧ fecha de `inicio` = hoy | `false` |
| `marcar_atendida` | `false` | `false` | `estado ∈ {AGENDADA, CONFIRMADA}` ∧ `llegada_en ≠ NULL` |

## 4. Flujos

### 4.A Confirmar asistencia (T-05)

1. El cliente abre el detalle de la cita (P-06), desde la lista (P-05) o desde el enlace del correo (§4.D).
2. Pulsa "Confirmar asistencia" → `POST /api/citas/{id}/confirmar`.
3. La cita pasa a `CONFIRMADA`. La UI muestra MSG-14 y actualiza la insignia.

### 4.B Cancelar (T-06)

1. **Cliente:** en P-06 pulsa "Cancelar cita" (habilitado solo si `acciones_permitidas.cancelar`). Un modal de confirmación pide "Sí, cancelar" → `POST /api/citas/{id}/cancelar`.
2. **Secretaria:** desde el panel C-14 en P-08, mismo endpoint y mismo modal.
3. La cita pasa a `CANCELADA`, el horario se libera y se emiten `agenda.cambio` y `disponibilidad.cambio`.
4. Si el cliente ya no cumple el plazo (R-C2-01) el botón está deshabilitado y la UI muestra MSG-03 ("Llama a recepción para cancelar o reprogramar").

### 4.C Reprogramar (T-07)

**Cliente (web), pantalla P-07:**
1. Pulsa "Reprogramar" en P-06 (`acciones_permitidas.reprogramar`).
2. Elige veterinario/fecha y horario (mismos componentes que `feature-core-01.md` §4). Las consultas de disponibilidad envían `excluir_cita_id` = id de la cita original (R-C1-13).
3. Al elegir horario se crea un apartado: `POST /api/citas/apartados` con `cita_origen_id` (`feature-core-01.md` §5.2).
4. Ve el resumen "Cita actual" → "Nueva cita" y confirma: `POST /api/citas/{nueva_id}/confirmar-apartado`. En una transacción, en este orden: original `REPROGRAMADA`, nueva `AGENDADA` (si falla algo, se revierte todo).
5. Si el apartado vence o falla la confirmación, **la cita original no se modifica**.

**Secretaria (recepción), modal M-04:**
1. Desde C-14 pulsa "Reprogramar", elige veterinario, fecha y horario.
2. `POST /api/citas/{id}/reprogramar`. En una transacción, en este orden: el original pasa a `REPROGRAMADA` y luego se inserta la nueva `AGENDADA` (T-02); si falla algo, se revierte todo. La consulta de horarios usa `excluir_cita_id` (R-C1-13).

### 4.D Confirmación y recordatorio por correo (RF-07)

**Tarea J-02 (cada 60 s):** selecciona las citas que cumplen R-C2-10, crea la `Notificacion` `RECORDATORIO` (`PENDIENTE`) y la envía. J-04 reintenta fallos.

**Correo de confirmación**
- Asunto: `Cita agendada: {mascota} el {DD/MM/YYYY} a las {hora}`
- Cuerpo: nombre del cliente, mascota, servicio, veterinario, fecha y hora, duración, y (solo si el cliente tiene cuenta) el enlace "Ver mi cita": `{APP_URL}/citas/{id}`.

**Correo de recordatorio**
- Asunto: `Recordatorio: cita de {mascota} el {DD/MM/YYYY} a las {hora}`
- Cuerpo: mismos datos y tres botones, solo si el cliente tiene cuenta:
  - "Confirmar asistencia" → `{APP_URL}/citas/{id}?accion=confirmar`
  - "Reprogramar" → `{APP_URL}/citas/{id}?accion=reprogramar`
  - "Cancelar" → `{APP_URL}/citas/{id}?accion=cancelar`
- Si el cliente no tiene cuenta: sin botones y con el texto "Para confirmar, cancelar o reprogramar, comunícate con recepción."
- Los enlaces exigen iniciar sesión (no llevan tokens). Tras el login, la app abre P-06 y muestra el modal de la acción indicada; el servidor valida igualmente las reglas.

### 4.E Registrar llegada (R-C2-07)

La secretaria, en P-08, abre la cita (C-14) y pulsa "Registrar llegada" → `POST /api/citas/{id}/llegada`. La cita muestra la insignia adicional "En sala" y se habilita "Marcar atendida" para el veterinario.

### 4.F Marcar atendida (T-08)

El veterinario, en P-09, pulsa "Marcar atendida" en la cita → `POST /api/citas/{id}/atendida`. La cita pasa a `ATENDIDA`.

### 4.G Inasistencia automática (T-09)

J-03 (cada 60 s) ejecuta una actualización atómica: `UPDATE cita SET estado = 'INASISTENCIA' WHERE estado IN ('AGENDADA','CONFIRMADA') AND llegada_en IS NULL AND ahora >= inicio + 15 min`, y emite `agenda.cambio` por cada cita afectada.

## 5. Contratos

### 5.1 Consultar citas

`GET /api/citas/mias?vista=proximas|historial&mascota_id={uuid?}&pagina=1&por_pagina=20` — rol `CLIENTE`

- `proximas`: estados `AGENDADA` y `CONFIRMADA`, orden `inicio` ascendente.
- `historial`: estados `ATENDIDA`, `CANCELADA`, `REPROGRAMADA` e `INASISTENCIA`, orden `inicio` descendente.
- `200` → `{ "items": [CitaDetalle], "total": n, "pagina": 1, "por_pagina": 20 }` (sin objeto `cliente`).

`GET /api/citas/{id}` — roles `CLIENTE` (propia), `SECRETARIA`, `VETERINARIO` (propia) → `200` `CitaDetalle`.

### 5.2 Acciones sobre la cita

Todas devuelven `200` con la cita actualizada salvo indicación.

| Método y ruta | Rol | Cuerpo | Efecto | Errores |
|---|---|---|---|---|
| `POST /api/citas/{id}/confirmar` | CLIENTE | — | `AGENDADA → CONFIRMADA` | `TRANSICION_INVALIDA`, `RECURSO_NO_ENCONTRADO` |
| `POST /api/citas/{id}/cancelar` | CLIENTE, SECRETARIA | — | → `CANCELADA`; libera horario | `PLAZO_VENCIDO` (solo cliente), `TRANSICION_INVALIDA`, `RECURSO_NO_ENCONTRADO` |
| `POST /api/citas/{id}/reprogramar` | SECRETARIA | `{ "veterinario_id": "uuid", "inicio": "2026-12-09T09:00:00-06:00" }` | Original → `REPROGRAMADA`, nueva `AGENDADA` | `RECURSO_NO_ENCONTRADO` (cita, o veterinario inexistente o inactivo), `TRANSICION_INVALIDA`, `FUERA_DE_HORARIO`, `LIMITE_CITAS_MASCOTA`, `HORARIO_NO_DISPONIBLE`, `VALIDACION` |
| `POST /api/citas/{id}/llegada` | SECRETARIA | — | Asigna `llegada_en` | `TRANSICION_INVALIDA`, `VALIDACION` (otro día) |
| `POST /api/citas/{id}/atendida` | VETERINARIO | — | → `ATENDIDA` | `TRANSICION_INVALIDA`, `SIN_PERMISO` (cita de otro veterinario) |

- `POST /api/citas/{id}/reprogramar` responde `200` `{ "cita_original": { "id": "uuid", "estado": "REPROGRAMADA" }, "cita_nueva": CitaDetalle }`.
- `POST /api/citas/{id}/reprogramar` con rol `CLIENTE` → `403 SIN_PERMISO` (el cliente usa el apartado con `cita_origen_id`, §4.C).
- Orden de validación común: autenticación y rol → existencia/pertenencia → estado y `llegada_en` (`TRANSICION_INVALIDA`) → plazo (`PLAZO_VENCIDO`) → reglas del nuevo horario.

## 6. Casos borde

| ID | Caso | Comportamiento esperado |
|---|---|---|
| EC-C2-01 | El cliente confirma una cita que la secretaria acaba de cancelar | 409 `TRANSICION_INVALIDA`; la UI muestra MSG-16 y refresca la cita |
| EC-C2-02 | Doble clic en "Cancelar" | La segunda llamada recibe 409 `TRANSICION_INVALIDA`; la UI refresca y muestra el estado real sin error alarmante |
| EC-C2-03 | Cancelar a exactamente 4 h 00 min 00 s del inicio | Se permite (R-C2-01) |
| EC-C2-04 | Cancelar a 3 h 59 min 59 s del inicio | 422 `PLAZO_VENCIDO` |
| EC-C2-05 | Reprogramar una cita `CONFIRMADA` | La nueva nace `AGENDADA` (R-C2-05) y puede volver a recibir recordatorio si cumple R-C2-10 |
| EC-C2-06 | El apartado de reprogramación vence | La original queda intacta; MSG-02 |
| EC-C2-07 | Durante los 5 min del apartado la original queda a < 4 h | Al confirmar: se elimina el apartado y responde 422 `PLAZO_VENCIDO`; la original sigue como estaba |
| EC-C2-08 | Reprogramar con la mascota en el límite de RN-07 | Se permite: el conteo excluye la cita original |
| EC-C2-09 | La secretaria intenta cancelar o reprogramar una cita con llegada registrada | 409 `TRANSICION_INVALIDA` (R-C2-02) |
| EC-C2-10 | El cliente llega después de los 15 min y ya está `INASISTENCIA` | No se revierte. La secretaria crea una cita nueva si hay horario (`feature-core-01.md` §6); el resto del bloque ya está libre porque `INASISTENCIA` no ocupa horario (RG-05) |
| EC-C2-11 | La secretaria registra llegada y J-03 corre al mismo tiempo | Gana quien confirme primero la actualización condicional; el perdedor recibe `TRANSICION_INVALIDA` y la UI refresca |
| EC-C2-12 | Llegada registrada en una cita de otro día | 422 `VALIDACION`: "La llegada solo se registra el día de la cita." |
| EC-C2-13 | El veterinario marca atendida sin llegada registrada | 409 `TRANSICION_INVALIDA` (el botón ya viene deshabilitado por `acciones_permitidas`) |
| EC-C2-14 | Un veterinario intenta marcar la cita de otro | 403 `SIN_PERMISO` |
| EC-C2-15 | Cita agendada con menos de 24 h de anticipación | No se envía recordatorio (R-C2-10); sí la confirmación |
| EC-C2-16 | Cita ya `CONFIRMADA` | No se envía recordatorio (R-C2-10) |
| EC-C2-17 | El servidor estuvo caído y se pasó la hora del recordatorio | J-02 lo envía al reanudarse mientras `inicio > ahora` y se cumpla R-C2-10 |
| EC-C2-18 | Cliente sin correo | No se crean notificaciones (RG-18) |
| EC-C2-19 | Cliente con correo pero sin cuenta | El correo se envía sin botones (§4.D) |
| EC-C2-20 | Falla el envío del correo | `Notificacion` reintenta hasta 3 veces (5 min entre intentos); luego `FALLIDA`. La cita no cambia |
| EC-C2-21 | Enlace del correo abierto sin sesión | Redirige a login y, tras autenticarse, abre `/citas/{id}?accion=…` |
| EC-C2-22 | El enlace del correo apunta a una cita ya cancelada o atendida | Se muestra el detalle con MSG-16 y sin acciones |

## 7. Estados de interfaz

Textos exactos en `ui-ux-spec.md` §8.

| Pantalla / sección | Vacío | Cargando | Éxito | Error |
|---|---|---|---|---|
| P-05 Mis citas — pestaña Próximas | "No tienes citas próximas" (MSG-35) + botón "Agendar cita" | 3 tarjetas-esqueleto | Lista de `TarjetaCita` | Banner con "Reintentar" |
| P-05 Mis citas — pestaña Historial | "Aún no hay citas en tu historial" (MSG-36) | 3 tarjetas-esqueleto | Lista paginada con insignias de estado | Banner con "Reintentar" |
| P-06 Detalle de cita | — | Esqueleto del detalle | Detalle con insignia, datos y acciones | Cita inexistente: "No encontramos esta cita" (MSG-29) + botón "Volver a mis citas"; fallo de red: banner con "Reintentar" |
| Acción confirmar / cancelar (modal) | — | Botón del modal con indicador y deshabilitado | Toast (MSG-14 o MSG-12) y detalle actualizado | `PLAZO_VENCIDO` → MSG-03; `TRANSICION_INVALIDA` → MSG-16 + refresco; otros → MSG-10 |
| P-07 Reprogramar | Sin horarios: igual que P-04 paso 3 | Esqueletos como P-04 | Toast MSG-13 y regreso a P-06 | `APARTADO_EXPIRADO` → modal MSG-02; `PLAZO_VENCIDO` → MSG-03 |
| C-14 Panel de cita (recepción) | — | Botón de acción con indicador | Toast (MSG-18, MSG-12 o MSG-13) y panel actualizado | Mensaje del catálogo en el panel; `TRANSICION_INVALIDA` refresca |
| P-09 Mi agenda (veterinario) | "No tienes citas para este día" (MSG-38) | Esqueletos de tarjetas | Lista cronológica de citas | Banner con "Reintentar" |
| P-09 Marcar atendida | — | Botón con indicador | Toast MSG-19 e insignia "Atendida" | `TRANSICION_INVALIDA` → MSG-16 + refresco |

## 8. Criterios de aceptación

**AC-C2-01 — Confirmar asistencia (T-05)**
**Given** una cita `AGENDADA` de Luna mañana a las 09:00 **When** su dueña pulsa "Confirmar asistencia" **Then** la cita queda `CONFIRMADA`, se muestra MSG-14 y `acciones_permitidas.confirmar` pasa a `false`.

**AC-C2-02 — Plazo vencido (RN-06, criterio del PRD)**
**Given** la cita es hoy a las 3:00 p. m. **When** el cliente intenta cancelarla en línea a las 12:00 m. d. **Then** responde 422 `PLAZO_VENCIDO`, la cita sigue `AGENDADA` y la UI indica que llame a recepción (MSG-03).

**AC-C2-03 — Límite exacto del plazo (R-C2-01)**
**Given** una cita hoy a las 3:00 p. m. **When** el cliente cancela a las 11:00 a. m. en punto **Then** se permite; **and when** lo intenta a las 11:00:01 a. m. **then** responde 422 `PLAZO_VENCIDO`.

**AC-C2-04 — Recepción sin plazo (RF-06)**
**Given** la misma cita a las 12:00 m. d. **When** la secretaria la cancela **Then** queda `CANCELADA`.

**AC-C2-05 — Horario liberado (R-C2-03)**
**Given** una cita `AGENDADA` de la Dra. Solís el 2026-12-08 a las 10:00 **When** se cancela **Then** ese horario vuelve a aparecer en `GET /api/disponibilidad` y los clientes reciben `disponibilidad.cambio` en ≤ 5 s.

**AC-C2-06 — Reprogramar en línea (T-07, RG-12)**
**Given** una cita `CONFIRMADA` a más de 4 h **When** el cliente aparta un nuevo horario con `cita_origen_id` y lo confirma **Then** la original queda `REPROGRAMADA`, la nueva queda `AGENDADA` con `cita_origen_id` apuntando a la original y el mismo `canal`, todo en una sola transacción.

**AC-C2-07 — Apartado de reprogramación vencido**
**Given** un apartado de reprogramación creado a las 09:00 **When** el cliente confirma a las 09:06 **Then** responde 410 `APARTADO_EXPIRADO` y la cita original sigue en su estado y horario.

**AC-C2-08 — Plazo vencido durante el apartado (EC-C2-07)**
**Given** la original empieza a las 13:00, el cliente aparta una reprogramación a las 08:58 **When** confirma a las 09:01 (faltan menos de 4 h) **Then** responde 422 `PLAZO_VENCIDO`, el apartado se elimina y la original no cambia.

**AC-C2-09 — Reprogramar por recepción**
**Given** una cita `AGENDADA` a 2 h **When** la secretaria la reprograma a un horario libre **Then** responde 200 con `cita_original.estado = REPROGRAMADA` y `cita_nueva.estado = AGENDADA`.

**AC-C2-10 — Recordatorio (RF-07, criterio del PRD)**
**Given** una cita `AGENDADA` el 2026-12-09 a las 09:00, agendada el 2026-12-05, y el cliente tiene correo **When** el reloj llega a 2026-12-08 09:00 **Then** en ≤ 60 s se crea y envía la `Notificacion` `RECORDATORIO` con asunto "Recordatorio: cita de Luna el 09/12/2026 a las 9:00 a. m." y los botones Confirmar, Reprogramar y Cancelar.

**AC-C2-11 — Sin recordatorio si ya está confirmada**
**Given** la misma cita pero `CONFIRMADA` **When** el reloj llega a 2026-12-08 09:00 **Then** no se crea `Notificacion` `RECORDATORIO`.

**AC-C2-12 — Sin recordatorio con poca anticipación**
**Given** una cita agendada hoy a las 08:00 para las 14:00 **When** se ejecuta J-02 **Then** no se crea recordatorio y sí existe la `Notificacion` `CONFIRMACION`.

**AC-C2-13 — Fallo de correo**
**Given** el servidor SMTP rechaza los envíos **When** J-04 agota 3 intentos **Then** la `Notificacion` queda `FALLIDA` y la cita conserva su estado.

**AC-C2-14 — Registrar llegada (RF-10)**
**Given** una cita `AGENDADA` hoy **When** la secretaria pulsa "Registrar llegada" **Then** `llegada_en` se asigna, el estado no cambia, la UI muestra "En sala" y `acciones_permitidas.marcar_atendida` es `true` para el veterinario.

**AC-C2-15 — Atendida exige llegada (RG-13)**
**Given** una cita `AGENDADA` sin `llegada_en` **When** el veterinario llama a `POST /api/citas/{id}/atendida` **Then** responde 409 `TRANSICION_INVALIDA`.

**AC-C2-16 — Marcar atendida (T-08)**
**Given** una cita con llegada registrada **When** su veterinario pulsa "Marcar atendida" **Then** la cita queda `ATENDIDA`.

**AC-C2-17 — Inasistencia (RN-09)**
**Given** una cita `AGENDADA` hoy a las 10:00 sin llegada **When** el reloj marca 10:14 **Then** sigue `AGENDADA`; **and when** marca 10:15 **then** en ≤ 60 s pasa a `INASISTENCIA`.

**AC-C2-18 — Llegada evita inasistencia**
**Given** la misma cita **When** la secretaria registra la llegada a las 10:14 **Then** a las 10:15 no pasa a `INASISTENCIA`.

**AC-C2-19 — Transiciones inválidas**
**Given** una cita `CANCELADA` **When** se intenta confirmarla, reprogramarla o marcarla atendida **Then** responde 409 `TRANSICION_INVALIDA` con `detalles.estado_actual = "CANCELADA"`.

**AC-C2-20 — Aislamiento entre clientes (RNF-06)**
**Given** la cita X pertenece al cliente A **When** el cliente B intenta cancelarla **Then** responde 404 `RECURSO_NO_ENCONTRADO` y la cita no cambia.

**AC-C2-21 — Vista de citas del cliente**
**Given** Luna tiene 1 cita `AGENDADA`, 1 `ATENDIDA` y 1 `CANCELADA` **When** su dueña pide `GET /api/citas/mias?vista=proximas` **Then** recibe solo la `AGENDADA`; **and with** `vista=historial` **then** recibe la `ATENDIDA` y la `CANCELADA` en orden de `inicio` descendente.

**AC-C2-22 — Inasistencia libera el horario (RG-05)**
**Given** una cita `AGENDADA` de *Cirugía programada* de la Dra. Solís hoy de 10:00 a 12:00, sin llegada, que J-03 marcó `INASISTENCIA` a las 10:15 **When** la secretaria consulta disponibilidad a las 10:20 (*Consulta*, 30 min, Dra. Solís) **Then** se ofrece el horario de 10:30 y puede crear una cita a esa hora (201); la cita `INASISTENCIA` ya no aparece en la agenda pero sigue en el historial y en los reportes.

**AC-C2-23 — Mover una cita 15 minutos por la web (R-C2-05)**
**Given** una cita `AGENDADA` de Luna con la Dra. Solís el 2026-12-08 de 10:00 a 10:30, a más de 4 h **When** el cliente aparta 10:15 con `cita_origen_id` (consultando con `excluir_cita_id`) y confirma **Then** el apartado se crea sin error de choque; al confirmar, la original queda `REPROGRAMADA` (10:00–10:30) y la nueva `AGENDADA` (10:15–10:45); el horario 10:00–10:15 queda libre.

**AC-C2-24 — Mover una cita 15 minutos por recepción (R-C2-05)**
**Given** la misma cita **When** la secretaria llama a `POST /api/citas/{id}/reprogramar` con `inicio = 2026-12-08T10:15:00-06:00` y el mismo veterinario **Then** responde 200 con `cita_original.estado = REPROGRAMADA` y `cita_nueva` de 10:15 a 10:45 `AGENDADA`.

**AC-C2-25 — Reversión si falla la reprogramación (R-C2-05)**
**Given** el Dr. Vargas tiene una cita de 10:30 a 11:00 y Luna una cita `AGENDADA` con él de 10:00 a 10:30 **When** la secretaria intenta reprogramar la de Luna a las 10:15 con el mismo veterinario (30 min) **Then** responde 409 `HORARIO_NO_DISPONIBLE` y la cita de Luna sigue `AGENDADA` de 10:00 a 10:30, sin cita nueva.

**AC-C2-26 — Recurso inexistente al reprogramar**
**Given** una sesión `SECRETARIA` **When** llama a `POST /api/citas/{id}/reprogramar` con un `veterinario_id` inexistente o inactivo **Then** responde 404 `RECURSO_NO_ENCONTRADO` y la cita no cambia.

**AC-C2-27 — Sin correo al cancelar (R-C2-13)**
**Given** una cita `AGENDADA` de un cliente con correo **When** la secretaria la cancela **Then** queda `CANCELADA` y no se crea ninguna `Notificacion` nueva.
