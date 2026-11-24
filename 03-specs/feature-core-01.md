# feature-core-01 — Disponibilidad y agendamiento de citas

> Depende de `architecture-spec.md` (entidades, estados, `RN-xx`, `RG-xx`, errores, `CitaDetalle`). No redefine nada de ese archivo.

## 1. Alcance y trazabilidad

| Elemento | Detalle |
|---|---|
| Funcionalidad | Consultar horarios libres, agendar en línea con apartado temporal, agendar por teléfono/presencial, ver la agenda compartida y bloquear horarios |
| Requerimientos PRD | RF-02, RF-03, RF-04, RF-05, RF-09 |
| Reglas | RN-01, RN-02, RN-03, RN-04, RN-05, RN-07, RN-08 · RG-02…RG-06, RG-08, RG-10, RG-11, RG-12, RG-17, RG-19 |
| No funcionales | RNF-01 (disponibilidad < 2 s), RNF-02 (atomicidad y sincronía < 5 s), RNF-03 (≤ 4 pasos) |
| Métricas del PRD | Cero choques · ≥ 50 % de citas en línea · agendar en < 2 min |
| Diagramas relacionados | Secuencia "agendar cita en línea" (con caminos alt de choque y apartado vencido) · Estados de `Cita` (T-01, T-02, T-03, T-04) · DFD nivel 1 (proceso de disponibilidad y agenda) |
| Pantallas | P-04, P-07 (pasos de horario), P-08 y modales M-01, M-03 (ver `ui-ux-spec.md`) |

## 2. Actores y precondiciones

| Actor | Precondiciones |
|---|---|
| CLIENTE | Sesión iniciada; tiene ≥ 1 mascota (RF-01); existe ≥ 1 servicio y ≥ 1 veterinario activos |
| SECRETARIA | Sesión iniciada; el cliente y la mascota existen, o los crea en el mismo flujo (`POST /api/clientes`) |
| Sistema | Servicios, horario de atención, turnos y días no laborables configurados (RF-08) |

## 3. Reglas de esta funcionalidad

| ID | Regla |
|---|---|
| R-C1-01 | Un horario es **ofrecido** solo si, para algún veterinario candidato, el intervalo `[inicio, inicio + duracion_min)` cumple **todas**: (a) cabe completo dentro de una intersección `HorarioAtencion ∩ Turno` del día; (b) la fecha no es `FERIADO` ni `DIA_LIBRE` de ese veterinario; (c) no choca con ninguna cita que ocupa horario (RG-05) ni con un `Bloqueo` (se ignora la cita indicada en `excluir_cita_id`, R-C1-13); (d) los apartados vencidos se tratan como libres. (RN-01, RN-02, RN-03, RN-08) |
| R-C1-02 | Los `inicio` ofrecidos caen en múltiplos de 15 min a partir del inicio de cada ventana (RG-02). Un servicio no puede cruzar el hueco entre dos turnos partidos ni exceder el cierre de la ventana. |
| R-C1-03 | Solo para rol `CLIENTE`: se ocultan los horarios con `inicio < ahora + 2 h` (RN-05) y con fecha posterior a `hoy + 60 días` (RG-03). Para `SECRETARIA`: se ocultan solo los horarios con `inicio < ahora`. |
| R-C1-04 | Al escoger horario en línea se crea una `Cita` `APARTADA` con `apartada_hasta = ahora + 5 min` (RN-04). Si no se confirma, se elimina (T-04). |
| R-C1-05 | Un cliente tiene como máximo un apartado vigente (RG-11). Crear otro elimina el anterior **antes** de evaluar RN-07. |
| R-C1-06 | RN-07: se cuentan las citas de la mascota en `APARTADA`, `AGENDADA` o `CONFIRMADA` (RG-04). Si el conteo es ≥ 2, se rechaza. En una reprogramación se excluye la cita original del conteo. |
| R-C1-07 | Si el cliente no elige veterinario, se asigna según RG-06 al crear el apartado. Si el candidato elegido falla por choque, se intenta el siguiente candidato; si todos fallan, `HORARIO_NO_DISPONIBLE`. |
| R-C1-08 | Confirmar un apartado **no** vuelve a validar RN-05 (se validó al apartar). Sí valida que `apartada_hasta > ahora`. |
| R-C1-09 | Las citas creadas por recepción nacen `AGENDADA` directamente (sin apartado), con `canal` `TELEFONO` o `PRESENCIAL`, con veterinario obligatorio (RG-08). RN-05 no aplica (solo rol `CLIENTE`). |
| R-C1-10 | Un `Bloqueo` no puede solaparse con citas que ocupan horario ni con otro `Bloqueo` (se rechaza; recepción debe cancelar/reprogramar primero). Un apartado vigente cuenta como cita que ocupa horario. |
| R-C1-11 | La agenda muestra citas en estados que ocupan horario (RG-05) y bloqueos. `CANCELADA`, `REPROGRAMADA` e `INASISTENCIA` no aparecen. El veterinario no ve `APARTADA`. |
| R-C1-12 | **Cliente único (RG-19).** `correo` es único entre los `Cliente`. Si recepción crea un cliente con un correo que ya existe, `POST /api/clientes` responde 409 `CORREO_DUPLICADO` con `detalles.cliente_id` y la UI ofrece seleccionar a ese cliente. Si alguien se registra en línea con el correo de un `Cliente` sin cuenta, se vincula a ese `Cliente` (no se crea otro) y conserva sus mascotas y citas. |
| R-C1-13 | **Reprogramación sobre el propio horario.** `GET /api/disponibilidad` y `GET /api/disponibilidad/dias` aceptan `excluir_cita_id` (la cita que se reprograma) y la ignoran al calcular choques; así se puede mover una cita a un horario que solapa con el suyo (ej. 10:00 → 10:15). La validación RN-01 del apartado y de la confirmación también ignora esa cita (V8, §5.3). |

