# Diseño Técnico: Módulo de Rifas y Boletos

> Basado en `specs/spec-rifas.md`. Estándar OpenSpec — Fase de diseño/architectura.

## 1. Arquitectura General

```
┌─────────────────────────┐        ┌──────────────────────────────┐
│  React + TS (Vite)      │  HTTP  │  .NET 9 Web API              │
│  ┌───────────────────┐  │◀──────▶│  Controllers/Minimal API     │
│  │ Grilla + Modal    │  │  JSON  │  Services (lógica negocio)   │
│  │ useRifas hook     │  │        │  Repos (Dapper/EF Core)      │
│  └───────────────────┘  │        └──────────────┬───────────────┘
└─────────────────────────┘                       │ UPDATE condicional
                                                  ▼
                                    ┌──────────────────────────────┐
                                    │  SQL Server                 │
                                    │  Rifas, Boletos (+ índices) │
                                    └──────────────────────────────┘
```

- **Patrón:** capas `API → Service → Repository` con DI.
- **Acceso a datos:** Dapper para escrituras condicionales (verificación de filas afectadas vía `OUTPUT`), EF Core opcional para consultas de listado.
- **Manejo de excepciones:** middleware global que mapea excepciones de negocio a `{ codigo, mensaje, detalle }`.

## 2. DDL — SQL Server

```sql
CREATE TABLE Rifas (
    Id              INT IDENTITY(1,1)  NOT NULL PRIMARY KEY,
    Nombre          NVARCHAR(200)      NOT NULL,
    PrecioBoleto    DECIMAL(10,2)      NOT NULL,
    TotalBoletos    INT                NOT NULL,
    Estado          NVARCHAR(20)       NOT NULL DEFAULT 'Abierta'
                    CHECK (Estado IN ('Abierta','Cerrada','Archivada')),
    FechaCierreUtc  DATETIME2          NOT NULL,
    FechaCreacionUtc DATETIME2         NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT CK_Rifas_TotalBoletos CHECK (TotalBoletos BETWEEN 1 AND 10000),
    CONSTRAINT CK_Rifas_Precio       CHECK (PrecioBoleto >= 0)
);

CREATE TABLE Boletos (
    Id                    INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    RifaId                INT               NOT NULL,
    Numero                INT               NOT NULL,
    Estado                NVARCHAR(20)      NOT NULL DEFAULT 'Disponible'
                          CHECK (Estado IN ('Disponible','Reservado','Pagado')),
    CompradorNombre       NVARCHAR(150)     NULL,
    CompradorEmail        NVARCHAR(200)     NULL,
    FechaReservaUtc       DATETIME2         NULL,
    FechaExpiracionReserva DATETIME2        NULL,
    FechaPagoUtc          DATETIME2         NULL,
    CONSTRAINT FK_Boletos_Rifas
        FOREIGN KEY (RifaId) REFERENCES Rifas(Id),
    CONSTRAINT UQ_Boletos_Rifa_Numero
        UNIQUE (RifaId, Numero)
);

-- Índices
CREATE INDEX IX_Boletos_RifaId_Estado
    ON Boletos (RifaId, Estado)
    INCLUDE (Numero, FechaExpiracionReserva);

CREATE INDEX IX_Boletos_Expiracion
    ON Boletos (FechaExpiracionReserva)
    WHERE Estado = 'Reservado';          -- apoya el job de TTL

ALTER DATABASE [TuBase] SET READ_COMMITTED_SNAPSHOT ON;
```

**Notas:**
- `UQ_Boletos_Rifa_Numero`: última línea de defensa ante duplicados concurrentes (RN-05).
- Índice filtrado de expiración: consulta barata del job de liberación (RF-031).
- `READ_COMMITTED_SNAPSHOT`: el `GET` de la grilla no bloquea escrituras.

## 3. Patrón C# — .NET 9 Web API

### 3.1 Transacción de reserva con `UPDATE` condicional (RN-03, C2)

