# Guardrails y Barreras de Contención del Proyecto

## 1. Comandos de Terminal Prohibidos
El agente tiene PROHIBIDO ejecutar o sugerir la ejecución de los siguientes comandos:
- `DROP DATABASE`, `TRUNCATE TABLE`, `ALTER TABLE ... DROP` (en SQL Server sin confirmación previa).
- `git push --force` o `git reset --hard` a ramas remotas.
- Eliminación recursiva de directorios fuera de `bin/` u `obj/`.

## 2. Protección de Secretos y Privacidad (PII)
- Queda estrictamente prohibido escribir contraseñas, tokens JWT, o ConnectionStrings en archivos `.json`, `.cs`, `.tsx` o `.md`.
- Toda cadena de conexión de base de datos DEBE usar `User Secrets` (`dotnet user-secrets`) o variables de entorno.

## 3. Gobernanza de Cambios en Código
- No se permite la generación masiva de código si no existe un archivo correspondiente en `specs/` con estado revisado.
- Todo endpoint nuevo de ASP.NET Core DEBE incluir manejo explícito de excepciones y log estructurado.