## 4. Consultar disponibilidad (RF-02)

### 4.1 Algoritmo

Entradas: `servicio_id`, `fecha`, `veterinario_id` (opcional).

1. Validar que el servicio y el veterinario (si viene) existen y están activos. Validar `fecha ≥ hoy` y, para `CLIENTE`, `fecha ≤ hoy + 60 días`.
2. Candidatos = el veterinario indicado, o todos los activos.
3. Si la fecha es `FERIADO` → resultado vacío.
4. Para cada candidato: si tiene `DIA_LIBRE` en la fecha → se descarta. Ventanas = `HorarioAtencion(dia) ∩ Turno(candidato, dia)`.
5. Generar inicios cada 15 min en cada ventana donde `inicio + duracion_min ≤ fin de la ventana`.
6. Quitar los inicios cuyo intervalo choque (semiabierto) con citas que ocupan horario o bloqueos del candidato (apartados vencidos = libres; la cita `excluir_cita_id`, si viene, se ignora).
7. Aplicar R-C1-03 (anticipación mínima o `inicio < ahora`).
8. Agrupar por `inicio`: cada horario lista los veterinarios que lo tienen libre. Ordenar por `inicio` ascendente.

### 4.2 Contrato

`GET /api/disponibilidad?servicio_id={uuid}&fecha=2026-12-08&veterinario_id={uuid?}&excluir_cita_id={uuid?}` — roles `CLIENTE`, `SECRETARIA`

```json
{
  "fecha": "2026-12-08",
  "servicio": { "id": "uuid", "nombre": "Consulta", "duracion_min": 30 },
  "slots": [
    { "inicio": "2026-12-08T08:00:00-06:00", "fin": "2026-12-08T08:30:00-06:00",
      "veterinarios": [ { "id": "uuid", "nombre_completo": "Dra. María Solís" } ] }
  ]
}
```

Sin horarios → 200 con `"slots": []`.