`OUTPUT INSERTED.Id` devuelve el Id **y** las filas afectadas en una sola operación (0 filas → result set vacío → `null`), eliminando la frágil dependencia de `SELECT @@ROWCOUNT` (S1). Si falla, un diagnóstico posterior clasifica 404 vs 409 **sin** un check previo (elimina el TOCTOU A1).

```csharp
// Repository/BoletoRepository.cs
public async Task<ReservaResultado> ReservarAsync(
    int rifaId, int numero, string nombre, string email, CancellationToken ct)
{
    const string sql = """
        UPDATE b
        SET b.Estado = 'Reservado',
            b.CompradorNombre = @nombre,
            b.CompradorEmail  = @email,
            b.FechaReservaUtc = @ahora,
            b.FechaExpiracionReserva = DATEADD(MINUTE, 15, @ahora)
        OUTPUT INSERTED.Id
        FROM Boletos b
        INNER JOIN Rifas r ON r.Id = b.RifaId
        WHERE b.RifaId = @rifaId
          AND b.Numero = @numero
          AND b.Estado = 'Disponible'                  -- condición crítica
          AND r.Estado = 'Abierta'
          AND r.FechaCierreUtc > @ahora;               -- cubre rifa cerrada/vencida
        """;

    await using var conn = new SqlConnection(_cs);
    await conn.OpenAsync(ct);
    await using var tx = await conn.BeginTransactionAsync(IsolationLevel.ReadCommitted, ct);

    var boletoId = await conn.ExecuteScalarAsync<int?>(sql,
        new { rifaId, numero, nombre, email, ahora = DateTime.UtcNow },
        transaction: tx);                               // null = 0 filas afectadas

    if (boletoId is null)
    {
        var diag = await DiagnosticarFalloAsync(conn, tx, rifaId, numero, ct);
        await tx.RollbackAsync(ct);
        return diag;                                    // 404 / 409 / 409 según causa
    }

    await tx.CommitAsync(ct);
    return ReservaResultado.Ok(boletoId.Value);
}

// Diagnóstico best-effort SOLO cuando el UPDATE falló (no hay check-then-act previo)
private static async Task<ReservaResultado> DiagnosticarFalloAsync(
    SqlConnection conn, SqlTransaction tx, int rifaId, int numero, CancellationToken ct)
{
    var rifa = await conn.QueryFirstOrDefaultAsync<(int Estado, DateTime? Cierre)>(
        """SELECT CAST(CASE Estado WHEN 'Abierta' THEN 1 ELSE 0 END AS BIT), FechaCierreUtc
           FROM Rifas WHERE Id = @rifaId""",
        new { rifaId }, transaction: tx, cancellationToken: ct);

    if (rifa == default) return ReservaResultado.NoEncontrado;         // rifa inexistente → 404

    var boleto = await conn.QueryFirstOrDefaultAsync<string>(
        "SELECT Estado FROM Boletos WHERE RifaId = @rifaId AND Numero = @numero",
        new { rifaId, numero }, transaction: tx, cancellationToken: ct);

    if (boleto is null) return ReservaResultado.NoEncontrado;          // boleto inexistente → 404
    if (rifa.Estado == 0 || (rifa.Cierre <= DateTime.UtcNow))
        return ReservaResultado.RifaCerrada;                           // → 409 RIFA_CERRADA
    return ReservaResultado.NoDisponible;                              // → 409 BOLETO_NO_DISPONIBLE
}
```

### 3.2 Mapeo a HTTP en el Service/Controller (C2, C3, A1)

Sin `RifaAbiertaAsync` previo (TOCTOU): la causa la determina el diagnóstico post-fallo, con un solo round-trip.

