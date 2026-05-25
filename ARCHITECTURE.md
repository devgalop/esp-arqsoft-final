# Arquitectura MentorSoft

La arquitectura propuesta para el proyecto MentorSoft se describe a continuación:

## Frontend

El frontend de MentorSoft se desarrollará en Angular utilizando SPA (Single Page Application). Esta elección se debe a la capacidad de Angular para crear interfaces de usuario dinámicas y responsivas, lo que es esencial para proporcionar una experiencia de usuario fluida. El frontend se encargará de la interacción con el usuario, mostrando los resultados del análisis de desempeño y las recomendaciones personalizadas.

## Backend

El backend se implementará en python utilizando FastAPI. FastAPI es un framework moderno y de alto rendimiento que facilita la creación de APIs RESTful. El backend se encargará de procesar las solicitudes del frontend, realizar el análisis de desempeño de los estudiantes, generar recomendaciones personalizadas y gestionar la lógica de negocio del sistema.

## ADR

|ID|Decisión Arquitectónica|Tecnología / Patrón|Justificación Principal|Ventajas|Riesgos / Desventajas|
|---|---|---|---|---|---|
|ADR-001|Framework backend|FastAPI|Necesidad de APIs rápidas, asincronía y soporte multiusuario concurrente|Alta concurrencia, bajo tiempo de respuesta, integración natural con IA/LLM, documentación automática|Complejidad en manejo async y concurrencia|
|ADR-002|Base de datos principal|PostgreSQL|Persistencia confiable y soporte transaccional para múltiples usuarios|ACID, escalabilidad, soporte concurrente, extensiones IA como pgvector|Mayor complejidad operativa|
|ADR-003|ORM backend|SQLAlchemy|Abstracción de acceso a datos y mantenibilidad|Seguridad, portabilidad entre DBs, soporte async, integración con FastAPI|Requiere optimización de consultas|
|ADR-004|Frontend SPA|Angular|Necesidad de interfaz modular y dinámica|Experiencia fluida, modularidad, escalabilidad frontend|Curva de aprendizaje alta|
|ADR-005|Integración LLM|Groq|Uso de IA generativa con bajo costo inicial|Baja latencia, planes gratuitos, integración sencilla|Límites de cuota y dependencia externa|
|ADR-006|Estrategia de concurrencia|Arquitectura async + consistencia eventual|Soportar múltiples estudiantes simultáneos y tareas IA|Mayor throughput, resiliencia, desacoplamiento|Mayor complejidad arquitectónica|
|ADR-007|Arquitectura desacoplada|SPA + API REST|Separación de frontend y backend|Escalabilidad independiente, reutilización de APIs|Necesidad de manejar CORS y autenticación distribuida|
|ADR-008 (Opcional MVP)|Base de datos temporal para MVP|SQLite|Reducir complejidad y acelerar desarrollo inicial|Simplicidad, bajo costo, despliegue rápido|Baja concurrencia y limitada escalabilidad|

## Modulos principales

- **Gestion de usuarios**: Registro, autenticación y gestión de usuarios (Estudiantes, Docentes, Administradores).
- **Evaluaciones**: Registro de preguntas, rubricas de evaluacion, resultados de evaluaciones y análisis de desempeño.
- **Recomendaciones**: Generación de recomendaciones personalizadas basadas en el desempeño del estudiante.
- **Gestion de contenidos**: Administración de recursos educativos y materiales de aprendizaje.
- **Calificación y feedback**: Sistema de calificación automática y generación de feedback detallado para los estudiantes.
- **Seguimiento y reportes**: Seguimiento del progreso de los estudiantes y generación de reportes para docentes y administradores.