`GET /api/disponibilidad/dias?servicio_id={uuid}&mes=2026-12&veterinario_id={uuid?}&excluir_cita_id={uuid?}` → `{ "mes": "2026-12", "dias": ["2026-12-02", "2026-12-03"] }` (fechas con ≥ 1 horario; alimenta el calendario).

`excluir_cita_id` es opcional y solo se usa al reprogramar: debe ser una cita `AGENDADA` o `CONFIRMADA` (del cliente, si el rol es `CLIENTE`); si no, `RECURSO_NO_ENCONTRADO`.

Errores: `VALIDACION` (parámetros o fecha pasada), `VENTANA_EXCEDIDA` (solo `CLIENTE`), `RECURSO_NO_ENCONTRADO` (servicio, veterinario o `excluir_cita_id` inexistente/inactivo).
Rendimiento: p95 < 2 s (RNF-01) con hasta 5 veterinarios y 10 000 citas en BD.

## 5. Agendar en línea (RF-03)

### 5.1 Flujo principal (rol `CLIENTE`, pantalla P-04)

1. **Paso 1:** elige mascota y servicio.
2. **Paso 2:** elige veterinario (o "Cualquiera disponible") y fecha.
3. **Paso 3:** ve los horarios libres (§4) y elige uno → `POST /api/citas/apartados`. El sistema aparta el horario 5 min.
4. **Paso 4:** ve el resumen con veterinario asignado, escribe el motivo (opcional) y confirma → `POST /api/citas/{id}/confirmar-apartado`.
5. La cita queda `AGENDADA`, aparece en la agenda de secretarias y veterinario (§7) y se crea la `Notificacion` `CONFIRMACION` (si hay correo).

### 5.2 Apartar

`POST /api/citas/apartados` — rol `CLIENTE`

```json
{ "mascota_id": "uuid", "servicio_id": "uuid", "veterinario_id": null,
  "inicio": "2026-12-08T10:00:00-06:00", "cita_origen_id": null }
```

`201`:

```json
{ "cita": { "...CitaDetalle con estado APARTADA y apartada_hasta": "..." }, "segundos_restantes": 300 }
```

**Orden de validación (se detiene en el primer fallo):**

| # | Validación | Error |
|---|---|---|
| V1 | Sesión y rol `CLIENTE` | `NO_AUTENTICADO` / `SIN_PERMISO` |
| V2 | Esquema; `inicio` en cuadrícula de 15 min (RG-02) | `VALIDACION` |
| V3 | La mascota pertenece al cliente; servicio y veterinario (si viene) activos | `RECURSO_NO_ENCONTRADO` |
| V4 | Si hay `cita_origen_id`: es del cliente, estado `AGENDADA`/`CONFIRMADA`, `llegada_en` NULL, misma mascota y servicio, y `origen.inicio − ahora ≥ 4 h` | `RECURSO_NO_ENCONTRADO` / `TRANSICION_INVALIDA` / `VALIDACION` / `PLAZO_VENCIDO` |
| V5 | `inicio ≥ ahora + 2 h` (RN-05) y la **fecha** de `inicio` ≤ hoy + 60 días (RG-03) | `ANTICIPACION_INSUFICIENTE` / `VENTANA_EXCEDIDA` |
| V6 | El intervalo cae en una ventana válida (RN-03) | `FUERA_DE_HORARIO` |
| V7 | Eliminar apartado previo del cliente (RG-11); conteo RN-07 < 2 | `LIMITE_CITAS_MASCOTA` |
| V8 | Transacción con bloqueo (arch. §5.4); RN-01 sin choque (ignorando la cita `cita_origen_id`, si viene); si no hay veterinario, aplicar RG-06 con reintento por candidato | `HORARIO_NO_DISPONIBLE` |
| V9 | Insertar `APARTADA` con `canal = WEB` | — |

### 5.3 Confirmar el apartado

`POST /api/citas/{id}/confirmar-apartado` — rol `CLIENTE` (dueño) — cuerpo `{ "motivo": "Vómito desde ayer" }` (opcional, 0–500).

