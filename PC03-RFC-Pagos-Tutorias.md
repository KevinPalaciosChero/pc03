# RFC-2026-001 · Pagos de Tutorías

*Rediseño arquitectónico del flujo de pago en Tutorías Universitarias*

| Campo | Valor |
| --- | --- |
| Curso | Arquitectura de Software |
| Evaluación | PC03 — Parte A: examen teórico (mini RFC) |
| Sistema base | Tutorías Universitarias |
| Funcionalidad | Pagos de Tutorías (nuevo servicio `ms-pagos`) |
| Estado del RFC | Propuesto para revisión técnica |

---

## 1. Contexto

Tutorías Universitarias es un sistema de microservicios donde un estudiante solicita una tutoría con un tutor disponible. Hoy el flujo funciona así: `ms-auth` entrega un token JWT; `ms-tutorias` orquesta la solicitud y valida al estudiante y al tutor llamando por HTTP a `ms-usuarios`; luego llama a `ms-agenda` para bloquear el horario (si ya estaba bloqueado, `ms-agenda` responde `409`); si el bloqueo se logra, `ms-tutorias` registra la tutoría y publica un evento a RabbitMQ, que `ms-notificaciones` consume para avisar al estudiante. El `tracking-dashboard` y Prometheus/Grafana permiten observar este flujo en tiempo real.

La nueva funcionalidad, Pagos de Tutorías, exige que una tutoría solo se confirme cuando exista un pago válido y trazable. Se incorpora `ms-pagos`, que se comunica con proveedores externos de tarjeta (Visa, Mastercard). Estos proveedores son un tercero no confiable en tiempo: pueden demorar más de lo esperado, rechazar, aprobar tarde, enviar respuestas duplicadas o notificar por un callback que llega después de que el cliente ya se cansó de esperar.

Restricciones relevantes para este diseño:

- El sistema ya combina llamadas HTTP síncronas (validaciones) con eventos asíncronos por RabbitMQ (notificaciones); este RFC mantiene ese mismo estilo mixto, ya validado en el proyecto.
- `ms-agenda` ya usa un bloqueo temprano con conflicto `409`; el pago debe respetar esa misma lógica: primero se asegura el horario, después se cobra.
- El entorno local usa Docker Compose, Toxiproxy para simular fallas y Prometheus/Grafana para métricas; el diseño en la nube debe conservar esa misma capacidad de observar fallas.
- Ningún proveedor externo debe controlar el tiempo de vida de una transacción interna de base de datos.

## 2. Problema

El diseño propuesto abre una transacción de base de datos en `ms-tutorias` y, sin cerrarla, hace una llamada HTTP en cadena hasta el proveedor externo de tarjetas. El problema no es que "el orden esté mal", sino que se mezclan dos mundos que no deberían mezclarse: una transacción local, que debería durar milisegundos y estar bajo control total del sistema, con una llamada de red a un tercero, cuya duración es impredecible y está fuera de nuestro control.

Una forma sencilla de explicarlo en la sustentación: es como si un cajero de banco dejara la bóveda abierta y a todo el personal esperando mientras llama por teléfono a otro banco para confirmar algo que puede tardar varios minutos. Mientras la transacción sigue abierta, la fila de la tutoría queda bloqueada (lock), cualquier otro proceso que necesite leer o escribir esa fila tiene que esperar, y si el proveedor tarda más que el timeout del cliente HTTP, el usuario puede recibir un error incluso si el pago terminó aprobándose segundos después.

**Idea central para defender: el orden aparente en el código no elimina las fallas parciales de un sistema distribuido.** El diseño propuesto no previene timeouts, dobles cobros por reintento, ni estados en los que el pago se aprueba después de que `ms-tutorias` ya abandonó o canceló la solicitud.

## 3. Riesgos

### Riesgos técnicos

- Transacción larga y bloqueo de recursos: la fila de la tutoría queda con lock mientras se espera al proveedor externo, afectando la concurrencia de todo el sistema.
- Acoplamiento temporal: `ms-tutorias`, `ms-pagos` y el proveedor deben estar disponibles y responder rápido, todos a la vez, o la operación completa falla.
- Timeout ambiguo: si el cliente HTTP corta la espera, no se sabe si el proveedor aprobó o no; el estado queda indefinido.
- Doble cobro: si el cliente reintenta, "se ejecuta todo el flujo de nuevo" según el diseño propuesto, incluyendo un nuevo cargo, porque no existe idempotencia.
- Estados parciales: agenda bloqueada pero pago nunca confirmado, o pago aprobado pero la tutoría ya no existe porque `ms-tutorias` revirtió su transacción.
- Dificultad de diagnóstico: sin un identificador de correlación compartido entre servicios, es difícil reconstruir en qué paso exacto ocurrió una falla.

