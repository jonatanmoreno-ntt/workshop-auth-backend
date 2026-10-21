# Proposal

## Why

El Issue [#12](https://github.com/jonatanmoreno-ntt/workshop-auth-backend/issues/12) compromete blindar el login contra inyección SQL y verificar el rechazo de intentos maliciosos sin regresiones en autenticación válida. Es importante porque los datos de login no deben modificar la búsqueda de credenciales ni permitir una autenticación no autorizada.

## What Changes

### Dentro del alcance
- Definir el comportamiento verificable que impide que los datos recibidos para login alteren la consulta lógica de credenciales.
- Exigir el rechazo de intentos de inyección SQL y conservar la autenticación válida.
- Formalizar escenarios de prueba para el rechazo y para la compatibilidad del contrato de login.

### Fuera del alcance
- Cambiar el formato de request o las respuestas HTTP documentadas.
- Cambiar emisión o contenido de tokens, expiración, CORS, endpoint de salud, almacenamiento de contraseñas o dependencias.
- Definir clases, bibliotecas, consultas, ni pasos técnicos de implementación.

### Comportamiento que debe preservarse
- `POST /auth/login` continúa sin requerir autenticación previa y recibe JSON.
- Login válido conserva `200 OK` y los campos de respuesta documentados: `token`, `expiresIn`, `user` y `debug`.
- Request nulo o credenciales ausentes/en blanco conserva `400`; credenciales inválidas conserva `401`.
- La respuesta de error inesperado conserva el contrato documentado en `docs/ENDPOINTS.md`; el Issue #12 no solicita cambiar errores HTTP.

La documentación de `docs/ENDPOINTS.md` describe un campo `exception` para `500`, mientras que el comportamiento actual de `AuthResource` devuelve un mensaje genérico. Esta discrepancia es previa y queda registrada, no resuelta ni ampliada por este Issue.

## Capabilities

### New Capabilities
- `user-authentication`: comportamiento observable del login, rechazo de intentos de inyección SQL y preservación del contrato HTTP existente.

### Modified Capabilities
- Ninguna. El repositorio no tiene specs base existentes.

## Impact

- Fuente de alcance: GitHub Issue [#12](https://github.com/jonatanmoreno-ntt/workshop-auth-backend/issues/12), derivado del hallazgo H-01.
- Baseline: `docs/ENDPOINTS.md`, en particular `POST /auth/login`.
- Áreas relevantes para una futura implementación y verificación: autenticación, consulta de credenciales y pruebas del endpoint. No se prescribe el diseño técnico.