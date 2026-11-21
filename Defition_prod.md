Actúa como Arquitecto de Software Senior + DevOps. Genera un **Monorepo Full-Stack** completo, funcional y listo para levantar con un solo comando `docker compose up --build`. NO generes placeholders vacíos: el CRUD debe funcionar contra PostgreSQL de verdad.

### 1. Stack Obligatorio
- **Monorepo:** pnpm Workspaces + Turborepo
- **Backend:** Node.js 20 + Express 4 + TypeScript
- **ORM:** Prisma 5
- **Base de Datos:** PostgreSQL 16
- **Frontend:** Next.js 14 (App Router) + Tailwind + TypeScript
- **Contenedores:** Docker multi-stage + docker-compose


## Alcance del proyecto 
se va a desarrollar una lista de articulos, precio, si tiene promo.
se contempla un admin panel para agregar articulos
si un usuario inicia sesion con:
- fb, intagram o google. 
podra apartar un articulo, este apartado generar un qr para pasar a pagar el porcentaje
que indique el admin. 

validara este apartado como, aprobado o pagado. 
- se tendra una lista de apartados ordenados por fecha del mas reciente,
con filtros de aprobados, pendientes.
cada apartado tiene una liberación de 2 horas ( configurable )

contara con un costo por promocion, cuando se quiera liberar el producto, el consto aplica, sobre un porcentaje menos al precio actual. 
- ¿como saber a que producto entra en promo?
- se tiene que indicar que producto entrar a promo.

esta promo entra en un draw & drop 
una lista de dos columnas mis producto, con buscador por nombre y al lado derecho,
los productos que entra en la promo. 
la otra vista sera un lista con check box, una lista diminuta. 

### Definicion de la DB 
los datos que se requiere es 

model User {
  id            String   @id @default(cuid())
  email         String?  @unique
  name          String?
  provider      String   // google | facebook | instagram
  providerId    String
  avatar        String?
  createdAt     DateTime @default(now())
  reservations  Reservation[]
}

model Admin {
  id        String   @id @default(cuid())
  email     String   @unique
  password  String   // hashed
  name      String?
  createdAt DateTime @default(now())
}

model Product {
  id          String   @id @default(cuid())
  name        String
  description String?
  price       Decimal  @db.Decimal(10, 2)
  cost        Decimal? @db.Decimal(10, 2)
  stock       Int      @default(0)
  gender      String?  // male | female | unisex
  isActive    Boolean  @default(true)
  isPromo     Boolean  @default(false)
  promoPercent Int?    // porcentaje de descuento (ej: 15)
  images      ProductImage[]
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  reservations Reservation[]
}

model ProductImage {
  id        String  @id @default(cuid())
  url       String
  order     Int     @default(0) // 1,2,3
  productId String
  product   Product @relation(fields: [productId], references: [id], onDelete: Cascade)
}

model SystemConfig {
  id                    String   @id @default(cuid())
  reservationHours      Int      @default(2)          // horas para que expire
  defaultPaymentPercent Int      @default(30)         // % que se debe pagar al apartar
  updatedAt             DateTime @updatedAt
  updatedBy             String?                       // id del admin que lo cambió
}