Precondiciones y errores:
- `id` inexistente o ya eliminado, o `apartada_hasta ≤ ahora` → `410 APARTADO_EXPIRADO` (se elimina el registro si aún existe).
- Cita de otro cliente → `404 RECURSO_NO_ENCONTRADO`.
- Estado distinto de `APARTADA` → `409 TRANSICION_INVALIDA`.
- Si tiene `cita_origen_id`: el original debe seguir `AGENDADA`/`CONFIRMADA`, sin `llegada_en` y con `inicio − ahora ≥ 4 h`; si no, se elimina el apartado y responde `TRANSICION_INVALIDA` o `PLAZO_VENCIDO`.

Efectos (una sola transacción, en este orden; si un paso falla se revierte todo): (0) bajo bloqueo del veterinario, se revalida RN-01 ignorando la cita original; (1) si hay `cita_origen_id`, el original pasa a `REPROGRAMADA` (T-07, RG-12); (2) `estado → AGENDADA`, `apartada_hasta → NULL`, `agendada_en = ahora`, se guarda `motivo`; (3) se crea `Notificacion` `CONFIRMACION` si el cliente tiene correo; (4) se emiten eventos (arch. §5.5).

`200` → `{ "cita": CitaDetalle (estado AGENDADA) }`.

### 5.4 Liberar el apartado

`DELETE /api/citas/{id}/apartado` — rol `CLIENTE` (dueño). `204` si se eliminó o ya no existía. Cita de otro cliente o en estado ≠ `APARTADA` → `404` / `409 TRANSICION_INVALIDA`.

## 6. Agendar por teléfono o presencial (RF-04, RF-05)

### 6.1 Flujo (rol `SECRETARIA`, modal M-01 desde P-08)

1. Busca al cliente (`GET /api/clientes?q=`) o lo crea (`POST /api/clientes`). Si el correo ya existe, recibe 409 `CORREO_DUPLICADO` y se le ofrece seleccionar al cliente existente (R-C1-12).
2. Elige una mascota del cliente o crea una (`POST /api/clientes/{id}/mascotas`).
3. Elige servicio, **veterinario**, fecha y horario (usa `GET /api/disponibilidad` con el veterinario fijo).
4. Elige canal (`TELEFONO` / `PRESENCIAL`), escribe motivo (opcional) y confirma.

### 6.2 Contrato

`POST /api/citas` — rol `SECRETARIA`

```json
{ "cliente_id": "uuid", "mascota_id": "uuid", "servicio_id": "uuid", "veterinario_id": "uuid",
  "inicio": "2026-12-08T10:00:00-06:00", "canal": "TELEFONO", "motivo": "Control" }
```

`201` → `{ "cita": CitaDetalle (estado AGENDADA) }`.

Validaciones en orden: V1 rol; V2 esquema y cuadrícula; V3 la mascota pertenece al cliente, servicio y veterinario activos (`RECURSO_NO_ENCONTRADO`); V4 `inicio ≥ ahora` (`VALIDACION`); V5 RN-03 (`FUERA_DE_HORARIO`); V6 RN-07 (`LIMITE_CITAS_MASCOTA`); V7 RN-01 con bloqueo transaccional (`HORARIO_NO_DISPONIBLE`). La secretaria **no** tiene excepciones a RN-01, RN-03 ni RN-07.

Efecto adicional: si el cliente tiene correo, se crea `Notificacion` `CONFIRMACION`.

## 7. Agenda compartida (RF-04) y bloqueos (RF-09)

### 7.1 Leer la agenda

`GET /api/agenda?fecha=2026-12-08&vista=dia&veterinario_id={uuid?}` — roles `SECRETARIA`, `VETERINARIO`

- `vista = dia | semana`. `semana` = lunes a domingo de la semana que contiene `fecha`.
- `SECRETARIA`: todos los veterinarios activos o el filtrado. `VETERINARIO`: solo el propio (el parámetro se ignora) y solo `vista = dia`; `vista = semana` → `VALIDACION`.

