Actúa como Arquitecto de Software Senior + DevOps. Genera un **Monorepo Full-Stack** completo, funcional y listo para levantar con un solo comando `docker compose up --build`. NO generes placeholders vacíos: el CRUD debe funcionar contra PostgreSQL de verdad.

### 1. Stack Obligatorio
- **Monorepo:** pnpm Workspaces + Turborepo
- **Backend:** Node.js 20 + Express 4 + TypeScript
- **ORM:** Prisma 5
- **Base de Datos:** PostgreSQL 16
- **Frontend:** Next.js 14 (App Router) + Tailwind + TypeScript
- **Contenedores:** Docker multi-stage + docker-compose

### 2. Estructura Exacta Requerida
.
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── index.ts              # bootstrap Express
│   │   │   ├── app.ts                # configuración express (cors, json, rutas)
│   │   │   ├── routes/
│   │   │   │   ├── health.routes.ts
│   │   │   │   └── users.routes.ts
│   │   │   ├── controllers/
│   │   │   │   └── users.controller.ts
│   │   │   ├── services/
│   │   │   │   └── users.service.ts
│   │   │   ├── middlewares/
│   │   │   │   └── errorHandler.ts
│   │   │   └── lib/prisma.ts         # singleton PrismaClient
│   │   ├── prisma/
│   │   │   ├── schema.prisma         # modelo User (id, email, name, createdAt)
│   │   │   └── seed.ts               # inserta 2 users de ejemplo
│   │   ├── Dockerfile
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── web/
│       ├── src/app/
│       │   ├── page.tsx              # lista catalogo 
│       │   ├── layout.tsx
│       │   └── globals.css
│       │   └── 
│       ├── src/lib/api.ts            # fetch wrapper al backend
│       ├── Dockerfile
│       ├── package.json
│       ├── next.config.js
│       └── tailwind.config.ts
├── packages/
│   ├── shared/
│   │   ├── src/types.ts              # interface User + CreateUserDto
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── tsconfig/
│       ├── base.json
│       └── package.json
├── docker-compose.yml
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── .env.example
└── README.md

### 3. Código Funcional Requerido (NO placeholders)

**Backend (Express):**
- `GET /health` → `{ status: "ok", timestamp: ... }`
- `GET /api/users` → lista todos los users desde Postgres
- `POST /api/users` → crea user (valida email único, retorna 201)
- `GET /api/users/:id` → obtiene uno
- `DELETE /api/users/:id` → elimina
- Middleware `errorHandler` centralizado (captura Prisma errors, 404, 500)
- CORS habilitado para `http://localhost:3000`
- `lib/prisma.ts` exporta un único `PrismaClient` (evita hot-reload leaks)

**Frontend (Next.js):**
- Página principal hace `fetch` a `GET /api/catalog` en Server Component
- Formulario (Client Component) hace `POST /api/catalog` con `name` y `email`
- Muestra lista de users con botón eliminar (DELETE)
- Revalida con `revalidatePath('/')` tras crear/eliminar
- `API_URL` leída de env var (`http://api:3001` en Docker, `http://localhost:3001` en local)

**Prisma:**
- `schema.prisma` con modelo `User { id String @id @default(cuid()), email String @unique, name String, createdAt DateTime @default(now()) }`
- `seed.ts` que inserta 2 users vía `prisma.user.createMany`
- Script `prisma:seed` en package.json

### 4. Docker (CRÍTICO — multi-stage)

**apps/api/Dockerfile:**
- Etapa `base`: node:20-alpine + pnpm
- Etapa `deps`: instala dependencias del workspace (usa `pnpm install --frozen-lockfile`)
- Etapa `build`: `pnpm --filter api build` (genera `dist/`)
- Etapa `runner`: copia solo `dist`, `node_modules` de prod y `prisma/`; corre `prisma migrate deploy` antes de `node dist/index.js`

**apps/web/Dockerfile:**
- Etapas `deps`, `builder`, `runner` (patrón oficial Next.js standalone)
- `output: 'standalone'` en `next.config.js`

**docker-compose.yml:**
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
      POSTGRES_DB: appdb
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    build: { context: ., dockerfile: apps/api/Dockerfile }
    environment:
      DATABASE_URL: postgresql://appuser:apppass@db:5432/appdb?schema=public
      PORT: 3001
      NODE_ENV: production
    ports: ["3001:3001"]
    depends_on:
      db: { condition: service_healthy }

  web:
    build: { context: ., dockerfile: apps/web/Dockerfile }
    environment:
      API_URL: http://api:3001
      NEXT_PUBLIC_API_URL: http://localhost:3001
    ports: ["3000:3000"]
    depends_on: [api]

volumes:
  pgdata:
