# ADR-001: Refactorización del módulo de autenticación por vulnerabilidades críticas de seguridad y violaciones de Clean Code

## Contexto

El módulo de autenticación del sistema (`UserRepository`, `AuthService` y `AuthController`) presenta vulnerabilidades críticas que lo hacen **no apto para producción**. 

Actualmente, el sistema implementa autenticación y registro de usuarios mediante una arquitectura que concatena directamente parámetros en consultas SQL sin preparación, utiliza MD5 sin salt para hash de contraseñas, expone datos sensibles en respuestas de API, y viola principios SOLID como Single Responsibility y Open/Closed. 

Durante la auditoría de seguridad realizada (Semana 3, Quiz de Arquitectura), se identificaron **10 problemas críticos**:
- **2 vulnerabilidades de SQL Injection** (riesgo Alto) en las métodos `findByUsername()` y `save()` que permiten acceso no autorizado y manipulación de base de datos
- **Credenciales hardcodeadas** en código fuente exponiendo usuario y contraseña administrativos
- **Hashing criptográfico débil con MD5** sin salt, permitiendo recuperación de contraseñas mediante rainbow tables
- **Exposición de hash de contraseña en respuesta de login**, capaz de ser reutilizado para ataques
- **Validación de contraseña extremadamente débil** (solo > 3 caracteres) permitiendo contraseñas triviales como "1234"
- **Recursos JDBC no cerrados**, causando memory leaks
- **Violación de encapsulación** mediante atributos públicos
- **Nombres de variables confusos** (s, u, p, c, x) reduciendo mantenibilidad

Esta situación afecta directamente a **usuarios finales** (riesgo de robo de credenciales), **equipo de desarrollo** (código difícil de mantener y extender), e **infraestructura** (riesgo de comprometer toda la base de datos). Las pruebas funcionales confirmaron todas estas vulnerabilidades en acción.

## Decisión

Se refactorizará completamente el módulo de autenticación aplicando **SOLID principles**, **Clean Code**, y **buenas prácticas de seguridad** mediante las siguientes decisiones concretas:

### 1. Eliminación de SQL Injection mediante PreparedStatement
**Decisión**: Reemplazar todas las consultas SQL construidas con concatenación de strings por `PreparedStatement` con parámetros vinculados.

**Implementación**:
```java
// Antes (VULNERABLE):
String q = "select username, email, password from users where username = '" + u + "'";
Statement s = c.createStatement();
ResultSet r = s.executeQuery(q);

// Después (SEGURO):
String q = "select username, email, password from users where username = ?";
PreparedStatement ps = c.prepareStatement(q);
ps.setString(1, u);
ResultSet r = ps.executeQuery();
```

**Por qué**: PreparedStatement separa la lógica de datos de los parámetros, haciendo imposible inyectar código SQL. OWASP y CWE-89 recomiendan esto como defensa principal contra SQL Injection.

### 2. Remover credenciales de código mediante inyección de configuración
**Decisión**: Eliminar strings hardcodeados de URL, usuario y password de BD. Usar `@ConfigurationProperties` de Spring Boot o variables de entorno.

**Implementación**:
```java
// Antes (VULNERABLE):
private String url = "jdbc:postgresql://db:5432/logincaos";
private String user = "admin";
private String pass = "admin123";

// Después (SEGURO):
@Value("${spring.datasource.url}")
private String url;
@Value("${spring.datasource.username}")
private String user;
@Value("${spring.datasource.password}")
private String pass;
```

**Por qué**: Las credenciales en variables de entorno no se comiten al repositorio, cumpliendo con CWE-798 (Hardcoded Credentials) y mejorando la seguridad en CI/CD.

### 3. Implementar hashing seguro con BCrypt + salt
**Decisión**: Reemplazar MD5 por `BCryptPasswordEncoder` de Spring Security con salt automático y ajustable.

**Implementación**:
```java
// Antes (INSEGURO):
private String md5(String s) throws Exception {
    MessageDigest d = MessageDigest.getInstance("MD5");
    // MD5 es débil sin salt
}

// Después (SEGURO):
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(10); // 10 rounds de salt
}

// En AuthService:
String hashedPassword = passwordEncoder.encode(rawPassword);
```

**Por qué**: BCrypt implementa Blowfish con salt automático y factor de trabajo ajustable. Es computacionalmente caro para atacantes, cumpliendo OWASP A02:2021 (Cryptographic Failures).

### 4. No exponer datos sensibles en respuestas de API
**Decisión**: Remover el campo "hash" de todas las respuestas de autenticación y registro.

**Implementación**:
```java
// Antes (VULNERABLE):
res.put("ok", true);
res.put("user", c.username);
res.put("hash", hp);  // EXPONE EL HASH

// Después (SEGURO):
res.put("ok", true);
res.put("user", c.username);
// NO incluir "hash" nunca
```

**Por qué**: El hash nunca debe ser transmitido al cliente. Solo el servidor necesita comparar hashes. Esto previene CWE-200 (Exposure of Sensitive Information).

### 5. Validación de contraseña robusta con criterios OWASP
**Decisión**: Implementar política de contraseñas que requiera mínimo 8 caracteres con complejidad (mayúscula, número, símbolo) o integrar con Spring Security Password Validator.

