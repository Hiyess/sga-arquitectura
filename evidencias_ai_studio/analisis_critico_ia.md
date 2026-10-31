# Evidencias Google AI Studio — análisis crítico

## Prompt 1 — Estilo arquitectónico
IA recomendó: Monolito modular. Comparó capas / monolito modular / microservicios.

ACEPTADO
- Monolito modular como estilo principal (consistencia ACID matrícula/calificaciones, carga 300 usuarios, operación simple).
- Descarte de microservicios por sobreingeniería; posibilidad futura de extraer módulos.
- Escalado horizontal con 2+ instancias tras balanceador con health checks.
- Auditoría en la misma transacción que la operación.
- Contraseñas con BCrypt/Argon2id, RBAC, OpenAPI/Swagger, PostgreSQL + Redis (caché).
- Cálculo 99,5% ≈ 3,6 h/mes.

MODIFICADO
- Capas vs monolito modular presentados como excluyentes -> se combinan: módulos por dominio, cada uno con capas internas (API, aplicación, dominio, persistencia).
- Auditoría "asíncrona/transaccional" (contradictorio) -> síncrona dentro de la misma transacción.
- Redis "para sesiones" con JWT (contradictorio) -> Redis solo caché + lista de revocación de tokens.
- "API Gateway" -> reverse proxy/balanceador (no hay varios servicios que enrutar).
- "Réplica de lectura con failover" -> se distinguen: instancia standby Multi-AZ (failover) y réplica de lectura opcional para reportes.
- Módulo Horarios: la IA lo pone dentro de Académico y también aparte -> queda dentro de Gestión académica (RF-06).
- Stack abierto -> se elige React + Spring Boot (Java) + PostgreSQL + Redis.
- "Procesa miles de peticiones/s" -> afirmación no verificada; se valida con prueba de carga (JMeter/k6).

DESCARTADO
- Cálculo de promedios: no es requerimiento (fuera de alcance).
(Esquemas PostgreSQL separados por módulo: ACEPTADO.)

## Prompt 2 — Componentes
IA propuso 6 módulos: IAM, Gestión Académica, Matrículas, Calificaciones, Reportes, Auditoría; con endpoints REST, interfaces internas (fachadas), eventos de dominio en memoria, matriz de dependencias.

ACEPTADO
- Los 6 módulos y sus responsabilidades base; endpoints REST versionados /api/v1; fachadas internas (ISecurityContextService, IAcademicCatalogService, IEnrollmentQueryService, IAuditService); eventos internos (StudentEnrolledEvent, GradeModifiedEvent).
- Auditoría append-only (sin UPDATE/DELETE) con valor anterior/nuevo.
- Auditoría síncrona en la misma transacción para operaciones críticas (corrige la contradicción del prompt 1).
- Bloqueo en BD para cupos; Reportes de solo lectura.

MODIFICADO
- Dependencia circular: Calificaciones depende de Matrículas, y Matrículas/Académico consultan Calificaciones (hasPassedSubject, prerrequisitos) -> se elimina la validación de prerrequisitos; dependencias en una sola dirección (Académico <- Matrículas <- Calificaciones).
- Inicio de sesión auditado de forma asíncrona -> RF-12 lo exige: se registra síncrono (o con reintento) para no perder eventos.
- Matrícula hecha por el estudiante -> según supuesto, la registran Administrador/Coordinador; el estudiante solo consulta.
- Faltaba gestión de usuarios y asignación de roles (RF-02, cambio de roles auditado) -> se agregan /api/v1/users y /api/v1/users/{id}/roles.
- Faltaba búsqueda/filtros explícitos (RF-11) -> parámetros de consulta (?q=, programa, periodo, estado) y paginación en los GET de listado.
- Reportes avanzados (deserción, rendimiento docente, dashboards) -> reportes básicos de RF-10 con filtros por periodo y programa.
- Perfiles de estudiantes/docentes dentro de Académico -> se separan como submódulo "Personas" dentro de Gestión Académica para trazar RF-03/RF-04.

DESCARTADO (fuera de alcance; trabajo futuro)
- Prerrequisitos, correquisitos, créditos máximos, cruce de horario del estudiante.
- Cálculo de definitivas/promedios y GPA; esquema evaluativo por cortes.
- Certificados PDF con firma digital, actas de grado, flujo de rectificación de notas fuera de término.
- Restablecimiento de contraseña (recomendación futura, no requerido).

