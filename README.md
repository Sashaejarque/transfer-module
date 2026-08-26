# Nexo — Sistema de Transferencias Bancarias (MVP portfolio)

Backoffice de transferencias entre 3 cuentas demo, con liquidación asíncrona real contra una arquitectura orientada a eventos en AWS: Cognito, API Gateway, dos Lambdas de Core API, SNS+SQS, un Worker Lambda, un mock de la cámara compensadora COELSA, DynamoDB como ledger append-only, Streams disparando una Lambda de generación de PDF, y S3 privado con presigned URLs.

**Demo en vivo:** https://frontend-chi-orpin-63.vercel.app
**Credenciales demo (públicas a propósito, ver más abajo):** `demo1@transfer.test` / `demo2@transfer.test` / `demo3@transfer.test`, contraseña `Admin123!`

La UI es deliberadamente simple — es un backoffice de 3 cuentas, no un producto. Lo que vale la pena mirar es lo que corre atrás: entrá a **`/dashboard/arquitectura`** una vez logueado — hay un diagrama en vivo de la arquitectura real que se anima con cada transferencia real que dispares, con el motivo de cada decisión un click de distancia.

## Arquitectura

![Arquitectura del sistema: Frontend autenticado con Cognito llama a API Gateway, que invoca Core API; Core API debita atómicamente en DynamoDB y publica a SNS, que entra a SQS y dispara al Worker; el Worker invoca Mock COELSA y según el resultado el ledger queda LIQUIDADO o REVERTIDO; si liquida, DynamoDB Streams dispara la Lambda de PDF que sube el comprobante a S3.](docs/architecture.svg)

Cada salto es real — nada de esto está simulado en el frontend; la página de arquitectura dispara una transferencia real de $0,01 y refleja eventos reales observados por polling contra `/ledger` y `/transfers/{id}/receipt`.

## Disciplina de costo: $0 por contrato, no "barato en la práctica"

Regla de oro del proyecto (ver `CLAUDE.md`): todo dentro de capas **always-free** de AWS, nunca free-tier legacy de 12 meses que después empieza a cobrar. Servicios prohibidos por no tener capa permanente: EC2, RDS, NAT Gateway, MSK, DocumentDB, ElastiCache. Cada decisión de servicio en `DECISIONS.md` (ADR-001 a ADR-015) trae su costo verificado explícitamente antes de tomarse, no asumido. Dos excepciones declaradas a propósito, no descuidos:

- **S3** no tiene capa always-free (solo 12 meses o créditos de cuenta nueva) — se mantiene igual por el valor de portfolio de demostrar el patrón bucket-privado + presigned URL, con fecha de revisión fijada para julio de 2027 (ADR-011).
- Al ser una demo pública con credenciales conocidas, se agregaron medidas anti-abuso $0 (throttle por ruta en API Gateway, lifecycle de 30 días en S3) — ver la nota "Medidas anti-abuso para demo pública" en `DECISIONS.md`.

## Los 7 repos

