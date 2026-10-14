# Especificación: Módulo de Rifas y Boletos

> Basado en `specs/proposal-rifas.md`. Estándar OpenSpec — Fase de requisitos.
> Stack: .NET 9 Web API + SQL Server + React/TypeScript.

## 1. Requisitos Funcionales

### 1.1 Gestión de Rifas
| ID | Requisito | Prioridad |
|----|-----------|-----------|
| RF-001 | El sistema permitirá crear una rifa con nombre, precio y N boletos numerados del 1 al N. | MVP |
| RF-002 | N debe ser un entero entre 1 y 10.000; fuera de rango → 400. | MVP |
| RF-003 | La generación de los N boletos será atómica: éxito total o rollback completo. | MVP |
| RF-004 | Tras la creación, el número y total de boletos son inmutables. | MVP |
| RF-005 | El sistema permitirá listar todos los boletos de una rifa con su estado. | MVP |
| RF-006 | Consultar una rifa inexistente responderá 404. | MVP |

### 1.2 Reserva de Boletos
| ID | Requisito | Prioridad |
|----|-----------|-----------|
| RF-010 | El sistema permitirá reservar un boleto `Disponible` asociándolo a un comprador (nombre + email). | MVP |
| RF-011 | Dos usuarios no podrán reservar el mismo boleto simultáneamente: exactamente una petición ganará. | MVP |
| RF-012 | La reserva generará `FechaExpiracionReserva = UTC ahora + 15 minutos` (TTL). | MVP |
| RF-013 | Reservar un boleto `Reservado` o `Pagado` responderá 409 con `BOLETO_NO_DISPONIBLE`. | MVP |
| RF-014 | Reservar en una rifa `Cerrada` o vencida responderá 409 con `RIFA_CERRADA`. | MVP |
| RF-015 | El doble submit del mismo comprador sobre el mismo boleto no creará doble reserva. | MVP |

### 1.3 Pago de Boletos
| ID | Requisito | Prioridad |
|----|-----------|-----------|
| RF-020 | El sistema permitirá marcar un boleto `Reservado` con TTL vigente como `Pagado`. | MVP |
| RF-021 | Pagar con reserva vencida responderá 409 con `RESERVA_EXPIRADA` y liberará el boleto a `Disponible`. | MVP |
| RF-022 | Pagar un boleto `Disponible` responderá 409 con `RESERVA_REQUERIDA` (transición prohibida). | MVP |
| RF-023 | Un boleto `Pagado` no podrá volver a `Reservado` ni a `Disponible`. | MVP |

### 1.4 Expiración (TTL)
| ID | Requisito | Prioridad |
|----|-----------|-----------|
| RF-030 | Las reservas no pagadas vencerán a los 15 minutos y el boleto volverá a `Disponible`. | MVP |
| RF-031 | La expiración se aplicará de forma perezosa al consultar el listado y/o por job periódico. | MVP |
| RF-032 | El pago concurrente con la expiración se resolverá con `UPDATE` condicional: solo una transición gana. | MVP |

## 2. Reglas de Negocio

| ID | Regla |
|----|-------|
| RN-01 | Máquina de estados: `Disponible → Reservado → Pagado`; `Reservado → Disponible` solo por expiración TTL. |
| RN-02 | Transiciones prohibidas: `Pagado → Reservado`, `Pagado → Disponible`, `Disponible → Pagado`. |
| RN-03 | El único mecanismo de concurrencia es el `UPDATE` condicional con verificación de filas afectadas vía `OUTPUT INSERTED.Id` (0 filas → null; nunca leer-then-escribir). |
| RN-04 | En reservas múltiples, los boletos se procesan en orden ascendente de `Id`; cualquier fallo revierte la transacción completa. |
| RN-05 | Constraint `UNIQUE (IdRifa, Numero)` es la última línea de defensa contra duplicados. |
| RN-06 | Los datos de comprador (PII) NO se exponen en `GET /api/rifas/{id}/boletos`; solo estado y número. |
| RN-07 | Todo error responde JSON `{ "codigo", "mensaje", "detalle" }` con el código HTTP correspondiente. |
| RN-08 | Nunca ejecutar `DROP DATABASE` ni `TRUNCATE` (governance `AGENTS.md`). |
| RN-09 | Sin secretos ni llaves de API en el código fuente (governance `AGENTS.md`). |

