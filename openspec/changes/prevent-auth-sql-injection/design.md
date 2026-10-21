# Design

## Context

See `proposal.md` for motivation and `specs/user-authentication/spec.md` for the required observable behavior.

El flujo actual recibe y valida la solicitud en `AuthResource`, delega autenticación a `AuthService` y obtiene el usuario mediante `LegacyDatabase.findByCredentials`. La consulta de búsqueda actual ya usa un parámetro enlazado para username; la contraseña se verifica después de recuperar el registro. `AuthResourceTest` contiene casos de request inválido, credenciales inválidas, login válido y `rejectsSqlInjectionAttempt`, aunque este último solo comprueba el estado HTTP.

El cambio debe mantenerse en el límite de búsqueda de credenciales y preservar el contrato de `POST /auth/login`. No requiere cambios de formato de datos ni migración de base de datos.

## Goals / Non-Goals

**Goals:**
- Mantener los datos de username y password como valores de credenciales, nunca como estructura ejecutable de una consulta.
- Hacer observable mediante pruebas que un intento de inyección no autentica ni devuelve datos de una sesión válida.
- Preservar los resultados de login válido, credenciales inválidas y validación de request definidos en la spec.
- Limitar el cambio a la búsqueda de credenciales y su verificación de contrato.

**Non-Goals:**
- Rediseñar las capas de autenticación, cambiar endpoints o modificar DTOs.
- Alterar tokens, contraseñas almacenadas, CORS, dependencias, esquema de datos o proceso de seed.
- Cambiar respuestas HTTP como parte de la mitigación SQL.
- Resolver la discrepancia preexistente entre el error `500` documentado y el error que devuelve actualmente el recurso.

## Contratos protegidos

- `POST /auth/login` acepta JSON y no requiere autenticación previa.
- Credenciales válidas conservan `200 OK` y los campos `token`, `expiresIn`, `user` y `debug`; `user` conserva `username`, `displayName` y `role`. El token se trata como valor opaco en estas pruebas.
- Request nulo o username/password ausentes o en blanco conserva `400 Bad Request` y los mensajes documentados.
- Credenciales inválidas, incluida una entrada que intente alterar la búsqueda, conserva `401 Unauthorized` y no devuelve datos de usuario autenticado.
- Un error inesperado no cambia debido a este trabajo. `docs/ENDPOINTS.md` documenta `message` y `exception` para `500`, mientras el código y una prueba actuales verifican un mensaje genérico sin `exception`; esa diferencia requiere una decisión separada y no se debe encubrir en esta implementación.

## Componentes afectados

- `LegacyDatabase.findByCredentials`: límite que realiza la búsqueda de identidad y verifica la contraseña; es el foco técnico de la protección.
- `AuthService.login`: delega la autenticación y mantiene la conversión de usuario encontrado/no encontrado a respuesta de servicio.
- `AuthResource.login`: contrato HTTP que debe permanecer estable; no requiere cambios de alcance.
- `AuthResourceTest`: pruebas de caracterización del contrato HTTP y del rechazo de intentos de inyección.

La inicialización de datos en `LegacyDatabase` no forma parte del alcance: no procesa valores recibidos del request de login.

## Decisions

### Mantener la protección en la búsqueda de credenciales

La consulta de identidad debe continuar separando el texto de consulta de los valores suministrados por el usuario. La revisión futura debe modificar solo esta frontera si la prueba de caracterización demuestra una brecha; el código actual ya usa un parámetro enlazado.

Alternativas consideradas:
- Escapar o filtrar manualmente caracteres de entrada: descartado porque es frágil y deja a la aplicación responsable de interpretar la sintaxis SQL.
- Reescribir la autenticación o moverla a otra capa: descartado porque el Issue #12 no solicita un cambio arquitectónico y el flujo actual ya separa HTTP, negocio y persistencia.

### Tratar la respuesta del token como opaca

Las pruebas de este cambio verificarán que el login válido conserva los campos documentados y no usarán valores concretos de tokens o secretos. Alterar el formato o firma del token queda fuera de alcance.

Alternativas consideradas:
- Afirmar el contenido literal del token: descartado porque acoplaría la prueba a un formato ajeno al Issue y podría exponer material sensible.

### No modificar errores HTTP ajenos al Issue

El manejo de `400`, `401` y `500` se mantiene fuera de la mitigación, excepto verificar que los casos cubiertos por la spec siguen con su contrato declarado. La diferencia de `500` entre documentación y código debe quedar visible y tratarse en un cambio separado.

Alternativas consideradas:
- Cambiar el error `500` para alinearlo durante este trabajo: descartado porque ampliaría el alcance comprometido en el Issue #12.
- Ajustar silenciosamente la spec al comportamiento actual: descartado porque `docs/ENDPOINTS.md` fue definido como baseline y el Issue no autoriza cambiarlo.

## Estrategia incremental

1. Ejecutar las pruebas de autenticación existentes antes de cualquier modificación y registrar su resultado como baseline.
2. Caracterizar el intento de inyección en el endpoint: debe responder `401`, no incluir `token` ni `user`, y conservar la respuesta de credenciales inválidas.
3. Completar la caracterización de login válido para el estado `200` y los campos documentados; conservar los casos existentes de `400` y `401`.
4. Si una modificación posterior resulta necesaria, limitarla a la búsqueda de credenciales y volver a ejecutar primero las pruebas de autenticación, y después la suite del proyecto.

## Pruebas de caracterización

Antes de modificar el comportamiento crítico, confirmar:
- `rejectsSqlInjectionAttempt`: fortalecer la observación para comprobar `401` y ausencia de campos de sesión/usuario en la respuesta, sin fijar un token ni credenciales reales.
- `returns401WhenCredentialsAreInvalid`: mantener la respuesta de credenciales inválidas para datos ordinarios no válidos.
- `logsInWithWorkshopUser`: verificar `200` y la presencia de todos los campos de respuesta definidos por el contrato, tratando el token como opaco.
- Los casos de request nulo, username vacío/en blanco y password vacío/en blanco siguen devolviendo `400` con sus mensajes documentados.
- Ejecutar el conjunto completo de pruebas de autenticación para detectar regresiones en el contrato HTTP.

La prueba existente de error `500` caracteriza el comportamiento actual, que difiere del contrato documentado. No se debe modificar esa prueba ni el error en el cambio de inyección SQL sin una decisión de alcance independiente.

## Risks / Trade-offs

- [La prueba de inyección solo cubre el username y una forma de entrada] -> Afirmar el resultado observable de no autenticación y no exposición de sesión; mantener la prueba acotada a entradas representativas sin depender de detalles SQL internos.
- [El query actual ya está parametrizado, por lo que el Issue podría no requerir cambio funcional adicional] -> Comparar la implementación con la prueba de caracterización antes de proponer una modificación; no introducir refactor si el comportamiento comprometido ya está satisfecho.
- [El contrato 500 documentado contradice el código y una prueba existente] -> Mantenerlo como discrepancia explícita y excluirlo de los cambios de este Issue; resolverlo por separado.
- [Cambiar respuestas de error o el token incidentalmente podría romper clientes] -> Verificar los contratos protegidos mediante pruebas HTTP de login válido, inválido y request inválido.

## Rollback

No se requiere migración ni rollback de datos. Si una futura modificación del lookup causa regresiones, revertir únicamente esa modificación y volver a desplegar la última versión aprobada; conservar las pruebas de caracterización para impedir que la regresión pase inadvertida. El rollback reintroduciría el riesgo de seguridad que motivó el Issue, por lo que debe ser temporal y rastreado como excepción operativa.