```json
{
  "vista": "dia", "desde": "2026-12-08", "hasta": "2026-12-08", "servidor_ahora": "2026-12-02T09:15:00-06:00",
  "veterinarios": [ {
    "id": "uuid", "nombre_completo": "Dra. María Solís",
    "turnos": [ { "dia_semana": 2, "hora_inicio": "08:00", "hora_fin": "16:00" } ],
    "dias_no_laborables": [ { "fecha": "2026-12-08", "tipo": "DIA_LIBRE", "motivo": "Capacitación" } ],
    "items": [
      { "tipo": "CITA", "cita": { "...CitaDetalle" } },
      { "tipo": "BLOQUEO", "id": "uuid", "inicio": "2026-12-08T12:00:00-06:00", "fin": "2026-12-08T13:00:00-06:00", "motivo": "Reunión" }
    ]
  } ]
}
```

Para `VETERINARIO`, `CitaDetalle` no incluye `cliente` (RG-17).

### 7.2 Sincronía

El cambio en cualquier cita o bloqueo emite `agenda.cambio` (staff) y `disponibilidad.cambio` (clientes) (arch. §5.5). La UI de agenda vuelve a pedir `GET /api/agenda` del rango visible al recibir el evento. Cualquier cita, sea `WEB`, `TELEFONO` o `PRESENCIAL`, bloquea el horario al instante porque todas comparten la misma tabla y el mismo control de choques (RF-05).

### 7.3 Bloqueos

`POST /api/bloqueos` — rol `SECRETARIA`

```json
{ "veterinario_id": "uuid", "inicio": "2026-12-08T12:00:00-06:00", "fin": "2026-12-08T13:00:00-06:00", "motivo": "Reunión" }
```

Validaciones: múltiplos de 15 min; `inicio < fin`; mismo día calendario; `inicio ≥ ahora`; sin choque con citas que ocupan horario (`BLOQUEO_CON_CITAS`, con `detalles.citas`) ni con otro bloqueo (`BLOQUEO_SOLAPADO`). No se exige que esté dentro de un turno. `201` → bloqueo creado.

`DELETE /api/bloqueos/{id}` — rol `SECRETARIA` → `204`; el horario vuelve a quedar disponible.

## 8. Casos borde

