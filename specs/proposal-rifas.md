# Propuesta: Módulo de Gestión de Rifas y Boletos

## 1. Problema / Objetivo
Permitir la creación de rifas digitales donde los usuarios puedan seleccionar boletos disponibles, registrarlos a su nombre y consultar el estado de su boleto (Disponible, Reservado, Pagado).

## 2. Alcance (MVP)
- **Backend (.NET 9):**
  - Endpoint `POST /api/rifas` para crear una nueva rifa con $N$ boletos.
  - Endpoint `GET /api/rifas/{id}/boletos` para listar boletos y su estado.
  - Endpoint `POST /api/boletos/reservar` con control de concurrencia (dos personas no pueden reservar el mismo boleto al mismo tiempo).
  - Endpoint `POST /api/boletos/pagar` para marcar un boleto `Reservado` como `Pagado`.
- **Frontend (React + TS):**
  - Grid/Cuadrícula interactiva para visualizar boletos numerados.
  - Formulario modal para ingresar datos de comprador y reservar.
  - Manejo visual de 409 Conflict: refrescar el boleto en la grilla.
- **Base de Datos (SQL Server):**
  - Tabla `Rifas` y Tabla `Boletos` con llave foránea.
  - Constraint `UNIQUE (IdRifa, Numero)` como defensa final ante inserciones concurrentes.
  - Columna `FechaExpiracionReserva` en `Boletos` para el TTL.

## 3. Máquina de Estados del Boleto

```
Disponible ──(reservar)──▶ Reservado ──(pagar)──▶ Pagado
    ▲                          │
    └──(expiración TTL 15 min)─┘
```

- **TTL de reserva:** 15 minutos. Una reserva no pagada vuelve a `Disponible` automáticamente (job/verificación al consultar).
- **Transiciones válidas:** `Disponible → Reservado`, `Reservado → Pagado`, `Reservado → Disponible` (solo por expiración).
- **Transiciones prohibidas:** `Pagado → Reservado`, `Pagado → Disponible`, `Disponible → Pagado` (debe pasar por reserva).

## 4. Estrategia de Concurrencia (SQL Server)

**Técnica elegida: `UPDATE` condicional con verificación de filas afectadas.**

```sql
UPDATE Boletos
SET Estado = 'Reservado', FechaExpiracionReserva = DATEADD(MINUTE, 15, GETUTCDATE()), ...
OUTPUT INSERTED.Id
WHERE Id = @id AND Estado = 'Disponible';
-- Si 0 filas (result set vacío → null) → diagnosticar causa → 409 Conflict o 404
```

- Cada transición de estado (`reservar`, `pagar`, `expiración`) usa la misma técnica: el `WHERE` incluye el estado esperado y se verifica la fila afectada vía `OUTPUT INSERTED.Id`.
- No se lee el estado antes de escribir (evita *lost update*); el diagnóstico de la causa (404/409) se hace solo cuando el `UPDATE` afectó 0 filas.
- Si se reservan boletos múltiples en una llamada: actualizar en orden de `Id` ascendente y abortar la transacción completa si algún boleto falla (rollback atómico).
- Aislamiento recomendado: `READ COMMITTED SNAPSHOT` para que el `GET` de la grilla no bloquee escrituras.
- Mapeo de errores: 0 filas porque el boleto no está `Disponible` → `409 Conflict` con cuerpo `{ "codigo": "BOLETO_NO_DISPONIBLE", "mensaje": "..." }`; rifa/boleto no existe → `404 Not Found`; rifa cerrada → `409 RIFA_CERRADA`.

## 5. Criterios de Aceptación (Given/When/Then)

### Reserva
1. **Reserva exitosa**
   - **Given** un boleto en estado `Disponible`
   - **When** un usuario envía `POST /api/boletos/reservar` con datos válidos
   - **Then** el boleto pasa a `Reservado`, `FechaExpiracionReserva` = ahora + 15 min y la API responde `201 Created`

2. **Conflicto por boleto ya tomado**
   - **Given** un boleto en estado `Reservado` o `Pagado`
   - **When** otro usuario intenta reservarlo
   - **Then** la API responde `409 Conflict` con JSON `{ "codigo": "BOLETO_NO_DISPONIBLE", ... }` y el estado no cambia