### Riesgos de negocio

- Pérdida de confianza del estudiante si se le cobra dos veces o si paga y no recibe la tutoría.
- Reclamos y contracargos por pagos aprobados tarde, cuando ya no hay tutoría asociada.
- Riesgo de cumplimiento (tipo PCI-DSS) si datos sensibles de tarjeta llegan a servicios que no deberían conocerlos, como `ms-tutorias` o los logs.
- Costo operativo alto: transacciones largas combinadas con reintentos sin control pueden saturar la base de datos en horas pico (por ejemplo, matrícula).

## 4. Diseño propuesto

Se propone reemplazar la transacción distribuida encadenada por una **Saga orquestada**: `ms-tutorias` conserva su rol actual de orquestador (igual que hoy coordina agenda y notificación), pero el pago se resuelve como un proceso propio, con estados explícitos, y no como una llamada bloqueante dentro de una transacción.

### Responsabilidades por servicio

| Servicio | Responsabilidad en Pagos de Tutorías |
| --- | --- |
| `ms-tutorias` | Orquesta el flujo de negocio. Crea la tutoría en estado `PENDIENTE_PAGO`. Nunca conoce datos de tarjeta. Espera confirmación de pago vía evento. Decide `CONFIRMADA` o `CANCELADA_POR_PAGO`. |
| `ms-pagos` | Dueño exclusivo de la comunicación con el proveedor externo. Gestiona la sesión de checkout, el webhook y los estados del intento de pago. Nunca persiste el número completo de tarjeta. |
| `ms-agenda` | Conserva su bloqueo optimista actual (`409` si el horario ya está tomado). El bloqueo ocurre antes de intentar el cobro. |
| `ms-notificaciones` | Sigue consumiendo eventos de RabbitMQ; ahora también reacciona al resultado del pago. |
| Proveedor externo (Visa/Mastercard) | Actor no confiable en tiempo. Confirma por webhook, de forma asíncrona, tras un checkout hospedado por él mismo. |

### Flujo principal

1. El cliente envía `POST /api/v1/tutorias` con `Idempotency-Key` en el header y `{ estudianteId, tutorId, horarioId }`.
2. `ms-tutorias` valida el JWT y valida estudiante/tutor con `ms-usuarios` (llamada síncrona y rápida, igual que hoy).
3. `ms-tutorias` pide a `ms-agenda` bloquear el horario. Si hay conflicto, responde `409` y el pago ni siquiera se intenta.
4. Si el horario se bloqueó, `ms-tutorias` crea la tutoría en estado `PENDIENTE_PAGO` (transacción local corta) y hace una llamada síncrona pero breve a `ms-pagos` para abrir una sesión de checkout. Esta llamada no espera al proveedor de tarjeta; solo genera un token/URL de pago.
5. `ms-tutorias` responde al cliente con `202 Accepted`: `{ tutoriaId, estado: PENDIENTE_PAGO, paymentUrl }`. El navegador del estudiante redirige a `paymentUrl`.
6. El estudiante completa el pago en la página hospedada por la pasarela (fuera de nuestros servicios). Ningún dato de tarjeta pasa por `ms-tutorias` ni por `ms-pagos`.
7. El proveedor confirma el resultado mediante un webhook a `ms-pagos` (`POST /api/v1/pagos/webhook`), aprobado o rechazado, o no confirma nada dentro del plazo máximo definido.
8. `ms-pagos` valida la firma del webhook, actualiza su propio registro de intento y publica un único evento `tutoria.pago.procesado` con el campo `resultado` en `APROBADO` o `RECHAZADO`.
9. `ms-tutorias` consume ese evento (con deduplicación vía Inbox, usando `eventoId`) y transiciona la tutoría a `CONFIRMADA` o `CANCELADA_POR_PAGO`, liberando el bloqueo de agenda en este último caso.
10. `ms-notificaciones` consume el evento final y avisa al estudiante, igual que hoy.
11. Si el estudiante abandona el checkout o el proveedor nunca confirma dentro del plazo máximo (por ejemplo 15 minutos), un job de expiración marca el intento como `EXPIRADO`, `ms-pagos` publica `tutoria.pago.procesado` con `resultado: RECHAZADO` y `motivo: EXPIRADO`, y `ms-tutorias` compensa liberando la agenda.

