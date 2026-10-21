# Spec Delta

## Purpose

Define el comportamiento observable del login para rechazar intentos de inyección SQL y conservar el contrato HTTP existente documentado para autenticación.

## ADDED Requirements

### Requirement: Los datos del login no alteran la autenticación
El sistema SHALL tratar los valores recibidos para username y password únicamente como datos de credenciales; valores sin credenciales válidas no SHALL permitir acceso ni cambiar la identidad autenticada.

#### Scenario: Un intento de inyección SQL no autentica
- **WHEN** se envían valores de login con sintaxis SQL que no corresponden a credenciales válidas
- **THEN** el sistema responde `401 Unauthorized` con el error documentado para credenciales inválidas y no devuelve una respuesta exitosa ni datos de usuario autenticado

#### Scenario: Credenciales inválidas no autentican
- **WHEN** se envían credenciales que no son válidas
- **THEN** el sistema responde `401 Unauthorized` según el contrato documentado

### Requirement: El login válido conserva su respuesta documentada
El sistema SHALL conservar el estado y los campos de respuesta documentados para una autenticación válida.

#### Scenario: Autenticación válida
- **WHEN** se envían credenciales válidas
- **THEN** el sistema responde `200 OK` con `token`, `expiresIn`, `user` y `debug`, y `user` contiene `username`, `displayName` y `role`

### Requirement: La validación de request conserva sus respuestas documentadas
El sistema SHALL conservar las respuestas `400 Bad Request` documentadas para requests nulos o con username o password nulos, vacíos o en blanco.

#### Scenario: Request nulo
- **WHEN** se envía un request de login nulo
- **THEN** el sistema responde `400 Bad Request` con el mensaje documentado `Request is null`

#### Scenario: Username ausente o en blanco
- **WHEN** el username se envía nulo, vacío o en blanco
- **THEN** el sistema responde `400 Bad Request` con el mensaje documentado `Username is required`

#### Scenario: Password ausente o en blanco
- **WHEN** el password se envía nulo, vacío o en blanco
- **THEN** el sistema responde `400 Bad Request` con el mensaje documentado `Password is required`

### Requirement: Los errores inesperados conservan el contrato HTTP documentado
El sistema SHALL conservar el estado y la estructura de respuesta `500 Internal Server Error` documentados para errores inesperados durante el login; este Issue no cambia ese contrato.

#### Scenario: Error inesperado durante el login
- **WHEN** ocurre un error inesperado durante el procesamiento del login
- **THEN** el sistema responde `500 Internal Server Error` con los campos `message` y `exception` descritos en `docs/ENDPOINTS.md`