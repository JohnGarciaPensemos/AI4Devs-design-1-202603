## Descripción
El sistema es una plataforma B2B de reclutamiento en la que las organizaciones publican y gestionan vacantes con un embudo claro, historial completo y trazabilidad (incluso si el proceso falló o la vacante se cerró y se reabre o se clona desde una oferta anterior), mientras las personas candidatas participan sin coste, crean un perfil estable y pueden postular a muchas vacantes de una o varias empresas sin que un contrato previo en otro proceso les bloquee de forma arbitraria. Se diferencia de muchos ATS genéricos al centrarse en decisiones defendibles y en la equidad del proceso—datos comparables por vacante, etapas explícitas, notas internas separadas de la experiencia del candidato y un compromiso explícito de veracidad del lado candidato, pensado para multi‑país desde identidad, privacidad y operación diaria. Para las empresas, la ventaja es orden, continuidad y memoria institucional del reclutamiento (menos dependencia de hojas sueltas y correos), con capacidad de reutilizar y reactivar procesos pasados; para los candidatos, la ventaja es un camino claro y respetuoso—menos opacidad sobre el estado de su candidatura y un trato más homogéneo frente a la dispersión típica entre portales, llamadas y entrevistas sin registro común.

## Lean Canvas
[Diagrama](Lean-Canvas-ATS.md)


## Casos de uso principales
[Diagrama CU](Casos_de_uso_principales.puml)

### Gestionar el ciclo de vida de la vacante (Gestión de vacantes)
Qué es: el cliente define la oferta, la hace visible (o la pausa) y la cierra cuando corresponde. Es la condición previa para que existan candidaturas válidas (nota junto a UC9: la candidatura referencia una vacante publicada).

Casos de uso del diagrama: UC4 (definir y editar), UC5 (publicar o pausar), UC6 (cerrar).
Actor principal: Reclutador (también puede implicar al admin si concentra roles).
Relación UML destacada: UC5 `<<include>>` UC4 — no se publica sin una vacante definida.

### Postular y dar seguimiento como candidato (Candidatura)
Qué es: la persona descubre ofertas públicas, crea o completa su perfil de candidato, envía la candidatura y consulta el estado de sus procesos.

Casos de uso del diagrama: UC7 (explorar vacantes públicas), UC8 (registrar perfil), UC9 (enviar candidatura), UC10 (consultar estado).
Actor: Candidato.
Relación UML destacada: UC9 `<<include>>` UC2 — postular requiere identidad autenticada (autenticación compartida con el cliente vía UC2).

### Operar la selección y comunicar con el candidato (Selección y comunicación)
Qué es: el equipo cliente ve el embudo por vacante, filtra, mueve candidatos de etapa, deja notas o evaluaciones solo internas (el candidato no accede a UC14, según la nota del diagrama), marca resolución (contratado/descartado) y envía mensajes al candidato; las notificaciones salen hacia el actor Canal de notificaciones.

Casos de uso del diagrama: UC11 a UC16.
Actores: Reclutador y Responsable de contratación (HM) (con distinto alcance según RBAC; el diagrama deja la comunicación del HM como política opcional).
Relaciones UML destacadas: UC11 `<<include>>` UC4 (el embudo sigue las etapas de la vacante); UC15 `<<extend>>` UC16 (al resolver, opcionalmente notificar según política); UC16 y opcionalmente UC13 hacia Notif (`<<send>>`).

## Modelo de datos
[Diagrama ER](modelo-datos-ats-er.puml)

## Diseño del sistema a alto nivel
[Diagrama de arquitectura](diagrama_de_arquitectura.png)

El archivo describe la arquitectura del sistema en microservicios sobre Oracle Cloud Infrastructure (OCI), orientada a un perfil de “moderate scaling”: mucha lógica de negocio separada en servicios, un punto de entrada único para la API y una base de datos compartida para simplificar operación y consistencia transaccional en esta fase.

Flujo general (de izquierda a derecha)
Frontend (caja azul) — Una SPA React/Vue y/o app móvil es lo que usan candidatos y usuarios de la empresa. El cliente no llama a cada microservicio por su cuenta: habla con la plataforma vía HTTPS.
OCI Edge (caja naranja) — Capa perimetral:
OCI CDN sirve assets estáticos (JS, CSS, imágenes) con baja latencia.
OCI Load Balancer recibe las llamadas a la API, reparte carga y suele ser donde termina TLS y a veces reglas básicas de seguridad.
Microservicios (caja verde) — Tras el balanceador, el tráfico entra a un API Gateway (REST/GraphQL), que enruta hacia el dominio correcto. Es el patrón típico “un solo borde API, muchos servicios detrás”.
Autonomous DB (caja morada) — Todos los servicios del núcleo están conectados a una OCI Autonomous Database compartida (“Shared DB”). Eso acelera consultas cruzadas y transacciones, a costa de acoplamiento en esquema y gobernanza de datos: conviene delimitar propietario por tabla/esquema aunque la instancia sea una.
Integraciones de terceros (caja dorada) — Proveedores externos conectados con líneas discontinuas: correo, SMS, calendarios, portales de empleo, pruebas, videollamada, HRIS, SSO, firma electrónica, verificación de antecedentes, referencias y Object Storage para archivos.
En conjunto, el diagrama dice: experiencia web/móvil → borde OCI → API unificada → dominios de negocio → persistencia central → mundo exterior vía adaptadores.