## 3. Endpoints JSON

### 3.1 `POST /api/rifas` — Crear rifa

**Request**
```json
{
  "nombre": "Rifa Solidaria 2026",
  "precioBoleto": 5.00,
  "totalBoletos": 100,
  "fechaCierreUtc": "2026-11-30T23:59:59Z"
}
```

**Response `201 Created`**
```json
{
  "id": 1,
  "nombre": "Rifa Solidaria 2026",
  "totalBoletos": 100,
  "estado": "Abierta",
  "fechaCierreUtc": "2026-11-30T23:59:59Z"
}
```

**Errores:** `400` (validación: N fuera de 1..10000, nombre vacío, precio negativo).

---

### 3.2 `GET /api/rifas/{id}/boletos` — Listar boletos

**Response `200 OK`**
```json
{
  "rifaId": 1,
  "nombre": "Rifa Solidaria 2026",
  "estado": "Abierta",
  "boletos": [
    { "numero": 1,  "estado": "Disponible" },
    { "numero": 2,  "estado": "Reservado", "expiraEnUtc": "2026-10-14T18:45:00Z" },
    { "numero": 3,  "estado": "Pagado" }
  ]
}
```

> Aplica expiración perezosa antes de responder (RF-031). No incluye datos del comprador (RN-06).

**Errores:** `404` (rifa inexistente).

---

### 3.3 `POST /api/boletos/reservar` — Reservar boleto

**Request**
```json
{
  "rifaId": 1,
  "numero": 2,
  "compradorNombre": "Ana Pérez",
  "compradorEmail": "ana@ejemplo.com"
}
```

**Response `201 Created`**
```json
{
  "boletoId": 42,
  "numero": 2,
  "estado": "Reservado",
  "expiraEnUtc": "2026-10-14T18:45:00Z"
}
```

**Errores:**
```json
{ "codigo": "BOLETO_NO_DISPONIBLE", "mensaje": "El boleto ya no está disponible", "detalle": "numero=2" }
```
- `409` `BOLETO_NO_DISPONIBLE` (RF-013)
- `409` `RIFA_CERRADA` (RF-014)
- `404` boleto/rifa inexistente
- `400` datos de comprador inválidos (nombre vacío, email con formato inválido)

---

### 3.4 `POST /api/boletos/pagar` — Pagar boleto

**Request**
```json
{ "rifaId": 1, "numero": 2 }
```

**Response `200 OK`**
```json
{ "boletoId": 42, "numero": 2, "estado": "Pagado" }
```

**Errores:**
- `409` `RESERVA_EXPIRADA` (RF-021) — y libera el boleto
- `409` `RESERVA_REQUERIDA` (RF-022)
- `409` `BOLETO_NO_DISPONIBLE` si ya está `Pagado`
- `404` inexistente

---

### 3.5 Formato de error uniforme (RN-07)

```json
{
  "codigo": "CODIGO_NEGOCIO",
  "mensaje": "Descripción legible para el usuario",
  "detalle": "Contexto técnico opcional"
}
```

| Código HTTP | Uso |
|---|---|
| 400 | Validación de input |
| 404 | Recurso inexistente |
| 409 | Conflicto de estado/concurrencia |
| 500 | Error inesperado (sin exponer stack trace) |

## 4. Criterios de Aceptación
Ver sección 5 de `specs/proposal-rifas.md` (13 criterios Given/When/Then). Esta especificación los hereda sin cambios.

## 5. Fuera de Alcance (MVP)
- Autenticación/autorización de usuarios.
- Pasarela de pago real (`pagar` marca estado, no procesa dinero).
- Notificaciones (email/SMS) de confirmación.
- Eliminación de rifas (solo restricción FK).
