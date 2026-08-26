# Decisiones de Arquitectura — Sistema de Transferencias (Portfolio)

Registro tipo ADR (Architecture Decision Record) de las decisiones tomadas durante el diseño, con el razonamiento completo para poder defenderlas en una entrevista técnica. Se va completando módulo por módulo, en el mismo orden del plan de días de CLAUDE.md.

---

## ADR-001: Repos separados por bounded context (polyrepo)

**Estado:** Decidido

**Contexto:** El objetivo principal del proyecto es demostrar comprensión de arquitectura en una entrevista técnica, no simplemente entregar rápido. Un monorepo con carpetas sería más simple de gestionar en solitario, pero no obliga a pensar en límites de servicio.

**Decisión:** Un repositorio por bounded context (DDD): Identity (Cognito), Transfers (Core API), Clearing (Mock COELSA), Ledger (Worker + DynamoDB), Documents (PDF Lambda), Infra (IaC/SAM).

**Alternativas consideradas:** Monorepo con carpetas por servicio (más simple, sin overhead de versionado cruzado).

**Consecuencias:**
- Hay que versionar tipos/DTOs compartidos entre repos — resuelto en ADR-004.
- El repo de Infra necesita artefactos de build de los demás repos — SAM asume por defecto código local al template, así que esto exige una estrategia explícita (pendiente de definir en Día 5).
- Overhead operativo real de mantener múltiples pipelines de CI/CD en solitario.

**Argumento central:** "Elegí polyrepo aplicando bounded contexts de DDD para trazar los límites de servicio, no arbitrariamente. Soy consciente del costo operacional que eso implica siendo un solo desarrollador — versionado cruzado de contratos, coordinación de builds para IaC — y lo asumí como parte del ejercicio de simular independencia de despliegue real."

---

## ADR-002: Core API y Mock COELSA corren en Lambda, nunca en EC2

**Estado:** Decidido

**Contexto:** CLAUDE.md prohíbe EC2 por no tener free tier permanente, pero originalmente no especificaba dónde corre la Core API (NestJS). Un framework HTTP tradicional asume un proceso long-running (`app.listen(puerto)`), que es la forma de EC2/contenedor, no de una función serverless.

**Decisión:** Toda la lógica de negocio (Core API, Mock COELSA) corre como funciones Lambda, invocadas on-demand, sin proceso persistente.

**Alternativas consideradas:** EC2 (descartado por costo — no tiene free tier permanente). Fargate (tampoco tiene free tier permanente, no evaluado en profundidad).

**Consecuencias:**
- Lambda always-free: 1.000.000 requests + 400.000 GB-segundos de cómputo, por mes, para siempre (no es free-tier de 12 meses). A escala de tráfico de demo, prácticamente $0 garantizado.
- Mock COELSA, al ser un proceso ya pensado como asíncrono (invocado por el Worker), encaja naturalmente como Lambda sin cambios de diseño.

**Argumento central:** "Descarté EC2 no solo por costo sino porque un servidor long-running es más trabajo operativo (levantarlo, mantenerlo corriendo, pagarlo aunque nadie lo use) que una función invocada on-demand. Lambda no fue un compromiso entre costo y simplicidad — optimiza las dos cosas a la vez."

---

## ADR-003: Node.js + Clean Architecture manual, en vez de NestJS empaquetado en Lambda

**Estado:** Decidido

**Contexto:** NestJS es viable en Lambda vía adapters (ej. `serverless-express`), pero su bootstrap (`NestFactory.create()`) hace trabajo de CPU síncrono antes de atender el primer request: lee metadata de `reflect-metadata` generada por decoradores, recorre todo el grafo de módulos, e instancia cada provider/controller — proporcional al tamaño de la app, no a lo que ese request puntual necesita. Ese costo se traduce directo en latencia y duración facturada en cold starts.

**Decisión:** Core API en Node.js + TypeScript plano, con Clean Architecture organizada a mano: dominio (entidades, reglas de negocio) → aplicación (casos de uso) → infraestructura (implementaciones DynamoDB/EventBridge) → interfaz (el handler de Lambda). Inyección de dependencias manual por constructor, compuesta una sola vez a nivel módulo (fuera del handler exportado), reutilizada en invocaciones warm.

**Alternativas consideradas:** NestJS en Lambda con mitigaciones (adapter Fastify en vez de Express, lazy loading de módulos, cachear la app bootstrapeada entre invocaciones).

**Consecuencias:**
- Cold start más rápido: sin reflection ni escaneo de grafo de módulos.
- Sin dependencia de un framework pesado — más control, pero también más código propio de "plomería" (routing básico, parsing del evento).

**Argumento central:** "Clean Architecture y 'usar un contenedor de DI' son ejes independientes — la arquitectura define la dirección de las dependencias, no cómo se instancian. Elegí DI manual porque el costo de bootstrap de un contenedor reflectivo (el de Nest) es medible y evitable, y en Lambda se traduce directo en costo y latencia percibida. No es que no sepa usar NestJS — es que entendí su modelo de ejecución en un contexto serverless y tomé una decisión deliberada en base a eso."

---

## ADR-004: Contratos compartidos vía repo `contracts` como git-dependency

**Estado:** Decidido

**Contexto:** El polyrepo (ADR-001) necesita compartir DTOs/tipos entre servicios en al menos cuatro seams: Frontend↔Core API (HTTP), Core API↔Worker (evento vía EventBridge/SQS), Worker↔Mock COELSA (invocación), y posiblemente Worker↔PDF Lambda (forma del item de DynamoDB). Duplicar a mano genera riesgo de drift silencioso sin contract testing (Pact), que está fuera de alcance para un proyecto portfolio en solitario.

