# Tácticas Arquitectónicas - MentorSoft

Este documento detalla cómo se aplica cada táctica arquitectónica seleccionada para los escenarios de calidad del proyecto MentorSoft, los servicios necesarios a nivel 3 del modelo C4 y los patrones de diseño o arquitectónicos asociados.

---

## Escenario 1: Estudiante envía evaluación sin finalizar

### 1. Autosave incremental

**Aplicación en el proyecto**: El sistema guarda automáticamente las respuestas del estudiante a intervalos regulares (por ejemplo, cada 30 segundos) o tras cada interacción significativa (selección de respuesta, cambio en campo de texto). Estas respuestas parciales se almacenan en un almacenamiento temporal de baja latencia antes de ser consolidadas al envío final de la evaluación.

**Servicios C4 Nivel 3**:
- `autosave-service`: Responsable de recibir y persistir los borradores incrementales de las respuestas del estudiante. Expone un endpoint `POST /evaluations/{id}/autosave` que acepta el estado actual de la evaluación.
- `autosave-scheduler`: Componente interno que gestiona la periodicidad del guardado automático y coordina la sincronización entre el almacenamiento temporal y el definitivo.
- `draft-store`: Almacén de datos temporal (por ejemplo, Redis) que mantiene los borradores activos de evaluaciones en curso.

**Patrones asociados**:
- **Patrón arquitectónico**: Event Sourcing (parcial) — cada autosave es un evento que representa el estado de la evaluación en un momento dado.
- **Patrón de diseño**: Observer — el frontend notifica al backend de cambios en el estado de la evaluación mediante eventos.
- **Patrón de diseño**: Debounce — para evitar escrituras excesivas, se agrupan múltiples cambios rápidos en una sola operación de guardado.

---

### 2. Persistencia temporal desacoplada

**Aplicación en el proyecto**: Las respuestas en curso de una evaluación se almacenan en un sistema de persistencia temporal separado de la base de datos transaccional principal. Esto evita que las escrituras frecuentes y de bajo valor durante la evaluación impacten el rendimiento de la base de datos definitiva. Al finalizar la evaluación, los datos se migran al almacén definitivo.

**Servicios C4 Nivel 3**:
- `temporal-storage-service`: Gestiona el ciclo de vida de los datos temporales. Expone operaciones CRUD para borradores y coordina la migración al almacenamiento definitivo.
- `migration-worker`: Proceso en background que transfiere respuestas temporalmente almacenadas al almacén definitivo cuando se confirma el envío de la evaluación o cuando expira el tiempo límite.
- `temporal-data-store`: Base de datos temporal (por ejemplo, Redis o DynamoDB con TTL) configurada con expiración automática para datos huérfanos.

**Patrones asociados**:
- **Patrón arquitectónico**: Sidecar — el servicio de persistencia temporal puede desplegarse como un componente lateral al servicio principal de evaluación.
- **Patrón de diseño**: Repository — abstrae el acceso al almacenamiento temporal y definitivo bajo una interfaz común.
- **Patrón de diseño**: Strategy — permite intercambiar la implementación del almacenamiento temporal sin afectar la lógica de negocio.

---

### 3. Session recovery

**Aplicación en el proyecto**: Cuando un estudiante reconecta tras una desconexión o cierre inesperado del navegador, el sistema restaura el último estado guardado de su evaluación. Se utiliza un token de sesión persistente (cookie con identificador) que permite asociar al estudiante con su borrador almacenado.

**Servicios C4 Nivel 3**:
- `session-recovery-service`: Responsable de detectar reconexiones y restaurar el estado de la evaluación desde el último autosave. Expone `GET /evaluations/{id}/recover`.
- `session-manager`: Gestiona el ciclo de vida de las sesiones de evaluación, incluyendo la detección de sesiones huérfanas y la limpieza de sesiones expiradas.
- `session-state-store`: Almacén que mantiene el mapeo entre tokens de sesión y el estado actual de la evaluación.

**Patrones asociados**:
- **Patrón arquitectónico**: Stateful Session — el sistema mantiene estado de sesión entre reconexiones.
- **Patrón de diseño**: Memento — captura y restaura el estado interno de la evaluación sin violar la encapsulación.
- **Patrón de diseño**: Token-based Authentication — el token de sesión permite identificar al usuario y su estado sin requerir re-autenticación completa.

---

### 4. Caché temporal distribuida

**Aplicación en el proyecto**: Se utiliza una caché distribuida (por ejemplo, Redis Cluster) para almacenar el estado de las evaluaciones activas, las preguntas de la evaluación actual y los metadatos del estudiante. Esto reduce la latencia de lectura y disminuye la carga sobre la base de datos principal durante picos de concurrencia.

**Servicios C4 Nivel 3**:
- `cache-service`: Gestiona las operaciones de lectura/escritura en la caché distribuida. Implementa estrategias de invalidación y expiración.
- `cache-coordinator`: Coordina la consistencia entre la caché y el almacenamiento definitivo, manejando invalidaciones y actualizaciones.
- `distributed-cache`: Infraestructura de caché distribuida (Redis Cluster o Memcached) que almacena datos de sesiones activas y preguntas frecuentes.

**Patrones asociados**:
- **Patrón arquitectónico**: Cache-Aside (Lazy Loading) — el servicio consulta primero la caché; si no existe, consulta la base de datos y popula la caché.
- **Patrón de diseño**: Decorator — la capa de caché envuelve el acceso al repositorio sin modificar la lógica de negocio.
- **Patrón de diseño**: Singleton — la instancia del cliente de caché distribuida se comparte entre solicitudes para optimizar conexiones.

---

## Escenario 2: Modificación de reglas de evaluación académica

### 5. Versionamiento de reglas

**Aplicación en el proyecto**: Cada modificación en la estructura de rúbricas genera una nueva versión de la regla. Las evaluaciones existentes referencian la versión de la regla vigente al momento de su creación, mientras que las nuevas evaluaciones utilizan la versión más reciente. Esto permite evolucionar las rúbricas sin afectar evaluaciones históricas.

**Servicios C4 Nivel 3**:
- `rules-versioning-service`: Gestiona la creación, consulta y activación de versiones de reglas de evaluación. Expone `POST /rules/{id}/versions` para crear nuevas versiones y `GET /rules/{id}/versions/{version}` para consultar una versión específica.
- `rule-evaluator`: Servicio que aplica la versión correcta de la regla según la evaluación que se está calificando.
- `rules-store`: Base de datos que almacena las reglas con su historial de versiones, incluyendo metadatos de cambios (autor, fecha, descripción).

**Patrones asociados**:
- **Patrón arquitectónico**: Event Sourcing — cada cambio en una regla se registra como un evento inmutable que genera una nueva versión.
- **Patrón de diseño**: Factory Method — crea instancias de evaluadores según la versión de la regla requerida.
- **Patrón de diseño**: Versioned Entity — las reglas se modelan como entidades con ciclo de vida versionado.

---

### 6. APIs desacopladas

**Aplicación en el proyecto**: Los módulos consumidores de las reglas de evaluación (módulo de calificación, módulo de recomendación, módulo de reportes) acceden a las reglas mediante interfaces de API estables y versionadas. Los cambios internos en la estructura de las reglas no afectan a los consumidores mientras se mantenga la compatibilidad del contrato de la API.

**Servicios C4 Nivel 3**:
- `rules-api-gateway`: Punto de entrada único para las APIs de reglas. Maneja el enrutamiento según la versión de la API solicitada (`/api/v1/rules`, `/api/v2/rules`).
- `rules-adapter-service`: Traduce entre las versiones de la API y la representación interna de las reglas, permitiendo evolución interna sin romper contratos externos.
- `contract-validator`: Valida que las respuestas de la API cumplan con el esquema definido para cada versión del contrato.

**Patrones asociados**:
- **Patrón arquitectónico**: API Gateway — centraliza el acceso a los servicios de reglas y maneja versionamiento de rutas.
- **Patrón de diseño**: Adapter — adapta la representación interna de las reglas al formato esperado por cada versión de la API.
- **Patrón de diseño**: Facade — proporciona una interfaz simplificada y estable hacia el subsistema complejo de reglas.

---

## Escenario 3: Protección contra modificación no autorizada de resultados

### 7. RBAC y políticas de autorización centralizadas

**Aplicación en el proyecto**: Se implementa un sistema de control de acceso basado en roles (RBAC) que define qué acciones puede realizar cada tipo de usuario (estudiante, docente, administrador). Las políticas de autorización se centralizan en un servicio dedicado que evalúa cada solicitud contra las reglas de acceso antes de permitir la operación.