model Reservation {
  id              String   @id @default(cuid())
  userId          String
  productId       String
  status          ReservationStatus @default(pending)
  reservedAt      DateTime @default(now())
  expiresAt       DateTime // reservedAt + reservationHours
  paymentPercent Int      // % que debe pagar (definido por admin)
  amountToPay     Decimal  @db.Decimal(10, 2)
  reservationCode String @unique // código legible (ej: APR-2026-00123)
  qrCode          String?  // base64 o url del QR
  // Control de admin
  approvedAt        DateTime?
  approvedBy        String?                           // id del admin
  rejectedAt        DateTime?
  rejectedBy        String?
  rejectionReason   String?
  paidAt            DateTime?
  user            User     @relation(fields: [userId], references: [id])
  product         Product  @relation(fields: [productId], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([status, expiresAt])
  @@index([userId])
  @@index([productId])
}

enum ReservationStatus {
  pending     // recién creado, esperando admin
  approved    // admin aprobó → stock descontado
  rejected    // admin rechazó
  expired     // pasó el tiempo sin acción
  paid        // se confirmó el pago
  cancelled   // el usuario canceló
}


### Definicion de servicio api
* `/api/login`
    - proveera servicion de redes sociales para inicio de sesion - crear cuenta. 
    -  with Passport.js

* `GET /api/admin/login`
    - proveera servicion login, correo y contraseña para la administración. 


* `GET /api/catalog`
    - se enlista el catalago disponible, utilizando paginacion en el el back 
    - y en el front cargar de forma infinita pero por partes. 

* `POST /api/catalog/create`
     - crear
     - actulizar datos
        - disminir stock
     - des-habilitar
* `UPDATE /api/catalog/:id`
    - Actuliza el articulo

* `GET /api/catalog/:id`
    - optine el detalle de un articulo/producto

* `GET /api/catalog?genero=male&stock=true`
    - filtro para listar por:
        - genero
        - disponibilidad

* `GET /api/reservations`
    - obtiene el listado de reservaciones:
        - middleware de auth solo para admin
        - vista protegida
* `UPDATE /api/reservations/:id`
    - actulizaciónes de la reserva
    - status

- agregar un nuevo articulo
 - nombre
 - foto
    - foto_1
    - foto_2
    - foto_3
 - precio 
 - costo
_una fotografia es oblicatoria, el resto puede ir por defecto en nullable_

### Comportamiento clave del stock 

| Acción                   | ¿Se descuenta stock?| Estado resultante |
| Usuario solicita apartado| No               | pending |
| Admin aprueba | Sí (-1)                     | approved |
| Admin rechaza | No                          | rejected |
| Expira por tiempo | No                      | expired |
| Se marca como pagado | Ya estaba descontado | paid |
| Usuario cancela | No (si estaba pending)    | cancelled |
  

# Actúa como un desarrollador frontend senior. A partir de ahora, utiliza estrictamente la siguiente paleta de colores y sistema de diseño para todos los componentes, estilos y vistas que creemos juntos.

  ## Rol diseño de fron-end 
  para el front se utilizara react con typescript, 
  teniendo los siguientes colores como paleta.
#### paleta de colores
    Primary color
    DFF69D
    RGB: 223–246–157
    RGB [%]: 87.5–96.5–61.6


    C9E86E
    RGB: 201–232–110
    RGB [%]: 78.8–91–43.1

    A6C843
    RGB: 166–200–67
    RGB [%]: 65.1–78.4–26.3

    89AC24
    RGB: 137–172–36
    RGB [%]: 53.7–67.5–14.1

    68880C
    RGB: 104–136–12
    RGB [%]: 40.8–53.3–4.7

    Secondary color #1
    A9E592
    RGB: 169–229–146
    RGB [%]: 66.3–89.8–57.3

    80D163
    RGB: 128–209–99
    RGB [%]: 50.2–82–38.8

    5DB43D
    RGB: 93–180–61
    RGB [%]: 36.5–70.6–23.9

    429A21
    RGB: 66–154–33
    RGB [%]: 25.9–60.4–12.9

    297A0B
    RGB: 41–122–11
    RGB [%]: 16.1–47.8–4.3

    Secondary color #2
    FFFBA3
    RGB: 255–251–163
    RGB [%]: 100–98.4–63.9

    F5EF73
    RGB: 245–239–115
    RGB [%]: 96.1–93.7–45.1

    D3CD47
    RGB: 211–205–71
    RGB [%]: 82.7–80.4–27.8

    B5AE26
    RGB: 181–174–38
    RGB [%]: 71–68.2–14.9

    8F890D
    RGB: 143–137–13
    RGB [%]: 56.1–53.7–5.1

### Definicion de rutas Front-end
- Tipografía: Inter o Geist.
    - `/home`
        -> Card_article
        -> Click_at_card
        - Página principal hace `fetch` a `GET /api/catalog` 
    - `/login`
    - `/admin/login`
    - `/admin/home`
        -> Selecion_admin_home
        - Página principal hace `fetch` a `GET /api/catalog`
    - `/admin/article/edit`
       - Formulario  hace  `UPDATE /api/catalog/:id`
    - `/admin/article/create`
        -  Formulario  hace `POST /api/catalog/create` 
    - Lista de Reservaciones (/admin/reservations)
        - hace  `fetch /api/reservations`
        - Columnas: Código, Producto, Usuario (foto + nombre), Fecha, Expira en, Estado, - Acciones
        - Filtros: Todos / Pendientes / Aprobados / Expirados / Pagados
        - Orden: más recientes primero
        - Botones rápidos: Aprobar / Rechazar / Ver detalle
    - Configuración (/admin/settings)
        - Campo: “Horas para expirar una reserva pendiente”
        - Campo: “Porcentaje por defecto a pagar al apartar

#### Card_article. 
    1. Card 
        a. Foto
        b. Name
        c. Talla
        d Price (tachado si tiene promo + precio con descuento)
        e. en stock (disponible / agotado)

    2. boton de compartir 
        a. copia el link para pedir en el whasapp
        b. con referencia al articulo

#### Click_at_card
    Cuando se le de click, 
    * se podra observar como details de este articulo,
    * foto, precio, nombre, stock

## Selecion_admin_home
    * Lista de articulos
        - nombre
        - preview photo 
        - precio
        - stock
        - acciones
    * Crear articulo
    * Editar articulo
    * desactivar articulo

### Crear articulo
    Los campos oblicatorios seran:
    - nombre
    - precio 
    - foto
### Editar articulo
    Se podra editar todos los campos

### Suspendier o desactivar articulo
    - tendremos dos tipos
    - el primero sera que no se va a mostrar el articulo ( accion directa en la lista de articulos)
    - el segundo sera que se mostrar pero indica que no hay disponible, stock = 0

## Consideraciones adicionales
* Validación de inputs con Zod (backend y frontend).
* Manejo de errores centralizado.
* Shared package (packages/shared) con tipos TypeScript.
* Configuración de Turborepo + pnpm workspaces correcta.

###

## 4. Docker (CRÍTICO — multi-stage)
Docker:
- docker-compose.yml con 3 servicios: postgres, api, web
- Multi-stage builds para api y web
- Healthchecks
- Volúmenes para postgres data y uploads
- Variables de entorno en .env.example
- Un solo comando: docker compose up --build
- Seed automático opcional (admin por defecto + algunos productos)