| ID | Caso | Comportamiento esperado |
|---|---|---|
| EC-C1-01 | Dos clientes eligen el mismo horario casi a la vez | El primero en obtener el bloqueo transaccional recibe el apartado (201). El segundo recibe 409 `HORARIO_NO_DISPONIBLE`; la UI muestra "Ese horario acaba de ser reservado" y recarga horarios del mismo día y servicio |
| EC-C1-02 | El apartado vence mientras el cliente está en el paso 4 | Al confirmar: 410 `APARTADO_EXPIRADO`. La UI vuelve al paso 3 con horarios recargados |
| EC-C1-03 | El cliente abre dos pestañas y aparta en ambas | El segundo apartado elimina el primero (RG-11). Confirmar el primero → 410 `APARTADO_EXPIRADO` |
| EC-C1-04 | Una secretaria bloquea un horario mientras el cliente está en el paso 3 | El evento `disponibilidad.cambio` actualiza la grilla; si ya había elegido ese horario, al apartar recibe 409 `HORARIO_NO_DISPONIBLE` |
| EC-C1-05 | Sin veterinario elegido y solo uno tiene libre el horario | Se asigna ese veterinario (R-C1-07) |
| EC-C1-06 | Sin veterinario elegido y varios libres | Se asigna por RG-06 (menos citas ese día; empate por `creado_en`) |
| EC-C1-07 | Fecha de hoy, cliente | Se ocultan horarios con `inicio < ahora + 2 h`. La secretaria sí ve horarios desde `ahora` |
| EC-C1-08 | Cliente sin mascotas | El paso 1 muestra estado vacío con acción "Registrar mascota" |
| EC-C1-09 | Mascota con 2 citas activas | En el paso 1 la mascota aparece deshabilitada (`citas_activas = 2`). Si se fuerza la llamada: 422 `LIMITE_CITAS_MASCOTA` |
| EC-C1-10 | Servicio de 120 min cerca del cierre del turno | No se ofrece un `inicio` cuyo `fin` exceda la ventana (R-C1-02) |
| EC-C1-11 | Veterinario con turno partido (ej. 08:00–12:00 y 14:00–17:00) | Ningún horario cruza el hueco 12:00–14:00 |
| EC-C1-12 | Reprogramar a un horario que solapa con la propia cita original (ej. 10:00 → 10:15) | Se ofrece (la consulta usa `excluir_cita_id`) y se permite: al confirmar, el original pasa a `REPROGRAMADA` **antes** de que la nueva quede `AGENDADA`, en la misma transacción. Si algo falla se revierte todo y el original no cambia |
| EC-C1-13 | Reprogramación de una mascota con 2 citas activas | Se permite: el conteo RN-07 excluye la cita original (R-C1-06) |
| EC-C1-14 | Bloqueo sobre un apartado vigente | 409 `BLOQUEO_CON_CITAS` (el apartado cuenta como ocupante); reintentar tras ≤ 5 min |
| EC-C1-15 | Secretaria crea cita para cliente sin correo | Se crea sin `Notificacion` (RG-18) |
| EC-C1-16 | Sesión expirada durante el asistente | 401 → login → vuelve a `/agendar` con el asistente vacío. El apartado previo vence solo |
| EC-C1-17 | Navegador en otra zona horaria | La UI muestra siempre `America/Costa_Rica` (RG-01) |
| EC-C1-18 | Secretaria intenta agendar en el pasado | 422 `VALIDACION` en `inicio` |
| EC-C1-19 | Feriado o día libre del veterinario | `slots = []`; el día aparece deshabilitado en el calendario |
| EC-C1-20 | Un cliente creado por recepción se registra después en línea con el mismo correo | Se vincula al `Cliente` existente (R-C1-12, RG-19); no hay un segundo `Cliente` |
| EC-C1-21 | Recepción crea un cliente con un correo que ya existe | 409 `CORREO_DUPLICADO` con `detalles.cliente_id`; la UI ofrece seleccionarlo |

## 9. Estados de interfaz

Textos exactos en `ui-ux-spec.md` §8.

| Pantalla / sección | Vacío | Cargando | Éxito | Error |
|---|---|---|---|---|
| P-04 Paso 1 (mascotas y servicios) | Sin mascotas: "Aún no tienes mascotas registradas" (MSG-32) + botón "Registrar mascota" | Esqueletos de 2 tarjetas | Tarjetas seleccionables | Banner MSG-10 + "Reintentar" |
| P-04 Paso 2 (fecha) | Mes sin disponibilidad: todos los días deshabilitados + texto "No hay fechas disponibles este mes" (MSG-33) | Calendario con esqueleto | Días con disponibilidad habilitados | Banner con "Reintentar" |
| P-04 Paso 3 (horarios) | "No hay horarios disponibles para este día" (MSG-34) + botón "Probar con otra fecha" (y "Ver todos los veterinarios" si había uno elegido) | 8 chips-esqueleto | Chips de hora agrupados Mañana / Tarde | Banner con "Reintentar"; errores 409 de apartado: ver abajo |
| P-04 Paso 3 → apartar | — | Botón "Continuar" con indicador; deshabilitado | Pasa al paso 4 con temporizador | `HORARIO_NO_DISPONIBLE`: MSG-01 y grilla recargada. `ANTICIPACION_INSUFICIENTE`, `LIMITE_CITAS_MASCOTA`, `FUERA_DE_HORARIO`: mensaje del catálogo, sin avanzar |
| P-04 Paso 4 (confirmar) | — | Botón "Confirmar cita" con indicador | Pantalla "¡Cita agendada!" (MSG-11) | `APARTADO_EXPIRADO`: modal MSG-02 con botón "Elegir otro horario" |
| P-08 Agenda | "No hay citas para este día" (MSG-37) sobre la cuadrícula con turnos visibles | Esqueleto de columnas | Cuadrícula con citas y bloqueos + indicador "Actualizado hace N s" | Banner "No se pudo actualizar la agenda. Reintentando…" (MSG-31) + "Reintentar"; conserva la última vista |
| M-01 Nueva cita | Búsqueda sin resultados: "No encontramos clientes" (MSG-44) + "Registrar nuevo cliente" | Indicador en el botón "Agendar cita" | Se cierra el modal y toast MSG-23 | Mensaje del error bajo el campo o en el modal; no se cierra |
| M-03 Bloquear horario | — | Indicador en el botón "Bloquear" | Se cierra y toast MSG-24 | `BLOQUEO_CON_CITAS` / `BLOQUEO_SOLAPADO`: mensaje en el modal |