**Servicios C4 Nivel 3**:
- `authz-service`: Servicio centralizado de autorización que evalúa permisos. Expone `POST /authz/check` que recibe contexto (usuario, recurso, acción) y retorna permitido/denegado.
- `role-manager`: Gestiona la definición de roles y la asignación de permisos a cada rol. Permite a administradores configurar políticas sin modificar código.
- `policy-store`: Almacén de políticas de acceso que define qué roles pueden ejecutar qué acciones sobre qué recursos.

**Patrones asociados**:
- **Patrón arquitectónico**: Policy Decision Point (PDP) / Policy Enforcement Point (PEP) — separa la decisión de autorización de su aplicación en los servicios.
- **Patrón de diseño**: Chain of Responsibility — las políticas se evalúan en cadena hasta encontrar una que permita o deniegue explícitamente.
- **Patrón de diseño**: Strategy — permite intercambiar diferentes estrategias de autorización (RBAC, ABAC) sin modificar los servicios consumidores.

---

### 8. Auditoría inmutable

**Aplicación en el proyecto**: Toda operación crítica (modificación de calificaciones, cambios en resultados diagnósticos, alteración de rutas de aprendizaje) se registra en un log de auditoría inmutable. Este log se almacena en un sistema append-only que no permite modificaciones ni eliminaciones de registros existentes.

**Servicios C4 Nivel 3**:
- `audit-service`: Recibe eventos de auditoría desde otros servicios y los persiste en el almacén inmutable. Expone `POST /audit/events` para registrar eventos y `GET /audit/events/{resource-id}` para consultar el historial de un recurso.
- `audit-log-store`: Almacén append-only (por ejemplo, base de datos con tablas inmutables o sistema de logs estructurado como Elasticsearch con políticas de retención) que garantiza la inmutabilidad de los registros.
- `audit-integrity-checker`: Proceso periódico que verifica la integridad del log de auditoría mediante hashes encadenados (similar a blockchain) para detectar manipulaciones.

**Patrones asociados**:
- **Patrón arquitectónico**: Event Sourcing — el log de auditoría es esencialmente un stream de eventos inmutables.
- **Patrón de diseño**: Observer — los servicios publican eventos de auditoría que el servicio de auditoría consume de forma asíncrona.
- **Patrón de diseño**: Decorator — la capa de auditoría envuelve las operaciones críticas sin modificar su lógica principal.

---

### 9. Validación server-side obligatoria

**Aplicación en el proyecto**: Todas las reglas de negocio, restricciones de integridad y validaciones de datos se ejecutan en el backend, independientemente de las validaciones realizadas en el frontend. El servidor nunca confía en los datos recibidos del cliente y valida cada solicitud contra las reglas vigentes antes de procesarla.

**Servicios C4 Nivel 3**:
- `validation-service`: Servicio centralizado que expone reglas de validación reutilizables. Otros servicios lo invocan antes de procesar solicitudes críticas.
- `input-sanitizer`: Componente que limpia y normaliza los datos de entrada antes de la validación (prevención de inyección, XSS, etc.).
- `rule-validator`: Motor de validación que evalúa las solicitudes contra las reglas de negocio definidas (por ejemplo, "un estudiante no puede modificar su propia calificación").

**Patrones asociados**:
- **Patrón arquitectónico**: Layered Architecture — la validación se ubica en una capa específica antes de la capa de dominio.
- **Patrón de diseño**: Specification — encapsula las reglas de validación como objetos reutilizables y combinables.
- **Patrón de diseño**: Guard — patrón de validación temprana que rechaza solicitudes inválidas antes de que alcancen la lógica de negocio.

---

### 10. Control transaccional ACID

**Aplicación en el proyecto**: Las operaciones críticas que involucran múltiples entidades relacionadas (por ejemplo, calificar una evaluación implica actualizar la calificación, registrar el resultado en el historial del estudiante y actualizar su ruta de aprendizaje) se ejecutan dentro de una transacción ACID que garantiza atomicidad, consistencia, aislamiento y durabilidad.

**Servicios C4 Nivel 3**:
- `transaction-coordinator`: Gestiona las transacciones distribuidas cuando una operación afecta múltiples servicios o bases de datos. Implementa el patrón Saga para transacciones de larga duración.
- `consistency-checker`: Verifica la consistencia de los datos tras operaciones transaccionales y detecta anomalías.
- `transactional-store`: Base de datos relacional (por ejemplo, PostgreSQL) que soporta transacciones ACID para operaciones críticas.