Con este diseño ninguna transacción de base de datos queda abierta esperando una red externa: crear la tutoría y abrir la sesión de checkout son pasos cortos y locales; el pago en sí ocurre en la página del proveedor, y su resultado llega después, de forma asíncrona, sin bloquear ningún lock.

## 5. Alternativas consideradas

**Alternativa elegida — Orquestación con confirmación diferida por eventos.** `ms-tutorias` llama a `ms-pagos` solo para "abrir la sesión de checkout" (respuesta rápida, sin esperar al proveedor); el resultado final llega por evento. Se acepta porque separa "aceptar la solicitud" de "esperar al proveedor", mantiene baja la latencia percibida por el estudiante y evita transacciones largas.

**Alternativa descartada — Coreografía pura.** Cada servicio publica y escucha eventos sin un orquestador central (agenda escucha "tutoría creada", pagos escucha "agenda bloqueada", etc.). Se descarta como diseño principal: con reglas de negocio claras y secuenciales (validar, bloquear, cobrar, confirmar), la coreografía pura dificulta saber "quién decide qué" y complica el diagnóstico. Sí se conserva como estilo para el tramo pago → notificación, que ya funciona así hoy.

**Alternativa descartada — Transacción distribuida síncrona (la propuesta original / 2PC).** Inviable porque el proveedor externo no participa en un protocolo de commit propio del sistema, y porque ata la disponibilidad de todo el flujo a la latencia de un tercero fuera de nuestro control.

Se elige la primera alternativa porque conserva el patrón de orquestación que el sistema ya usa (`ms-tutorias` coordinando agenda y notificación) y solo añade el pago como un participante más de la Saga, sin introducir un mecanismo de coordinación nuevo y más complejo.

## 6. Contratos

Se usa un patrón de **checkout hospedado por el proveedor**: `ms-tutorias` nunca recibe datos de tarjeta, solo entrega al cliente una URL de pago (`paymentUrl`). El resultado final llega por un único evento de dominio, no por dos eventos separados de aprobado/rechazado, para mantener el contrato simple.

### HTTP `POST /api/v1/tutorias` (Solicitud de reserva y pago)

El cliente inicia el flujo enviando un `Idempotency-Key` en el header. Esta llamada bloquea la agenda de forma síncrona (igual que hoy) y abre la sesión de checkout; no espera al proveedor de tarjeta.

- **Headers:** `Idempotency-Key: uuid-unico-de-peticion`
- **Payload:**
  ```json
  {
    "estudianteId": 123,
    "tutorId": 456,
    "horarioId": 789
  }
  ```
- **Respuesta `202 Accepted`:**
  ```json
  {
    "tutoriaId": "tut-999",
    "estado": "PENDIENTE_PAGO",
    "paymentUrl": "https://pasarela.com/checkout/token"
  }
  ```
- **Errores:** `409` si el horario ya está bloqueado (mismo criterio que `ms-agenda` hoy) · `422` si estudiante/tutor inválido · `400` si falta `Idempotency-Key` · `409` si la misma `Idempotency-Key` ya generó una tutoría distinta.

### HTTP `POST /api/v1/pagos/webhook` (Notificación del proveedor)

Endpoint expuesto por `ms-pagos`. Recibe la confirmación asíncrona del proveedor externo cuando el estudiante termina el checkout.

- Exige firma/HMAC del proveedor en el header; rechaza con `401` cualquier payload sin firma válida.
- Es idempotente por `transaccionId`: si el proveedor reenvía el mismo webhook, se responde `200` sin reprocesar.

### Evento RabbitMQ: `tutoria.pago.procesado`

Publicado por `ms-pagos` una sola vez que el webhook del proveedor confirma la transacción (aprobada o rechazada). Es el único evento de resultado; el campo `resultado` distingue el desenlace.

```json
{
  "eventoId": "evt-555",
  "tutoriaId": "tut-999",
  "transaccionId": "tx-visa-001",
  "resultado": "APROBADO",
  "motivo": null,
  "timestamp": "2026-07-01T18:42:00Z",
  "correlationId": "corr-321"
}
```

`resultado` puede ser `APROBADO`, `RECHAZADO` o `RECHAZADO` con `motivo: EXPIRADO` (cuando el estudiante no completó el checkout a tiempo).

### Versionamiento y compatibilidad

