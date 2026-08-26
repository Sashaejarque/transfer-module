# Tickets — Día 1: Fundaciones (Identity + Core API + datos)

Backlog de trabajo del Día 1, ya con arquitectura resuelta (ADR-002 a 007). Dos grupos: configuración de AWS (consola/CLI, sin IaC todavía — SAM llega en Día 5) y desarrollo del Core API. Cada ticket tiene su criterio de aceptación: no se marca hecho hasta que se cumple y se prueba, no solo "se escribió el código".

Orden sugerido al pie. Vamos ticket por ticket, con la misma dinámica de siempre — antes de programar cada pieza, un par de preguntas para asegurarnos que el porqué quedó claro, no solo el qué.

---

## AWS — Configuración de infraestructura

### AWS-1: Cuenta AWS + usuario IAM + CLI configurado ✅
- [x] Confirmar/crear la cuenta AWS del proyecto.
- [x] Crear un usuario IAM con permisos acotados para desarrollo — nunca usar el usuario root de forma operativa.
- [x] Instalar y configurar AWS CLI v2 localmente con las credenciales de ese usuario.
- **Criterio de aceptación:** `aws sts get-caller-identity` devuelve el usuario IAM creado, no la cuenta root.
- **Hecho (2026-08-13):** cuenta nueva bajo modelo "Free plan" ($100 crédito/185 días), usuario IAM `sasha-transfer-module` con `AdministratorAccess`, CLI configurado vía `aws configure` en `us-east-1`. Verificado: `arn:aws:iam::627551519292:user/sasha-transfer-module`.

### AWS-1b: Freno automático de gasto (Budget Action) ✅
- [x] Zero Spend Budget (alerta a $0).
- [x] AWS Budgets Action a $1/mes: adjunta automáticamente policy `Deny *` al usuario `sasha-transfer-module` si se supera el monto.
- **Hecho (2026-08-14):** rol `BudgetsActionExecutionRole` (trust policy para `budgets.amazonaws.com` + inline policy `AllowBudgetsAttachDenyPolicy` con los `iam:Attach*/Detach*Policy` necesarios), policy standalone `DenyAllOnBudgetExceededPolicy` (Effect Deny, Action `*`), y la Budget Action con aprobación automática ya configurada apuntando a ese rol y esa policy. Motivo: usuario opera con tarjeta prepaga casi siempre en $0, esta es la segunda capa de protección además del Zero Spend Budget.

### AWS-2: Cognito User Pool + 3 usuarios demo ✅
- [x] Crear el User Pool (política de contraseña, atributos mínimos).
- [x] Crear un App Client sin secret (el frontend es público/SPA).
- [x] Crear los 3 usuarios demo con contraseña definida.
- **Criterio de aceptación:** loguear cualquiera de los 3 usuarios (CLI `initiate-auth` o consola) y recibir un JWT válido.
- **Hecho (2026-08-18):** User Pool `us-east-1_UHsuel1W6` (nickname "User pool - 0aqxkx") en `us-east-1`. App Client `transfer-module`, Client ID `6temeulgf3n53kg2r4e7ukt6tb`, sin secret, flujos `ALLOW_USER_PASSWORD_AUTH` + `ALLOW_USER_SRP_AUTH` + `ALLOW_REFRESH_TOKEN_AUTH` habilitados. 3 usuarios demo (`demo1@transfer.test`, `demo2@transfer.test`, `demo3@transfer.test`, contraseña `Admin123!`) creados por consola con contraseña temporal, pasada a permanente por CLI (`admin-set-user-password --permanent`), y email verificado marcado desde la consola. Verificado: `initiate-auth --auth-flow USER_PASSWORD_AUTH` con `demo1` devuelve `AuthenticationResult` con tokens válidos.

### AWS-3: Tablas DynamoDB — Accounts y Ledger ✅
- [x] Tabla `Accounts`: PK `accountId`, atributo `balance`.
- [x] Tabla `Ledger`: PK `transferId`, SK `timestamp` (estado queda como atributo normal del item, no en la clave) + GSI por `accountId` para historial.
- [x] Modo provisioned, repartiendo 25 WCU / 25 RCU entre las dos tablas (ADR-007).
- **Criterio de aceptación:** las dos tablas existen, capacidad provisioned visible en consola, suma combinada ≤ 25/25.
- **Hecho (2026-08-18):** `Accounts` (PK `accountId` String, sin SK) 5 WCU/5 RCU. `Ledger` (PK `transferId` String, SK `timestamp` String) 15 WCU/15 RCU, con GSI `accountId-timestamp-index` (PK `accountId`, SK `timestamp`, proyección "All") ajustado a 5 WCU/5 RCU (nace heredando la capacidad de la tabla base, hay que bajarlo a mano después de crear la tabla). Total combinado en la cuenta: 25/25, dentro del free tier permanente de DynamoDB (25 WCU + 25 RCU combinados, no es por tabla). `balance` de `Accounts` se guarda como entero en centavos, no float, para evitar errores de redondeo — el formateo a moneda queda del lado de cada consumidor (frontend), no en la API. El "costo estimado" que muestra la consola al editar capacidad (ej. "USD 11.61/mes") es precio de lista sin descontar el free tier — no confundir con el cargo real.

### AWS-4: Rol IAM de ejecución para la Core API Lambda ✅
- [x] Rol con permisos mínimos: `dynamodb:GetItem`, `UpdateItem`, `TransactWriteItems` **solo** sobre `Accounts` y `Ledger` — nada de `*`.
- **Criterio de aceptación:** policy revisada línea por línea, sin wildcards de recurso innecesarios.
- **Hecho (2026-08-25):** `CoreApiCreateTransferLambdaRole` (ARN `arn:aws:iam::627551519292:role/CoreApiCreateTransferLambdaRole`). Trust policy: solo `lambda.amazonaws.com` puede asumirlo. Policy de permisos inline (no managed policy de AWS, para evitar el `Resource: "*"` que trae `AWSLambdaBasicExecutionRole` por defecto). Cuatro statements finales: `dynamodb:TransactWriteItems` sobre los 2 ARNs exactos de `Accounts`/`Ledger`, `dynamodb:UpdateItem` solo sobre `Accounts`, `dynamodb:PutItem` solo sobre `Ledger`, y los 3 permisos de logs (`CreateLogGroup`/`CreateLogStream`/`PutLogEvents`) escopeados al log group de esta Lambda puntual (`/aws/lambda/core-api-create-transfer`), no `logs:*`. CloudWatch Logs confirmado **always-free** (5GB/mes combinado, no free tier de 12 meses) — verificado antes de usarlo, no asumido. **Corrección post-deploy (durante CORE-7): la primera versión de esta policy solo tenía `TransactWriteItems`**, bajo la suposición de que esa acción de alto nivel alcanzaba sin permisos separados para las operaciones internas de la transacción — **esa suposición era incorrecta**. El deploy real de la Lambda (CORE-7) lo reveló con un `AccessDeniedException` real en CloudWatch Logs pidiendo `dynamodb:UpdateItem` explícito, aunque `aws iam simulate-principal-policy` ya daba `allowed` para `TransactWriteItems` solo — el simulador prueba la acción de alto nivel, no que la transacción completa con sus operaciones internas funcione de punta a punta contra el servicio real. Se agregaron `UpdateItem`/`PutItem` escopeados cada uno a su tabla exacta. Verificado con `aws iam simulate-principal-policy`: `TransactWriteItems`/`UpdateItem` sobre `Accounts` → `allowed`; `DeleteTable` → `implicitDeny`; `Scan` sobre cualquier tabla → `implicitDeny` (sin permisos de lectura). Rol adjuntado a la función Lambda real en CORE-7.