**Decisión:** Repo `contracts` separado, con solo interfaces/tipos TypeScript (cero código runtime, cero dependencias). Los demás repos lo instalan como git-dependency fijado a un tag de release (`github:usuario/contracts#v1.0.0`), nunca a una rama mutable. Se commitea `package-lock.json` para fijar el commit SHA resuelto. Repo público (no hay necesidad de simular privacidad para un portfolio, y evita gestionar PATs en CI). Mismo mecanismo aplicado de forma uniforme en los cuatro seams, sin mezclar estrategias.

**Alternativas consideradas:**
- Duplicar tipos a mano en cada repo (sin red de contract testing, riesgo de drift).
- EventBridge Schema Registry con code binding generation (nativo de AWS, evaluado para el seam Core API↔Worker específicamente, pero se descartó por ahora a favor de un único mecanismo consistente en todos los seams).
- Paquete npm publicado a un registry (GitHub Packages u npm privado) — descartado por agregar un paso de publish/CI innecesario cuando el git-dependency directo ya resuelve el problema a costo $0.

**Consecuencias:**
- Un repo más que mantener, pero sin infraestructura de CI de publish — el "release" es simplemente taggear.
- Si en el futuro el repo pasara a privado, hace falta un PAT de GitHub con scope mínimo guardado como secret de CI (nunca embebido en la URL), siguiendo el mismo principio de "nunca hardcodear credenciales" que ya rige para AWS.

**Argumento central:** "Elegí compartir contratos vía un repo de solo-tipos versionado con git tags en vez de duplicar código o montar un registry privado — es el punto de menor complejidad operativa que aún da single-source-of-truth con versionado explícito. Fijé a tags/commit SHA, no a una rama, para que la resolución de dependencias sea reproducible."

---

## ADR-005: API Gateway HTTP API con JWT authorizer de Cognito, en vez de Lambda Function URL

**Estado:** Decidido

**Contexto:** La Core API necesita exponerse a internet y validar que cada request lleve un JWT de Cognito vigente. El login en sí no pasa por la Core API — Cognito valida credenciales y emite el JWT directamente al frontend. Lo que hay que resolver es dónde se valida ese JWT en cada llamada posterior.

**Decisión:** API Gateway HTTP API, con JWT authorizer nativo conectado al User Pool de Cognito, parado delante de la Lambda de la Core API. Tokens inválidos/vencidos se rechazan con 401 en el gateway, sin invocar la Lambda.

**Alternativas consideradas:** Lambda Function URL — más cercano a "always-free literal", pero solo soporta auth IAM o ninguna. La validación del JWT quedaría adentro del código de la Lambda (librería tipo `aws-jwt-verify`), lo que implica invocar (y pagar) la Lambda incluso con tokens inválidos, y exponer código propio a input no confiable en vez de un componente maduro de AWS.

**Consecuencias:**
- Costo real: ~$1 por millón de requests en API Gateway HTTP API — no es "always free" estricto como Lambda, pero trivial a escala de demo.
- Menos superficie propia expuesta a tráfico no autenticado; fail-fast en el borde.
- Nota aparte, no resuelta acá: la validación del JWT es autenticación (quién manda el request), no protege contra injection en el contenido del body — eso se resuelve por separado con validación de los DTOs de entrada en cada endpoint.

**Argumento central:** "Elegí pagar un costo marginal (~$1/millón de requests) a cambio de que la validación de identidad ocurra en un componente maduro de AWS antes de que mi código corra, en vez de reimplementar verificación de JWT a mano dentro de la Lambda. Es una decisión consciente de dónde ubicar la frontera de confianza en el ciclo de vida del request."

---

## ADR-006: Balance protegido por escritura atómica + Saga con transacción compensatoria

**Estado:** Decidido

**Contexto:** La regla append-only del ledger ("nunca UPDATE, solo INSERT") no dice qué pasa con el *balance* de una cuenta. Sin definir esto hay riesgo de double-spend: si el balance se tocara recién cuando Mock COELSA responde, dos transferencias simultáneas de la misma cuenta podrían leer el mismo saldo disponible antes de que ninguna termine.

**Decisión:** Saga pattern por coreografía (vía EventBridge/SQS, no Kafka) con transacción compensatoria:
1. Core API, al aceptar la transferencia (antes de responder y antes de emitir el evento), reserva/debita el balance de forma atómica: `UpdateItem` con `ConditionExpression: balance >= :monto`, junto con un `INSERT` de estado `DEBITADO` en el ledger, ambos en una sola `TransactWriteItems` (todo o nada).
2. Recién ahí se emite el evento y el Worker invoca Mock COELSA de forma asíncrona.
3. Éxito → `INSERT` en el ledger de estado `LIQUIDADO` (el balance ya estaba correcto, no se vuelve a tocar).
4. Fallo → transacción compensatoria: `INSERT` de estado `REVERTIDO` + crédito atómico de vuelta al balance. Nunca se edita ni se borra el `INSERT` original.

Idempotencia obligatoria (clave = `transferId`) en cada paso, porque SQS es at-least-once.

**Alternativas consideradas:**
- Balance 100% derivado (sumar todo el ledger en cada validación) — descartado: correcto en teoría pero caro de leer a cualquier escala real.
- Debitar recién cuando COELSA confirma éxito — descartado: dos transferencias concurrentes podrían pasar la validación de saldo con el mismo balance "disponible" (double-spend).

**Consecuencias:**
- El balance vive como campo mutable en Accounts, pero **toda mutación pasa exclusivamente por escritura atómica condicionada** — nunca por lock de aplicación.
- El ledger se mantiene 100% append-only incluso para reversiones (nuevo INSERT, nunca edición).

**Argumento central:** "No podía usar una transacción ACID única entre mi banco y una cámara compensadora externa, así que apliqué Saga con transacciones compensatorias: reservo el balance de forma atómica y síncrona antes de tocar nada externo, y si la cámara compensadora falla, la reversión es un nuevo hecho en el ledger, no una edición del original."

---

