# Glosario técnico — Sistema de Transferencias

Términos usados en `DECISIONS.md` y `CLAUDE.md`, explicados en simple. Cuando aplica, se conecta con el ADR donde se usó la decisión, para poder citarlo en la entrevista con contexto real, no solo la definición de libro.

---

## AWS — Cómputo y costos

**Lambda** — Función que corre solo cuando se la invoca, no un servidor prendido todo el tiempo. Se le paga por invocación + duración. Es donde vive toda la lógica de negocio del proyecto (ADR-002).

**Cold start** — La primera vez que se invoca una Lambda (o después de estar inactiva un rato), AWS tiene que preparar el entorno de ejecución antes de correr tu código — eso tarda más que una invocación normal. Una invocación posterior que reutiliza ese mismo entorno ya preparado se llama **warm** (caliente), y es mucho más rápida. Central en ADR-003.

**GB-segundo** — Unidad con la que se factura Lambda: memoria asignada (en GB) × tiempo que tardó en ejecutarse (en segundos). El free tier de Lambda incluye 400.000 GB-segundos por mes, siempre.

**Always-free vs. Free Tier de 12 meses** — Dos cosas distintas que se llaman parecido. *Always-free* no vence nunca (ej. Lambda, Cognito hasta 50k usuarios). *Free Tier de 12 meses* es un descuento para cuentas nuevas de AWS que después empieza a cobrar (ej. API Gateway REST API v1). El proyecto evita depender de la segunda.

**Provisioned Concurrency** — Una configuración de Lambda que mantiene entornos "pre-calentados" listos para responder sin cold start. Cobra por hora reservada exista o no tráfico — **no tiene free tier permanente**. Se descartó explícitamente como trampa de costo.

---

## AWS — API y autenticación

**Cognito / User Pool** — Servicio de AWS que gestiona usuarios, contraseñas y login por vos — no programás lógica de autenticación propia. Un *User Pool* es el directorio de usuarios (las 3 cuentas demo viven ahí). Always-free hasta 50.000 MAU.

**MAU** — *Monthly Active Users*, usuarios distintos que se loguean en un mes. Es la métrica que usa Cognito para su límite de free tier.

**JWT** — *JSON Web Token*. Un token firmado digitalmente que Cognito entrega al loguearse, con información sobre quién es el usuario (identidad) y hasta cuándo es válido. No dice nada sobre si los *datos* del request son seguros — eso es un problema aparte (ver ADR-005).

**JWT authorizer** — Componente de API Gateway que valida un JWT automáticamente (firma, vencimiento, audiencia) antes de que tu Lambda se ejecute. Si el token es inválido, rechaza el request ahí mismo — la Lambda ni se entera (ADR-005).

**API Gateway (HTTP API vs. REST API)** — Servicio que expone una Lambda a internet como una API. *HTTP API* es la versión más nueva y barata (~$1/millón de requests, sin free tier permanente). *REST API* (v1) tiene free tier de 1M requests/mes pero solo por 12 meses. El proyecto usa HTTP API (ADR-005).

**Lambda Function URL** — Una URL HTTPS que se le puede pegar directo a una Lambda, sin pasar por API Gateway. Gratis (no es un servicio aparte), pero solo soporta auth IAM o ninguna — no tiene JWT authorizer nativo. Se evaluó y se descartó para la Core API (ADR-005).

---

## AWS — Eventos y mensajería

**EventBridge** — Bus de eventos: recibe un evento y lo enruta a quien esté suscripto (acá, a una cola SQS). Reemplaza lo que en otro contexto sería Kafka — no usado en este proyecto.

**SQS** — Cola de mensajes. Recibe el evento ruteado por EventBridge y dispara al Worker. Garantiza entrega **at-least-once** (ver abajo).

**DLQ (Dead-Letter Queue)** — Una cola secundaria donde van a parar los mensajes que fallaron demasiadas veces (superan `maxReceiveCount`) sin poder procesarse. Evita que un mensaje "envenenado" bloquee la cola principal reintentando para siempre.

**Event source mapping** — La configuración nativa que hace que una Lambda se dispare automáticamente cuando llega un mensaje a SQS — sin que vos escribas código de polling manual.