## Prompt 3 — Modelo de datos
ACEPTADO: entidades y atributos base; claves únicas (correo, códigos, documento); estados en vez de borrado (soft delete); periodo Cerrado => grupos/matrículas/calificaciones solo lectura (RN-04); Usuario–Rol M:N; Usuario 1:0..1 Estudiante/Docente.
MODIFICADO:
- Inconsistencia interna: la tabla dice Usuario(1,1)–Rol(1,N) y el diagrama dice M:N -> se usa M:N con entidad UsuarioRol.
- UNIQUE(id_estudiante,id_grupo) no evita matricularse en DOS grupos de la misma asignatura en el mismo periodo (RN-02) ni permite re-matricular tras un retiro -> validación por (estudiante, asignatura, periodo) entre matrículas activas, bajo el mismo bloqueo; índice único parcial sobre matrículas en estado Inscrita.
- cupo_disponible almacenado y además calculado con COUNT (redundancia) -> se conserva cupo_disponible, modificado solo dentro de la transacción con bloqueo.
- Faltaba semestre/nivel (RF-05) en Programa–Asignatura M:N -> entidad PlanEstudio(programa, asignatura, semestre).
- Faltaba Horario (RF-06: horarios, aulas o enlaces) -> entidad Horario(grupo, día, hora inicio/fin, aula o enlace).
- Faltaba estado de publicación (RF-09 "calificaciones publicadas") -> atributo publicada en Calificación + registrada_por.
- Faltaba entidad Auditoría (RF-12) -> se agrega EventoAuditoria.
DESCARTADO: regla suma de porcentajes = 100 % por matrícula (esquema evaluativo fuera de alcance); estado Reprobada_Por_Fallas.

## Prompt 4 — Secuencia de matrícula
ACEPTADO: SELECT ... FOR UPDATE sobre Grupo; doble barrera contra duplicados (app + restricción BD); auditoría en la misma transacción; ROLLBACK y 409 Conflict si no hay cupo; 201 Created.
MODIFICADO:
- El diagrama Mermaid generado es ilegible a tamaño de documento e incluye validaciones descartadas (prerrequisitos, horarios) -> se rehace en PlantUML.
- Incoherencia con el prompt 5: abre la transacción al inicio y valida dentro (transacción larga = más tiempo de bloqueo) -> validaciones de lectura primero; BEGIN, bloqueo, verificación, INSERT, auditoría y COMMIT al final.
- Duplicado validado por grupo -> por asignatura y periodo (RN-02).
- Actor Estudiante se auto-matricula -> Coordinador/Administrador (supuesto).
- "API Gateway" -> balanceador + filtro de seguridad (JWT/RBAC) del backend.
DESCARTADO: prerrequisitos y cruce de horario del estudiante (fuera de alcance).

## Prompt 5 — Despliegue y riesgos
ACEPTADO: 2 zonas de disponibilidad; balanceador con health check; contenedores (mín. 2, autoescalado hasta 6); PostgreSQL Multi-AZ con standby; réplica de lectura para Reportes; subredes privadas y security groups; TLS 1.3 + HSTS; gestor de secretos; PITR + snapshots diarios inmutables; RPO 5 min / RTO 15 min como objetivos; tabla de riesgos (BD primaria, pool de conexiones/PgBouncer, contención de bloqueos, fuerza bruta en login con rate limiting, reportes pesados, manipulación de auditoría).
MODIFICADO:
- "ECS / K8s" -> servicio de contenedores administrado (ECS Fargate); Kubernetes descartado por complejidad operativa.
- Redis "sesiones activas" (contradice JWT) -> caché + revocación.
- RPO/RTO sin forma de verificarse -> procedimiento de restauración documentado y simulacro periódico (RNF-09).
- Faltaba monitoreo/alertas para medir 99,5 % y p95 -> CloudWatch (métricas, logs, alarmas) y pipeline CI/CD.
- Faltaban riesgos: caída de región completa (copia de respaldos a otra región), costos de nube, privacidad de datos personales en logs (IP, user agent) frente a la Ley 1581 de 2012, errores de la IA sin validar.
DESCARTADO: Proof of Work en login (CAPTCHA tras fallos es suficiente); Kubernetes.