## ADR-007: DynamoDB — provisioned fijo (25/25) y multi-tabla (Accounts + Ledger)

**Estado:** Decidido — reemplaza la directiva original de CLAUDE.md ("modo on-demand, nunca provisioned")

**Contexto:** Dos preguntas quedaban abiertas: (a) capacity mode, dado que on-demand es pay-per-request real y no está bajo el paraguas contractual de "always free" como Lambda o Cognito; (b) diseño de tabla, dado que single-table design es el patrón que AWS suele promover para DynamoDB.

**Decisión — capacity mode:** Provisioned, fijo dentro del free tier perpetuo de AWS (25 WCU + 25 RCU). Importante: ese presupuesto es **por cuenta de AWS, no por tabla** — se reparte entre Accounts y Ledger, no es 25/25 para cada una. A volumen de portfolio, sobra margen repartido.

**Decisión — diseño de tabla:** Multi-tabla — Accounts y Ledger como tablas separadas. En palabras simples, por qué no single-table:

> En DynamoDB, single-table design junta todos los tipos de dato en una sola tabla, agrupados en "particiones" que normalmente le pertenecen a una sola entidad — por ejemplo, una cuenta y su propio historial juntos. El problema es que una transferencia no le pertenece a una sola cuenta: la plata sale de una y entra a otra, son dos dueños al mismo tiempo. Para que las dos cuentas puedan ver su historial rápido, habría que guardar el mismo movimiento dos veces — una copia en la partición de la cuenta que envía, otra en la de la cuenta que recibe. Eso funciona, pero es trabajo extra: hay que mantener las dos copias sincronizadas, y si algo se corrige hay que acordarse de tocar las dos. Con tablas separadas, cada transferencia se guarda **una sola vez**, con su propio ID, y para ver el historial de una cuenta simplemente se le pregunta a la tabla "dame todos los movimientos donde esta cuenta participó" — sin duplicar nada.

**Alternativas consideradas:**
- On-demand (ya estaba en CLAUDE.md) — descartado porque no es $0 garantizado por contrato, solo barato en la práctica.
- Single-table design — descartado para este dominio puntual porque una transferencia es una relación de dos partes (cuenta origen + cuenta destino), no el historial de una sola entidad. Forzarlo exige duplicar el item por cuenta o resolver con un GSI extra — complejidad real sin beneficio de costo en este volumen.

**Consecuencias:**
- Requiere disciplina de capacidad (25/25 repartido entre 2 tablas), pero a escala de demo no hay riesgo real de exceder eso.
- Se actualizó CLAUDE.md (sección Arquitectura objetivo, punto 7) para reflejar este cambio sobre la directiva original.

**Argumento central:** "Evalué single-table design, que es lo que Amazon suele recomendar como patrón por defecto en DynamoDB, pero en este dominio una transferencia es una relación de dos cuentas, no el historial de una sola — forzar single-table hubiera significado duplicar cada movimiento o resolverlo con un GSI extra. Con solo dos entidades y patrones de acceso distintos, tablas separadas es la elección que evita complejidad sin pagar costo real en dólares, dado que el free tier de capacidad se reparte por cuenta de AWS, no por tabla."

---

## ADR-008: SNS + SQS en vez de EventBridge + SQS para el event bus

**Estado:** Decidido — reemplaza la decisión original de CLAUDE.md (punto 4 de arquitectura, "Amazon EventBridge → Amazon SQS")

**Contexto:** Al arrancar el Día 2, se verificó el costo real de EventBridge antes de crearlo (regla del proyecto: declarar el costo explícito de cada servicio nuevo). Hallazgo: los eventos *custom* de EventBridge (los que publica la propia aplicación, como `TransferRequested`) se cobran desde el primer evento — $1 USD por millón, sin ningún tramo gratuito, ni siquiera de 12 meses. Esto contradice la regla de oro del proyecto ("$0 USD siempre... dentro de capas always-free, no free-tier de 12 meses que después empieza a cobrar").

**Decisión:** Amazon SNS (tópico pub/sub) → Amazon SQS (cola suscrita al tópico, dispara al worker). SNS confirmado **always-free permanente**: 1.000.000 requests/mes, y las entregas de SNS hacia SQS no generan costo adicional. Cubre exactamente el mismo caso de uso que se buscaba con EventBridge — desacoplar el productor (Core API) del consumidor (Worker) con entrega confiable, y capacidad de fan-out a más de un suscriptor si en el futuro hiciera falta — a costo $0 garantizado por contrato, no solo bajo en la práctica.

**Alternativas consideradas:**
- Mantener EventBridge y aceptar el costo real — descartado: a volumen de demo el costo es una fracción de centavo (cubierto además por el Budget Action de AWS-1b), pero rompe la regla de oro tal como está escrita, y el proyecto ya tiene precedente de priorizar "$0 por contrato" sobre "barato en la práctica" (ver ADR-007, DynamoDB on-demand descartado por el mismo motivo).
- SQS directo, sin capa de pub/sub (Core API publica directo a la cola) — descartado por ahora: más simple y también $0 por contrato, pero pierde la capacidad de fan-out a múltiples consumidores. Hoy el proyecto solo tiene un consumidor (el Worker), así que esta alternativa hubiera sido válida igual — se prefirió SNS+SQS porque preserva el patrón pub/sub que la arquitectura original ya buscaba (desacople real, no solo cola punto a punto), sin pagar el costo de EventBridge por eso.

**Consecuencias:**
- El puerto `TransferEventPublisher` (CORE-5) no cambia — solo el adaptador real que lo implementa. El caso de uso `CreateTransfer` no se toca (ver STUDY.md, CORE-5).
- Si en el futuro se necesitara filtrado de eventos por contenido (algo que EventBridge ofrece nativo con event patterns y SNS no), habría que reevaluar — no es un requisito actual del proyecto.
- Se actualizó CLAUDE.md (sección Arquitectura objetivo, punto 4) para reflejar este cambio sobre la directiva original.