- Todos los endpoints usan el prefijo `/api/v1/`; un cambio incompatible se publica como `/api/v2/` y ambas versiones conviven mientras los consumidores migran.
- Los campos nuevos se agregan como opcionales, tanto en HTTP como en el evento; no se eliminan campos existentes sin subir de versión.
- Los consumidores del evento `tutoria.pago.procesado` ignoran campos desconocidos (*tolerant reader*) para no romperse ante cambios aditivos, por ejemplo si más adelante se agrega el medio de pago usado.

## 7. Manejo de fallas

**Idempotencia:** el `Idempotency-Key` del cliente viaja hasta `ms-pagos`. Tanto `ms-tutorias` como `ms-pagos` guardan las claves ya procesadas; si la misma clave llega dos veces (doble clic, retry, refresh), se devuelve la respuesta anterior sin repetir el cobro ni duplicar la tutoría.

**Inbox:** `ms-tutorias` guarda una tabla Inbox con los `eventoId` de `tutoria.pago.procesado` ya procesados. Si el mismo evento llega dos veces (reintento del broker, redelivery), se descarta sin volver a transicionar el estado de la tutoría.

**Idempotencia del webhook:** `ms-pagos` deduplica por `transaccionId`. Si el proveedor reenvía el mismo webhook (algo común en pasarelas reales), se responde `200` sin publicar un segundo `tutoria.pago.procesado`.

**Timeouts y expiración del checkout:** la sesión de pago (`paymentUrl`) tiene un plazo máximo de validez, por ejemplo 15 minutos. Si el estudiante no completa el pago o el proveedor no confirma en ese plazo, un job de expiración cierra el intento como `EXPIRADO` y `ms-pagos` publica `tutoria.pago.procesado` con `resultado: RECHAZADO`, `motivo: EXPIRADO`.

**Compensación:** si el resultado es `RECHAZADO` (por el proveedor o por expiración), `ms-tutorias` libera el bloqueo en `ms-agenda` y marca la tutoría como `CANCELADA_POR_PAGO`. Si el proveedor confirma `APROBADO` después de que la tutoría ya se canceló, no se reactiva automáticamente: se dispara un reembolso y la tutoría queda en `CONFIRMADA_CON_REEMBOLSO`.

**DLQ:** si `tutoria.pago.procesado` no puede procesarse tras los reintentos configurados, se mueve a una cola de error (*dead-letter*) para revisión manual, igual que ya contempla el proyecto para notificaciones, de modo que ninguna falla se pierda en silencio.

### Estados de la tutoría

| Estado | Cuándo ocurre |
| --- | --- |
| `PENDIENTE_PAGO` | Agenda bloqueada, tutoría creada, sesión de checkout abierta. |
| `CONFIRMADA` | Pago aprobado dentro del plazo. |
| `CANCELADA_POR_PAGO` | Pago rechazado o expirado; agenda liberada. |
| `CONFIRMADA_CON_REEMBOLSO` | Pago aprobado tarde, tras cancelación; se reembolsa en vez de revivir la tutoría. |

## 8. Observabilidad

**Correlation ID:** cada solicitud genera un `X-Correlation-ID` en `ms-tutorias`, igual que hoy soporta el `tracking-dashboard`. Ese `correlationId` se entrega dentro de `paymentUrl` como parámetro de retorno, y `ms-pagos` lo incluye en el evento `tutoria.pago.procesado`, para poder rastrear una misma solicitud desde el clic inicial hasta la confirmación del webhook.

**Logs estructurados:** cada servicio registra en formato JSON el `correlationId`, el `eventoId`/`transaccionId`, el estado anterior y el nuevo, y el resultado de la operación, permitiendo reconstruir la línea de tiempo completa de una tutoría pagada.

**Métricas (Prometheus):** tasa de resultados `APROBADO`/`RECHAZADO`/`EXPIRADO`, latencia entre la creación del checkout y la llegada del webhook, tamaño de la DLQ de pagos y cantidad de sesiones de checkout abandonadas. Se configuran alertas si la DLQ crece o si la tasa de `EXPIRADO` sube de forma inusual (señal de que el proveedor está degradado).

**Trazas distribuidas:** el `correlationId` se propaga entre `ms-tutorias`, `ms-pagos` y los consumidores del evento `tutoria.pago.procesado`, permitiendo ver en una sola vista el recorrido completo de una solicitud, similar a lo que hoy hace el dashboard para agenda y notificación.