### AWS-5: API Gateway HTTP API + JWT authorizer ✅
- [x] Crear la HTTP API.
- [x] Configurar JWT authorizer apuntando al User Pool/App Client (AWS-2).
- [x] Ruta `POST /transferencias` protegida por el authorizer (integración puede quedar mock hasta que exista la Lambda de CORE-7).
- **Criterio de aceptación:** request sin token → 401. Request con JWT válido de un usuario demo → pasa el authorizer.
- **Hecho (2026-08-25):** HTTP API `core-api-http` (`ApiId 6d5igy4es2`, endpoint `https://6d5igy4es2.execute-api.us-east-1.amazonaws.com`), stage `$default` con `auto-deploy`. Authorizer JWT nativo (`CognitoJwtAuthorizer`) apuntando a `Issuer: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_UHsuel1W6` + `Audience: 6temeulgf3n53kg2r4e7ukt6tb` (App Client de AWS-2), identity source `$request.header.Authorization`. Integración real (no mock — se decidió aprovechar que CORE-7 ya tenía código listo y hacer el deploy real ahí mismo, ver CORE-7). Permiso de invocación en la Lambda (resource-based policy, `lambda:InvokeFunction` para `apigateway.amazonaws.com`) escopeado con `Condition.ArnLike` al ARN exacto de esta API + ruta — no alcanza con el rol de ejecución de AWS-4 (ese controla qué hace la Lambda hacia afuera, no quién puede invocarla). Ambos criterios verificados contra el endpoint real: sin token → `401 {"message":"Unauthorized"}`; con `IdToken` real de `demo1@transfer.test` (vía `initiate-auth`) → `202`, transferencia debitada de verdad en `Accounts`/`Ledger`.

---

## Core API — Desarrollo (Node.js + Clean Architecture)

### CORE-1: Repo `contracts` — DTOs iniciales ✅
- [x] `TransferRequestDTO`, `TransferResponseDTO`, `Account`, `LedgerEntry`.
- [x] Tag de release `v0.1.1`.
- **Criterio de aceptación:** se instala como git-dependency desde un repo de prueba.
- **Parcial (2026-08-24):** los 4 tipos escritos en `contracts/src/` (más `TransferStatus` en archivo propio), compilan sin errores (`tsc` con `declaration`+`emitDeclarationOnly`), commit local hecho (`9e6dfba`).
- **Hecho (2026-08-25):** repo público `github.com/Sashaejarque/contracts-transfer-module`, push de `master`. Tag `v0.1.0` inicial resultó **roto**: `dist/` está en `.gitignore` (correcto, no se commitea build output) pero faltaba el script `prepare`, que es el que npm corre automáticamente al instalar una git-dependency — sin él, el paquete instalaba sin ningún tipo. Detectado con un smoke test real (`npm install github:...#v0.1.0` en un proyecto de prueba en el scratchpad, `node_modules/` quedaba sin `dist/`). Fix: agregado `"prepare": "tsc"` a `package.json`, cortado `v0.1.1` (se decidió no mover el tag `v0.1.0` ya pusheado — coherente con ADR-004, "nunca a una rama mutable", un tag tampoco se reescribe una vez publicado — y dejarlo como release roto en el historial en vez de reescribir). Segundo smoke test contra `v0.1.1`: `node_modules/contracts-transfers-modules/dist/*.d.ts` presentes — criterio de aceptación cumplido y verificado, no solo asumido.

### CORE-2: Esqueleto Clean Architecture ✅
- [x] Carpetas `domain/`, `application/`, `infrastructure/`, `interface/` en el repo `core-api`.
- [x] Sin lógica todavía — solo la regla de dependencias (domain no importa de infrastructure).
- **Criterio de aceptación:** regla documentada (o de lint) que impida que `domain/` importe de `infrastructure/`.
- **Hecho (2026-08-24):** repo `core-api/` (git anidado, como `contracts/`) con `src/{domain,application,infrastructure,interface}/`. Regla de capas aplicada con `eslint-plugin-boundaries` (`boundaries/element-types`) + `eslint-import-resolver-typescript` (necesario para que el plugin resuelva imports `.ts`, si no la regla no detecta nada). Tabla de capas permitidas: domain→nada, application→domain, infrastructure→domain+application, interface→domain+application+infrastructure. Verificado con un archivo de prueba real (`domain` importando de `infrastructure`): el lint falla con `boundaries/element-types`, exit code 1. Archivo de prueba borrado después, no quedó en el repo. Commit local `d0887ee`.

### CORE-3: Dominio — entidades y reglas de negocio ✅
- [x] Entidad `Account` (invariante: balance nunca negativo).
- [x] Entidad `Transfer` (estados válidos: DEBITADO, LIQUIDADO, REVERTIDO).
- [x] Cero AWS SDK en esta capa.
- **Criterio de aceptación:** tests unitarios puros, sin mocks de AWS, validando las reglas.
- **Hecho (2026-08-25):** `core-api/src/domain/account.entity.ts` y `transfer.entity.ts`, constructor privado + factory estático (`Account.create` / `Transfer.create`) que valida el invariante al nacer, sin forma de bypasearlo. Inmutables: `debit`/`credit`/`liquidate`/`revert` devuelven una **nueva** instancia en vez de mutar la existente. `Transfer` modela el invariante como máquina de estados: `liquidate`/`revert` solo son válidos desde `DEBITADO`, cualquier otra transición tira `InvalidTransferTransitionError`. Errores de dominio propios (`domain/errors/`, clase base `DomainError` con `code` — pensado para que CORE-7 mapee a status HTTP sin matchear strings) en vez de excepciones genéricas. `TransferStatus` vive como tipo propio de `domain/`, no reusa el de `contracts/` — el dominio no depende de la forma del contrato de red. Test runner: Vitest (elegido por el usuario sobre el runner nativo de Node, mejor DX). 19 tests unitarios en memoria, sin AWS SDK, cubriendo invariantes y transiciones inválidas. Fix de build: `tsconfig.json` excluía sin querer `dist/` de los `*.test.ts` — se agregó `"exclude": ["src/**/*.test.ts"]` porque `tsc` los compilaba a `dist/` con un `import` de `vitest` (solo devDependency), lo que hubiera roto el zip de deploy de la Lambda en CORE-7.

### CORE-4: Puerto `AccountRepository` + adaptador DynamoDB ✅
- [x] Interfaz en `application/`.
- [x] Adaptador real en `infrastructure/` con `TransactWriteItems` (debita con `ConditionExpression: balance >= :monto` + INSERT en Ledger).
- **Criterio de aceptación:** test de integración donde dos escrituras concurrentes con el mismo balance disponible solo dejan pasar una.
- **Hecho (2026-08-25):** `application/ports/account-repository.ts` (interfaz `AccountRepository.debitForTransfer`, solo el primer paso de la Saga de ADR-006 — liquidar/revertir se agregan cuando se conecte el Worker) + `infrastructure/dynamodb/dynamodb-account-repository.ts` (implementación con `TransactWriteCommand`: `Update` condicionado sobre `Accounts` + `Put` sobre `Ledger`, atómico). Cliente `DynamoDBDocumentClient` inyectado por constructor (no leído de env dentro del adaptador), así el mismo código sirve contra DynamoDB real o Local sin cambiar una línea. Ajuste chico en `InsufficientBalanceError`: `attemptedBalance` pasó a opcional, porque el adaptador no lo conoce sin un `GetItem` extra que reabriría la carrera que la transacción evita. Test de integración real contra **DynamoDB Local** (Docker, `amazon/dynamodb-local`, cero costo AWS) — 3 casos: happy path, balance insuficiente, y el del criterio de aceptación (dos débitos concurrentes con balance para solo uno → 1 resuelve, 1 rechaza con `InsufficientBalanceError`, balance final `0`, no negativo ni duplicado). `npm test` (rápido, sin Docker) separado de `npm run test:integration` (requiere Docker corriendo). Nota de tooling: el filtro de archivos de `vitest run` no acepta un glob con `**` como argumento posicional — se resolvió pasando `integration.test.ts` como substring de filtro, no como path glob.
- **Fix de idempotencia (2026-08-25, durante AWS-5/CORE-9):** releyendo CORE-9 se detectó que esta implementación original **no era idempotente de verdad** — el `Put` al Ledger usaba `timestamp = new Date().toISOString()` fresco en cada llamada, así que reinvocar con el mismo `transferId` generaba una fila nueva y debitaba dos veces (contradice ADR-006/CLAUDE.md: "Idempotencia obligatoria... porque SQS es at-least-once"). Fix: se agregó un **tercer item** a la misma `TransactWriteItems` — una marca de idempotencia con clave fija (`timestamp: "DEBITADO#IDEMPOTENCY_KEY"`, no un timestamp real) por `transferId`, condicionada a `attribute_not_exists(transferId)`. Un reintento con el mismo `transferId` hace que esa condición falle y cancele la transacción completa; el `catch` ahora inspecciona `error.CancellationReasons` (array por posición de `TransactItem`) para distinguir **cuál** condición falló: si falló la de la marca (posición 2) es un reintento → no-op idempotente, se retorna sin error; si falló la del balance (posición 0) → `InsufficientBalanceError` como antes. No hizo falta tocar el schema de la tabla `Ledger` (AWS-3) ni pedir permisos IAM nuevos (sigue siendo `PutItem`/`TransactWriteItems`, ya concedidos en AWS-4). Verificado en 3 niveles: integration test contra DynamoDB Local (4 tests, incluido el nuevo caso de idempotencia), y contra AWS real vía dos `curl` consecutivos al mismo endpoint de API Gateway con el mismo `transferId` — balance se movió una sola vez, el ledger quedó con 2 filas (histórico + marca), no 4.