3. **Carrera concurrente**
   - **Given** 10 peticiones simultáneas para el mismo boleto `Disponible`
   - **When** se procesan en paralelo
   - **Then** exactamente 1 responde `201` y 9 responden `409`; nunca hay doble reserva

4. **Doble submit idempotente**
   - **Given** un usuario que envía la misma reserva dos veces (doble clic/doble pestaña)
   - **When** la segunda petición llega con los mismos datos del mismo comprador
   - **Then** no se crean dos reservas y el estado final sigue siendo `Reservado`

### Pago
5. **Pago de reserva vigente**
   - **Given** un boleto `Reservado` con TTL no vencido
   - **When** se envía `POST /api/boletos/pagar`
   - **Then** el boleto pasa a `Pagado` y responde `200 OK`

6. **Pago de reserva expirada**
   - **Given** un boleto `Reservado` con `FechaExpiracionReserva` vencida
   - **When** se envía `POST /api/boletos/pagar`
   - **Then** responde `409 Conflict` con `{ "codigo": "RESERVA_EXPIRADA", ... }` y el boleto vuelve a `Disponible`

### Expiración (TTL)
7. **Liberación automática**
   - **Given** un boleto `Reservado` hace más de 15 minutos sin pago
   - **When** se consulta `GET /api/rifas/{id}/boletos` o ejecuta el job de expiración
   - **Then** el boleto aparece/vuelve como `Disponible`

### Creación de rifa
8. **Boletos generados**
   - **Given** `POST /api/rifas` con `N` boletos
   - **When** la creación es exitosa
   - **Then** se insertan exactamente $N$ boletos numerados del 1 al $N$ de forma atómica o se revierte todo
9. **Validación de N**
   - **Given** `N < 1` o `N > 10000`
   - **When** se envía la petición
   - **Then** responde `400 Bad Request` con detalle de validación

### Consulta
10. **Listado y rendimiento**
    - **Given** una rifa con 10.000 boletos
    - **When** se llama `GET /api/rifas/{id}/boletos`
    - **Then** responde `200` con menos de 300 ms y el frontend renderiza por paginación/virtualización
11. **Rifa inexistente**
    - **Given** un `id` inexistente
    - **When** se consulta cualquier endpoint
    - **Then** responde `404 Not Found`

### Errores
12. **Formato de error uniforme**
    - **Given** cualquier error de validación o de negocio
    - **When** se produce una respuesta fallida
    - **Then** el cuerpo es JSON con `{ "codigo", "mensaje", "detalle" }` y el código HTTP apropiado (400/404/409)

13. **Integridad referencial**
    - **Given** una rifa con boletos asociados
    - **When** se intenta eliminar la rifa
    - **Then** la BD lo restringe (FK Restrict) o se archiva, nunca borra en cascada silenciosa

## 6. Casos Borde y su Manejo

| Caso borde | Manejo |
|---|---|
| Reservar en rifa `Cerrada` o tras fecha límite | `409 Conflict` con `{ "codigo": "RIFA_CERRADA" }` |
| Reserva que nunca se paga | TTL 15 min → vuelve a `Disponible` |
| Pago concurrente con expiración | El `UPDATE` condicional decide: solo uno gana |
| Usuario ve grilla desactualizada y reserva | Frontend maneja `409` refrescando el boleto en la grilla |
| Doble pestaña del mismo usuario | Mismo `409`/idempotencia del criterio 4 |
| Crear rifa con $N$ grande (10.000+) | Insert atómico en lote dentro de una transacción |
| Boleto duplicado por inserción concurrente | Constraint `UNIQUE (IdRifa, Numero)` devuelve `500`/mapeado a `409` |
| Datos de comprador vacíos o inválidos | Validación `[Required]`/email → `400` con detalle |
| Boleto `Pagado` intentado re-reservar | `409` (transición prohibida) |
| Eliminar/editar rifa con reservas activas | FK Restrict + negocio de archivado, sin borrado en cascada |

## 7. Seguridad y Privacidad
- Validación de todas las entradas en servidor (no confiar en el cliente).
- Datos de comprador son PII: no exponerlos en `GET /api/rifas/{id}/boletos` (retornar solo estado y número).
- Sin llaves de API ni secretos en el código fuente (regla de `AGENTS.md`).
- Consultas parametrizadas vía EF Core/Dapper (sin SQL concatenado).