## 10. Criterios de aceptación

**AC-C1-01 — Cálculo de disponibilidad (RF-02, RN-02, RN-03, RN-08)**
**Given** la Dra. Solís tiene turno martes 08:00–16:00, la veterinaria atiende ese martes 08:00–17:00, tiene una cita 10:00–10:30 y un bloqueo 12:00–13:00 el 2026-12-08, y hoy es 2026-12-02
**When** se consulta `GET /api/disponibilidad` con servicio *Consulta* (30 min), esa fecha y su `veterinario_id`
**Then** se devuelven 23 horarios: 08:00 a 09:30 (7), 10:30 a 11:30 (5) y 13:00 a 15:30 (11), cada 15 min, y no aparecen 09:45, 10:00, 10:15, 11:45, 12:00–12:45 ni 15:45.

**AC-C1-02 — Choque de horario por teléfono (RF-05, criterio del PRD)**
**Given** la secretaria registró por teléfono una cita con la Dra. Solís el martes 2026-12-08 a las 10:00
**When** un cliente consulta disponibilidad web para ese martes
**Then** las 10:00 con la Dra. Solís ya no aparece disponible.

**AC-C1-03 — Apartado simultáneo (RF-03, criterio del PRD)**
**Given** dos clientes eligen el mismo horario casi al mismo tiempo
**When** el primero lo aparta
**Then** el segundo recibe 409 `HORARIO_NO_DISPONIBLE`, ve "Ese horario acaba de ser reservado" y se le muestran otras opciones.

**AC-C1-04 — Apartado temporal (RN-04)**
**Given** un cliente aparta el 2026-12-08 a las 10:00 con la Dra. Solís a las 09:00:00 del 2026-12-02
**When** otro cliente consulta disponibilidad para ese día a las 09:03
**Then** las 10:00 de la Dra. Solís no aparece; **y when** consulta a las 09:05:01 sin que se haya confirmado **then** vuelve a aparecer.

**AC-C1-05 — Confirmar apartado (T-03)**
**Given** un apartado vigente **When** el cliente envía `confirmar-apartado` con `motivo` **Then** la cita queda `AGENDADA`, `apartada_hasta` es NULL, aparece en la agenda de secretaria y veterinario y se crea la `Notificacion` `CONFIRMACION`.

**AC-C1-06 — Apartado vencido (RN-04)**
**Given** un apartado creado a las 09:00 **When** el cliente intenta confirmarlo a las 09:05:30 **Then** responde 410 `APARTADO_EXPIRADO` y el horario queda libre.

**AC-C1-07 — Anticipación mínima (RN-05)**
**Given** son las 09:00 del 2026-12-02 **When** un cliente intenta apartar ese día a las 10:30 **Then** responde 422 `ANTICIPACION_INSUFICIENTE`; **and when** aparta a las 11:00 exactas **then** se permite.

**AC-C1-08 — Límite por mascota (RN-07)**
**Given** Luna tiene 1 cita `AGENDADA` y 1 `CONFIRMADA` **When** su dueña intenta apartar otra para Luna **Then** responde 422 `LIMITE_CITAS_MASCOTA`.

