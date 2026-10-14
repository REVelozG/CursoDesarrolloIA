# Reglas y Gobernanza de Agentes - Curso Desarrollo IA

## Contexto del Entorno
- **Tecnologías:** .NET 9 (C#), React (TypeScript), SQL Server.
- **Herramienta de Agentes:** OpenCode.
- **Uso de APIs:** Modelos gratuitos en la nube (Google AI Studio / Groq / OpenCode Zen).

## Reglas para los Agentes
1. **Límites de Ejecución:**
   - NO borrar bases de datos ni ejecutar scripts destructivos sin confirmación explícita.
   - NO subir credenciales, claves API o datos sensibles al control de versiones.
2. **Estilo de Código:**
   - C# debe seguir principios SOLID y arquitectura limpia.
   - React debe estructurarse en componentes funcionales con TypeScript estricto.
3. **Metodología:**
   - Respetar el flujo de especificaciones antes de escribir código masivo (`proposal.md` -> `spec.md` -> `tasks.md`).