**Diagnóstico de incidentes:** ante un reclamo tipo "me cobraron pero no tengo tutoría", el procedimiento es: buscar el `transaccionId` o `correlationId` en logs, revisar el estado del intento en `ms-pagos` y el estado de la tutoría en `ms-tutorias`, y si están desalineados, revisar la Inbox pendiente y la DLQ.

## 9. Seguridad

**Límites de confianza:** el proveedor externo es una zona no confiable; cualquier dato que llegue desde él (webhook) se valida con firma/HMAC antes de actuar. `ms-tutorias` nunca llama directamente al proveedor; solo `ms-pagos` cruza ese límite.

**Datos sensibles:** al usar un checkout hospedado por el proveedor, el número de tarjeta y el CVV se capturan directamente en la página de la pasarela y nunca llegan a `ms-tutorias`, a `ms-pagos`, a RabbitMQ, a los logs ni al dashboard. Esto reduce el alcance de cumplimiento de datos de tarjeta al nivel más bajo posible (equivalente a *SAQ A* en PCI-DSS), porque ningún servicio propio procesa ni almacena datos de tarjeta.

**Autenticación entre servicios:** igual que hoy `ms-auth` emite JWT para el usuario final, se añade autenticación servicio a servicio (JWT de servicio o mTLS interno) para que `ms-pagos` verifique que la solicitud de abrir un checkout proviene realmente de `ms-tutorias`. Solo `ms-tutorias` puede iniciar una sesión de pago; ningún cliente externo llama a `ms-pagos` directamente.

**Webhook del proveedor:** el endpoint `POST /api/v1/pagos/webhook` exige verificación de firma/HMAC, rechaza payloads no firmados con `401`, y es idempotente por `transaccionId` para tolerar reintentos del propio proveedor sin duplicar efectos.

**Secretos:** las credenciales del proveedor y las claves HMAC se gestionan como secretos (variables de entorno inyectadas de forma segura o un vault), nunca dentro del repositorio ni en los logs.

## 10. Despliegue y costo

| Opción | Dónde encaja y por qué |
| --- | --- |
| Contenedores (Docker Compose / Kubernetes) | `ms-tutorias`, `ms-usuarios` y `ms-agenda`: tráfico estable, necesitan locks de agenda consistentes y control fino de recursos. |
| PaaS (plataforma de contenedores administrada) | Reduce la carga operativa de mantener el clúster; buen equilibrio para los servicios de validación con tráfico predecible. |
| Serverless (funciones) | `ms-pagos` y el worker de webhooks: tráfico variable con picos (matrícula), carga dominada por espera de red; escala a cero en horas valle y absorbe picos sin sobreaprovisionar. |

Recomendación: mantener `ms-tutorias`, `ms-usuarios` y `ms-agenda` en contenedores o PaaS, y mover `ms-pagos` junto con el worker de webhooks a un esquema serverless o de autoescalado agresivo, ya que su carga es más intermitente y su costo por invocación conviene frente a tener instancias siempre encendidas. RabbitMQ se mantiene como servicio administrado para no perder eventos durante despliegues o migraciones.

El diseño con Saga y eventos agrega el costo operativo de mantener Inbox, DLQ y monitoreo adicional; ese costo es menor que el riesgo de doble cobro, reclamos y bloqueos de base de datos que tenía el diseño original.

## 11. Decisión

Se recomienda reemplazar la transacción distribuida síncrona por una Saga orquestada por `ms-tutorias`, con `ms-pagos` como dueño exclusivo del contacto con el proveedor externo mediante un checkout hospedado. La llamada inicial es HTTP corta (crear la sesión de checkout) y la confirmación final llega por un único evento `tutoria.pago.procesado`, apoyada en `Idempotency-Key`, deduplicación por Inbox y DLQ.

### Riesgos residuales

- Las aprobaciones tardías después de una cancelación seguirán ocurriendo; se mitigan con reembolso automático, no se eliminan por completo.
- La complejidad operativa aumenta frente al diseño original (más colas, más estados), lo que exige mantener buena observabilidad para no perder trazabilidad.

### Condiciones para revisar esta decisión

- Si el volumen de pagos crece y el tráfico de `ms-pagos` deja de tener picos y se vuelve constante, conviene reevaluar serverless frente a contenedores dedicados.
- Si el proveedor de pagos ofrece garantías de latencia muy bajas y confiables, se podría simplificar el flujo síncrono inicial, siempre sin abrir una transacción de base de datos que espere a un tercero externo.