**Argumento central:** "Empecé con EventBridge porque es el servicio 'de catálogo' para pub/sub en AWS, pero antes de crearlo verifiqué su modelo de costo real: los eventos custom no tienen ningún tramo gratuito, se cobran desde el primer evento. Como el proyecto tiene una regla explícita de costo cero garantizado por contrato (no solo barato en la práctica), reemplacé EventBridge por SNS — mismo patrón de desacople productor/consumidor, mismo potencial de fan-out a futuro, pero con free tier permanente real. Es la misma disciplina que ya había aplicado antes con DynamoDB (ADR-007): preferir la garantía contractual de AWS sobre 'total no debería costar casi nada'."

---

## ADR-009: Polling simple para "tiempo real" en el frontend (no WebSockets, no AppSync)

**Estado:** Decidido — resuelve el pendiente que había quedado abierto para Día 4

**Contexto:** Al arrancar Día 4 se verificó el costo real de las dos alternativas candidatas antes de elegir (misma disciplina de ADR-008). La nota pendiente original decía "AppSync (always-free 250K ops/mes)" — ese dato era **incorrecto**, se corrige acá: el tramo de 250K operaciones de AppSync es de 12 meses, no permanente (después: $4/millón de queries, $2/millón de updates en tiempo real). WebSockets vía API Gateway tiene el mismo problema — 1M mensajes/750K minutos de conexión, también de 12 meses — y como la cuenta de este proyecto es post-15/07/2025 ("Free Plan", ver nota de sesión 2026-08-13), ni siquiera aplica ese tramo legacy: facturaría desde la primera conexión, cubierto solo por el crédito inicial de $100/185 días.

**Decisión:** Polling simple desde el frontend contra el HTTP API que ya existe (AWS-5) — sin sumar ningún servicio nuevo. El HTTP API tampoco es *always-free* (~$1/millón de requests), pero esa excepción **ya está aceptada y documentada en ADR-005** ("costo trivial a escala de demo, sin alternativa $0 real porque Cognito JWT necesita un componente maduro delante"). Polling no abre una categoría de costo nueva — solo agrega volumen de requests sobre algo que el proyecto ya decidió pagar.

**Alternativas consideradas:**
- AppSync — descartado: no es always-free (12 meses), y además sumaría un servicio nuevo (GraphQL, suscripciones) para un caso de uso de 3 usuarios demo que no lo necesita.
- WebSockets vía API Gateway — descartado por el mismo motivo: no es always-free, y en esta cuenta puntual (creada después del cambio de modelo de AWS) ni siquiera tiene tramo legacy de 12 meses.

**Consecuencias:**
- El frontend necesita una ruta nueva de lectura (`GET /accounts` o similar) en el HTTP API existente, protegida por el mismo JWT authorizer de AWS-5 — no hace falta un authorizer nuevo.
- Frecuencia de polling a definir en Día 4 (ej. cada 3-5 segundos) — suficiente para una demo de "tiempo real" sin generar volumen real de costo.
- Si en algún futuro el proyecto necesitara latencia sub-segundo real (no es el caso de una demo), habría que reevaluar.

**Argumento central:** "Evalué WebSockets y AppSync para el tiempo real del backoffice, pero ninguno de los dos es always-free — ambos tienen tramo de 12 meses, y mi cuenta ni siquiera calificaba para el legacy. Polling simple sobre el HTTP API que ya tenía desplegado no suma ninguna categoría de costo nueva, solo reutiliza una excepción de costo que ya había aceptado y documentado (ADR-005) para el propio endpoint de transferencias. Para 3 usuarios demo, la latencia de polling cada pocos segundos es imperceptible — no hay necesidad real de push."

---

## ADR-010: DynamoDB Streams dispara la Lambda de PDF (no invocación directa del Worker)

**Estado:** Decidido — resuelve el pendiente que había quedado abierto para Día 5

**Contexto:** La generación del PDF de comprobante podía dispararse de dos formas: (a) el Worker invoca directamente a la Lambda de PDF después de liquidar, o (b) una Lambda de PDF separada consume los cambios de la tabla `Ledger` vía **DynamoDB Streams**, filtrando por `status = LIQUIDADO`.

**Decisión:** DynamoDB Streams. Un fallo generando el PDF (librería rota, timeout, bucket sin permisos) **nunca** puede afectar el resultado real de la transferencia — que es la parte que importa. Streams desacopla completamente la generación del comprobante del camino crítico del Worker: el Worker ni sabe que la Lambda de PDF existe.

**Costo verificado antes de decidir** (misma disciplina de todo el proyecto): DynamoDB Streams es **always-free** cuando el consumidor es una Lambda vía event source mapping nativo — no se cobra por las llamadas `GetRecords` que hace el propio Lambda (solo se cobraría para consumidores no-Lambda, $0.02/100.000 llamadas). Habilitar Streams en una tabla sin leer de él tampoco tiene costo. El único costo real es la invocación de la Lambda de PDF en sí, que ya cae dentro del millón de invocaciones always-free mensuales de Lambda.

**Alternativas consideradas:**
- Invocación directa desde el Worker — descartada: acopla el camino crítico (liquidar una transferencia) con una operación secundaria (generar un PDF). Un bug en la librería de PDF no debería poder impedir que una transferencia se liquide.

**Consecuencias:**
- Ventana de eventual consistency: el PDF no existe en el instante exacto en que la transferencia pasa a `LIQUIDADO` — aparece unos segundos después, cuando Streams entrega el evento. El endpoint de lectura (`GET /transfers/{id}/receipt`) devuelve `404` mientras tanto, no un error — el frontend puede reintentar.
- Streams entrega **at-least-once**: la Lambda de PDF necesita su propia idempotencia. Se resolvió igual que en CORE-4/WORKER-1: la protección real es un `ConditionExpression: attribute_not_exists(receiptKey)` en el `UpdateItem` final — una reentrega que regenera y resube el mismo PDF a S3 es aceptable (mismo key, sobrescribe), solo la escritura en DynamoDB está protegida contra duplicarse.
- Streams también entrega la fila de marca de idempotencia (mismo mecanismo de CORE-4) como su propio evento `INSERT` — la Lambda de PDF filtra por `timestamp` real (no `#IDEMPOTENCY_KEY`) antes de procesar.