| Repo | Qué es |
|---|---|
| [contracts](https://github.com/Sashaejarque/contracts-transfer-module) | DTOs/tipos TypeScript compartidos, sin runtime — versionado por git tag (ADR-004) |
| [core-api](https://github.com/Sashaejarque/core-api-transfer-module) | `POST /transferencias`, `GET /accounts`, `/ledger`, `/transfers/{id}/receipt` |
| [worker](https://github.com/Sashaejarque/worker-transfer-module) | Consume SQS, liquida o revierte, idempotente |
| [mock-coelsa](https://github.com/Sashaejarque/mock-coelsa-transfer-module) | Simula la cámara compensadora: 1.5s de latencia, ~15% de fallas |
| [pdf-receipt](https://github.com/Sashaejarque/pdf-receipt-transfer-module) | Genera el comprobante PDF disparado por DynamoDB Streams, lo sube a S3 |
| [frontend](https://github.com/Sashaejarque/frontend-transfer-module) | Next.js — backoffice + diagrama de arquitectura en vivo |
| [infra](https://github.com/Sashaejarque/transfer-module-infra) | Template de AWS SAM de toda la infraestructura real (ver ADR-015) |

Polyrepo deliberado, un repo por bounded context (ADR-001) — no es indecisión, es simular límites de servicio reales incluso desarrollando en solitario.

## Cómo correr cada parte

Cada Lambda (`core-api`, `worker`, `mock-coelsa`, `pdf-receipt`) es un proyecto Node.js + TypeScript independiente:

```bash
npm install
npm test        # vitest, con fakes escritos a mano -- sin mockear AWS
npm run build   # compila a dist/
```

El frontend:

```bash
cd frontend
npm install
cp .env.example .env.local   # completar con el API endpoint y el Client ID de Cognito reales
npm run dev
```

La infraestructura (`infra/`) es documentación viva del estado real de la cuenta, no el mecanismo de deploy actual — ver el README de ese repo antes de tocarlo. Los recursos reales se desplegaron a mano, sesión por sesión, con `aws-cli`, y así siguen: `sam validate`/`sam build` corren local sin tocar AWS; `sam deploy` contra la cuenta real **no está habilitado** todavía (recursos con datos reales en uso, sin CloudFormation de por medio — el camino de import queda documentado, no ejecutado).

## Credenciales demo públicas — por qué

Es un portfolio, no un sistema con usuarios reales. Las tres cuentas demo y su contraseña están escritas en la propia pantalla de login a propósito, para que cualquiera pueda probar el flujo completo sin pedir acceso. Eso trae un riesgo real (cualquiera puede generar transferencias en loop), mitigado con throttle en `POST /transferencias` y lifecycle de 30 días en el bucket de comprobantes — ver la nota "Medidas anti-abuso para demo pública" en `DECISIONS.md`.

## Cómo se construyó con IA

Todo el proyecto se hizo en pair-programming con Claude Code, pero no de la misma forma de punta a punta — el modo de trabajo cambió a propósito según la etapa:

**Días 1 a 4 (núcleo funcional): mentor, no autocomplete.** `CLAUDE.md` fija el modo de trabajo explícitamente: antes de cada pieza nueva, preguntas conceptuales para chequear que la elección se entendía (no que "andaba"), plan aprobado antes de escribir código, y costo explícito de cada servicio de AWS nuevo (always-free vs. free-tier de 12 meses vs. consume créditos) verificado antes de usarlo, no asumido después. `STUDY.md` es el residuo de eso: preguntas que quedaron pendientes de masticar en el momento para no cortar el ritmo, respondidas completas al final para repasar.

**Día 5/6 (cierre): dos agentes autónomos en paralelo, con permisos distintos a propósito.** Para terminar el proyecto en una sesión, se lanzaron dos subagentes de fondo en simultáneo — uno para Día 5 (IaC con SAM + README), otro para Día 6 (X-Ray, CloudWatch Budgets, CloudTrail) — cada uno con un límite de autoridad distinto según qué tan reversible era lo que iban a tocar:

- El agente de **Día 6** trabajaba solo con recursos nuevos y aditivos (nada que reemplazar ni borrar), así que tuvo autonomía completa: investigar el costo real, implementar, desplegar contra AWS de verdad, y verificar el resultado real (no solo "se creó" — una traza real de X-Ray, un evento real en CloudTrail).
- El agente de **Día 5** iba a describir en SAM infraestructura que ya estaba viva y en uso real (usuarios de Cognito, todo el historial del ledger) — se le prohibió explícitamente ejecutar un deploy que pudiera reemplazar o borrar esos recursos. Podía escribir y validar los templates (`sam validate`/`sam build`, ambos locales), pero el paso de traer los recursos existentes bajo control de CloudFormation quedó documentado como decisión pendiente, no ejecutado.

Terminados los dos, cada resultado se volvió a verificar a mano contra AWS real antes de darlo por bueno (`aws budgets describe-budgets`, `aws cloudtrail describe-trails`, `aws lambda get-function-configuration` para X-Ray, el repo nuevo existiendo de verdad en GitHub) — no alcanza con que el agente diga que lo hizo.

## Documentación de decisiones

Cada decisión de arquitectura no obvia está en `DECISIONS.md`, formato ADR (Architecture Decision Record): contexto, decisión, costo verificado, alternativas consideradas y consecuencias. `TICKETS.md` tiene el backlog día por día tal como se ejecutó. `GLOSSARY.md` y `STUDY.md` son notas de estudio del curso base de AWS que precedió a este proyecto.