**AC-C1-09 — Un solo apartado por cliente (RG-11)**
**Given** un cliente tiene un apartado vigente a las 10:00 **When** aparta otro a las 11:00 **Then** el de las 10:00 se elimina y vuelve a estar disponible.

**AC-C1-10 — Asignación automática (RG-06)**
**Given** la Dra. Solís tiene 3 citas que ocupan horario ese día y el Dr. Vargas 1, ambos libres a las 14:00 **When** un cliente aparta las 14:00 sin elegir veterinario **Then** se asigna al Dr. Vargas y el resumen lo muestra.

**AC-C1-11 — Servicio largo (R-C1-02)**
**Given** *Cirugía programada* (120 min) y turno de la Dra. Solís hasta las 16:00 **When** se consulta disponibilidad **Then** el último horario ofrecido es 14:00.

**AC-C1-12 — Cita por recepción (RF-04)**
**Given** una secretaria con cliente y mascota existentes **When** crea una cita `TELEFONO` válida **Then** responde 201 con estado `AGENDADA`, sin apartado, y el horario deja de aparecer en la web en ≤ 5 s.

**AC-C1-13 — Recepción sin excepciones (RN-01, RN-03)**
**Given** un horario ocupado o fuera de turno **When** la secretaria intenta crear la cita **Then** responde 409 `HORARIO_NO_DISPONIBLE` o 422 `FUERA_DE_HORARIO`, respectivamente.

**AC-C1-14 — Bloqueo (RN-08, RF-09)**
**Given** el Dr. Vargas sin citas el 2026-12-08 de 14:00 a 15:00 **When** la secretaria crea un bloqueo de 14:00 a 15:00 **Then** responde 201 y esas horas dejan de ofrecerse en la web.

**AC-C1-15 — Bloqueo con cita existente**
**Given** el Dr. Vargas tiene una cita 14:30–15:00 **When** la secretaria intenta bloquear 14:00–15:00 **Then** responde 409 `BLOQUEO_CON_CITAS` con el id de esa cita y no crea el bloqueo.

**AC-C1-16 — Veterinario ve solo lo suyo (RG-17)**
**Given** una sesión `VETERINARIO` (Dra. Solís) **When** pide `GET /api/agenda` **Then** solo recibe su bloque, sin `cliente` en las citas y sin apartados; y con `vista = semana` responde 422 `VALIDACION`.

**AC-C1-17 — Sincronía (RNF-02)**
**Given** dos secretarias con la agenda abierta **When** una crea una cita **Then** la otra la ve en ≤ 5 s sin recargar.

**AC-C1-18 — Reprogramación en línea**
**Given** una cita `AGENDADA` a más de 4 h **When** el cliente aparta un nuevo horario con `cita_origen_id` **Then** el original sigue `AGENDADA` hasta que confirme (ver `feature-core-02.md`).

**AC-C1-19 — Mover una cita 15 minutos (RF-06, R-C1-13)**
**Given** Luna tiene una cita `AGENDADA` con la Dra. Solís el 2026-12-08 de 10:00 a 10:30, a más de 4 h **When** el cliente consulta `GET /api/disponibilidad` con `excluir_cita_id` igual a esa cita **Then** `10:15` aparece disponible; **and without** `excluir_cita_id` **then** `10:15` no aparece.

**AC-C1-20 — Cliente creado por recepción que luego se registra (RG-19)**
**Given** recepción creó al cliente Ana (`ana@correo.com`, sin cuenta) con la mascota Luna y una cita `AGENDADA` **When** Ana se registra en línea con `ana@correo.com` **Then** no se crea otro `Cliente`: el existente queda con `usuario_id` de la nueva cuenta, y Luna y la cita aparecen en su P-03 y P-05.

**AC-C1-21 — Correo duplicado por recepción (R-C1-12)**
**Given** existe el `Cliente` Ana con `ana@correo.com` **When** la secretaria intenta crear otro cliente con `ANA@correo.com` **Then** responde 409 `CORREO_DUPLICADO` con `detalles.cliente_id` de Ana y no se crea el cliente.