**Argumento central:** "Elegí que la generación del PDF fuera un consumidor pasivo de un stream de cambios, no una llamada directa del Worker, específicamente para que un fallo ahí nunca pueda tocar el resultado de la transferencia real. Verifiqué el costo de DynamoDB Streams antes de usarlo — es always-free cuando el consumidor es Lambda, a diferencia de otras alternativas de streaming que evalué en otras partes del proyecto (como EventBridge) que sí cobran desde el primer evento."

---

## ADR-011: Bucket S3 de comprobantes — se mantiene, pero con fecha de revisión declarada (no es always-free por contrato)

**Estado:** Decidido (2026-08-26)

**Contexto:** Revisando qué le faltaba al frontend para poder descargar el comprobante (el botón nunca se construyó después de PDF-2), surgió una pregunta más de fondo: el PDF se genera una sola vez (Streams → Lambda `pdf-receipt`) y se guarda en S3 de forma permanente; el endpoint de lectura solo devuelve una presigned URL al objeto ya guardado, nunca lo regenera. A diferencia de cada otro servicio del proyecto, AWS-10 (creación del bucket) nunca declaró el costo real de S3 — y a diferencia de DynamoDB, Lambda, SNS/SQS o Cognito, **S3 no tiene una capa always-free**: depende del régimen de free tier de la cuenta.

Se verificó el estado real de la cuenta usada en este proyecto: creada en 2026 (posterior al cambio de AWS del 15/07/2025) y con método de pago cargado (Paid plan) — esto descarta el riesgo de cierre automático a los 6 meses que aplica al Free Plan sin tarjeta, pero no aclara si el almacenamiento de S3 sigue cubierto por el esquema legacy de "primeros 12 meses gratis" o solo por los créditos de bienvenida (que vencen a los 12 meses de creada la cuenta o antes si se agotan). En cualquiera de los dos escenarios, el horizonte real es **~12 meses desde la creación de la cuenta**, no "gratis para siempre".

**Decisión:** Mantener S3 + presigned URL para los comprobantes — es un patrón de mucho valor de portfolio (bucket privado, Block Public Access, presigned URL de expiración corta) muy consultado en entrevistas de fintech, y no tiene sentido sacrificarlo por un costo que a volumen de demo es centavos. Pero, siguiendo la misma disciplina que el resto del proyecto, se declara explícitamente en vez de asumir "$0 para siempre" sin más: **fecha de revisión/decommission fijada para julio de 2027** (margen conservador dentro de la ventana de ~12 meses desde la creación de la cuenta). En esa fecha hay que confirmar el estado real de facturación en la consola de Billing y, si S3 ya no cae en ningún tramo gratuito, vaciar y eliminar el bucket `transfer-module-receipts-627551519292` (o migrar la Lambda de lectura a generar el PDF al vuelo desde el `LedgerEntry`, sin persistencia — alternativa evaluada y descartada por ahora solo por el valor de portfolio de S3, no por costo).

**Alternativas consideradas:**
- Generar el PDF on-demand en cada `GET /transfers/{id}/receipt`, sin subir nada a S3 — sería $0 por contrato de verdad (el `LedgerEntry` ya es inmutable por el diseño append-only, así que el PDF generado siempre sería reproducible), y más simple (sin Streams disparando una Lambda aparte, sin idempotencia de escritura a S3). Descartada por ahora: se prefiere demostrar el patrón S3-privado-con-presigned-URL para el portfolio, aceptando el costo trivial y con fecha límite en vez de ignorarlo.

**Consecuencias:**
- Costo real mientras tanto: a volumen de demo (unas pocas decenas de transferencias, unos KB por PDF) el total nunca supera unos pocos MB — facturado a precio de S3 Standard ($0.023/GB/mes) serían centavos, pero la fecha límite es lo que evita que esto se vuelva "gratis para siempre" por descuido.
- Queda anotado en TICKETS.md como recordatorio para no perderlo de vista entre sesiones.

**Argumento central:** "Guardo el comprobante en S3 en vez de regenerarlo cada vez porque quería demostrar el patrón de bucket privado + presigned URL de expiración corta, que es exactamente cómo se resuelven documentos sensibles en la banca real. Pero como S3 no tiene capa always-free como el resto de mi stack, lo dejé explícitamente declarado con fecha de revisión en vez de asumir que era gratis para siempre sin chequearlo — la misma disciplina de costo que apliqué en cada decisión del proyecto."

---

## ADR-012: AWS X-Ray activo en las 5 Lambdas del proyecto

**Estado:** Decidido (2026-08-26)

**Contexto:** El pipeline real de una transferencia cruza 5 Lambdas (`core-api-create-transfer`, `worker-process-transfer-requested`, `mock-coelsa`, `pdf-receipt`, `core-api-read`) más SNS, SQS y DynamoDB Streams. Hasta ahora la única visibilidad era CloudWatch Logs por función, sin correlación automática entre los saltos de un mismo pedido — para reconstruir el camino de una transferencia había que buscar a mano por `transferId` en los logs de cada Lambda por separado.

**Decisión:** Tracing activo (`TracingConfig: Mode=Active`) en las 5 Lambdas, agregando la policy administrada `AWSXRayDaemonWriteAccess` a cada rol de ejecución existente (aditivo — no se tocó ninguna policy inline ya existente).

