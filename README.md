# FrogPay API

NestJS + Prisma + PostgreSQL (Supabase, con RLS) + RabbitMQ + Redis.

## Estructura

```
apps/api/
├── prisma/
│   ├── schema.prisma         # Modelo de datos (fuente: ERD de Vertabelo)
│   ├── migrations/           # Migraciones generadas por Prisma. No se editan a mano
│   └── seed.ts               # Datos iniciales: planes Free y Premium
├── src/
│   ├── main.ts               # Bootstrap de la app
│   ├── app.module.ts         # Módulo raíz: solo importa módulos
│   ├── config/               # Carga y validación de variables de entorno (falla al arrancar si falta alguna)
│   ├── shared/               # Código transversal, SIN lógica de negocio
│   │   ├── domain/           # Clases base: Entity, DomainEvent, Result
│   │   ├── events/           # Interfaz EventBus, implementación RabbitMQ, catálogo de nombres de eventos
│   │   ├── database/         # PrismaService y contexto de tenant para RLS
│   │   ├── auth/             # Guards (JWT, ApiKey, Roles) y decoradores (@CurrentTenant, @Roles)
│   │   ├── http/             # Filtros de excepción, interceptores y formato estándar de error
│   │   └── utils/            # Helpers puros (fechas, dinero, ids)
│   └── modules/              # Un módulo = un dominio de negocio
│       ├── identity/         # Tenants, usuarios, invitaciones, login, API Keys
│       ├── plans/            # Planes Free/Premium y límites de uso
│       ├── payments/         # Núcleo de pagos (ciclo de vida, idempotencia)
│       ├── provider-adapters/# Integraciones con proveedores (Adapter + Strategy)
│       ├── webhooks/         # Envío firmado (HMAC) a tenants, reintentos y DLQ
│       ├── notifications/    # Correos (invitación de owner)
│       ├── audit/            # Log de auditoría inmutable (append-only)
│       └── health/           # Endpoint /health
└── test/
    ├── integration/          # Pruebas contra BD y bus reales (docker compose)
    └── e2e/                  # Flujos completos por HTTP
```

### Estructura interna de cada módulo de negocio

```
modules/identity/
├── domain/                   # El "qué": reglas de negocio puras, sin NestJS ni Prisma
│   ├── entities/             # Tenant, User, ApiKey
│   ├── value-objects/        # Email, TaxId, UserStatus (invited/active)
│   ├── events/               # Eventos que emite el módulo: tenant.creado, usuario.activado
│   └── repositories/         # INTERFACES (puertos) de persistencia
├── application/              # El "cómo": orquesta el dominio
│   ├── use-cases/            # CreateTenant, ActivateAccount, GenerateApiKey…
│   ├── dto/                  # Entrada/salida validada (class-validator)
│   └── event-handlers/       # Reacciones a eventos de OTROS módulos
├── infrastructure/           # Detalles técnicos intercambiables
│   └── persistence/          # Implementación Prisma de los repositorios
├── presentation/
│   └── http/                 # Controllers REST (delgados: validan y llaman a un use-case)
└── identity.module.ts
```

Módulos especiales:

- `provider-adapters/`: `ports/` define la interfaz `PaymentProviderPort`; `adapters/stripe` y `adapters/qr-bcb` la implementan; `registry/` elige el adapter por configuración (Strategy). Agregar un proveedor = agregar una carpeta en `adapters/` sin tocar `payments/` (RF-16, RNF-07).
- `health/`: `indicators/` revisa la API, la BD y RabbitMQ; responde 200 o 503.

### Dónde va cada tarea del Sprint 1

| Tarea | Carpeta |
|---|---|
| TSK-ARQ/DEVs-100 Modelo E-R y migraciones | `prisma/`, `docs/erd/` |
| TSK-DEV1-101 POST /tenants | `modules/identity` |
| TSK-BACK1-102 API Key / Secret | `modules/identity` (`infrastructure/` para el hash) |
| TSK-BACK1-103 Correo de invitación | `modules/notifications` (escucha `tenant.creado`) |
| TSK-BACK2-201 / 202 Planes | `modules/plans`, `prisma/seed.ts` |
| TSK-BACK2-203 RLS | `shared/database`, `prisma/migrations` |
| TSK-ARQ/BACK1-104 Bus de eventos | `shared/events`, `infra/rabbitmq` |
| TSK-BACK1-105 /health | `modules/health` |

## Reglas de la arquitectura

1. **Regla de dependencia:** `presentation → application → domain`. `domain/` nunca importa NestJS, Prisma ni nada de `infrastructure/`.
2. **Los módulos no se importan entre sí por dentro.** Se comunican por **eventos** del bus. La única excepción son consultas síncronas imprescindibles (p. ej. "límite restante del plan"), que se hacen a través de un servicio que el módulo **exporta explícitamente**.
3. **El `tenant_id` nunca viene del body.** Siempre se obtiene del JWT o de la API Key (`@CurrentTenant`).
4. **RLS con Prisma:** la API se conecta a Supabase con un rol **sin** `BYPASSRLS` (nunca como `postgres`). En cada request, `shared/database` ejecuta la consulta dentro de una transacción que primero hace `set_config('app.current_tenant', <id>, true)`, y las políticas RLS filtran por ese valor.
5. **Nombres de eventos:** `<dominio>.<acción en participio>` en español: `tenant.creado`, `pago.aprobado`. Las colas se nombran `<modulo>.<evento>` y su DLQ `<cola>.dlq`.
6. **Pruebas unitarias** junto al archivo (`create-tenant.use-case.spec.ts`). Las de integración y e2e van en `test/`.
7. **Idioma:** el código (clases, carpetas, variables) va en inglés; los eventos y los mensajes al usuario, en español.