### CORE-5: Puerto `TransferEventPublisher` + adaptador no-op ✅
- [x] Interfaz con `publishTransferRequested`.
- [x] Adaptador no-op/console.log por ahora — el real (EventBridge) se conecta en Día 2, sin tocar el caso de uso.
- **Criterio de aceptación:** el caso de uso llama al puerto sin saber que todavía no hace nada real.
- **Hecho (2026-08-25):** `application/ports/transfer-event-publisher.ts` (interfaz `publishTransferRequested(TransferRequestedEvent)`) + `infrastructure/events/console-transfer-event-publisher.ts` (adaptador que solo hace `console.log`, cero dependencias). El caso de uso `CreateTransfer` (CORE-6) va a recibir el publisher por constructor sin saber qué implementación es — el día que se conecte EventBridge en Día 2, la única línea que cambia es la de la composition root (donde se decide qué adaptador instanciar), no el caso de uso. Test chico verificando que el adaptador resuelve sin lanzar y loguea el evento.

### CORE-6: Caso de uso `CreateTransfer` ✅
- [x] Orquesta: valida el DTO, invoca `AccountRepository`, invoca `TransferEventPublisher`.
- [x] DI manual por constructor, sin contenedor.
- **Criterio de aceptación:** test del caso de uso con los puertos mockeados a mano.
- **Hecho (2026-08-25):** `application/use-cases/create-transfer.ts`. Orden: `Transfer.create(...)` primero (valida reglas de negocio sin I/O, falla rápido) → `accountRepository.debitForTransfer(...)` → `eventPublisher.publishTransferRequested(...)` al final (nunca se anuncia un cambio de estado que todavía no se confirmó — ver `STUDY.md`). `contracts` conectado como dependencia real de `core-api` por primera vez (`TransferRequestDTO`/`TransferResponseDTO`, git-dependency a `v0.1.1`). Test con los puertos mockeados a mano (clases fake propias, sin librería de mocking) verificando: orden de llamadas, que un monto inválido o cuentas iguales cortan antes de tocar cualquier puerto, y que si el débito falla el evento nunca se publica y el error se propaga. Detalle de diseño documentado en `STUDY.md`: `Account.debit()` (CORE-3) no se usa en este flujo — el debito real ocurre atómico en DynamoDB vía `AccountRepository`, sin lectura previa en memoria.

### CORE-7: Handler Lambda (capa de interfaz) ✅
- [x] Parsea el evento de API Gateway (claims del JWT ya validado + body).
- [x] Composición de dependencias una sola vez a nivel módulo, fuera del handler exportado.
- [x] Invoca el caso de uso, responde 202.
- **Criterio de aceptación:** deploy manual (zip) conectado a la ruta de AWS-5, responde 202 real.
- **Parcial (2026-08-25):** código completo y testeado — `interface/http/create-transfer.handler.ts` (lógica pura, `buildHandler(createTransfer)`, sin efectos secundarios, testeable sin AWS) + `create-transfer.lambda.ts` (entrypoint real: composition root a nivel de módulo, arma `DynamoDbAccountRepository` + `ConsoleTransferEventPublisher` + `CreateTransfer`, exporta `handler`). Mapeo de errores de dominio → status HTTP vía tabla `DOMAIN_ERROR_STATUS` indexada por `error.code` (400 para validación, 409 para conflictos de balance/estado, 500 genérico sin filtrar detalles internos para lo inesperado). 7 tests cubriendo body inválido, JSON malformado, campo faltante, error de dominio mapeado, y que un error inesperado no filtra su mensaje interno al cliente. **Cerrado (2026-08-25):** deploy real hecho — función Lambda `core-api-create-transfer` (Node.js 22.x, rol de AWS-4, zip de 1.5MB con `npm ci --omit=dev`, sin devDependencies). **Gotcha real encontrado y corregido**: el archivo `create-transfer.lambda.ts` tenía dos puntos en el nombre base (`create-transfer` + `.lambda`), y el parser del handler de Lambda para Node.js/ESM no resuelve bien nombres de archivo con más de un punto — fallaba con `Cannot find module 'create-transfer'`, perdiendo el resto de la ruta. Se renombró a `create-transfer-lambda.ts` (guión, no punto) y se resolvió. **Segundo hallazgo real, corrigiendo algo que se había afirmado mal en AWS-4**: la policy de AWS-4 solo tenía `dynamodb:TransactWriteItems`, asumiendo (incorrecto) que esa acción de alto nivel alcanzaba sin permisos separados de `UpdateItem`/`PutItem`. El error real de AWS (`AccessDeniedException` en los logs de CloudWatch) mostró que **sí hacen falta los permisos de las operaciones internas de la transacción** además del permiso de alto nivel — se corrigió la policy de AWS-4 agregando `dynamodb:UpdateItem` sobre `Accounts` y `dynamodb:PutItem` sobre `Ledger`, cada uno escopeado solo a la tabla que realmente usa. Después de corregir la policy, tardó varios minutos en propagar (eventual consistency de IAM, documentado por AWS, más lento de lo típico esta vez). Invocación directa de la Lambda (sin pasar por API Gateway, para aislar la causa) confirmó **202 real**: `{"transferId":"tr-direct-invoke-1","status":"DEBITADO","acceptedAt":"..."}`, verificado además contra la tabla real (`Accounts`/`Ledger`) — balance debitado, fila `DEBITADO` insertada.

### CORE-8: Script de seed — 3 cuentas demo ✅
- [x] Crea las 3 cuentas en `Accounts` con balance inicial.
- **Criterio de aceptación:** correrlo dos veces no duplica cuentas (idempotente).
- **Hecho (2026-08-25):** `core-api/scripts/seed-accounts.ts`, corrido con `npm run seed` (vía `tsx`, sin paso de build separado). `accountId` = `sub` de Cognito de cada uno de los 3 usuarios demo (decisión tomada en esta sesión: mapeo 1 a 1 cuenta↔usuario logueado, no IDs desacoplados — ver `contracts`/AWS-2 para los subs). Balances iniciales variados a propósito para poder demostrar transferencias de distinto tamaño y el caso de balance insuficiente: Demo Uno 500.000 centavos, Demo Dos 300.000, Demo Tres 100.000. Idempotencia real (no simulada): `PutCommand` con `ConditionExpression: attribute_not_exists(accountId)` — mismo principio de CORE-4 (chequeo atómico del lado del servidor, no lectura previa desde la app). Corrido dos veces contra la tabla `Accounts` real de AWS: primera vez `created` x3, segunda vez `skipped` x3, verificado con `aws dynamodb scan` — exactamente 3 items, balances intactos.