**Costo verificado antes de decidir:** X-Ray tiene **100.000 trazas grabadas/mes always-free**, más 1.000.000 trazas recuperadas y 1.000.000 trazas escaneadas por mes también gratis (fuente: `aws.amazon.com/xray/pricing`). A volumen de demo, ni cerca del límite.

**Hallazgo real durante la implementación:** el HTTP API (`core-api-http`, ApiId `6d5igy4es2`) **no genera segmento de X-Ray a nivel de API Gateway** — esa integración de tracing de borde solo existe para REST API (v1), no para HTTP API (v2). No se migró a REST API por esto: contradiría ADR-005, que eligió HTTP API específicamente por el JWT authorizer nativo de Cognito y el costo. La traza arranca en la Lambda (no en el borde), lo cual sigue siendo suficiente para ver el cruce entre Lambdas, que era el objetivo real.

**Verificado real:** se disparó una transferencia real (`POST /transferencias`) y se confirmó con `aws xray batch-get-traces` un trace real con segmentos de `worker-process-transfer-requested` y `mock-coelsa` correlacionados como parte del mismo trace — tracing distribuido cross-Lambda funcionando de punta a punta, no solo "activado en la consola".

**Alternativas consideradas:** ninguna real — X-Ray es la opción nativa de AWS sin costo comparable; un colector propio de OpenTelemetry implicaría correr infraestructura adicional, fuera del alcance y la regla de $0 de este proyecto.

**Argumento central:** "Activé X-Ray en las 5 Lambdas para poder seguir una transferencia de punta a punta por trace ID, no solo por logs sueltos. Descubrí en el camino que HTTP API v2 no da tracing de borde como sí lo hace REST API v1 — decisión consciente de no migrar solo por eso, porque HTTP API ya estaba elegido por motivos de costo y JWT authorizer nativo que pesan más."

---

## ADR-013: CloudWatch Budgets — alerta temprana complementaria al guardarail existente

**Estado:** Decidido (2026-08-26)

**Contexto:** La cuenta ya tenía un budget (`DenyAllOnBudgetExceeded`, $1/mes) con una **Budgets Action automática** que aplica una IAM policy de deny sobre el usuario `sasha-transfer-module` al 80% de gasto real, con notificación por email. Es un guardarail real y efectivo, pero agresivo y sin aviso previo — corta el acceso sin avisar antes de que eso pase.

**Decisión:** se agrega un segundo budget, `TransferModuleCostWatch` ($5/mes), **sin ninguna Budgets Action** (solo notificación), con dos alertas por email a `sashax191@gmail.com`: gasto real >50% y gasto proyectado (forecasted) >100%. Da aviso temprano antes de que el deny automático del budget existente llegue a dispararse.

**Costo verificado antes de decidir:** un budget con solo notificación (sin Budgets Actions asociadas) es **gratis sin límite declarado** — confirmado en la página oficial de pricing de AWS Budgets ("Budget monitoring and notifications are completely free"). El límite de 62 *action-enabled budget-days*/mes gratis (~2 budgets con acción automática) solo aplica a budgets **con** Action — el único budget con Action de todo el proyecto sigue siendo el preexistente, muy por debajo del límite.

**Verificado real:** `aws budgets describe-budgets` confirma los 2 budgets activos; `describe-notifications-for-budget` confirma las 2 alertas del budget nuevo. Gasto real del mes verificado con `aws ce get-cost-and-usage`: **$0** — margen completo antes de tocar cualquiera de los dos umbrales.

**Alternativas consideradas:** subir el límite del budget existente en vez de crear uno nuevo — descartada: el deny automático a $1 es un guardarail correcto tal como está (agresivo a propósito), lo que faltaba era una alerta previa, no cambiar ese límite.

**Argumento central:** "Ya había un budget con acción automática de deny al superar $1 — lo dejé intacto porque es el guardarail duro correcto. Le sumé uno de solo notificación con un umbral más alto y una alerta de gasto proyectado, para tener aviso previo en vez de enterarme cuando el acceso ya se cortó."

---

## ADR-014: CloudTrail — trail de management events, bucket dedicado

**Estado:** Decidido (2026-08-26)

**Contexto:** Ver nota en "Pendientes" de sesiones anteriores: CloudTrail registra quién hizo qué *en la cuenta de AWS* (cambios de IAM, creación/borrado de recursos) — audita la infraestructura, distinto del ledger append-only del proyecto (que audita el negocio). No estaba prendido.

**Decisión:** trail `transfer-module-trail`, multi-región (incluye eventos globales de IAM/STS), **solo management events** (`ReadWriteType: All`, `DataResources: []` — sin Data Events), **sin Insights**, un solo copy a un bucket S3 nuevo y dedicado `transfer-module-cloudtrail-627551519292` (separado del bucket de comprobantes `transfer-module-receipts-...` para no mezclar propósitos), con Block Public Access en los 4 flags y SSE-S3, y bucket policy escopeada al ARN exacto de este trail (no genérica).

**Costo verificado antes de decidir (vuelto a confirmar en esta sesión, no asumido de la nota anterior):** el primer trail por cuenta/región que entrega solo management events a S3 es **gratis** (fuente: `aws.amazon.com/cloudtrail/pricing`). Data Events cuestan $2/100.000 eventos, Insights e integración con CloudTrail Lake cobran desde el primer evento — ninguno de los tres se activó. Costo real remanente: almacenamiento S3 de los logs entregados, mismo caveat que ADR-011 (no *always-free* más allá del tier de la cuenta, pero trivial a este volumen).

**Verificado real:** `aws cloudtrail lookup-events` mostró eventos de management reales y recientes de esta misma sesión (incluido el propio `DescribeBudgetActionsForBudget` de instantes antes de crear el trail). `get-event-selectors` confirmó explícitamente `DataResources: []`; `get-insight-selectors` devolvió `InsightNotEnabledException`, confirmando que Insights no está activo.