**Patrones asociados**:
- **Patrón arquitectónico**: Unit of Work — agrupa múltiples operaciones en una única transacción que se confirma o revierte como un todo.
- **Patrón de diseño**: Saga — para transacciones distribuidas entre servicios, coordina una secuencia de transacciones locales con compensación en caso de fallo.
- **Patrón de diseño**: Command — encapsula la operación transaccional como un objeto que puede ejecutarse, revertirse y registrarse.

---

### 11. Principio de menor privilegio

**Aplicación en el proyecto**: Cada usuario y servicio recibe únicamente los permisos mínimos necesarios para realizar sus funciones. Los estudiantes solo pueden acceder a sus propias evaluaciones, los docentes a las evaluaciones de sus cursos, y los servicios internos solo a los endpoints que necesitan para su operación.

**Servicios C4 Nivel 3**:
- `least-privilege-enforcer`: Middleware que intercepta solicitudes y verifica que el solicitante tenga el mínimo privilegio necesario para la operación solicitada.
- `service-identity-manager`: Gestiona las identidades de los servicios internos y asigna permisos granulares a cada uno (por ejemplo, el servicio de recomendación solo tiene acceso de lectura a los resultados de evaluación).
- `privilege-auditor`: Monitorea y reporta permisos otorgados para detectar configuraciones que violen el principio de menor privilegio.

**Patrones asociados**:
- **Patrón arquitectónico**: Zero Trust Architecture — ningún usuario o servicio se considera confiable por defecto; cada solicitud se verifica.
- **Patrón de diseño**: Proxy — intercepta las solicitudes y aplica verificaciones de privilegio antes de permitir el acceso al recurso.
- **Patrón de diseño**: Middleware — la verificación de privilegios se implementa como middleware en la pipeline de procesamiento de solicitudes.

---

## Escenario 4: Respuesta rápida durante evaluaciones en línea

### 12. Escalamiento horizontal

**Aplicación en el proyecto**: El módulo de evaluación se despliega en múltiples instancias detrás de un balanceador de carga. Cuando la carga aumenta (por ejemplo, durante evaluaciones masivas), se añaden automáticamente nuevas instancias para distribuir la carga y mantener tiempos de respuesta bajos.

**Servicios C4 Nivel 3**:
- `evaluation-service` (múltiples instancias): Servicio principal de evaluación desplegado en múltiples nodos. Cada instancia procesa solicitudes de forma independiente.
- `load-balancer`: Balanceador de carga (por ejemplo, NGINX, HAProxy o un balanceador de nube) que distribuye las solicitudes entre las instancias disponibles.
- `auto-scaler`: Componente que monitorea métricas de carga (CPU, memoria, latencia) y escala automáticamente el número de instancias del servicio de evaluación.

**Patrones asociados**:
- **Patrón arquitectónico**: Stateless Services — los servicios de evaluación no mantienen estado local, permitiendo que cualquier instancia procese cualquier solicitud.
- **Patrón de diseño**: Load Balancer — distribuye la carga entre múltiples instancias del servicio.
- **Patrón de diseño**: Circuit Breaker — protege las instancias sobrecargadas dejando de enviarles solicitudes temporalmente.

---

### 13. Escritura asíncrona controlada

**Aplicación en el proyecto**: Cuando un estudiante envía una respuesta, el sistema la acepta inmediatamente y la coloca en una cola de procesamiento. La persistencia definitiva en la base de datos se realiza de forma asíncrona por un worker que consume la cola. El estudiante recibe confirmación inmediata sin esperar la escritura completa.

**Servicios C4 Nivel 3**:
- `async-write-service`: Recibe las respuestas de los estudiantes y las publica en la cola de mensajería. Retorna confirmación inmediata al cliente.
- `write-worker`: Proceso que consume mensajes de la cola y persiste las respuestas en la base de datos definitiva. Maneja reintentos y errores de forma controlada.
- `write-queue`: Cola de mensajería (por ejemplo, RabbitMQ, Apache Kafka o AWS SQS) que bufferiza las respuestas pendientes de persistencia.

