## 1. Caracterizacion antes del cambio funcional

- [x] Ejecutar `mvn -Dtest=AuthResourceTest test` y confirmar el baseline de las pruebas de autenticacion antes de modificar el comportamiento.
- [x] Fortalecer `rejectsSqlInjectionAttempt` para comprobar `401`, el mensaje documentado de credenciales invalidas y la ausencia de `token` y `user` en la respuesta.
- [x] Completar `logsInWithWorkshopUser` para comprobar `200`, `token`, `expiresIn`, `user` y `debug`, incluidos `user.username`, `user.displayName` y `user.role`; tratar `token` como valor opaco.
- [x] Confirmar la caracterizacion de credenciales invalidas y de requests nulos, ausentes, vacios o en blanco; agregar solo las variantes que falten para cubrir `400` y sus mensajes documentados.

### Evidencia de caracterizacion

- Comando desde `workshop-auth-backend`: `.tools/maven/bin/mvn.cmd -Dtest=AuthResourceTest test`
- Baseline previo a los cambios de pruebas: 15 tests, 0 fallas, 0 errores; `BUILD SUCCESS`.
- Despues de completar la caracterizacion: 19 tests, 0 fallas, 0 errores; `BUILD SUCCESS`.

## 2. Implementacion incremental

- [x] Revisar `LegacyDatabase.findByCredentials` como unica frontera de busqueda: los valores de username y password deben seguir siendo datos, no estructura ejecutable de consulta.
- [x] Si la caracterizacion o la revision demuestra una brecha, aplicar la correccion minima en la busqueda de credenciales, sin filtros/escape manual, cambios arquitectonicos ni modificaciones a tokens, DTOs o respuestas HTTP. La consulta ya usa `PreparedStatement` con username enlazado y verifica la password despues del lookup; no se encontro brecha que requiera cambio funcional.
- [x] Mantener sin cambios el flujo de `AuthService` y el contrato HTTP de `AuthResource`; el intento de inyeccion debe continuar resultando en credenciales invalidas y `401`.

## 3. Validacion

- [x] Ejecutar `mvn -Dtest=AuthResourceTest test` despues de la implementacion y confirmar las pruebas de rechazo SQL, login valido, credenciales invalidas y validacion del request.
- [x] Ejecutar `mvn test` y confirmar que la suite del backend pasa sin regresiones.

### Evidencia de aplicacion y validacion

- Revision de `LegacyDatabase.findByCredentials`: SQL fijo con `WHERE username=?`, valor enlazado mediante `PreparedStatement.setString` y verificacion de password posterior a la consulta; no se modifico codigo funcional.
- Comando desde `workshop-auth-backend`: `.tools/maven/bin/mvn.cmd -Dtest=AuthResourceTest test` — 19 tests, 0 fallas, 0 errores; `BUILD SUCCESS`.
- Comando desde `workshop-auth-backend`: `.tools/maven/bin/mvn.cmd test` — 22 tests, 0 fallas, 0 errores; `BUILD SUCCESS`.

La discrepancia preexistente del contrato de error `500` queda fuera de este Issue: no cambiar su comportamiento ni las pruebas que lo caracterizan en este trabajo.