**Alternativas consideradas:** CloudTrail Lake — descartada explícitamente por costo (cobra desde el primer evento, sin capa gratuita).

**Consecuencias:** cierra el gap que la sección "Producción real" de este documento marcaba explícitamente como faltante.

**Argumento central:** "El ledger append-only audita transferencias; CloudTrail audita la cuenta de AWS en sí — quién cambió una IAM policy, quién creó un recurso. Son dos capas de auditoría distintas y ambas hacen falta en un sistema bancario real. Lo dejé acotado a management events porque Data Events e Insights cobran desde el primer evento, y acá el objetivo era demostrar el patrón sin salirme de la regla de $0."

---

## ADR-015: IaC de Día 5 — un solo `template.yaml`, sin nested stacks; API Gateway declarado crudo

**Estado:** Decidido (2026-08-26)

**Contexto:** Cerrando el pendiente de IaC con AWS SAM (ver ADR-001, nota de Consecuencias, y TICKETS.md), había dos decisiones de diseño del template en sí: (a) un solo `template.yaml` vs. nested stacks para separar por dominio (datos / mensajería / cómputo / API), y (b) usar el shorthand `AWS::Serverless::HttpApi` para el HTTP API vs. declarar los recursos `AWS::ApiGatewayV2::*` crudos.

**Decisión:**
- **Un solo template**, sin nested stacks. A esta escala (~15 recursos) separar en stacks anidados agrega indirección (parámetros cruzados entre stacks, más superficie para `sam build`) sin ganar nada real — nested stacks pagan su complejidad recién cuando varios equipos despliegan partes distintas de forma independiente, que no es el caso de un proyecto de una sola persona.
- **API Gateway declarado crudo** (`AWS::ApiGatewayV2::Api`/`Route`/`Stage`/`Integration`), no con `AWS::Serverless::HttpApi`. El shorthand de SAM no expone `RouteSettings` por ruta individual, y `POST /transferencias` necesita un throttle más estricto (2 req/s) que el resto de las rutas (20 req/s) — medida anti-abuso agregada en esta misma sesión (ver nota de "Medidas anti-abuso" más abajo). Usar el shorthand hubiera significado renunciar a esa protección puntual o pelear contra la abstracción.

**Verificado real:** cada recurso del template (tablas, colas, roles IAM, funciones, rutas) se armó leyendo la configuración real de la cuenta con `aws-cli` (`describe-table`, `get-role-policy`, `get-queue-attributes`, `get-routes`, etc.), no de memoria ni de los tickets. `sam validate --lint` y `sam build` corren en limpio contra el código real de los repos hermanos (`core-api`, `worker`, `mock-coelsa`, `pdf-receipt`) — confirma que el `CodeUri`/`Handler`/entry point de cada función coincide con lo que está desplegado. No se corrió `sam deploy`: los recursos reales no están bajo CloudFormation y tienen datos en uso (ver README de `transfer-module-infra`).

**Alternativas consideradas:**
- Nested stacks por dominio — descartado, ver arriba.
- `AWS::Serverless::HttpApi` + una `AWS::ApiGatewayV2::Stage` propia superpuesta para el throttle por ruta — técnicamente posible, pero el shorthand ya crea su propio stage `$default` internamente y colisiona con uno declarado a mano; más simple declarar el API completo crudo que pelear esa colisión.

**Consecuencias:** el repo nuevo (`transfer-module-infra`) documenta la infra real en código, pero **no es todavía la fuente de verdad** — los recursos existen sin CloudFormation. El camino para que lo sea (`CloudFormation import`) queda documentado en el README de ese repo mismo, sin ejecutar — es una operación real sobre recursos con datos en uso, necesita aprobación humana explícita y puntual, no algo para correr en automático.

**Argumento central:** "Escribí el SAM template al final, desde el estado real verificado de la cuenta, no antes — no tiene sentido templar algo mientras todavía estás decidiendo la arquitectura. Elegí un solo template porque nested stacks son una herramienta para coordinar despliegues de varios equipos, no algo que sume valor en un proyecto de una persona; y declaré el API Gateway crudo, sin el shorthand de SAM, específicamente porque necesitaba throttle distinto por ruta y el shorthand no lo expone. Y dejé el `CloudFormation import` documentado pero sin correr: automatizar una migración sobre recursos con datos reales en uso es exactamente el tipo de acción que no debería ejecutar solo un agente sin luz verde puntual."

---

## Pendientes / próximas decisiones abiertas

- **Revisar costo real de S3 (bucket de comprobantes) antes de julio de 2027** — ver ADR-011. Confirmar en la consola de Billing si sigue en algún tramo gratuito; si no, vaciar/eliminar el bucket o migrar a generación on-demand.
- **Confirmar que el límite de concurrencia de Lambda de la cuenta subió de 10 a 1.000** (ver "Trampas" abajo) — chequear con `aws lambda get-account-settings`. Si en unos días sigue en 10, abrir caso de soporte gratuito con AWS (Billing/Account) para que lo levanten a mano.

## Medidas anti-abuso para demo pública (2026-08-26)

Al ser un proyecto de portfolio, las credenciales demo (`demo1/2/3@transfer.test`) van a quedar visibles en el login y el README — cualquiera puede loguearse y generar transferencias en loop. Dos medidas $0 (configuración nativa de servicios ya usados, no agregan servicio nuevo ni facturan aparte):

- **S3 Lifecycle** en `transfer-module-receipts-627551519292`: expira objetos con prefijo `receipts/` a los 30 días. Acota el almacenamiento sin importar cuánta gente pruebe la demo.
- **Throttle en API Gateway** (`6d5igy4es2`): 20 req/s / burst 40 en el stage `$default`, y un override más estricto de 2 req/s / burst 5 puntual sobre `POST /transferencias` (la única ruta que crea recursos nuevos y dispara un PDF).