### CORE-9: Prueba end-to-end manual ✅
- [x] Login con un usuario demo → JWT.
- [x] `POST /transferencias` con ese JWT y un `transferId` propio.
- [x] Verificar: 202, balance debitado en `Accounts`, fila `DEBITADO` en `Ledger`.
- [x] Reinvocar con el mismo `transferId` → confirmar que no duplica el débito (idempotencia — el disparo real por SQS llega en Día 3, acá se simula a mano).
- **Criterio de aceptación:** los cuatro puntos anteriores confirmados a ojo (consola de DynamoDB) o por script.
- **Hecho (2026-08-25):** los 4 puntos corridos contra el sistema real desplegado (no local, no simulado). (1) `aws cognito-idp initiate-auth` con `demo2@transfer.test` → `IdToken` real. (2) `curl -X POST` a `https://6d5igy4es2.execute-api.us-east-1.amazonaws.com/transferencias` con ese token y `transferId: tr-core9-e2e-1787677831` → `202`. (3) Verificado por script contra DynamoDB real: `Accounts` de Demo Dos bajó de 300.000 a 296.000 (debitó los 4.000 exactos), `Ledger` con la fila `DEBITADO` correspondiente. (4) Reinvocado el mismo request con el mismo `transferId` → también `202` (respuesta idéntica en forma, `acceptedAt` distinto porque es un timestamp nuevo, pero sin volver a tocar el balance) — balance se mantuvo en 296.000, `Ledger` se mantuvo en 2 filas para ese `transferId` (no 4). Este último punto solo pasa porque se corrigió el gap de idempotencia real encontrado durante AWS-5 (ver nota de fix en CORE-4) — sin ese fix, este ticket hubiera fallado en el punto 4. **Día 1 completo**: los 14 tickets (AWS-1 a AWS-5, CORE-1 a CORE-9) cerrados y verificados contra AWS real, no solo localmente.

---

## Orden sugerido

AWS-1 → AWS-2 → AWS-3 → AWS-4 → (AWS-5 en paralelo, no depende de código) → CORE-1 → CORE-2 → CORE-3 → CORE-4 → CORE-5 → CORE-6 → CORE-7 → CORE-8 → CORE-9

---

# Tickets — Día 2: Mensajería (SNS + SQS + Mock COELSA)

Backlog de Día 2, con arquitectura resuelta (ADR-008: SNS+SQS reemplaza a EventBridge+SQS por costo — ver DECISIONS.md). Sin DLQ todavía: eso es Día 3, junto con el Worker que la necesita para tener sentido. Mock COELSA queda como Lambda separada (decisión tomada en sesión: simula mejor un límite real hacia un sistema externo, mismo costo $0 que cualquier Lambda).

---

## Mensajería — infraestructura

### MSG-1: Tópico SNS `transfer-events` ✅
- [x] Crear el tópico SNS.
- **Criterio de aceptación:** tópico visible en la consola/CLI, ARN documentado acá.
- **Hecho (2026-08-25):** `arn:aws:sns:us-east-1:627551519292:transfer-events`.

### MSG-2: Cola SQS `transfer-requested-queue` suscripta al tópico ✅
- [x] Crear la cola SQS (standard, sin DLQ todavía).
- [x] Suscribir la cola al tópico de MSG-1 (subscription SNS→SQS).
- [x] Queue policy que permita al tópico de MSG-1 específicamente (no `Principal: "*"`) publicar en la cola.
- **Criterio de aceptación:** publicar un mensaje de prueba en el tópico (`aws sns publish`) y verificar que aparece en la cola (`aws sqs receive-message`).
- **Hecho (2026-08-25):** cola `arn:aws:sqs:us-east-1:627551519292:transfer-requested-queue` (URL `https://sqs.us-east-1.amazonaws.com/627551519292/transfer-requested-queue`). Queue policy escopeada con `Condition.ArnEquals` al ARN exacto del tópico de MSG-1 (no principal abierto). Suscripción con `RawMessageDelivery=true` — el body que llega a la cola es el JSON publicado tal cual, sin el sobre (`Message`/`MessageId`/`TopicArn`) que SNS agrega por defecto, para que el Worker (Día 3) no tenga que desenvolverlo. Verificado real: `aws sns publish` de un mensaje de prueba → apareció en `aws sqs receive-message` con el body crudo, sin sobre. Mensaje de prueba borrado de la cola después.

### AWS-6: Rol IAM para la Lambda Mock COELSA ✅
- [x] Rol con permisos mínimos — solo logs de esta función puntual, nada de DynamoDB ni SNS/SQS (Mock COELSA no toca ningún otro recurso, solo devuelve una respuesta simulada).
- **Criterio de aceptación:** policy revisada línea por línea, sin wildcards de recurso innecesarios (misma disciplina que AWS-4).
- **Hecho (2026-08-25):** `MockCoelsaLambdaRole` (`arn:aws:iam::627551519292:role/MockCoelsaLambdaRole`). Un solo statement: los 3 permisos de logs, escopeados al log group `/aws/lambda/mock-coelsa` únicamente. Sin ningún permiso hacia DynamoDB/SNS/SQS — el handler no llama a ningún SDK de AWS, solo procesa el evento de entrada y responde.

---

## Core API — conectar el publisher real

### CORE-10: Adaptador `SnsTransferEventPublisher` ✅
- [x] Implementación real del puerto `TransferEventPublisher` (CORE-5) usando `@aws-sdk/client-sns`, reemplaza a `ConsoleTransferEventPublisher` en la composition root de `create-transfer-lambda.ts`.
- [x] Permiso `sns:Publish` agregado al rol de AWS-4, escopeado al ARN exacto del tópico de MSG-1 — nada más.
- **Criterio de aceptación:** `POST /transferencias` real (como en CORE-9) produce un mensaje real en la cola de MSG-2, verificado con `aws sqs receive-message`. `CreateTransfer` no se modifica (solo la composition root).
- **Hecho (2026-08-25):** `infrastructure/events/sns-transfer-event-publisher.ts`, cliente `SNSClient` inyectado por constructor (mismo patrón de DI que `DynamoDbAccountRepository`). `create-transfer-lambda.ts` (composition root) actualizado para usarlo, leyendo el ARN del tópico de `TRANSFER_EVENTS_TOPIC_ARN` (env var, no hardcodeado). `ConsoleTransferEventPublisher` (CORE-5) se deja en el repo sin usar — documenta el adaptador no-op original, no rompe nada dejarlo. Verificado real: `POST /transferencias` con JWT válido → `202` → mensaje real recibido en la cola de MSG-2 con la forma exacta de `TransferRequestedEvent`, sin sobre de SNS (gracias al `RawMessageDelivery` de MSG-2). `CreateTransfer` no se tocó — confirma en la práctica el payoff de DI discutido en CORE-5/STUDY.md.

---

## Mock COELSA

### COELSA-1: Lambda Mock COELSA ✅
- [x] Handler propio, sin dependencia de ningún otro componente del proyecto.
- [x] Latencia artificial ~1.5s.
- [x] Éxito: devuelve `coelsaId` de 22 caracteres. Fallo: ~15% de las invocaciones, con un motivo de rechazo.
- [x] Tests unitarios puros para la lógica de generación de ID y la tasa de fallas (sin depender de sleep real de 1.5s en el test).
- **Criterio de aceptación:** invocación directa (`aws lambda invoke`) contra la función real desplegada, corrida varias veces, muestra tanto el caso de éxito como el de fallo.
- **Hecho (2026-08-25):** repo nuevo `mock-coelsa/` (git propio, mismo patrón que `contracts`/`core-api`), sin dependencia de `contracts` ni `core-api` — no necesita ningún DTO compartido, su input/output es un contrato propio y mínimo (`{transferId}` in, `{success, coelsaId?, reason?}` out). `generateCoelsaId`/`processTransfer` reciben la fuente de aleatoriedad como parámetro inyectable (default `Math.random`) — permite tests deterministas sin depender de que el azar real caiga del lado correcto. `coelsa-mock-handler.ts` (guión, no punto en el nombre — aplicando la lección de CORE-7) hace el `sleep` real de 1.5s y delega en la lógica pura. 7 tests unitarios, cero AWS SDK como dependencia de producción (zip de 26KB). Lambda `mock-coelsa` desplegada con `MockCoelsaLambdaRole` (AWS-6). Invocada 20 veces reales: 17 éxito (`coelsaId` de 22 caracteres) / 3 fallo (con `reason`) — 15% casi exacto.

