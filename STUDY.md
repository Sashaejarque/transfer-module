# STUDY.md — Guía de repaso

Preguntas conceptuales que quedaron sin masticar del todo durante el desarrollo (para no frenar el ritmo), con la respuesta completa acá para repasar cuando termine el proyecto. Orden cronológico por ticket.

---

## CORE-6 — Caso de uso `CreateTransfer`

**Pregunta 1:** ¿En qué orden hace `CreateTransfer` sus tres pasos (validar DTO, debitar, publicar evento), y por qué el publish va al final?

**Respuesta:** Orden: (1) construir/validar las entidades de dominio a partir del DTO — sin tocar AWS, falla rápido si el input es inválido (monto ≤ 0, cuentas iguales) — (2) `accountRepository.debitForTransfer(...)`, (3) `eventPublisher.publishTransferRequested(...)` recién al final.

El motivo de que el publish vaya último: publicar el evento es lo que va a disparar al Worker (Día 2) para que arranque el proceso async hacia COELSA. Si publicáramos *antes* de debitar y el débito después fallara (balance insuficiente, error de red con DynamoDB), estaríamos anunciándole al resto del sistema "esta transferencia está en curso" para una transferencia que nunca quedó registrada — el Worker recibiría un evento fantasma sobre algo que no existe en el ledger. Publicar al final garantiza que el evento solo se emite para una transferencia que ya quedó grabada de forma atómica y durable como `DEBITADO`. Es el mismo principio de fondo que un patrón *outbox*: nunca anunciás un cambio de estado que todavía no confirmaste.

**Pregunta 2:** ¿`CreateTransfer` tiene que leer el `Account` actual del repositorio, llamar `account.debit()` (CORE-3) en memoria, y guardar eso — o llama directo a `accountRepository.debitForTransfer()` sin leer el balance antes?

**Respuesta:** Directo al repositorio, sin lectura previa. Si leyéramos el balance primero (`GetItem`), construyéramos un `Account` y llamáramos `account.debit()` en memoria, y recién después intentáramos persistir ese nuevo balance, reintroduciríamos exactamente la carrera que resolvimos en CORE-4: dos requests concurrentes podrían leer el mismo balance "suficiente" antes de que ninguno haya escrito nada, decidir cada uno por su cuenta que está OK, e intentar escribir los dos — el `ConditionExpression` evaluado en el momento de la escritura en DynamoDB es lo que realmente evita el double-spend, no ningún chequeo que hagamos antes en JS.

**Observación honesta**: esto significa que `Account.debit()` (el método que armamos en CORE-3) **no se llama en ningún lugar de este flujo real**. Sigue teniendo valor — modela el invariante para sus propios tests unitarios en aislamiento, y serviría si algún día necesitáramos una simulación en memoria sin tocar DynamoDB — pero el camino de producción salta directo al repositorio. Lo que sí se usa de CORE-3 en este caso de uso es `Transfer.create(...)`, para validar las reglas de negocio (monto positivo, cuentas distintas) sin I/O antes de tocar AWS.

---

## CORE-7 — Handler Lambda

**Pregunta 1:** La composition root (donde se decide qué adaptador concreto usar) tiene que vivir a nivel de módulo, *fuera* del handler exportado — nunca adentro de la función que Lambda invoca en cada request. ¿Por qué importa dónde vive ese código?

**Respuesta:** El modelo de ejecución de Lambda reutiliza el mismo contenedor/proceso Node entre invocaciones mientras esté "caliente" (no se crea un proceso nuevo por cada request). El código que está *afuera* del handler exportado corre una sola vez por contenedor — en el primer cold start — y después se reusa para todas las invocaciones siguientes que caigan en ese mismo contenedor. Si la composition root (armar el `DynamoDBDocumentClient`, instanciar los repositorios) estuviera *adentro* del handler, se repetiría en cada request: reconectar el cliente de DynamoDB, releer variables de entorno, todo de nuevo — lastre de latencia innecesario, y en un caso real hasta riesgo de agotar conexiones. Por eso el proyecto terminó con dos archivos separados: `create-transfer.handler.ts` (la lógica pura, `buildHandler(createTransfer)`, testeable sin AWS) y `create-transfer.lambda.ts` (el entrypoint real, con la composition root a nivel de módulo y el `export const handler` que Lambda invoca).

**Pregunta 2:** ¿Por qué separar el handler en dos archivos (`create-transfer.handler.ts` puro + `create-transfer.lambda.ts` con la composition root), en vez de tener todo junto en un solo archivo como se planeó al principio?

**Respuesta:** Se descubrió en la práctica, no fue una decisión previa: al escribir el test del handler, con solo hacer `import { buildHandler } from "./create-transfer.handler.js"` ya explotaba, porque ese archivo tenía la composition root a nivel de módulo (`requireEnv("ACCOUNTS_TABLE_NAME")`, etc.) — y en el entorno de test no hay esas variables de entorno seteadas. El problema de fondo: en JS/TS, importar un módulo ejecuta *todo* su código de nivel superior, no solo lo que declarás usar. Si ese código de nivel superior tiene efectos secundarios (leer env vars, crear un cliente AWS), cualquiera que solo quiera importar una función pura de ese archivo se lleva puestos esos efectos secundarios igual. La solución fue separar "lógica testeable" de "wiring con efectos secundarios" en dos archivos distintos — patrón común en Lambda: un módulo delgado de entrada (el entrypoint real que declarás en la config de Lambda) y la lógica de negocio en un módulo aparte, sin importaciones que disparen nada al cargarlas.

**Pregunta 3 (mapeo de errores):** `DomainError` tiene un campo `code` (`INVALID_AMOUNT`, `INSUFFICIENT_BALANCE`, etc., de CORE-3/CORE-4). En el handler, ese código se mapea a un status HTTP con una tabla (`Record<string, number>`) en vez de una cadena de `if (error instanceof InvalidAmountError) ... else if (error instanceof InsufficientBalanceError) ...`. ¿Qué ventaja tiene la tabla?

**Respuesta:** Con la cadena de `instanceof`, agregar un nuevo error de dominio significa tocar el handler (agregar un `else if` más). Con la tabla `DOMAIN_ERROR_STATUS`, agregar un nuevo error de dominio en CORE-3 solo requiere agregar una fila a esa tabla (`NUEVO_CODIGO: 400`) — el resto del código del handler no cambia, y si alguien se olvida de agregar la fila, cae en el default (400) en vez de romper. También separa una decisión puramente de infraestructura (qué status HTTP corresponde a qué código de negocio) de la lógica del dominio, que no sabe nada de HTTP.