**Hallazgo al testear el throttle:** el límite de concurrencia de Lambda de toda la cuenta está en **10** (`aws lambda get-account-settings` → `ConcurrentExecutions: 10`), compartido entre las 5 funciones del proyecto. Es muy bajo — el default estándar de AWS es 1.000 (confirmado al intentar `aws service-quotas request-service-quota-increase --quota-code L-B99A9384`, que rechazó el pedido porque *"debe ser mayor al default de 1000"* — o sea, el 10 es una restricción temporal de cuenta nueva, no el límite real asignado). Bajo carga (60 requests concurrentes reales a `POST /transferencias`), 43/60 volvieron `503` por agotar esos 10 slots — un riesgo de confiabilidad real bajo tráfico de demo normal (dos personas viendo la app a la vez), independiente del throttle que se configuró arriba. Se espera que se levante solo en 24-48hs de uso real de la cuenta; si no, abrir caso de soporte.

## Trampas de costo detectadas (para no caer)

- **Provisioned Concurrency** en Lambda: sin free tier permanente, cobra por hora reservada exista o no tráfico. No usar para "arreglar" cold starts en este proyecto.
- **API Gateway REST API (v1)**: free tier de 1M calls/mes es de 12 meses (legacy), no permanente.
- **DynamoDB on-demand**: pay-per-request sin línea de always-free (a diferencia del modo provisioned dentro de 25/25 WCU/RCU). En volumen de demo el costo es de centavos, pero no es contractualmente $0.

---

## Producción real — qué falta fuera del alcance del MVP

Este proyecto es un portfolio de aprendizaje con regla de costo $0, desarrollado por una sola persona con acceso administrativo directo a una única cuenta de AWS. Ninguna de esas tres cosas es cierta en un entorno de producción real, sobre todo en banca. Lista de lo que faltaría, para tenerla a mano en una entrevista.

### Modelo de acceso y credenciales
- Durante esta sesión, el asistente de IA ejecutó comandos de AWS CLI usando las credenciales del usuario (usuario IAM `sasha-transfer-module`, `AdministratorAccess`), tal como lo haría el propio usuario desde su terminal. Válido para desarrollo en solitario; **nunca así contra producción**.
- En un entorno real: nadie (humano o asistente) tiene `AdministratorAccess` permanente. Los humanos acceden con **IAM Identity Center (SSO)** — credenciales temporales de pocas horas, nunca access keys estáticas. Los cambios de infraestructura pasan por **CI/CD** (GitHub Actions / CodePipeline) usando un rol de deploy acotado, autenticado por **OIDC**, nunca por un secret guardado.
- Es exactamente lo que Día 5 (IaC con AWS SAM) empieza a resolver: infraestructura como templates versionados aplicados por un pipeline controlado, no comandos sueltos por CLI.

### Ambientes (dev / staging / prod)
- AWS no tiene el concepto nativo de ambientes — se arman con **cuentas separadas** (una por ambiente), agrupadas con **AWS Organizations**. Es aislamiento real de billing, permisos y blast radius, no solo una convención de nombres. Este proyecto usó una única cuenta personal para todo.

### Protección de datos sensibles
- Hoy: DynamoDB encriptado en reposo con clave default de AWS, TLS en tránsito siempre. Para nivel bancario: **KMS con clave propia (customer-managed key)**, para controlar rotación y poder revocar acceso apagando la clave.
- **Secrets Manager / Parameter Store** para cualquier credencial que una Lambda necesite (este proyecto no hardcodea nada, pero tampoco tiene secretos reales que gestionar todavía — el día que Mock COELSA sea una integración real con una API key, ahí es donde iría).
- Repensar qué PII vive en la tabla transaccional (`ownerName` en `Accounts` hoy) — en un banco real, identidad/KYC suele vivir en un servicio aparte del ledger.

### Auditoría e infraestructura de red
- **CloudTrail**: registro inmutable de quién hizo qué *en la cuenta de AWS* (quién cambió una policy, quién creó un recurso) — activo desde Día 6 (ver ADR-014), management events únicamente. Complementa al ledger append-only (que audita el negocio, no la infraestructura).
- **VPC + subnets privadas + VPC endpoints (PrivateLink)**: hoy las Lambdas corren en la red compartida de AWS, hablando con DynamoDB por internet (aunque sea TLS). Aislarlo de verdad requiere una red privada propia. **Tensión real con la regla de $0**: la forma clásica de dar salida a internet desde una subnet privada es un NAT Gateway — uno de los servicios explícitamente prohibidos en este proyecto por costo (~$32/mes solo por existir). Aislamiento de red de nivel bancario cuesta plata real; este proyecto lo sacrificó a propósito.
- **Alarmas reales**: hoy nada avisa si un mensaje cae en la DLQ. En producción, un mensaje en la DLQ es una transferencia real colgada — necesita page a un humano en minutos, no descubrirse por casualidad revisando la consola.
- **MFA** obligatorio en usuarios IAM; rotación de access keys si llegaran a existir (lo ideal es no tenerlas).
- **Point-in-time recovery** en DynamoDB (backup continuo) — tiene costo real (no always-free), pero para datos bancarios probablemente valga la pena aceptarlo.

**Cómo resumirlo en una entrevista:** "Esto es un MVP de portfolio con una regla de costo cero explícita — accedí a una sola cuenta de AWS con credenciales administrativas propias, sin CI/CD, sin aislamiento de red. Sé exactamente qué le falta para producción real: separación de cuentas por ambiente, acceso federado sin credenciales permanentes, deploys por pipeline en vez de CLI manual, aislamiento de VPC, y alarmas reales sobre la DLQ. Elegí no construir eso acá porque cuesta plata real (VPC endpoints, NAT Gateway, backups) y no aporta a lo que el proyecto necesitaba demostrar — pero la lista de qué falta y por qué es parte del diseño, no un punto ciego."