```csharp
// Services/BoletoService.cs
public async Task<IResult> ReservarAsync(ReservarRequest req, CancellationToken ct)
{
    if (!Validador.EsValido(req))                       // capa API, no el repo (S4): 400
        return Error(400, "DATOS_INVALIDOS", "Nombre y email requeridos");

    var r = await _repo.ReservarAsync(req.RifaId, req.Numero,
                                      req.CompradorNombre, req.CompradorEmail, ct);

    return r switch
    {
        // C3: Location apunta al listado real (no existe GET /api/boletos/{id})
        ReservaResultado.Ok(var id)     => Results.Created(
                                               $"/api/rifas/{req.RifaId}/boletos",
                                               new BoletoDto(req.Numero, "Reservado", r.ExpiraEnUtc)),
        ReservaResultado.NoEncontrado   => Error(404, "NO_ENCONTRADO",
                                               "Rifa o boleto inexistente"),
        ReservaResultado.RifaCerrada    => Error(409, "RIFA_CERRADA",
                                               "La rifa está cerrada o vencida"),
        ReservaResultado.NoDisponible   => Error(409, "BOLETO_NO_DISPONIBLE",
                                               "El boleto ya no está disponible"),
        _                               => Error(500, "ERROR_INTERNO", null)
    };
}

// Mismo patrón para Pagar: WHERE Estado = 'Reservado' AND FechaExpiracionReserva > @ahora
// 0 filas → diagnóstico: RESERVA_EXPIRADA (liberar) / RESERVA_REQUERIDA / BOLETO_NO_DISPONIBLE (ya Pagado).
```

> `ReservaResultado` es un *discriminated union* (record + `static` factory) con los
> casos `Ok(int Id, DateTime? ExpiraEnUtc)`, `NoEncontrado`, `RifaCerrada`, `NoDisponible`.

### 3.3 Reserva múltiple atómica (RN-04)
- `ORDER BY Numero ASC` sobre los `UPDATE` dentro de **una sola** transacción (orden consistente evita deadlocks).
- Ante el primer `OUTPUT` que retorne `null` (0 filas) → `tx.Rollback()` y responder con el diagnóstico de la causa (404/409) del boleto conflictivo.

### 3.4 Expiración perezosa (RF-031)
```csharp
// Antes de listar: UPDATE condicional masivo
UPDATE Boletos SET Estado = 'Disponible',
       CompradorNombre = NULL, CompradorEmail = NULL,
       FechaReservaUtc = NULL, FechaExpiracionReserva = NULL
WHERE Estado = 'Reservado' AND FechaExpiracionReserva < @ahora;
```
+ `BackgroundService` (IHostedService) que ejecuta el mismo `UPDATE` cada 60 s.

### 3.5 Middleware de errores (RN-07, C1)

**Nunca exponer `ex.Message` en 500** — los mensajes de `SqlException` revelan nombres de objetos, usuario de login, servidor y timeouts. El detalle técnico solo va al log.

```csharp
// Program.cs
app.UseExceptionHandler(handler => handler.Run(async context =>
{
    var ex = context.Features.Get<IExceptionHandlerFeature>()?.Error;
    var (status, codigo) = ex switch {
        ValidacionException v => (400, v.Codigo),
        NoEncontradoException => (404, "NO_ENCONTRADO"),
        ConflictoEstadoException c => (409, c.Codigo),
        _ => (500, "ERROR_INTERNO")
    };

    if (status == 500)
    {
        context.RequestServices.GetRequiredService<ILogger<Program>>()
            .LogError(ex, "Error inesperado");         // detalle solo en log
        await context.Response.WriteAsJsonAsync(new {
            codigo, mensaje = "Error interno del servidor", detalle = (string?)null
        });
    }
    else
    {
        await context.Response.WriteAsJsonAsync(new {
            codigo, mensaje = ex.Message, detalle = (ex as DetallableException)?.Detalle
        });
    }
    context.Response.StatusCode = status;
}));
```