---

## Orden sugerido Día 2

MSG-1 → MSG-2 → CORE-10 → AWS-6 → COELSA-1

---

# Tickets — Día 3: Worker (liquidar/revertir) + DLQ

Backlog de Día 3. Dos decisiones de diseño tomadas en sesión, aplicando lecciones de Día 1/2:
- El Worker **no** duplica la entidad de dominio `Transfer` (CORE-3) — va directo al patrón repositorio con `TransactWriteItems` + marca de idempotencia (mismo mecanismo de CORE-4), porque ya vimos en CORE-6 que el chequeo atómico real vive en DynamoDB, no en un objeto en memoria. La entidad de dominio queda como lo que es: una herramienta de testeo/documentación de las reglas, no parte del camino de producción.
- El Lambda del Worker procesa **un mensaje SQS por invocación** (`BatchSize: 1` en el event source mapping) — evita la complejidad de reportar fallos parciales de un batch (`ReportBatchItemFailures`), sin costo real (Lambda es always-free hasta 1M invocaciones/mes).

Repo nuevo: `worker/` (git propio, mismo patrón que `contracts`/`core-api`/`mock-coelsa`).

---

## Mensajería — DLQ

### MSG-3: Dead-Letter Queue de `transfer-requested-queue` ✅
- [x] Crear la cola `transfer-requested-dlq`.
- [x] Redrive policy en `transfer-requested-queue` apuntando a la DLQ, con `maxReceiveCount` razonable (3).
- **Criterio de aceptación:** `RedrivePolicy` visible en los atributos de la cola principal, ARN de la DLQ correcto.
- **Hecho (2026-08-25):** `arn:aws:sqs:us-east-1:627551519292:transfer-requested-dlq`. `RedrivePolicy` en `transfer-requested-queue`: `maxReceiveCount: 3` — un mensaje se mueve a la DLQ recién después de que el Worker lo reciba y falle en procesarlo 3 veces sin borrarlo (no falla real todavía, se prueba en WORKER-6).

## Worker — infraestructura

### AWS-7: Rol IAM de ejecución para el Worker Lambda ✅
- [x] Permisos mínimos: `dynamodb:TransactWriteItems`/`UpdateItem`/`PutItem` solo sobre `Accounts`/`Ledger` (según lo que el código realmente llame — mismo criterio que AWS-4, no copiar de una lista prospectiva). `lambda:InvokeFunction` solo sobre la función `mock-coelsa`. `sqs:ReceiveMessage`/`DeleteMessage`/`GetQueueAttributes` solo sobre `transfer-requested-queue` (necesario para el event source mapping — el pull de SQS lo hace el servicio Lambda usando los permisos del rol de ejecución, no una resource policy como con API Gateway).
- **Criterio de aceptación:** policy revisada línea por línea, sin wildcards de recurso innecesarios.
- **Hecho (2026-08-25):** `WorkerProcessTransferRequestedLambdaRole` (`arn:aws:iam::627551519292:role/WorkerProcessTransferRequestedLambdaRole`). `UpdateItem` sobre `Accounts` (solo `revert` la usa) + `PutItem` sobre `Ledger` (ambos métodos) + `TransactWriteItems` sobre las dos, aplicando la lección de AWS-4/CORE-7 desde el principio (no hubo que corregir esta vez). `lambda:InvokeFunction` escopeado al ARN exacto de `mock-coelsa`. Permisos de SQS escopeados a `transfer-requested-queue` únicamente.

## Worker — desarrollo (repo nuevo)

### WORKER-1: Puerto `LedgerRepository` + adaptador DynamoDB ✅
- [x] Interfaz con `liquidate(input)` y `revert(input)`.
- [x] `liquidate`: `TransactWriteItems` — crédito atómico al destino + INSERT histórico `LIQUIDADO` + marca de idempotencia.
- [x] `revert`: `TransactWriteItems` — crédito atómico de vuelta a `Accounts` + INSERT histórico `REVERTIDO` + marca de idempotencia.
- **Criterio de aceptación:** test de integración contra DynamoDB Local — reinvocar `liquidate`/`revert` con el mismo `transferId` no duplica el efecto (mismo criterio de idempotencia que CORE-4).
- **Hecho (2026-08-25):** `worker/src/infrastructure/dynamodb/dynamodb-ledger-repository.ts`. Mismo mecanismo de idempotencia que CORE-4 (marca con clave fija por `transferId`+estado, `error.CancellationReasons` para distinguir no-op idempotente de error real). `revert` no lleva `ConditionExpression` en el crédito a `Accounts` — acreditar de vuelta nunca puede dejar el balance negativo, a diferencia de debitar. 4 tests de integración contra DynamoDB Local, incluida la idempotencia real de ambos métodos.
- **BUG REAL encontrado por el usuario probando la app (2026-08-25, después de FE-5)**: `liquidate` nunca acreditaba la cuenta destino — en ningún paso del sistema (`debitForTransfer` de CORE-4, `liquidate` ni `revert` del Worker) se movía plata hacia `toAccountId`. El sistema debitaba el origen y emitía un recibo, pero nunca completaba la transferencia. El error estaba en el ADR-006 original (de una sesión anterior a esta), que nunca especificó el crédito al destino en ningún paso. Fix: se agregó el `Update` de crédito a `toAccountId` dentro de la misma `TransactWriteItems` de `liquidate` (no en el débito inicial) — la plata queda disponible en destino recién cuando COELSA confirma, igual que una cámara compensadora real. `revert` no necesitó cambios: como el destino nunca se acreditaba, no hay nada que revertirle ahí. Tests de integración actualizados (crédito real + idempotencia del crédito). Verificado real contra AWS: transferencia de $5 de Demo Tres a Demo Dos, Demo Tres bajó $5 y Demo Dos subió $5 exactos. **Nota importante**: todas las transferencias `LIQUIDADO` de antes de este fix perdieron plata del sistema (se debitó el origen sin acreditar nunca el destino) — los balances habían quedado en ~$3.144 combinados contra los $9.000 originales. Reseteados manualmente a los valores iniciales de CORE-8 (500.000/300.000/100.000 centavos) vía `aws dynamodb update-item`, a pedido del usuario — el historial del ledger no se tocó, solo los balances actuales de `Accounts`.

### WORKER-2: Puerto `CoelsaClient` + adaptador Lambda real ✅
- [x] Interfaz con `process(transferId)`.
- [x] Adaptador real: `lambda:InvokeFunction` (RequestResponse) contra `mock-coelsa` (COELSA-1) via `@aws-sdk/client-lambda`.
- **Criterio de aceptación:** test de integración con invocación real a la función `mock-coelsa` desplegada.
- **Hecho (2026-08-25):** `worker/src/infrastructure/coelsa/lambda-coelsa-client.ts`. Maneja `FunctionError` de la respuesta de `InvokeCommand` (si `mock-coelsa` tirara una excepción no controlada, la API igual devuelve 200 a nivel de transporte). Test de integración real: invocación real contra la función desplegada, ~2.5s (1.5s de latencia simulada + overhead de red).

### WORKER-3: Caso de uso `ProcessTransferRequested` ✅
- [x] Orquesta: invoca `CoelsaClient.process`, según el resultado llama `LedgerRepository.liquidate` o `.revert`.
- [x] DI manual por constructor, sin contenedor (mismo patrón que CORE-6).
- **Criterio de aceptación:** test con los puertos mockeados a mano, cubriendo los dos caminos (éxito → liquidate, fallo → revert).
- **Hecho (2026-08-25):** `worker/src/application/use-cases/process-transfer-requested.ts`. 3 tests con fakes escritos a mano, cubriendo éxito→liquidate, fallo→revert, y el motivo por defecto cuando COELSA falla sin `reason` explícito.