**Implementación**:
```java
// Antes (DÉBIL):
if (p.length() > 3)  // Solo 4+ caracteres

// Después (ROBUSTO):
if (p.length() < 8 || !p.matches(".*[A-Z].*") || 
    !p.matches(".*\\d.*") || !p.matches(".*[!@#$%^&*].*")) {
    throw new WeakPasswordException("Contraseña no cumple criterios");
}
// O mejor aún: usar PasswordValidator de Spring Security
```

**Por qué**: OWASP y NIST recomiendan 8-12 caracteres mínimo con complejidad. Previene CWE-521 (Weak Password Requirements).

### 6. Aplicar SRP mediante inyección de dependencias y separación de responsabilidades
**Decisión**: Crear clases especializadas:
- `JdbcConnectionPool`: Gestionar conexiones (con try-with-resources)
- `UserRepository`: Solo lógica de persistencia
- `PasswordEncoder`: Solo hashing (delegado a Spring)
- `AuthService`: Orquestar autenticación

**Implementación**:
```java
// UserRepository - Solo persistencia
public User findByUsername(String username) throws Exception {
    try (Connection c = dataSource.getConnection();
         PreparedStatement ps = c.prepareStatement(
             "select username, email, password from users where username = ?")) {
        ps.setString(1, username);
        try (ResultSet r = ps.executeQuery()) {
            if (r.next()) return mapUser(r);
        }
    }
    return null;
}

// AuthService - Lógica de autenticación
public Map<String, Object> login(String u, String p) throws Exception {
    User user = userRepository.findByUsername(u);
    if (user != null && passwordEncoder.matches(p, user.getPassword())) {
        return Map.of("ok", true, "user", user.getUsername());
    }
    return Map.of("ok", false);
}
```

**Por qué**: Cumple SOLID SRP (cada clase una responsabilidad), facilita testing unitario, y permite reutilizar componentes (PasswordEncoder, ConnectionPool).

### 7. Encapsulación de datos mediante getters/setters
**Decisión**: Convertir atributos públicos de `User` en privados con getters/setters validados.

**Implementación**:
```java
// Antes (EXPUESTO):
public class User {
    public String username;
    public String email;
    public String password;
}

// Después (ENCAPSULADO):
public class User {
    private String username;
    private String email;
    private String password;
    
    public String getUsername() { return username; }
    public void setUsername(String u) { /* validar */ }
    // ... getters/setters para email y password
}
```

**Por qué**: Permite validación en setters, facilita evolución futura, cumple OCP (Open/Closed Principle).

## Consecuencias

### Consecuencias Positivas

✓ **Seguridad crítica mejorada**: SQL Injection eliminada, hashing resistente a ataques, credenciales protegidas  
✓ **Cumplimiento normativo**: Alineado con OWASP Top 10, NIST, CWE best practices  
✓ **Mantenibilidad incrementada**: Código legible con nombres descriptivos, separación clara de responsabilidades  
✓ **Testabilidad mejorada**: Inyección de dependencias permite mocking y testing cercano a 100% de cobertura  
✓ **Rendimiento**: Try-with-resources cierra automáticamente conexiones, evitando memory leaks  
✓ **Extensibilidad**: Arquitectura modular facilita agregar autenticación multi-factor, OAuth2, auditoría  
✓ **Confianza del usuario**: Datos sensibles no se exponen, contraseñas seguras requeridas  

### Consecuencias Negativas / Riesgos

⚠️ **Esfuerzo de refactoring**: Estimado 4-5 días de desarrollo (reescribir 3 clases, tests, validación)  
⚠️ **Riesgo de regresiones**: Requiere testing funcional y de integración exhaustivo before/after  
⚠️ **Deuda técnica temporal**: Sistema vulnerabilidad durante transitorio (mitigar con feature flags)  
⚠️ **Capacitación necesaria**: Equipo debe entender PreparedStatement, BCrypt, SOLID  
⚠️ **Dependencias adicionales**: Spring Security (size aumento ~3MB), pero beneficio > costo  

## Alternativas consideradas

### Alternativa 1: Parches puntuales sin refactoring
**Decisión descartada**:
- Reemplazar solo MD5 con BCrypt manteniendo concatenación SQL
- Mantener código existente con cambios mínimos
- **Por qué descartar**: 
  - SQL Injection seguiría siendo explotable
  - Violaciones de SRP persistirían (difícil mantener después)
  - Code smell continuo afectaría moral del equipo
  - Riesgo de "parches parciales" sin visión integral causarían bugs futuros
  - **Riesgo residual muy alto**: ✗ NO VIABLE

### Alternativa 2: Reescribir desde cero con framework completo (Spring Security + JPA)
**Decisión descartada**:
- Usar Spring Security + Spring Data JPA + Hibernate
- Reescribir completamente con nuevas tecnologías
- **Por qué descartar**:
  - Curva de aprendizaje alta (semanas vs. días)
  - Risk de incompatibilidades con integración actual
  - Overkill para aplicación simple
  - Costo > beneficio en contexto actual
  - **Tiempo de implementación**: Excesivo para urgencia actual ✗ NO ÓPTIMO

### Alternativa 3: Refactoring integral (ELEGIDA)
**Razón**: 
- Máxima seguridad (PreparedStatement, BCrypt, sin datos sensibles)
- Máxima sostenibilidad (SOLID, Clean Code)
- Esfuerzo razonable (4-5 días)
- Costo-beneficio optimal
- ✓ RECOMENDADO

---

**Decisión final**: Proceder con refactoring integral (Alternativa 3) con timeline de 1 semana e iteraciones de testing incrementales.