Los microservicios del núcleo (qué problema resuelve cada uno)
Vacancy Lifecycle Management Service — Ciclo de vida de la vacante (crear, editar, publicar, pausar, cerrar, reabrir/clonar según el producto). Se enlaza con job boards para publicación de ofertas.
Candidate Application & Tracking Service — Corazón del embudo de candidatos: candidaturas, etapas, avances y resoluciones. Concentra muchas integraciones operativas (comunicación transaccional, evaluaciones, referencias, firma de oferta, etc., según las líneas del diagrama).
Company & Recruiter Management Service — Organización, usuarios internos, roles y datos del “lado cliente”. Suele alimentar permisos y contexto multi‑tenant.
Internal Notes & Fairness Service — Notas y evaluaciones internas (no visibles al candidato) y apoyo a equidad del proceso; enlaces típicos a slots de entrevista, CVs y adjuntos (a menudo vía Object Storage) y coordinación con videollamada para entrevistas.
Audit & Traceability Service — Trazabilidad y cumplimiento: registro de acciones y eventos para reconstruir “quién hizo qué y cuándo” (historial de contratación, exportaciones, auditorías).
Identity & Privacy Service — Autenticación, autorización, políticas de privacidad, consentimientos y federación con SSO corporativo (SAML/OIDC).
La idea del reparto es aislar cambios frecuentes (vacantes vs. candidaturas vs. notas) y centralizar preocupaciones transversales (identidad, auditoría).

## Diagrama C4
[Diagrama](arquitectura-c4.puml)

El archivo PlantUML contiene tres niveles del modelo C4 alineados con el diagrama OCI:

Nivel 1 — Contexto: el ATS y sus actores (candidatos y usuarios cliente) frente a los terceros (email, calendario, HRIS, background check, assessments, video, e‑signature, SSO, job boards).
Nivel 2 — Contenedores: SPA + CDN + Load Balancer + API Gateway frente a los microservicios de dominio (Vacancy Lifecycle, Candidate Application & Tracking, Company & Recruiter Mgmt, Identity & Privacy, Internal Notes & Fairness, Audit & Traceability) y la capa de Integrations / Adapters, todos sobre Autonomous DB + Object Storage + Cache/Bus.
Nivel 3 — Componentes: zoom en el Candidate Application & Tracking Service.
/Pensemos/processes/Innovation/training/ia4devs/AI4Devs-design-1/arquitectura-c4.puml

Nivel 3 (foco) — Candidate Application & Tracking Service
Este servicio es el corazón operativo del ATS: convierte un “CV recibido” en un proceso vivo, auditable y respetuoso para ambos lados. Es donde más reglas de negocio, concurrencia y eventos confluyen, así que conviene describirlo con cuidado.

Responsabilidad y límites (qué hace y qué NO hace)
Hace: gestionar el ciclo de vida de la Candidatura dentro de una Vacante: alta, avance/retroceso de etapas, decisión final, documentos, comunicación al candidato y vista de “mi estado” para el candidato.
No hace:
Definir vacantes ni etapas (lo hace Vacancy Lifecycle; este servicio las lee).
Gestionar identidad/SSO o consentimientos (lo hace Identity & Privacy; este servicio los consulta).
Crear notas o rúbricas (lo hace Notes & Fairness; este servicio las lee para apoyar la decisión).
Hablar directo con terceros (delega en Integrations).
Componentes internos
API (controllers)

Application API — Endpoints públicos (candidato: postular) y privados (cliente: listar/ver). Aplica idempotencia y authz.
Pipeline API — Cambio de etapa y resolución; soporta operaciones masivas (mover varios candidatos).
Documents API — Subida y descarga vía URLs firmadas a Object Storage; antivirus asíncrono antes de marcar “limpio”.
Candidate Status API — Vista del candidato sobre su propio proceso (filtra notas internas y campos sensibles).
Application services (orquestadores de casos de uso)