### WORKER-4: Handler SQS + composition root ✅
- [x] Parsea un record de SQS (`BatchSize: 1`), extrae el `TransferRequestedEvent`.
- [x] Composición de dependencias a nivel módulo, igual que CORE-7.
- **Criterio de aceptación:** mismo patrón de separación (lógica pura testeable + entrypoint real) que CORE-7, por el bug de imports que encontramos ahí.
- **Hecho (2026-08-25):** `worker/src/interface/sqs/process-transfer-requested.handler.ts` (puro, `buildHandler(useCase)`) + `process-transfer-requested-lambda.ts` (guión, no punto — entrypoint real con la composition root). `TransferRequestedEvent` importado de `contracts-transfers-modules` v0.2.0, no duplicado. 2 tests, incluido procesar más de un record del batch (aunque la config real use `BatchSize: 1`, el handler no asume eso).

### WORKER-5: Deploy real + event source mapping + prueba E2E ✅
- [x] Deploy zip de la función Worker con el rol de AWS-7.
- [x] Event source mapping SQS (`transfer-requested-queue`) → Worker Lambda, `BatchSize: 1`.
- **Criterio de aceptación:** `POST /transferencias` real → el Worker procesa el mensaje solo (sin invocación manual) → el ledger termina en `LIQUIDADO` o `REVERTIDO` según el resultado real de `mock-coelsa`, verificado contra DynamoDB real.
- **Hecho (2026-08-25):** función `worker-process-transfer-requested` desplegada con `WorkerProcessTransferRequestedLambdaRole`. Probada primero con invocación directa (evento SQS fake, aislado de la cola real) antes de conectar el event source mapping — mismo criterio de debugging que en CORE-7. Event source mapping creado (`BatchSize: 1`), estado `Enabled`. **Pipeline completo verificado extremo a extremo, sin ninguna invocación manual más allá del `POST` inicial**: `POST /transferencias` real → `202 DEBITADO` → (SNS publica → SQS entrega → Worker se dispara solo) → `mock-coelsa` invocada → `LIQUIDADO` insertado, ~2 segundos después. Cola verificada en `0` mensajes visibles/en vuelo tras el procesamiento — consumido y borrado correctamente, no quedó atascado.

### WORKER-6: Prueba de la DLQ ✅
- [x] Forzar que el Worker falle repetidamente para un mensaje (ej. rol sin permiso temporalmente, o un `transferId` que dispare una excepción no controlada).
- **Criterio de aceptación:** tras superar `maxReceiveCount` (MSG-3), el mensaje aparece en `transfer-requested-dlq`, verificado con `aws sqs receive-message` sobre la DLQ.
- **Hecho (2026-08-25):** en vez de tocar IAM temporalmente (más riesgo de olvidarse de restaurarlo), se mandó un mensaje con body malformado (`{esto no es JSON valido`) directo a `transfer-requested-queue` con `aws sqs send-message`, sin pasar por SNS — el `JSON.parse(record.body)` del handler lo revienta de forma confiable y repetible. Logs de CloudWatch confirmaron 3 fallos reales (`SyntaxError`) espaciados ~30s (el visibility timeout default de la cola). El mensaje apareció en `transfer-requested-dlq` con `ApproximateReceiveCount: 4`, verificado con `aws sqs receive-message`. Cola principal quedó en `0` mensajes — el mecanismo protege contra un mensaje envenenado sin bloquear el resto de la cola ni perderlo. Mensaje de prueba borrado de la DLQ después. **Día 3 completo.**

---

## Orden sugerido Día 3

MSG-3 → WORKER-1 → WORKER-2 → WORKER-3 → WORKER-4 → AWS-7 → WORKER-5 → WORKER-6

---

# Tickets — Día 4: Frontend Next.js (backoffice)

Backlog de Día 4, con arquitectura resuelta (ADR-009: polling, no WebSockets/AppSync). Dos partes: rutas de lectura nuevas en `core-api` (necesarias para que el frontend tenga algo que mostrar) y el frontend en sí.

---

## Core API — rutas de lectura

### CORE-11: `AccountsReader` + `ListAccounts` ✅
- [x] Puerto de solo lectura (`listAll(): Promise<Account[]>`) + adaptador DynamoDB (`Scan` sobre `Accounts`).
- [x] Caso de uso `ListAccounts`, DI manual.
- **Criterio de aceptación:** test con el puerto mockeado a mano.
- **Hecho (2026-08-25):** `application/ports/accounts-reader.ts` + `infrastructure/dynamodb/dynamodb-accounts-reader.ts`. Reusa el DTO `Account` de `contracts`, no inventa uno nuevo.

### CORE-12: `LedgerReader` + `ListLedgerEntries` ✅
- [x] Puerto de solo lectura (`listRecent(limit): Promise<LedgerEntry[]>`) + adaptador DynamoDB (`Scan` sobre `Ledger`, filtrando las marcas de idempotencia — `timestamp` que termina en `#IDEMPOTENCY_KEY` no es un evento real —, ordenado por `timestamp` descendente).
- [x] Caso de uso `ListLedgerEntries`, DI manual.
- **Criterio de aceptación:** test con el puerto mockeado a mano, incluyendo que las marcas de idempotencia no aparecen en el resultado.
- **Hecho (2026-08-25):** filtro y orden extraídos como funciones puras (`excludeIdempotencyMarkers`, `sortByTimestampDescending`) testeadas sin DynamoDB — mismo patrón que `processTransfer` en `mock-coelsa`.

### CORE-13: Handler de lectura (`GET /accounts`, `GET /ledger`) ✅
- [x] Una Lambda, dos rutas — despacha por `event.routeKey`.
- [x] Mismo patrón de separación lógica-pura/entrypoint que CORE-7/WORKER-4.
- **Criterio de aceptación:** test del despacho por ruta con los casos de uso mockeados.
- **Hecho (2026-08-25):** `interface/http/read-api.handler.ts` (puro) + `read-api-lambda.ts` (entrypoint real, guión no punto). Ruta desconocida → 404. Error inesperado → 500 sin filtrar el detalle interno (mismo criterio que `create-transfer.handler.ts`).

### AWS-8: Rol IAM para la Lambda de lectura ✅
- [x] Permisos mínimos: `dynamodb:Scan` solo sobre `Accounts`/`Ledger` — de solo lectura, nada de escritura.
- **Criterio de aceptación:** policy revisada línea por línea.
- **Hecho (2026-08-25):** `CoreApiReadLambdaRole`. Un solo permiso de datos (`dynamodb:Scan` sobre las 2 tablas) + logs escopeados a `/aws/lambda/core-api-read`. Sin ningún permiso de escritura — coherente con que esta Lambda solo lee.

### AWS-9: Rutas nuevas en el HTTP API existente + deploy real ✅
- [x] `GET /accounts` y `GET /ledger` agregadas al HTTP API de AWS-5 (mismo JWT authorizer, no uno nuevo).
- **Criterio de aceptación:** ambas rutas responden real contra AWS, protegidas por JWT (401 sin token).
- **Hecho (2026-08-25):** función `core-api-read` desplegada, dos rutas nuevas en el HTTP API `6d5igy4es2` (mismo `CognitoJwtAuthorizer` de AWS-5), permisos de invocación escopeados por ruta (`.../accounts`, `.../ledger`). Verificado real: sin token → `401` en ambas; con `IdToken` válido → `200` con datos reales (3 cuentas, 8 entradas de ledger, sin marcas de idempotencia). **Nota para el frontend (FE-1)**: el primer intento con un token guardado horas antes dio `401` — el `IdToken` de Cognito expira (default ~1h). El frontend necesita manejar esto (re-login o refresh token), no asumir que el JWT dura para siempre. **Segundo hallazgo, durante FE-2**: probando desde el navegador real (no `curl`), las peticiones se colgaban en el preflight `OPTIONS` — el HTTP API nunca tuvo `CorsConfiguration`, porque hasta ese momento solo se había probado con `curl` (que no dispara preflight). Se agregó vía `aws apigatewayv2 update-api --cors-configuration` (`AllowOrigins: ["http://localhost:3000"]` por ahora, `AllowHeaders: ["authorization","content-type"]`) — pendiente sumar el dominio real de Vercel en FE-5.

