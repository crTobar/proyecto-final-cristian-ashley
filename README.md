# VetCitas – Sistema Web de Citas para Veterinaria

**Proyecto final – Análisis y Diseño de Sistemas (ADS1, 3Q 2026)**
**Autores:** Cristian Martínez y Ashley Retana
**Docente:** Wilian Martínez

## Resumen

Las citas de la veterinaria se piden por teléfono, WhatsApp o en el local y se anotan a mano, lo que provoca choques de horario, citas con veterinarios que no atienden ese día y espacios perdidos por inasistencias. VetCitas es un sistema web donde el cliente agenda en línea viendo solo los horarios libres, y las secretarias usan la misma agenda sincronizada para las citas por teléfono o presenciales. Incluye apartado temporal del horario, recordatorios por correo, registro de llegada, inasistencias automáticas y reportes.

## Estructura

| Carpeta / archivo | Contenido |
|---|---|
| `01-PRD.md` | Documento de requerimientos del producto (12 RF, 6 RNF, 10 reglas de negocio) |
| `02-diagramas/` | Un archivo por requerimiento (`RF-01` a `RF-12`), cada uno con los 4 diagramas: casos de uso, DFD (nivel 0 y 1), secuencia y estados. En `.excalidraw` y `.png` |
| `03-specs/` | `architecture-spec.md`, `feature-core-01.md`, `feature-core-02.md`, `ui-ux-spec.md` |
| `04-evidencia-ia/` | `prompts-utilizados.md`, `enlace-prototipo.md` y capturas |

## Prototipo

- **Vista previa navegable:** https://stitch.google.com/preview/16056898836837622110?node-id=848ae421ad79498cbe0f74062796931e
- **Proyecto en Stitch:** https://stitch.google.com/projects/16056898836837622110
- Generado con Google Stitch a partir de las specs. El detalle de los prompts y las iteraciones está en `04-evidencia-ia/prompts-utilizados.md`.

## Herramientas usadas

| Herramienta | Uso |
|---|---|
| Excalidraw | Diagramas (casos de uso, DFD, secuencia y estados por requerimiento) |
| Claude | Redacción y revisión de las specs |
| Google Stitch | Prototipo visual navegable |

## Trazabilidad

| Requerimiento | Spec principal | Pantallas |
|---|---|---|
| RF-01 Registro y mascotas | architecture §3.6 | P-01, P-02, P-03 |
| RF-02 Consultar disponibilidad | feature-core-01 §4 | P-04 |
| RF-03 Agendar en línea | feature-core-01 §5 | P-04 |
| RF-04 Agenda sincronizada | feature-core-01 §7 | P-08 |
| RF-05 Validación de choques | architecture §3.3, §5.4 · feature-core-01 | P-04, P-08 |
| RF-06 Reprogramar y cancelar | feature-core-02 §4.B–4.C | P-05, P-06, P-07, P-08 |
| RF-07 Confirmaciones y recordatorios | feature-core-02 §4.D | P-06 |
| RF-08 Configuración | architecture §3.6 | P-10 |
| RF-09 Bloquear horarios | feature-core-01 §7.3 | P-08 |
| RF-10 Llegada y atención | feature-core-02 §4.E–4.G | P-08, P-09 |
| RF-11 Historial | architecture §3.6 | P-05, M-05 |
| RF-12 Reportes | architecture §3.6 | P-11 |
