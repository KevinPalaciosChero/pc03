# PC03 — Parte A: RFC de arquitectura (Pagos de Tutorías)

Entrega individual de la evaluación **PC03 — Parte A: examen teórico**, curso Arquitectura de Software, sistema base **Tutorías Universitarias**.

Esta parte es teórica (no requiere código): el producto evaluado es un mini RFC que analiza el diseño deficiente propuesto para *Pagos de Tutorías* y presenta una arquitectura alternativa.

## Contenido

- [`/PC03-RFC-Pagos-Tutorias.md`](./PC03-RFC-Pagos-Tutorias.md) — RFC completo en Markdown (11 secciones: Contexto, Problema, Riesgos, Diseño propuesto, Alternativas, Contratos, Manejo de fallas, Observabilidad, Seguridad, Despliegue y costo, Decisión).
- [`/PC03-RFC-Pagos-Tutorias.docx`](./PC03-RFC-Pagos-Tutorias.docx) — mismo contenido en formato Word.

## Resumen de la decisión

Se reemplaza la transacción distribuida síncrona propuesta (que abre una transacción en `ms-tutorias` y encadena una llamada HTTP hasta el proveedor externo) por una **Saga orquestada**: `ms-tutorias` sigue coordinando el flujo, pero el pago se resuelve mediante un checkout hospedado por el proveedor y se confirma de forma asíncrona con un único evento `tutoria.pago.procesado`, apoyada en `Idempotency-Key`, deduplicación (Inbox) y una DLQ para fallas no recuperables.