---

## Frontend

### FE-1: Scaffold Next.js + login Cognito ✅
- [x] Next.js nuevo (App Router), login contra el User Pool de AWS-2 (`USER_PASSWORD_AUTH`, mismo flujo que se usó por CLI toda la sesión), JWT guardado del lado del cliente.
- **Criterio de aceptación:** login real con un usuario demo, JWT visible/usable para las siguientes llamadas.
- **Hecho (2026-08-25):** repo `frontend/` (Next.js 16, React 19, App Router, Tailwind). `lib/cognito.ts` llama directo a la API pública de Cognito (`fetch` plano con `X-Amz-Target`, sin SDK de AWS en el bundle — el App Client no tiene secret, AWS-2). Sesión guardada en `localStorage` (`lib/session.ts`). Probado real en navegador (Chrome vía MCP): login con `demo1@transfer.test` → redirige a `/dashboard` con la sesión activa.

### FE-2: Dashboard — cuentas y balances ✅
- [x] Lista de las 3 cuentas demo con balance, refrescada por polling (`GET /accounts`).
- **Criterio de aceptación:** balances reales, se actualizan solos después de una transferencia sin recargar la página.
- **Hecho (2026-08-25):** `components/AccountsList.tsx`, polling cada 3s. **Hallazgo real probando en navegador**: `GET /accounts`/`GET /ledger` se colgaban en el preflight `OPTIONS` — el HTTP API nunca tuvo CORS configurado (solo se había probado con `curl`, que no dispara preflight). Corregido con `CorsConfiguration` en AWS-9. `lib/api.ts` maneja el refresh de token automático si el backend devuelve 401 (`REFRESH_TOKEN_AUTH`).

### FE-3: Formulario de transferencia ✅
- [x] Form que dispara `POST /transferencias` con un `transferId` generado en el cliente.
- **Criterio de aceptación:** transferencia real de punta a punta disparada desde la UI.
- **Hecho (2026-08-25):** `components/TransferForm.tsx`, `transferId` vía `crypto.randomUUID()`. Probado real en navegador: transferencia de $12,50 de Demo Uno a Demo Dos disparada desde la UI, `202` real.

### FE-4: Actividad reciente (ledger en tiempo real) ✅
- [x] Lista de movimientos recientes por polling (`GET /ledger`), mostrando el estado (`DEBITADO`/`LIQUIDADO`/`REVERTIDO`).
- **Criterio de aceptación:** se ve en la UI la transición `DEBITADO → LIQUIDADO` (o `REVERTIDO`) sin recargar, ~2 segundos después de disparar la transferencia — el pago visual de todo Día 1-3.
- **Hecho (2026-08-25):** `components/LedgerFeed.tsx`. **Confirmado visualmente en el navegador**: balance de Demo Uno bajó de $4.947,50 a $4.935,00, fila `DEBITADO` apareció a las 07:26:48 y pasó sola a `LIQUIDADO` a las 07:26:51 (3s después) — sin recargar la página. Screenshot real tomado como evidencia.

### FE-5: Deploy a Vercel ✅
- [x] Deploy real (no local) — `CLAUDE.md` ya definido: frontend en Vercel, no S3+CloudFront.
- **Criterio de aceptación:** URL pública de Vercel, funcionando contra el backend real de AWS.
- **Hecho (2026-08-25):** proyecto `sashaejarques-projects/frontend` en Vercel, conectado al repo de GitHub (`frontend-transfer-module`, deploys automáticos en cada push a `main`). Variables `NEXT_PUBLIC_*` configuradas en producción y preview (no sensibles — Client ID de Cognito y URL del HTTP API son datos públicos del lado del cliente de todas formas). Deploy a producción: **https://frontend-chi-orpin-63.vercel.app**. Dominio de Vercel agregado al `CorsConfiguration` del HTTP API (junto a `localhost:3000`). Verificado real en el navegador contra la URL pública: login, cuentas reales, transferencia de punta a punta — funciona igual que en local, contra el mismo backend de AWS. **Día 4 completo.**

---

## Orden sugerido Día 4

CORE-11 → CORE-12 → CORE-13 → AWS-8 → AWS-9 → FE-1 → FE-2 → FE-3 → FE-4 → FE-5

---

# Tickets — Día 5: PDF de comprobantes a S3

Backlog de la parte de PDF+S3 de Día 5 (arquitectura resuelta en ADR-010: DynamoDB Streams, no invocación directa del Worker). La parte de IaC con AWS SAM y el README quedan pendientes, no forman parte de este avance — se hacen directamente con el usuario, no en background.

## Infraestructura

### AWS-10: Bucket S3 privado para comprobantes ✅
- [x] Bucket privado, Block Public Access ON sin excepciones.
- **Criterio de aceptación:** `get-public-access-block` confirma los 4 flags en `true`.
- **Hecho (2026-08-25):** `transfer-module-receipts-627551519292` (us-east-1). Block Public Access con los 4 flags en `true`, sin excepciones. Encriptación por defecto SSE-S3 (AES256) y `BucketOwnerEnforced` (ACLs deshabilitadas) agregados además del mínimo pedido.

### AWS-11: DynamoDB Streams en la tabla Ledger ✅
- [x] Habilitar Streams (`NEW_IMAGE`) sobre `Ledger`.
- **Criterio de aceptación:** costo verificado antes de habilitar (ver ADR-010) — always-free con consumidor Lambda.
- **Hecho (2026-08-25):** `StreamViewType: NEW_IMAGE`, `LatestStreamArn` confirmado. `StartingPosition: LATEST` en el event source mapping (no reprocesa el historial previo a la conexión, solo eventos nuevos).

### AWS-12: Rol IAM para la Lambda `pdf-receipt` ✅
- [x] Permisos mínimos: consumir el stream de `Ledger` (`DescribeStream`/`GetRecords`/`GetShardIterator`/`ListStreams` escopeados al stream ARN), `UpdateItem` solo sobre `Ledger`, `PutObject` solo sobre el prefijo `receipts/*` del bucket de AWS-10.
- **Criterio de aceptación:** policy revisada línea por línea, verificada con `aws iam simulate-principal-policy`.
- **Hecho (2026-08-25):** `PdfReceiptLambdaRole`. Verificado: `UpdateItem`/`PutObject` → `allowed`; sin ningún permiso de lectura sobre `Accounts` ni `GetObject` sobre el bucket (no los necesita, solo escribe).

### AWS-13: Permiso `s3:GetObject` para `core-api-read` ✅
- [x] La Lambda de lectura necesita poder firmar (no solo generar la URL, sino que la firma tiene que ser honrada al usarla) `GetObject` sobre el prefijo `receipts/*`.
- **Criterio de aceptación:** una presigned URL generada por el endpoint se puede descargar de verdad sin credenciales de AWS del lado del cliente.
- **Hecho (2026-08-25):** agregado a `CoreApiReadLambdaRole` (también se agregó `dynamodb:Query` sobre `Ledger`, necesario para `findLiquidated`, que no estaba cubierto por el `Scan` existente). Verificado real: `curl` directo a la presigned URL, sin ningún header de autenticación, descargó el PDF real (`200`, `PDF document, version 1.3`).

## Código

### PDF-1: Lambda `pdf-receipt` (repo nuevo) ✅
- [x] Handler disparado por DynamoDB Streams, filtra records `INSERT` de `LIQUIDADO` reales (excluye la marca de idempotencia de CORE-4/WORKER-1).
- [x] Genera el PDF (`pdfkit`, sin binarios nativos) y lo sube a S3 con key `receipts/{transferId}.pdf`.
- [x] Marca el item de `Ledger` con `receiptKey` (nunca una URL), protegido con `ConditionExpression: attribute_not_exists(receiptKey)` para idempotencia ante reentrega de Streams.
- **Criterio de aceptación:** tests unitarios del filtro y de la idempotencia con fakes escritos a mano; invocación directa contra la función real desplegada confirma el PDF en S3 y `receiptKey` en el item.
- **Hecho (2026-08-25):** repo `pdf-receipt/` (git propio, sin `domain/`, mismo patrón que `mock-coelsa`). `receiptKey` agregado a `LedgerEntry` en `contracts` (v0.3.0). Mismo patrón de separación lógica-pura/entrypoint que CORE-7/WORKER-4 (`receipt.handler.ts` + `receipt-lambda.ts`, guión en el entrypoint). 9 tests. Invocado directo contra un `LIQUIDADO` real ya existente en la tabla → PDF real de 1899 bytes en S3, `receiptKey` seteado en el item real.