Además, el repositorio captura `SqlException.Number` (1205 = deadlock, 8152/8152 = truncamiento) y los traduce a excepciones de negocio (`ConflictoEstadoException` / `ValidacionException`) para que jamás lleguen crudas al middleware. Reintento con backoff ante 1205 (A10).

### 3.6 Estructura de proyectos
```
backend/
  src/Rifas.Api/            Program.cs, Middleware/, Endpoints/
  src/Rifas.Application/    Services/, Dtos/, Exceptions/
  src/Rifas.Infrastructure/ Repositories/, Dapper/, Scripts/ (DDL)
  tests/Rifas.UnitTests/
```

### 3.7 Configuración: secretos y CORS (A6, A7)

**Secretos (RN-09):** la cadena de conexión con credenciales **no va** en `appsettings.json` (ese archivo se commitea). Usar `dotnet user-secrets` en desarrollo y variables de entorno/Key Vault en producción:

```bash
dotnet user-secrets init --project src/Rifas.Api
dotnet user-secrets set "ConnectionStrings:RifasDb" "Server=...;Database=...;User Id=...;Password=..." \
    --project src/Rifas.Api
```
```csharp
// Program.cs — el proveedor de user-secrets/variables de entorno tiene prioridad sobre appsettings.json
builder.Configuration.AddEnvironmentVariables();
var cs = builder.Configuration.GetConnectionString("RifasDb")
    ?? throw new InvalidOperationException("Falta ConnectionStrings:RifasDb");
```
`appsettings.json` solo guarda la versión sin credenciales (`Server=localhost;Database=Rifas`). Agregar `appsettings.*.json` con secretos al `.gitignore`.

**CORS:** sin él, el frontend Vite (`localhost:5173`) no podrá llamar a la API:

```csharp
// Program.cs
const string CorsPolicy = "Frontend";
builder.Services.AddCors(o => o.AddPolicy(CorsPolicy, p => p
    .WithOrigins(builder.Configuration["Cors:FrontendOrigin"] ?? "http://localhost:5173")
    .AllowAnyHeader()
    .AllowAnyMethod()));

var app = builder.Build();
app.UseCors(CorsPolicy);          // antes de MapControllers/Map*
```
Orígenes permitidos por configuración (no `AllowAnyOrigin` en ningún entorno).

### 3.8 Consulta de listado (A8) — contrato SQL del `GET`

El `GET /api/rifas/{id}/boletos` se define aquí para poder validar el CA-10 (< 300 ms con 10.000 boletos):

```sql
-- Proyección mínima: sin PII (RN-06), cubierta por IX_Boletos_RifaId_Estado
SELECT b.Numero, b.Estado, b.FechaExpiracionReserva
FROM Boletos b
WHERE b.RifaId = @rifaId
ORDER BY b.Numero;
```

- Devuelve a lo sumo `TotalBoletos ≤ 10.000` filas pequeñas (~30 B) → medir el CA-10 con datos reales.
- `expiraEnUtc` solo se emite si `Estado = 'Reservado'`; los demás van `null`.
- `Content-Type: application/json` + ETag derivado de `MAX(FechaExpiracionReserva) + COUNT(*)` permite condicionales `If-None-Match` → `304` barato en refetch repetido.
- Si la medición excede 300 ms → paginación (`$skip/$take` sobre `Numero`) sin cambiar el contrato del MVP (la grilla ya pagina).

## 4. Estructura React + TypeScript

