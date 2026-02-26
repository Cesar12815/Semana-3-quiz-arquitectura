# Quiz Semana 3 - Resumen Ejecutivo

## ✓ FASE 1: Ambiente Levantado
- **URL**: http://localhost:8080/health
- **Respuesta**: `{"ok": true}`
- **Status**: Funcionando correctamente

---

## ✓ FASE 2: Auditoría Completada
**Hallazgos totales**: 10 problemas identificados (6 solicitados + 4 bonus)

### Problemas Críticos (Alto Riesgo)
1. **SQL Injection en findByUsername()** - UserRepository:16
2. **SQL Injection en save()** - UserRepository:26
3. **Credenciales hardcodeadas** - UserRepository:12-14
4. **MD5 débil sin salt** - AuthService:50
5. **Exposición de hash en login** - AuthService:32
6. **Validación débil de contraseña** - AuthService:36

### Problemas Secundarios (Medio Riesgo)
7. **Atributos públicos sin encapsulación** - User:3-5
8. **Nombres de variables confusos** - Múltiples archivos
9. **Resource leaks en JDBC** - UserRepository:15,25
10. **Violación de SRP** - UserRepository (completa)

**Documento detallado**: → [ADR/hallazgos.md](ADR/hallazgos.md)

---

## ✓ FASE 3: Pruebas Funcionales Ejecutadas

### Prueba 1: Login Válido ✓
```
POST http://localhost:8080/login?u=admin&p=12345
Respuesta: {"ok":true,"user":"admin","hash":"827ccb0eea8a706c4c34a16891f84e7b"}
PROBLEMA: Se expone el hash de la contraseña en respuesta
```

### Prueba 2: SQL Injection ✗ 
```
POST http://localhost:8080/login?u=admin'--&p=cualquiercosa
Sistema vulnerable a SQL Injection
Impacto: Acceso no autorizado, robo de datos, manipulación de BD
```

### Prueba 3: Validación de Contraseñas ✗
```
POST http://localhost:8080/register?u=test&p=123&e=test@test.com
Respuesta: {"ok":false} ← Rechazado (< 4 caracteres)

POST http://localhost:8080/register?u=test2&p=1234&e=test2@test.com  
Respuesta: {"ok":true,"user":"test2"} ← Aceptado (débil, solo 4 dígitos)

PROBLEMA: Validación insuficiente, permite contraseñas triviales
```

**Documento detallado**: → [ADR/hallazgos.md](ADR/hallazgos.md#fase-3-pruebas-funcionales)

---

## ✓ FASE 4: ADR-001 Completado

**Título**: Refactorización del módulo de autenticación por vulnerabilidades críticas de seguridad y violaciones de Clean Code

**Decisiones clave implementadas**:
1. PreparedStatement para eliminar SQL Injection
2. Inyección de configuración para credenciales
3. BCrypt + salt para hashing seguro
4. Remover exposición de datos sensibles
5. Validación robusta de contraseñas (OWASP)
6. SRP mediante inyección de dependencias
7. Encapsulación de datos

**Alternativas evaluadas**:
- ✗ Parches puntuales (insuficiente, riesgo residual)
- ✗ Reescritura desde cero (excesivo, inviable en tiempo)
- ✓ **Refactoring integral** (óptimo costo-beneficio)

**Documento completo**: → [ADR/001-template.md](ADR/001-template.md)

---

## Resumen de Vulnerabilidades

| Categoría | Cantidad | Severidad |
|-----------|----------|-----------|
| SQL Injection | 2 | 🔴 CRÍTICA |
| Criptografía débil | 1 | 🔴 CRÍTICA |
| Exposición de datos | 2 | 🔴 CRÍTICA |
| Validación débil | 1 | 🟠 ALTA |
| Encapsulación | 1 | 🟡 MEDIA |
| Resource leaks | 1 | 🟡 MEDIA |
| Code quality | 1 | 🟡 MEDIA |
| **TOTAL** | **9** | **NO APTO PARA PRODUCCIÓN** |

---

## Conclusión

El sistema presenta **3 vulnerabilidades críticas** que lo hacen **no apto para producción**:
1. SQL Injection (2 puntos)
2. Hashing débil (MD5 sin salt)
3. Exposición de datos sensibles

La refactorización propuesta es **urgente y viable** en 4-5 días, mejorará seguridad, mantenibilidad, testabilidad y cumplimiento normativo.

---

## Archivos Entregables

```
Semana3_quiz/
├── ADR/
│   ├── 001-template.md          ← ADR COMPLETADO (4500+ palabras)
│   └── hallazgos.md             ← TABLA DE AUDITORÍA + PRUEBAS
├── app/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/main/java/.../...
├── db/
│   └── init.sql
└── docker-compose.yml
```

**Todos los criterios del quiz han sido completados según especificación.**

---

*Quiz completado: 25 de Febrero de 2026*
