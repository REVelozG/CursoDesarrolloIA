# Tareas: Módulo de Rifas y Boletos

> Basado en `specs/spec-rifas.md` y `specs/design-rifas.md`. Estándar OpenSpec — Fase de implementación.
> Regla: tareas pequeñas, secuenciales y verificables. Completar en orden.
> Incluye correcciones del Revisor Adversarial (C1-C3, A1, A6-A9).

## Fase A — Base de Datos
1. Crear script `001_rifas_ddl.sql` con las tablas `Rifas` y `Boletos` (DDL de `design-rifas.md`).
2. Agregar constraints: `CHECK` de estados/total, `UNIQUE (RifaId, Numero)` y FK `Boletos → Rifas`.
3. Crear índices `IX_Boletos_RifaId_Estado` e `IX_Boletos_Expiracion` (filtrado).
4. Habilitar `READ_COMMITTED_SNAPSHOT` en la base (ejecutar sin transacciones abiertas — S8).
5. Ejecutar el script en SQL Server local y verificar con `sys.indexes`/`INFORMATION_SCHEMA` que todo existe.

## Fase B — Backend: scaffolding
6. Crear solución `backend/Rifas.sln` con los proyectos `Rifas.Api`, `Rifas.Application`, `Rifas.Infrastructure` (estructura de `design-rifas.md` §3.6).
7. **(A6)** Inicializar `dotnet user-secrets` en `Rifas.Api` y guardar la cadena de conexión ahí; `appsettings.json` solo con versión sin credenciales. Agregar `appsettings.*.json` al `.gitignore` (RN-09).
8. **(A7)** Configurar política de CORS con orígenes desde configuración (`localhost:5173` en dev, sin `AllowAnyOrigin`) y `app.UseCors` antes de los endpoints (`design-rifas.md` §3.7).
9. Registrar servicios e interfaces en DI en `Program.cs`.
10. **(C1)** Agregar middleware global de excepciones: mensaje genérico + log para 500, `ex.Message` solo para 400/404/409 (`design-rifas.md` §3.5).

## Fase C — Backend: casos de uso
11. Definir DTOs y uniones: `EstadoBoleto`, `CrearRifaRequest`, `ReservarRequest`, `PagarRequest`, `ErrorApi`, `ReservaResultado` (Ok/NoEncontrado/RifaCerrada/NoDisponible).
12. Implementar `POST /api/rifas`: validar N ∈ [1..10000], nombre y precio (RF-001/002/400).
13. Implementar inserción atómica de N boletos numerados en una transacción (RF-003).
14. **(A8)** Implementar `GET /api/rifas/{id}/boletos` con SQL de proyección sin PII + `ORDER BY Numero`, expiración perezosa previa (RF-031) y ETag/304.
15. **(C2/A1)** Implementar `ReservarAsync` con `UPDATE ... WHERE Estado='Disponible'` + `OUTPUT INSERTED.Id` (sin `@@ROWCOUNT`, sin check previo de rifa).
16. **(C2)** Implementar `DiagnosticarFalloAsync` (solo cuando el UPDATE falla): distinguir rifa/boleto inexistente → 404, rifa cerrada → 409 `RIFA_CERRADA`, tomado → 409 `BOLETO_NO_DISPONIBLE` (CA-11).
17. **(C3)** `201 Created` con `Location: /api/rifas/{rifaId}/boletos` (endpoint real; no existe `GET /api/boletos/{id}`).
18. Validar nombre/email y longitudes ≤ columnas (150/200) en la capa API → 400 (RN-07, A11).
19. Implementar `POST /api/boletos/pagar` con `WHERE Estado='Reservado' AND FechaExpiracionReserva > @ahora` (RF-020).
20. **(C2)** Diagnóstico en pago con 0 filas afectadas: liberar si TTL vencido (`RESERVA_EXPIRADA`), `RESERVA_REQUERIDA` si `Disponible`, `BOLETO_NO_DISPONIBLE` si ya `Pagado` (RF-021/022/023).
21. Capturar `SqlException` 1205/8152 en repositorios → traducir a excepción de negocio + reintento con backoff ante deadlock (A10).
22. Implementar `BackgroundService` que libera reservas vencidas cada 60 s (RF-031).

## Fase D — Backend: pruebas
23. Prueba unitaria: creación con N inválido → 400.
24. Prueba unitaria: máquina de estados — todas las transiciones prohibidas → 409 (RN-01/02).
25. Prueba de integración: reserva exitosa cambia a `Reservado` con TTL = +15 min y `Location` correcto (CA-1, C3).
26. Prueba de concurrencia: 10 peticiones paralelas al mismo boleto → exactamente 1×201 y 9×409 (CA-3).
27. Prueba de integración: boleto/rifa inexistente → **404** (no 409) — regresión de C2 (CA-11).
28. Prueba de integración: rifa cerrada → 409 `RIFA_CERRADA` (RF-014).
29. Prueba de integración: pago con reserva vencida → 409 `RESERVA_EXPIRADA` y boleto liberado (CA-6).
30. Prueba de idempotencia: doble submit del mismo comprador no duplica reserva (CA-4).
31. Prueba del middleware: excepción interna → 500 con mensaje genérico, sin texto de `SqlException` (C1).
32. Prueba de CORS: `Origin: http://localhost:5173` → headers `Access-Control-Allow-*` presentes (A7).
33. Prueba de rendimiento: rifa con 10.000 boletos, `GET` < 300 ms (CA-10, A8).
34. Ejecutar suite completa (`dotnet test`) y dejarla en verde.

## Fase E — Frontend: scaffolding
35. Crear proyecto Vite `frontend/` con React + TypeScript estricto.
36. Definir tipos `rifa.ts` (`EstadoBoleto`, `Boleto`, `Rifa`, `ErrorApi`).
37. Implementar `client.ts` con manejo de 400/404/409 → `ErrorApi` y soporte de `AbortSignal` (A9).

## Fase F — Frontend: UI
38. **(A9)** Implementar `TickProvider` + `useTick`: un solo `setInterval(1000)` global en `App.tsx`.
39. Implementar `useRifas` hook: carga con estados `idle/loading/error/ready`, `AbortController` en unmount y refetch superpuesto (A9).
40. Implementar `RifaGrid` + `BoletoCell` (`React.memo`, `key={boleto.numero}`) con colores por estado (A9).
41. Agregar virtualización/paginación si la rifa supera 1.000 boletos (CA-10).
42. Implementar `ReservaModal` con formulario nombre/email, validación cliente y botón deshabilitado durante submit (A9, RF-015).
43. Conectar modal a `POST /api/boletos/reservar`; éxito → actualización optimista + `Countdown` 15 min.
44. Manejar 409 en el cliente: `ErrorToast` + `refetch` del boleto en la grilla (caso borde §6 de la propuesta).
45. Implementar `Countdown` consumiendo `useTick()` (sin `setInterval` propio) + `SkeletonGrid` de carga (A9).
46. Ejecutar `npm run build` y `tsc --noEmit` sin errores.

## Fase G — Verificación final
47. Revisar los 13 criterios Given/When/Then de `proposal-rifas.md` uno por uno contra la implementación.
48. Revisar que se corrigieron C1, C2, C3, A1, A6, A7, A8, A9 del reporte adversarial.
49. Verificar `AGENTS.md`: sin secretos en el código (user-secrets/env), sin `DROP/TRUNCATE` en scripts.
50. Correr lint/format (`dotnet format`, `npm run lint`) y dejar limpio.
51. Actualizar `CURSO.md` con el avance de la sesión.