```
frontend/src/
├── api/
│   ├── client.ts            // fetch wrapper: maneja 400/404/409 → ErrorApi {codigo,mensaje}
│   └── rifasApi.ts          // crearRifa, getBoletos, reservar, pagar
├── types/
│   └── rifa.ts              // type EstadoBoleto = 'Disponible'|'Reservado'|'Pagado'
│                            // interface Boleto, Rifa, ErrorApi
├── hooks/
│   ├── useRifas.ts          // estado de grilla, refetch, optimistic handling de 409
│   └── useTick.ts           // A9: UN solo intervalo global de 1 s (contexto TickProvider)
├── components/
│   ├── RifaGrid/
│   │   ├── RifaGrid.tsx     // cuadrícula; virtualización si > 1.000 boletos
│   │   └── BoletoCell.tsx   // React.memo; badge de color + Countdown (sin timer propio)
│   ├── ReservaModal/
│   │   ├── ReservaModal.tsx // formulario nombre+email, validación, botón deshabilitado en submit
│   │   └── useReservaForm.ts
│   └── common/
│       ├── ErrorToast.tsx   // muestra {mensaje} de ErrorApi
│       ├── SkeletonGrid.tsx // A9: estado de carga del listado
│       └── Countdown.tsx    // TTL 15:00 restante; consume useTick(), NO crea setInterval
└── App.tsx                  // <TickProvider> envuelve la app
```

**Rendimiento y concurrencia de UI (A9):**
1. **Tick global único:** `TickProvider` mantiene **un** `setInterval(1000)` en la raíz; los `Countdown` leen el valor del contexto. Cientos de `setInterval` por celda provocan re-render storms.
2. **Re-renders mínimos:** `BoletoCell` envuelta en `React.memo` + `key={boleto.numero}` estable; `RifaGrid` solo se re-renderiza si cambia `boletos` (referencia), no el tick — el tick se pasa por contexto a las celdas que lo usan, no como prop.
3. **Peticiones cancelables:** `client.ts` acepta `AbortSignal`; `useRifas` aborta en unmount y en refetch superpuesto (evita que una respuesta vieja pise una optimista) y controla estados `idle/loading/error/ready`.
4. **Contención del 409:** manejar en el `catch` de `rifasApi.reservar`; refetch solo del boleto afectado (o de la grilla) y toast con `mensaje`.

**Flujos clave:**
1. `useRifas` carga `GET /{id}/boletos` (mostrar `SkeletonGrid` mientras); expiración perezosa del server reflejada en cada `refetch`.
2. Clic en boleto `Disponible` → `ReservaModal`; submit → `POST /api/boletos/reservar` con botón deshabilitado hasta terminar (evita doble submit).
3. **409 → `ErrorToast` + refetch del boleto** (caso borde de grilla desactualizada); no reintentar automáticamente.
4. Éxito → actualización optimista del boleto a `Reservado` + `Countdown` (tick global) de 15 min.
5. Botón "Pagar" (solo dueño del flujo MVP) → `POST /api/boletos/pagar`.

**Colores de estado:** `Disponible` gris/verde · `Reservado` ámbar · `Pagado` azul.

## 5. Decisiones Técnicas Resumen

| Tema | Decisión |
|---|---|
| Concurrencia | `UPDATE` condicional + `OUTPUT INSERTED.Id` (nunca leer-then-escribir; sin `@@ROWCOUNT` frágil) |
| Clasificación de errores | Diagnóstico post-fallo (404/409 correctos); sin check previo TOCTOU |
| Exposición de errores | 500 → mensaje genérico + log; `ex.Message` jamás al cliente |
| TTL reservas | 15 min, expiración perezosa + `BackgroundService` cada 60 s |
| Aislamiento | `READ COMMITTED SNAPSHOT` |
| Última línea de defensa | `UNIQUE (IdRifa, Numero)` |
| Secretos | `dotnet user-secrets` / env vars; `appsettings.json` sin credenciales |
| CORS | Política con orígenes configurados (`localhost:5173` en dev), nunca `AllowAnyOrigin` |
| ORM | Dapper para updates condicionales; consultas simples con Dapper/EF |
| Listado | Proyección sin PII, `ORDER BY Numero`, ETag/304, paginación si CA-10 falla |
| PII | `Comprador*` no sale en `GET` de grilla |
| Errores | Middleware global → `{codigo, mensaje, detalle}` |
| React perf | 1 tick global, `React.memo`, `AbortSignal`, estados de carga explícitos |