Application Service — Crea la candidatura, valida vacante publicada, llama al Duplicate & Identity Resolver, asigna etapa inicial.
Pipeline Orchestrator — Aplica transiciones; consulta Stage Transition Policy; dispara eventos.
Decision Service — Marca contratado/descartado; condiciona efectos colaterales (HRIS sync, oferta a firma, comunicación al candidato).
Communication Coordinator — Selecciona plantilla y contexto; no envía, sino que pide a Integrations por evento.
Interview Scheduling — Persiste cita y dispara sincronización con calendario externo.
Duplicate & Identity Resolver — Resuelve persona ya existente vs. nueva; aplica política de re‑postulación.
Dominio (DDD)

Application Domain Model — Aggregate Candidatura con invariantes (no se puede saltar a “contratado” sin pasar por etapas previas si la política lo exige; no se puede mover en vacante cerrada salvo política de cierre administrativo).
Stage Transition Policy — Tabla/política de transiciones permitidas por vacante y por rol.
Consent Guard — Garantiza que para acciones sensibles (background check, transferencia internacional de datos) exista consentimiento vigente.
Persistencia

Application Repository — Estado actual de la candidatura.
Application Event Store — Append‑only de eventos (StageChanged, Hired, Rejected, CommunicationSent…). Es la base del “historial completo” del cliente y del derecho del candidato a saber.
Document Repository — Solo metadatos; binarios en Object Storage.
Mensajería (event‑driven)

Domain Event Publisher — Patrón outbox para publicar de forma fiable a un bus (Kafka / OCI Streaming).
Event Consumer — Reacciona a eventos externos: BackgroundCheckCompleted, AssessmentCompleted, VacancyClosed, etc.
Cross‑cutting

Idempotency & Concurrency — Claves idempotentes para POSTs y locks optimistas (versionado) en transiciones para evitar dobles avances simultáneos.
Audit Hook — Cada caso de uso publica una traza estructurada hacia Audit & Traceability.
AuthZ Adapter — Resuelve permisos por rol (admin/reclutador/HM/candidato dueño) con datos de Identity & Privacy y Company & Recruiter Mgmt.
Flujos principales (cómo conviven los componentes)
1. Candidato postula a una vacante

Application API recibe POST /applications (con clave de idempotencia).
AuthZ valida candidato autenticado y vacante visible.
Application Service consulta vacante a Vacancy Lifecycle (estado y etapas).
Duplicate Resolver decide si reutilizar persona y si la candidatura es nueva o re‑postulación.
Application Repository persiste; Application Event Store registra ApplicationCreated.
Domain Event Publisher emite el evento; Notifications envía confirmación; Audit lo registra.
2. Reclutador mueve a “entrevista”

Pipeline API recibe PATCH /applications/{id}/stage.
Stage Transition Policy valida que la transición es legal en esa vacante y para ese rol.
Pipeline Orchestrator actualiza con lock optimista; emite StageChanged.
Communication Coordinator (si la política lo marca) prepara invitación; Interview Scheduling propone slots y delega calendario en Integrations.
3. Llega resultado de background check

Integrations recibe webhook del proveedor; transforma a evento BackgroundCheckCompleted.
Event Consumer lo aplica: si pasa, sigue avanzando; si no, marca etapa de revisión y notifica internamente sin exponer datos sensibles al candidato.
Todo queda en Application Event Store y se replica a Audit.
4. Decisión final (contratado)

Pipeline API → Decision Service.
Verifica Consent Guard para HRIS sync.
Emite Hired → Integrations dispara HRIS y oferta a firma; Communication Coordinator envía mensaje al candidato.
Aplica reglas de cierre (la vacante puede pasar a cerrada desde Vacancy Lifecycle según política, no desde aquí).
Datos y consistencia
Mantiene su propio subesquema sobre la Autonomous DB compartida (no “tablas de otros servicios”), aislado por organizacion_id.
Consistencia fuerte dentro del aggregate Candidatura; eventual entre servicios vía bus.
Outbox evita pérdidas: el evento se persiste en la misma transacción que el cambio de estado.
Para multi‑país, residencia de datos se decide a nivel de tenant (bucket regional en Object Storage; segmentación lógica en DB).
Seguridad y privacidad específicas
Tokens JWT validados en el API Gateway, scopes finos por endpoint.
Aislamiento estricto candidato vs. cliente: Candidate Status API jamás expone notas internas, motivos internos de descarte ni evaluaciones.
Consent Guard bloquea cualquier acción que requiera consentimiento ausente o caducado.
Borrado/exportación de datos del candidato son eventos de dominio que disparan limpieza coordinada en repos, Object Storage y Audit (con sello legal de cumplimiento).
Observabilidad y SLOs sugeridos
SLO de latencia para POST /applications (impacto directo en abandono): p95 < 500 ms.
SLO de éxito de eventos: < 0,1 % de mensajes en DLQ por día.
Métricas de negocio expuestas: tiempo en cada etapa, tasa de respuesta del cliente al candidato, abandono en formulario.