**Patrones asociados**:
- **Patrón arquitectónico**: Message Queue — desacopla la recepción de la persistencia mediante una cola intermedia.
- **Patrón de diseño**: Producer-Consumer — el servicio de escritura produce mensajes y el worker los consume.
- **Patrón de diseño**: Fire and Forget (con confirmación) — el servicio acepta la solicitud y retorna inmediatamente, delegando el procesamiento asíncrono.

---

### 14. Uso de colas de mensajería

**Aplicación en el proyecto**: Se implementa una cola de mensajería como buffer entre la captura de respuestas de los estudiantes y su procesamiento/persistencia. La cola absorbe los picos de carga durante evaluaciones masivas, evitando que el sistema se sature y permitiendo un procesamiento ordenado y controlado.

**Servicios C4 Nivel 3**:
- `message-queue-broker`: Broker de mensajería (por ejemplo, RabbitMQ, Apache Kafka) que gestiona las colas, el enrutamiento de mensajes y la garantía de entrega.
- `queue-monitor`: Monitorea el estado de las colas (profundidad, tasa de procesamiento, mensajes fallidos) y alerta cuando se superan umbrales críticos.
- `dead-letter-handler`: Procesa mensajes que no pudieron ser consumidos tras múltiples intentos, registrándolos para análisis y re-procesamiento manual.

**Patrones asociados**:
- **Patrón arquitectónico**: Message Broker — centraliza la gestión de colas y el enrutamiento de mensajes entre servicios.
- **Patrón de diseño**: Publisher-Subscriber — múltiples servicios pueden suscribirse a los eventos de la cola para procesamiento paralelo.
- **Patrón de diseño**: Competing Consumers — múltiples workers compiten por consumir mensajes de la cola, permitiendo procesamiento paralelo.

---

### 15. CQRS (Command Query Responsibility Segregation)

**Aplicación en el proyecto**: Se separa el modelo de escritura (comandos: enviar respuestas, calificar evaluaciones) del modelo de lectura (consultas: visualizar preguntas, consultar resultados). Cada modelo se optimiza de forma independiente: el modelo de escritura se enfoca en consistencia y rendimiento de inserción, mientras que el modelo de lectura se optimiza para consultas rápidas y escalables.

**Servicios C4 Nivel 3**:
- `command-service`: Maneja todas las operaciones de escritura (comandos). Expone endpoints `POST` para enviar respuestas, calificar evaluaciones, etc. Escribe en el `write-store`.
- `query-service`: Maneja todas las operaciones de lectura (consultas). Expone endpoints `GET` para visualizar preguntas, consultar resultados, etc. Lee del `read-store`.
- `read-store`: Base de datos optimizada para lectura (por ejemplo, Elasticsearch o una base de datos con índices denormalizados) que mantiene una vista actualizada de los datos.
- `write-store`: Base de datos optimizada para escritura (por ejemplo, PostgreSQL) que mantiene la fuente de verdad de los datos.
- `event-publisher`: Publica eventos tras cada operación de escritura para que el `read-store` se actualice de forma asíncrona.

**Patrones asociados**:
- **Patrón arquitectónico**: CQRS — separación explícita de modelos de lectura y escritura.
- **Patrón arquitectónico**: Event Sourcing (opcional) — los comandos se almacenan como eventos que generan el estado actual, facilitando la reconstrucción de vistas de lectura.
- **Patrón de diseño**: Mediator — el servicio de comandos y consultas se comunican mediante un mediator que despacha las solicitudes al manejador apropiado.
- **Patrón de diseño**: Materialized View — el `read-store` mantiene vistas materializadas actualizadas desde los eventos de escritura.

---

## Resumen de patrones arquitectónicos por escenario

| Escenario | Tácticas | Patrones arquitectónicos principales |
|-----------|----------|-------------------------------------|
| 1 - Envío sin finalizar | Autosave, Persistencia temporal, Session recovery, Caché distribuida | Event Sourcing, Sidecar, Cache-Aside, Stateful Session |
| 2 - Modificación de reglas | Versionamiento, APIs desacopladas | Event Sourcing, API Gateway, Layered Architecture |
| 3 - Integridad de resultados | RBAC, Auditoría, Validación server-side, ACID, Menor privilegio | PDP/PEP, Event Sourcing, Zero Trust, Unit of Work, Saga |
| 4 - Respuesta rápida | Escalamiento horizontal, Escritura asíncrona, Colas, CQRS | Stateless Services, Message Queue, CQRS, Publisher-Subscriber |