**At-least-once delivery** — Garantía de SQS: un mensaje puede entregarse (y procesarse) **más de una vez**, nunca menos. Por eso el Worker tiene que ser idempotente — ver *idempotencia* más abajo.

---

## DynamoDB

**WCU / RCU** — *Write/Read Capacity Unit*. La unidad con la que se mide y se paga (o se reserva, en modo provisioned) la capacidad de una tabla. El free tier permanente de DynamoDB es 25 WCU + 25 RCU — importante: **por cuenta de AWS, no por tabla** (ADR-007).

**On-demand vs. Provisioned** — Dos formas de pagar la capacidad de una tabla. *On-demand* es pay-per-request, escala solo, pero no tiene línea de always-free — el costo real es bajísimo pero no $0 garantizado. *Provisioned* reserva una capacidad fija (ej. 25/25) que sí está cubierta por el free tier permanente, a cambio de tener que planificarla vos. El proyecto eligió provisioned (ADR-007).

**Conditional write / `ConditionExpression`** — Una escritura en DynamoDB que solo se ejecuta si se cumple una condición (ej. `balance >= :monto`). Si no se cumple, la base de datos rechaza la escritura sola — sin que el código tenga que implementar ningún lock. Así se evita el double-spend (ver ADR-006).

**`TransactWriteItems`** — Operación de DynamoDB que agrupa varias escrituras (incluso en tablas distintas) en una sola transacción atómica: todo se aplica, o nada. Se usa para debitar el balance y escribir el ledger en un solo paso indivisible (ADR-006).

**PK / SK (Partition Key / Sort Key)** — Las claves con las que DynamoDB organiza y busca los datos de una tabla. La PK decide en qué "partición" física vive el dato; la SK ordena los datos dentro de esa misma partición.

**GSI (Global Secondary Index)** — Un índice alternativo sobre una tabla de DynamoDB, para poder buscar por una clave distinta a la PK/SK original (ej. "todos los movimientos de esta cuenta" en la tabla Ledger).

**Single-table design** — Patrón de DynamoDB donde todas las entidades de una app viven en una sola tabla, usando PK/SK genéricos. Promovido por AWS como buena práctica a escala, pero se evaluó y descartó para este proyecto por la relación de dos cuentas en cada transferencia (ver el ADR-007 completo en DECISIONS.md).

---

## Patrones de arquitectura y datos

**Ledger** — Registro histórico de movimientos. En este proyecto es **append-only**: nunca se edita ni se borra una fila, cada cambio de estado es una fila nueva. Es el mecanismo real que usan los sistemas bancarios para tener auditoría completa (ADR-006).

**Saga pattern** — Forma de manejar una operación que cruza varios sistemas sin una transacción ACID única (acá: tu banco + una cámara compensadora externa). En vez de eso, es una secuencia de pasos locales, cada uno con su reverso por si algo falla más adelante. Este proyecto usa la variante *choreography* (coordinada por eventos, sin un orquestador central) (ADR-006).

**Transacción compensatoria** — El "reverso" de un paso de un saga. No es un rollback automático — es una acción nueva y explícita que deshace el efecto de un paso anterior (ej. acreditar de vuelta el balance si Mock COELSA falla).

**Idempotencia / idempotency key** — Que una operación dé el mismo resultado final aunque se ejecute más de una vez. Necesario porque SQS es at-least-once: si el mismo mensaje llega dos veces, la *idempotency key* (acá, el `transferId`) permite detectar que ya se procesó y no duplicar el efecto.

**Double-spend** — Cuando dos operaciones concurrentes logran gastar el mismo saldo porque ninguna se enteró de la otra a tiempo. Es el riesgo concreto que se evita debitando el balance de forma atómica y síncrona, antes de cualquier proceso async (ADR-006).

**Race condition / concurrencia optimista** — Una *race condition* es un bug que depende del orden en que corren dos operaciones simultáneas. *Concurrencia optimista* es una forma de evitarlo sin locks: se intenta la escritura y se deja que la base de datos la rechace si algo cambió — es lo que hace el `ConditionExpression` de DynamoDB.

**Lost update problem** — El mecanismo concreto detrás de una race condition cuando "leer → decidir → escribir" son 3 pasos separados. Ejemplo con una cuenta de $100 y dos transferencias de $80 saliendo casi al mismo tiempo:

| Tiempo | Transferencia A ($80) | Transferencia B ($80) |
|---|---|---|
| t=0ms | Lee saldo → **$100** | |
| t=2ms | | Lee saldo → **$100** (todavía no vio ningún cambio, porque A ni escribió nada aún) |
| t=5ms | Chequea 100≥80 ✓ → escribe saldo=**$20** | |
| t=7ms | | Chequea 100≥80 ✓ (con el $100 que leyó, ya viejo) → escribe saldo=**$20** |

Resultado: salieron $160 de una cuenta con $100 (dos transferencias "exitosas"), y el saldo final quedó en $20 en vez de reflejar el sobregiro — cada escritura pisó el resultado de la otra sin que el sistema se diera cuenta. No es un problema de demoras/timing, es que las dos lecturas ocurren *antes* de que cualquiera de las dos escriba. Se soluciona haciendo que el chequeo y la escritura sean una sola operación atómica (`TransactWriteItems` + `ConditionExpression`), no dos pasos separados — así, si dos transferencias compiten, DynamoDB garantiza que solo una gana y la otra recibe `ConditionalCheckFailedException` (ADR-006).

**Event sourcing** — Guardar el estado de un sistema como una secuencia de eventos históricos, en vez de un valor mutable. El ledger de este proyecto es parcialmente event sourcing (nunca se edita), pero el balance en sí *no* es 100% derivado de los eventos — es un valor mutable protegido (ver la discusión completa en ADR-006).

**Clean Architecture** — Forma de organizar el código en capas donde el dominio (reglas de negocio) no depende de detalles de infraestructura (AWS, base de datos, HTTP). Es ortogonal a usar o no un contenedor de DI — se puede hacer con o sin uno (ADR-003).

**Dependency Injection (DI) / contenedor de DI** — Pasar las dependencias de un objeto desde afuera (por constructor, típicamente) en vez de que el objeto las cree él mismo. Un *contenedor de DI* (como el de NestJS) automatiza ese armado leyendo metadata de decoradores — con un costo de bootstrap que en Lambda se traduce en cold start más lento (ADR-003).

**Bounded context (DDD)** — Concepto de Domain-Driven Design: un límite claro dentro del cual un modelo de datos y un vocabulario tienen un significado consistente. Se usó para trazar los límites de los repos del polyrepo (ADR-001).

**Polyrepo / Monorepo** — *Polyrepo*: un repositorio de git por servicio. *Monorepo*: todo en un solo repositorio, con carpetas por servicio. El proyecto usa polyrepo, por bounded context (ADR-001).

**ADR (Architecture Decision Record)** — Formato corto para documentar una decisión de arquitectura: contexto, decisión, alternativas consideradas, consecuencias. Es el formato de `DECISIONS.md`.

**BFF (Backend-for-Frontend)** — Una capa fina del lado del frontend que resuelve necesidades específicas de la UI (tokens, agregar llamadas) sin contener lógica de negocio. En este proyecto, el rol de Next.js/Vercel — la lógica de negocio real vive en la Core API sobre AWS.

---

## Seguridad

**PAT (Personal Access Token)** — Un token de acceso personal de GitHub, usado para autenticarse en vez de usuario+contraseña. Se usaría solo si el repo `contracts` pasara a privado, con scope mínimo, guardado como secret de CI (ADR-004).

**Git-dependency** — Instalar un paquete de npm directo desde un repositorio de git (no desde un registry), fijado idealmente a un tag o commit SHA para que sea reproducible. Mecanismo elegido para compartir tipos entre repos (ADR-004).

**Supply-chain attack** — Ataque que compromete una dependencia de terceros (no tu propio código) para infiltrar código malicioso en tu proyecto. No aplica al *git-dependency* de este proyecto porque el repo `contracts` es propio, no de un tercero.

**Presigned URL** — Una URL temporal y firmada que da acceso a un objeto privado de S3 (ej. un PDF) por un tiempo limitado, sin necesitar que el bucket sea público. Se genera al momento de la descarga, con expiración corta.

---

## Otros

**202 Accepted** — Código de respuesta HTTP que significa "recibí tu request y lo voy a procesar, pero todavía no terminó". Lo usa la Core API para responder de inmediato mientras el saga corre en background — el contrato explícito de que el procesamiento es asíncrono.
