# Gobernanza y Guía de Agentes para OpenCode

## 1. Perfil del Proyecto
- **Propósito:** Desarrollo de aplicaciones Web de alto desempeño.
- **Backend:** .NET 9 (C#), ASP.NET Core Web API, Entity Framework Core / Dapper, SQL Server.
- **Frontend:** React con TypeScript, Vite, componentes modulares.
- **Entorno de IA:** OpenCode con modelos gratuitos basados en la nube (Gemini Flash, Groq, MiMo).

## 2. Convenciones y Arquitectura
- **C# / Backend:**
  - Seguir principios SOLID y Clean Code.
  - Usar Inyección de Dependencias y controladores o Minimal APIs estructurados.
  - Manejo de excepciones centralizado.
- **React / Frontend:**
  - Componentes funcionales bien tipados con TypeScript.
  - Separación clara entre lógica de UI y llamadas a API (hooks personalizados).
  - Manejo de estado limpio.

## 3. Reglas de Ejecución del Agente
- **SIEMPRE** consultar el archivo `CURSO.md` para entender la fase actual del desarrollo.
- **NO** ejecutar scripts de borrado en SQL Server (`DROP DATABASE`, `TRUNCATE`).
- **NO** almacenar llaves de API, credenciales o secretos en el código fuente.
- Antes de proponer cambios extensos en el código, documentar la propuesta en un archivo de especificación breve (`proposal.md`).