### PDF-2: `GET /transfers/{transferId}/receipt` en core-api ✅
- [x] `LedgerReader.findLiquidated(transferId)` (Query por partition key, no Scan) + puerto `ReceiptUrlSigner` + caso de uso `GetTransferReceipt`.
- [x] Presigned URL de expiración corta (5 minutos) — nunca se expone la URL directa del bucket.
- [x] Ruta nueva en la Lambda de lectura existente (`read-api.handler.ts`, ya despachaba por `event.routeKey` desde CORE-13) — no una Lambda aparte.
- **Criterio de aceptación:** `404` si la transferencia no está liquidada o el comprobante todavía no se generó (ventana de Streams); `200` con una URL que descarga el PDF real cuando sí existe.
- **Hecho (2026-08-25):** 3 tests nuevos con fakes a mano (URL firmada, 404 sin liquidar, 404 en la ventana de eventual consistency antes de que Streams procese). Verificado real de punta a punta: una transferencia nueva disparada por `POST /transferencias` llegó sola a `LIQUIDADO` con `receiptKey`, y `GET /transfers/{id}/receipt` devolvió una presigned URL que efectivamente descargó el PDF — sin ninguna invocación manual entre el `POST` inicial y la descarga final.

---

## Orden sugerido Día 5 (PDF+S3)

AWS-10 → AWS-11 → PDF-1 → AWS-12 → AWS-13 → PDF-2

---

## IaC + README (cola de Día 5)

### INFRA-1: Template de AWS SAM de toda la infraestructura real ✅
- [x] Repo nuevo `infra` (`transfer-module-infra` en GitHub), mismo patrón polyrepo que el resto (ADR-001).
- [x] `template.yaml` con DynamoDB (Accounts/Ledger, Streams, GSI), S3 de comprobantes, SNS+SQS+DLQ, Cognito, las 5 Lambdas con sus roles de mínimo privilegio, y el HTTP API con throttle por ruta.
- **Criterio de aceptación:** cada recurso verificado contra la cuenta real con `aws-cli` (no de memoria); `sam validate --lint` y `sam build` corren en limpio, local, sin tocar AWS.
- **Hecho (2026-08-26):** decisiones de diseño del template (un solo `template.yaml` sin nested stacks, API Gateway declarado crudo en vez del shorthand `AWS::Serverless::HttpApi`) documentadas en ADR-015 de `DECISIONS.md`. `sam build` compiló las 5 funciones contra el código real de `core-api`/`worker`/`mock-coelsa`/`pdf-receipt` (repos hermanos). No se corrió `sam deploy` — los recursos reales no están bajo CloudFormation y tienen datos en uso; el camino de `CloudFormation import` queda documentado en el README de `transfer-module-infra`, sin ejecutar, pendiente de aprobación humana puntual.

### INFRA-2: README del proyecto ✅
- [x] `README.md` en la raíz: qué es el proyecto, arquitectura real, disciplina de costo $0, cómo correr/desplegar cada repo, links a los 6+1 repos, nota sobre las credenciales demo públicas (ADR-011).
- **Hecho (2026-08-26):** ver `README.md` en la raíz de `transfer-module/`.

---

# Tickets — Día 6: Observabilidad (X-Ray, Budget Alerts, CloudTrail)

Backlog de Día 6, resuelto en una sola sesión con deploy real contra AWS. Decisiones y costos documentados en ADR-012/013/014 de `DECISIONS.md`.

## X-Ray

### XRAY-1: Tracing activo en las 5 Lambdas del proyecto ✅
- [x] `TracingConfig: Mode=Active` en `core-api-create-transfer`, `core-api-read`, `worker-process-transfer-requested`, `mock-coelsa`, `pdf-receipt`.
- [x] Policy administrada `AWSXRayDaemonWriteAccess` agregada a cada rol de ejecución (aditivo, sin tocar las policies inline existentes).
- **Criterio de aceptación:** costo verificado antes de activarlo (100.000 trazas grabadas/mes always-free); una transferencia real produce un trace real visible con `aws xray batch-get-traces`.
- **Hecho (2026-08-26):** verificado con `aws lambda get-function-configuration` en las 5 funciones (`TracingConfig.Mode: Active`, `LastUpdateStatus: Successful`). Disparada una transferencia real (`POST /transferencias`), `aws xray get-trace-summaries` mostró trazas reales de los minutos siguientes, y `aws xray batch-get-traces` sobre una de ellas confirmó segmentos correlacionados de `worker-process-transfer-requested` y `mock-coelsa` como parte de un mismo trace — tracing distribuido cross-Lambda funcionando de punta a punta. **Hallazgo:** el HTTP API (`core-api-http`) no genera segmento de X-Ray a nivel de gateway (esa integración solo existe para REST API v1) — ver ADR-012 para el porqué de no migrar por esto.

## CloudWatch Budgets

### BUDGET-1: Alerta de costo temprana, complementaria al guardarail existente ✅
- [x] Budget `TransferModuleCostWatch` ($5/mes), sin Budgets Actions (solo notificación), dos alertas por email a `sashax191@gmail.com`: gasto real >50% y gasto proyectado >100%.
- **Criterio de aceptación:** costo verificado antes de crearlo (budgets solo-notificación son gratis sin límite, distinto de los *action-enabled* que sí tienen tope gratuito de ~2/cuenta); `aws budgets describe-budgets` confirma el budget activo.
- **Hecho (2026-08-26):** la cuenta ya tenía un budget previo (`DenyAllOnBudgetExceeded`, $1/mes, con Budgets Action automática de deny IAM al 80% real) — se lo dejó intacto como guardarail duro y se sumó este segundo, sin acción, como aviso previo. Verificado con `describe-budgets` + `describe-notifications-for-budget`. Gasto real del mes verificado con `aws ce get-cost-and-usage`: `$0`.

## CloudTrail

### TRAIL-1: Bucket S3 dedicado para los logs ✅
- [x] Bucket nuevo `transfer-module-cloudtrail-627551519292`, separado del bucket de comprobantes (no mezclar propósitos). Block Public Access los 4 flags en `true`, SSE-S3, bucket policy escopeada al ARN exacto del trail (no genérica).
- **Criterio de aceptación:** `get-public-access-block` confirma los 4 flags en `true`.
- **Hecho (2026-08-26):** verificado con `aws s3api get-public-access-block`.

### TRAIL-2: Trail de management events únicamente ✅
- [x] Trail `transfer-module-trail`, multi-región (incluye eventos globales de IAM/STS), `IncludeManagementEvents: true` / `ReadWriteType: All` / `DataResources: []` — explícitamente sin Data Events, sin Insights, sin CloudTrail Lake.
- **Criterio de aceptación:** costo verificado antes de crearlo (primer trail de management events por cuenta/región es gratis; Data Events/Insights/Lake sí cobran desde el primer evento — se volvió a chequear en esta sesión, no se asumió de la nota de la sesión anterior); `aws cloudtrail lookup-events` muestra eventos de management reales.
- **Hecho (2026-08-26):** `get-event-selectors` confirmó `DataResources: []`; `get-insight-selectors` devolvió `InsightNotEnabledException` (Insights no activo). `start-logging` confirmado con `get-trail-status` (`IsLogging: true`). `aws cloudtrail lookup-events` mostró eventos reales y recientes de esta misma sesión de trabajo (incluido el propio `DescribeBudgetActionsForBudget` de instantes antes).

---

## Orden sugerido Día 6

XRAY-1 → BUDGET-1 → TRAIL-1 → TRAIL-2
