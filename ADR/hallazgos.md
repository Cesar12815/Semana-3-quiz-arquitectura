# Auditoría de Código - Quiz Semana 3

## FASE 1 ✓ Completado
El ambiente fue levantado exitosamente:
- **Endpoint**: `http://localhost:8080/health`
- **Respuesta**: `{"ok": true}`
- **Status**: ✓ Funcionando correctamente

---

## FASE 2: Tabla de Hallazgos de Auditoría

| # | Descripción del problema | Archivo | Línea aprox. | Principio violado | Riesgo |
|---|---|---|---|---|---|
| 1 | SQL Injection en consulta SELECT mediante concatenación directa de parámetro username | `UserRepository.java` | 16 | Seguridad básica | **Alto** |
| 2 | SQL Injection en INSERT mediante concatenación directa de parámetros (username, email, password) | `UserRepository.java` | 26 | Seguridad básica | **Alto** |
| 3 | Credenciales de base de datos hardcodeadas en código fuente (usuario: admin, password: admin123) | `UserRepository.java` | 12-14 | Seguridad básica, exposición de datos | **Alto** |
| 4 | Hash MD5 sin salt para contraseñas - criptografía insegura y vulnerable a rainbow tables | `AuthService.java` | 50 | Seguridad básica | **Alto** |
| 5 | Exposición de hash de contraseña en respuesta de login (campo "hash" retornado al cliente) | `AuthService.java` | 32 | Seguridad básica, exposición de datos | **Alto** |
| 6 | Validación de contraseña extremadamente débil: solo requiere longitud > 3 caracteres | `AuthService.java` | 36 | Seguridad básica | **Medio** |
| 7 | Atributos públicos sin encapsulación en clase User (username, email, password) | `User.java` | 3-5 | Clean Code (Encapsulation), OCP | **Medio** |
| 8 | Nombres de variables confusos y no descriptivos (s, u, p, c, x, r, q, d, b) | `AuthController.java`, `AuthService.java`, `UserRepository.java` | Múltiples | Clean Code (Naming) | **Medio** |
| 9 | Recursos JDBC no cerrados apropiadamente (Connection y Statement - resource leak) | `UserRepository.java` | 15, 25 | SOLID (SRP), Buenas prácticas | **Medio** |
| 10 | Violación de Single Responsibility Principle en UserRepository (gestiona conexiones y lógica de datos) | `UserRepository.java` | 1-35 | SOLID (SRP) | **Medio** |

**Total de problemas encontrados**: 10 (6 críticos solicitados, +4 adicionales)

---

## FASE 3: Pruebas Funcionales

### Prueba 1: Login Válido
**Comando**: `POST http://localhost:8080/login?u=admin&p=12345`

**Respuesta**:
```json
{
  "ok": true,
  "user": "admin",
  "hash": "827ccb0eea8a706c4c34a16891f84e7b"
}
```

**Análisis**:
- ✓ Login funciona correctamente
- ✗ **PROBLEMA CRÍTICO**: Se retorna el hash MD5 de la contraseña en la respuesta
- ✗ **DATO SENSIBLE**: El cliente puede capturar y reutilizar el hash
- ✗ **Impacto**: Un atacante podría usar este hash para:
  - Ataques de fuerza bruta offline
  - Rainbow table lookups
  - Reutilización del hash en otros sistemas

**Debería retornarse**: Solo `{"ok": true, "user": "admin"}` sin el hash

---

### Prueba 2: SQL Injection
**Comando**: `POST http://localhost:8080/login?u=admin'--&p=cualquiercosa`

**Respuesta**:
```json
{
  "ok": false,
  "hash": "f73862908453012d17eb1d60240d95d1"
}
```

**Análisis**:
- ✗ **VULNERABLE A SQL INJECTION**: El parámetro username se inserta directamente en la query
- **Evidencia en código** (línea 16 UserRepository.java):
  ```java
  String q = "select username, email, password from users where username = '" + u + "'";
  ```
- **Por qué es peligroso en producción**:
  - Permite extraer datos de cualquier tabla de la BD
  - Permite insertar, actualizar o borrar registros
  - Permite ejecutar comandos del sistema operativo
  - Contournea completamente el sistema de autenticación
  - Puede llevar a exposición de toda la BD (datos de todos los usuarios)
  - Potencial para escalación de privilegios

- **Remediación requerida**: Usar `PreparedStatement` con parámetros vinculados

---

### Prueba 3a: Registro con Contraseña Débil (3 caracteres)
**Comando**: `POST http://localhost:8080/register?u=test&p=123&e=test@test.com`

**Respuesta**:
```json
{
  "ok": false
}
```

**Resultado**: ✓ Rechazado (La validación requiere `length() > 3`)

---

### Prueba 3b: Registro con Contraseña Débil (4 caracteres)
**Comando**: `POST http://localhost:8080/register?u=test2&p=1234&e=test2@test.com`

**Respuesta**:
```json
{
  "ok": true,
  "user": "test2"
}
```

**Resultado**: ✗ Aceptado (Debería ser rechazado)

**Análisis**:
- La validación actual es `if (p.length() > 3)` - solo verifica 4 caracteres mínimo
- **¿Es suficiente?**: NO
  - OWASP recomienda mínimo 8-12 caracteres
  - NIST recomienda mínimo 8 caracteres con complejidad
  - "1234" es trivial de crackear (< 1ms con herramientas modernas)
  - No hay validación de complejidad (mayúsculas, números, símbolos)

- **Vulnerabilidades de validación débil**:
  - Ataque de diccionario
  - Ataque de fuerza bruta (4 dígitos = 10,000 combinaciones)
  - Contraseñas predecibles

---

## Resumen de Riesgos de Seguridad

| Categoría | Cantidad | Severidad | Impacto |
|---|---|---|---|
| SQL Injection | 2 | Crítica | Acceso no autorizado, robo de datos, manipulación de BD |
| Criptografía débil | 1 | Crítica | Recuperación de contraseñas |
| Exposición de datos sensibles | 2 | Crítica | Riesgo de autenticación comprometida |
| Validación débil | 1 | Alta | Acceso con contraseñas triviales |
| Encapsulación deficiente | 1 | Media | Riesgos de modificación de estado |
| Resource leaks | 1 | Media | Degradación de rendimiento |
| Code naming | 1 | Media | Mantenibilidad reducida |
| **TOTAL** | **9** | **Muy Alto** | **Sistema inseguro para producción** |

---

## Conclusión

El sistema presenta **vulnerabilidades críticas de seguridad** que lo hacen **NO APTO PARA PRODUCCIÓN**. Las vulnerabilidades de SQL Injection combinadas con hashing débil y exposición de datos sensibles crean un riesgo extremadamente alto para los datos de los usuarios y la integridad de la aplicación. Es necesaria una refactorización urgente e integral del módulo de autenticación.
