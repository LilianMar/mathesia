# Quiz USB Cali v2 — Guía de construcción desde cero

**Fecha:** 28 de septiembre de 2026
**Revisado:** 9 de octubre de 2026 — *ampliar tiempo de una evaluación en curso,
preguntas de bonificación y quices en pareja (§7.2–§7.4).*
**Sustituye a:** `ARQUITECTURA_V2.md` como documento de trabajo. Aquel queda como
registro histórico y fuente de las lecciones de la v1 (§14 resume las que se heredan).
**Stack:** Node.js + TypeScript (API) · React + Vite + TypeScript (web) · PostgreSQL 16 · Docker Compose
**Arranque:** la v2 empieza **limpia**. No se migran datos de la v1. La v1 queda como archivo de consulta.

---

## 0. Cómo leer este documento

- **§1** son las decisiones cerradas. No se reabren durante la construcción.
- **§2–§9** son la arquitectura: qué existe y qué reglas no se pueden romper.
- **§10** es el mapa de rutas con su protección, la fuente de verdad de la seguridad.
- **§12** es el plan de construcción por etapas. Cada etapa termina con algo usable.
- Las reglas marcadas con **🔒** son invariantes de seguridad. Una PR que rompa una de ellas no entra, aunque los tests pasen.

---

## 1. Decisiones

### 1.1 Adoptadas

| Área | Decisión |
|---|---|
| Lenguaje | **TypeScript en todo el monorepo**, en modo `strict`. |
| Sesiones | **En Postgres**, con cookie `httpOnly`. Web y API en **el mismo dominio** (nginx sirve la web y reenvía `/api`). Sin CORS. |
| Trabajos en segundo plano | **Cola sobre Postgres** (`pg-boss`). Sin Redis. |
| Modelo académico | **Materia → Periodo → Curso → Grupo → Matrícula** como entidades separadas (§6.2). |
| Temas | **Tabla `topics`** compartida por preguntas, bitácora, práctica, insignias y ayudas. |
| Borrado | **Lógico** para usuarios, cursos y bancos. Las notas nunca se borran; se revisan (`grade_revisions`). |
| Evaluaciones | **Máquina de estados explícita**: borrador → programada → abierta → cerrada → publicada (§7). |
| Seguridad | Segundo factor opcional para docentes y administradores. Reautenticación en acciones sensibles. Retrasos progresivos en vez de bloqueo duro. Contraseña más exigente para docentes. Aviso al estudiante cuando su contraseña se cambió por código. |
| Panel docente | **Organizado por curso, con Inicio como bandeja de pendientes**, conservando las áreas globales del plan anterior que tienen sentido fuera de un curso (Biblioteca, Análisis). Ver §9. |
| Capturas y copia | **Solo web**: bloqueo de copia y selección, y **marca de agua visible** con nombre y código del estudiante, además de la invisible. Sin app nativa (§8). |
| Bitácora | **La ven los estudiantes.** Los pendientes tienen visibilidad: solo docente, o docente y estudiantes (§5). |
| Alcance de evaluación | Una evaluación pertenece a **un grupo** o a **todo el curso** (todos sus grupos). |
| Ampliar tiempo *(9/10)* | Desde el monitor (docente) y desde el panel de administración, a **toda la evaluación** o a **un intento**. El administrador puede ampliar tiempo, pero no ver respuestas ni notas (§7.2). |
| Bonificación *(9/10)* | Preguntas opcionales al final, en cantidad y temas configurables. Cada una bien respondida **recupera** una pregunta fallada, sin superar la nota máxima (§7.3). |
| Quices en pareja *(9/10)* | Un intento compartido por dos estudiantes, con parejas asignadas por la docente o formadas con un código. Responde un dispositivo y el otro queda vigilado. La nota es la misma para ambos (§7.4). |
| Roles | Estudiante, docente y **administrador**, con **varios docentes desde el inicio**. |
| Datos de la v1 | **No se migran.** La v2 arranca limpia. |
| Diseño visual | Se conserva la identidad de `arcade.css` (colores, tipografía, tono), llevada a tokens. Minimalista: cada pantalla responde una pregunta. |

### 1.1b Ajustes confirmados con la docente (9/10) — **prevalecen sobre el resto del documento**

Donde una sección posterior diga otra cosa, manda esta tabla. Las secciones afectadas llevan la marca *(ver §1.1b)*.

| # | Área | Decisión |
|---|---|---|
| A1 | **Alta de docentes** | **Autorregistro con código de invitación.** El administrador genera un código de un solo uso (token aleatorio, caduca, se guarda solo su hash). El docente se registra con ese código; al usarlo queda consumido. Contraseña de docente: mínimo 12 (§4.6). |
| A2 | **Alta de estudiantes** | Autorregistro con **código de estudiante, nombre, número de identificación (cédula), correo**. El código de estudiante y la cédula se escriben **dos veces, sin pegar**. Ambos son únicos. |
| A3 | **Recuperar la cuenta** | Vía principal: **correo con un código** de restablecimiento enviado con **Resend** (`RESEND_API_KEY`). Vías secundarias: **cédula + código de estudiante** (activable/desactivable por configuración) y **código de un solo uso generado por la docente**. Sustituye a la vía «enlace por SMTP». |
| A4 | **Administrador** | Ve **todos los cursos, bancos de preguntas y evaluaciones** de todos los docentes (lectura y gestión). **No ve información de estudiantes ni notas**: ni listas, ni intentos, ni respuestas, ni resultados, ni eventos de integridad. Sustituye a «el administrador no es un superdocente» (§4.1). |
| A5 | **Cursos** | **El docente crea sus cursos** y dentro de ellos los grupos, y gestiona todo lo demás del curso. El administrador ya no crea cursos ni asigna titulares. |
| A6 | **Retiro** | Solo el docente retira estudiantes de su curso. El estudiante no puede salirse por su cuenta. |
| A7 | **Bancos** | Son **del docente** y los usa en todos sus cursos y periodos. No son del administrador ni de un curso. |
| A8 | **Corregir la respuesta correcta** | Editar una pregunta solo afecta a evaluaciones futuras (lo presentado queda congelado), **excepto** si se corrige la **respuesta correcta**: se trata como corrección de un error y se aplica a las evaluaciones en curso, recalculando los intentos afectados con su registro en `grade_revisions`. |
| A9 | **Reglas de selección** | Una regla puede filtrar por **varios temas** (casillas) y **una o más dificultades** a la vez, además de «todos». Ver §7.0. |
| A10 | **Pareja** | Se elige en la creación si la evaluación es individual o en pareja. En pareja, se escoge el compañero de una **lista desplegable con los estudiantes matriculados en el curso**. Solo queda activo el perfil de quien inicia; **el perfil del compañero queda inactivo mientras dura la presentación**. La evaluación, la nota y la retroalimentación quedan en el perfil de los dos. |
| A11 | **Presencialidad** | Si la evaluación es presencial y el estudiante no está presente en la asistencia de la clase, su perfil queda inhabilitado para presentarla. Las evaluaciones en casa no lo exigen. |
| A12 | **Presentación del examen** | Se porta **el motor de la v1** (`quiz_app/public/js/pages/quiz.js` y `attempt.service.js`), que ya funciona bien en producción: antitrampas (incluida la congelación por pantalla dividida y la pantalla compartida), vidas, latido con secuencia, retroalimentación y vista de resultados. |
| A13 | **Reporte de notas del estudiante** | Además de su nota, el estudiante ve el **desglose** de cada evaluación y un **reporte de notas** del curso. |
| A14 | **Bonificación** | Se configura al crear la evaluación (§7.3, sin cambios). |
| A15 | **Ampliar tiempo** | Lo tienen **todos los docentes** del curso sobre sus evaluaciones en curso (toda la evaluación o un intento). El administrador **también** la conserva, solo sobre la evaluación completa (no ve estudiantes, A4). |
| A16 | **Pareja: quién la forma** | **El estudiante** que inicia elige a su compañero en un desplegable con los estudiantes matriculados en el curso (cualquier grupo del curso). La docente no arma parejas. Desaparecen la formación «asignada» y el código de 4 dígitos de §7.4. |
| A17 | **Pareja: cuenta del compañero** | Mientras dura el intento, la cuenta del compañero queda **inactiva para toda la plataforma**: cualquier pantalla muestra «Estás presentando *Quiz 3* en pareja con *Nombre*» y solo permite cerrar sesión. Se reactiva al entregar o vencer el intento. Sustituye al «acompañante vigilado» de §7.4. |
| A18 | **Corrección de la clave: alcance** | Se aplica a evaluaciones **abiertas y cerradas** (recalcula y deja `grade_revisions`). En las **publicadas** no se aplica sola. |
| A19 | **Materia y periodo** | Los escribe **el docente al crear el curso**. Se reutiliza la materia o el periodo si ya existe con el mismo nombre normalizado (sin tildes ni mayúsculas); si no, se crea. El periodo tiene formato `AAAA-N`. |
| A20 | **Temas** | Los temas son **los del banco de preguntas**: pertenecen al banco (y por tanto al docente), se crean al escribir o importar preguntas, y no hay catálogo por materia. La bitácora de clase y la bonificación eligen entre los temas de los bancos del docente. Sustituye a «los temas son de la materia» (§6.3). |
| A21 | **Varios intentos** | Nota configurable: **mejor nota** o **promedio**. Se fija en la evaluación para todo el grupo y se puede **cambiar para un estudiante concreto**. |
| A22 | **Grupos** | Un estudiante está en **un solo grupo por curso** (confirmado). Puede estar en varios cursos. |
| A23 | **Varios docentes y bancos** | Un curso puede tener **varios docentes**. Cada docente usa **sus** bancos en cualquier curso donde enseñe, sin importar la materia. |
| A24 | **Pareja: confirmación en dos pasos** | El compañero tiene que **aceptar desde su propio dispositivo**, con su sesión iniciada. Al elegirlo, su pantalla muestra «*Nombre* quiere presentar *Quiz 3* contigo» con un **código de 6 dígitos** y un botón **Rechazar**. Quien inicia escribe ese código; si coincide, empieza el intento y la cuenta del compañero queda inactiva (A17). El código dura 2 minutos, admite 3 intentos fallidos (después la solicitud se cancela) y queda en la auditoría. Nadie puede inactivar la cuenta de otro sin que esté presente y con su sesión abierta. |

### 1.2 No adoptadas (y su riesgo, para que quede escrito)

| Recomendación | Riesgo que queda abierto |
|---|---|
| Almacenamiento de archivos intercambiable (disco / S3) | Los archivos viven en un volumen de Docker del mismo servidor. Si el servidor se pierde, se pierden. Si algún día hay dos instancias de la API, no comparten archivos. **Mitigación mínima:** todo acceso a archivos pasa por un único módulo `platform/files.ts`, así cambiarlo después es tocar un archivo y no cien. |
| Copias de seguridad diarias con prueba de restauración | **Es el riesgo más grave de este documento.** La base contendrá notas reales y no hay forma de recuperarlas si el volumen se corrompe o se borra por error. Se recomienda reconsiderarlo antes de la primera evaluación calificada (§13, decisión pendiente **P1**). |
| Dominio propio para la aplicación *(9/10)* | Se despliega bajo un prefijo, `https://lidis.usbcali.edu.co/mathesia/`, que comparte origen con las demás aplicaciones de `lidis`. Un XSS en cualquiera de ellas sería del mismo origen que mathesia y podría llamar a su API con la sesión del usuario. La cookie pasa a ser `__Secure-sid` con `Path=/mathesia/` (`__Host-` exige `Path=/`). **Mitigación:** CSRF ligado a sesión, CSP estricta, reautenticación en acciones sensibles. La app admite servirse en la raíz si algún día hay subdominio (`PUBLIC_URL`). |

---

## 2. Stack y estructura del repositorio

### 2.1 Dependencias elegidas

| Capa | Herramienta | Por qué esta |
|---|---|---|
| Monorepo | `pnpm` workspaces | Simple, sin herramientas de build adicionales. |
| API | Express 5 | Conocido, y los manejadores async ya propagan errores. |
| SQL | **Kysely** | SQL tipado sin ORM: el SQL sigue siendo SQL, pero una columna renombrada rompe la compilación. |
| Migraciones | Migraciones de Kysely, en archivos numerados | Una sola herramienta para consultas y esquema. |
| Validación / contratos | **Zod** en `packages/contracts` | El mismo esquema valida en el servidor y tipa en el cliente. |
| OpenAPI | Generado **desde Zod** (`zod-to-openapi`) | No se escribe a mano: un segundo contrato escrito a mano se desincroniza. |
| Cola | `pg-boss` | Trabajos, reintentos y programados sobre la misma base. |
| Contraseñas | `argon2` (Argon2id) | Igual que la v1. |
| Segundo factor | `otplib` (TOTP) | Estándar, funciona con cualquier app autenticadora. |
| Web | React 19 + Vite + React Router | — |
| Estado del servidor en la web | **TanStack Query** | Caché, reintentos y sondeo (`refetchInterval`) sin Redux. |
| Tests | Vitest (unidad e integración), Supertest (HTTP), Playwright (flujos de punta a punta) | — |
| Logs | `pino`, en JSON | Un log por petición con `requestId` y `actorId`. |

### 2.2 Estructura

```
quiz-v2/
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── http/           Rutas. Declaran auth + política, validan con contracts, llaman a UN servicio.
│   │   │   ├── services/       Casos de uso. Orquestan y abren la transacción. Sin SQL, sin req/res.
│   │   │   ├── domain/         Lógica pura: cálculo de notas, estados, reglas. Sin dependencias.
│   │   │   ├── policies/       🔒 Quién puede qué. Único lugar donde se decide acceso.
│   │   │   ├── repositories/   Único lugar con SQL. Toda consulta recibe al actor o un filtro de alcance.
│   │   │   ├── jobs/           Manejadores de pg-boss (abrir/cerrar evaluaciones, calificación, etc.).
│   │   │   ├── db/             Conexión, migraciones, semilla de desarrollo.
│   │   │   └── platform/       Sesiones, CSRF, límites, archivos, correo, logs, errores, config.
│   │   └── tests/
│   │       ├── security/       Matriz de acceso cruzado (§11.2) y registro de rutas (§11.1).
│   │       ├── integration/
│   │       └── domain/
│   ├── worker/                 Proceso aparte que consume la cola. Mismo código que api/, otro punto de entrada.
│   └── web/
│       └── src/
│           ├── app/            Enrutado, layouts por rol, guardas de navegación.
│           ├── features/       auth, courses, classes, calendar, banks, assessments, exam, grades, admin…
│           ├── shared/         Componentes, cliente de API tipado, hooks.
│           └── design/         Tokens y estilos base.
├── packages/
│   └── contracts/              Esquemas Zod de entrada y salida de cada endpoint. Fuente única.
├── docker/
│   └── nginx.conf              Sirve web/ y reenvía /api a la API. Un solo origen.
└── docker-compose.yml          postgres · api · worker · web(nginx)
```

### 2.3 Configuración

Toda variable de entorno se valida con Zod **al arrancar**. Si falta una obligatoria, el proceso no arranca y dice cuál falta.

```bash
# .env.example — sin valores reales
DATABASE_URL=postgres://quiz:quiz@postgres:5432/quiz
SESSION_SECRET=              # 32+ bytes aleatorios
TOTP_ENCRYPTION_KEY=         # 32 bytes en base64; cifra los secretos TOTP en la base
FILES_DIR=/data/files        # volumen de Docker
PUBLIC_URL=https://quiz.example.edu.co

# Opcionales: si faltan, la función se oculta sola (regla heredada de la v1)
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
MAIL_FROM=
ANTHROPIC_API_KEY=           # calificación por fotos (etapa 7)
```

**Regla:** una función que depende de configuración opcional **no aparece en la interfaz** mientras esa configuración no exista. La API expone `GET /api/v1/features` con lo que está activo.

---

## 3. Arquitectura del backend

### 3.1 Capas y sus reglas

| Capa | Puede | No puede |
|---|---|---|
| `http/` | Declarar auth y política, validar con `contracts`, llamar **un** servicio, dar forma a la respuesta | Tocar repositorios, tener lógica de negocio |
| `services/` | Abrir transacciones, llamar repositorios, políticas, dominio y encolar trabajos | Leer `req`/`res`, escribir SQL |
| `domain/` | Calcular | Importar nada con efectos (BD, red, reloj: el reloj entra como parámetro) |
| `policies/` | Decidir si un actor puede hacer algo, leyendo con repositorios de alcance | Modificar datos |
| `repositories/` | SQL | Decidir permisos por su cuenta (pero sí **filtrar por alcance**, §4.3) |

Si una ruta necesita dos servicios, falta un caso de uso.

### 3.2 Contexto de la petición

Cada petición autenticada construye un `Actor` una sola vez, en `platform/session.ts`:

```ts
export type Actor =
  | { kind: "student"; id: string; sessionId: string; reauthAt: Date | null }
  | { kind: "teacher"; id: string; sessionId: string; reauthAt: Date | null; isAdmin: boolean };
```

El `Actor` se pasa explícitamente a los servicios. No se lee de variables globales ni de `AsyncLocalStorage`: así un test puede construirlo sin levantar HTTP.

### 3.3 Transacciones

Con `pg`, una transacción es una conexión. Todo servicio que escribe en más de una tabla la abre, y todos los repositorios reciben el ejecutor:

```ts
// services/assessments/close.ts
export async function closeAssessment(actor: Actor, assessmentId: string) {
  return db.transaction().execute(async (tx) => {
    const a = await assessmentsRepo.findForTeacher(tx, actor.id, assessmentId); // 🔒 alcance
    if (!a) throw notFound();
    assertTransition(a.state, "cerrada");                    // domain/assessment/states.ts
    await attemptsRepo.autoSubmitOpen(tx, a.id);
    await assessmentsRepo.setState(tx, a.id, "cerrada");
    await auditRepo.log(tx, actor, "assessment.close", a.id);
    await jobs.enqueue(tx, "assessment.closed", { assessmentId: a.id }); // se encola DENTRO de la transacción
  });
}
```

`pg-boss` acepta encolar con la misma conexión. Si la transacción falla, el trabajo no existe. Así no quedan trabajos huérfanos ni cierres sin trabajo.

### 3.4 Errores

Hay un tipo `AppError` con código estable (`NOT_FOUND`, `FORBIDDEN_ACTION`, `INVALID_STATE`, `VALIDATION`, `REAUTH_REQUIRED`, `RATE_LIMITED`…). El manejador global lo traduce a HTTP y **nunca** devuelve una traza al cliente.

La web detecta «HTML donde esperaba JSON» y lo dice con esas palabras: el WAF de la universidad ya lo hizo en la v1.

### 3.5 Trabajos en segundo plano (`apps/worker`)

| Trabajo | Disparo | Qué hace |
|---|---|---|
| `assessment.open` | Programado a `opens_at` | programada → abierta |
| `assessment.close` | Programado a `closes_at` | abierta → cerrada, entrega los intentos abiertos |
| `assessment.closed` | Al cerrar | Calcula notas automáticas y (etapa 8) reparte XP |
| `attempt.unfreeze` | 10 min después de congelar | Reanuda el intento si nadie lo liberó |
| `grading.photo` | Botón de la docente (etapa 7) | Llama a la API de Claude, guarda observaciones |
| `sessions.cleanup` | Diario | Borra sesiones vencidas |

Todos los manejadores son **idempotentes**: si se ejecutan dos veces, el resultado es el mismo.

---

## 4. Identidad, sesiones y acceso 🔒

### 4.1 Roles

| Rol | Cómo se obtiene | Qué hace |
|---|---|---|
| **Estudiante** | Se registra solo *(ver §1.1b A2)* | Ve los cursos y evaluaciones de los grupos en que está matriculado |
| **Docente** | Se autorregistra con un código de invitación del administrador *(ver §1.1b A1)* | Crea y gestiona sus cursos *(A5)* |
| **Administrador** | Marca `is_admin` sobre una cuenta docente | Gestiona usuarios, materias y periodos; asigna docentes a cursos; ve la auditoría |

*(Reemplazado por §1.1b A4.)* **El administrador no es un superdocente.** No ve ni edita notas ni evaluaciones de cursos que no dicta. Si necesita entrar a un curso para dar soporte, se asigna como docente auxiliar y queda en la auditoría. Un rol con acceso invisible a todo es exactamente lo que no debe existir.

*(Revisar contra §1.1b A4 y A15.)* **Única excepción operativa: ampliar tiempo** (§7.2). El administrador ve la lista de evaluaciones **abiertas en este momento** (curso, título, intentos en curso, hora de cierre) y puede sumarles tiempo. No ve preguntas, respuestas, eventos ni notas. Exige reautenticación y motivo, queda en la auditoría, y la docente del curso lo ve en su Inicio. La excepción existe porque una caída del servidor en mitad de un parcial es urgente y la docente puede estar sin acceso al panel.

### 4.2 Sesiones

```sql
CREATE TABLE sessions (
  id            TEXT PRIMARY KEY,          -- SHA-256 del token; el token solo vive en la cookie
  user_id       UUID NOT NULL REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at    TIMESTAMPTZ NOT NULL,      -- vencimiento absoluto
  reauth_at     TIMESTAMPTZ,               -- última vez que confirmó contraseña/TOTP
  ip            INET,
  user_agent    TEXT,
  revoked_at    TIMESTAMPTZ
);
CREATE INDEX idx_sessions_user ON sessions(user_id) WHERE revoked_at IS NULL;
```

| | Estudiante | Docente / admin |
|---|---|---|
| Inactividad | 8 h | **2 h** |
| Vencimiento absoluto | 30 días | **12 h** |
| Durante un examen | El latido renueva `last_seen_at`: **la sesión no vence con un intento abierto** | — |
| Reautenticación | No aplica | Válida por 10 min (§4.5) |

La cookie es `__Secure-sid`, con `HttpOnly; Secure; SameSite=Lax; Path=/mathesia/` (la app vive bajo ese prefijo, §1.2). Si algún día se sirve en la raíz de un dominio propio, pasa a `__Host-sid` con `Path=/`; ambos salen de `PUBLIC_URL`.

**CSRF:** con `SameSite=Lax` y un solo origen, las peticiones cruzadas ya no llevan la cookie. Aun así, toda petición que cambia estado exige la cabecera `X-CSRF-Token`, ligada a la sesión. Son dos defensas independientes, a propósito.

En su perfil, **cada usuario ve sus sesiones activas y puede cerrarlas.** Cambiar la contraseña cierra todas las demás.

### 4.3 Reglas de acceso — las que no se negocian

1. **Denegar por defecto.** Toda ruta declara `auth` (`public`, `student`, `teacher`, `admin`, o una combinación) y una política. Una ruta sin declaración **no arranca**: el registro lanza error al iniciar (§11.1).
2. **El acceso se deriva de la matrícula o de la asignación docente, nunca del ID que llega en la URL.**
3. **Consultas con alcance.** Los repositorios exponen variantes por actor. No existe `findAssessment(id)` accesible desde servicios de estudiante o docente:

   ```ts
   // repositories/assessments.ts
   findForStudent(tx, studentId, assessmentId)   // solo si está matriculado en su grupo/curso y es visible
   findForTeacher(tx, teacherId, assessmentId)   // solo si dicta el curso
   ```

   ```sql
   -- findForStudent
   SELECT a.*
   FROM assessments a
   JOIN sections    s ON s.course_id = a.course_id
   JOIN enrollments e ON e.section_id = s.id
   WHERE a.id = $2
     AND e.student_id = $1
     AND e.status = 'active'
     AND (a.section_id IS NULL OR a.section_id = s.id)   -- evaluación de todo el curso, o de su grupo
     AND a.state IN ('abierta','cerrada','publicada')
     AND a.deleted_at IS NULL;

   -- findForTeacher
   SELECT a.*
   FROM assessments a
   JOIN course_teachers ct ON ct.course_id = a.course_id
   WHERE a.id = $2 AND ct.user_id = $1 AND ct.removed_at IS NULL;
   ```

   Si un servicio olvida comprobar el permiso, la consulta igual no devuelve nada ajeno.
4. **Lo ajeno responde 404, nunca 403.** Un 403 confirma que el recurso existe. El 403 se reserva para «es tuyo, pero no puedes hacer esto ahora» (por ejemplo, editar una evaluación ya abierta).
5. **IDs expuestos como UUID.** No se pueden adivinar ni recorrer.
6. **Los archivos pasan por la misma política.** No hay carpeta pública: `GET /api/v1/files/:id` resuelve el dueño lógico (examen, material, foto) y aplica su política.
7. **Estudiante de otro grupo = ajeno.** Un estudiante del grupo A de Cálculo no ve una evaluación asignada solo al grupo B, ni siquiera por URL directa.
8. **Matrícula inactiva = sin acceso.** Retirarse del grupo corta el acceso a evaluaciones abiertas. Conserva, en solo lectura, sus notas ya publicadas.

### 4.4 Políticas

```ts
// policies/course.ts
export const teachesCourse = (param = "courseId"): Policy =>
  async (actor, req, tx) =>
    actor.kind === "teacher" &&
    (await courseTeachersRepo.isActive(tx, req.params[param], actor.id));

export const enrolledInCourse = (param = "courseId"): Policy =>
  async (actor, req, tx) =>
    actor.kind === "student" &&
    (await enrollmentsRepo.isActiveInCourse(tx, actor.id, req.params[param]));

// http/courses.ts
route({
  method: "GET",
  path: "/courses/:courseId/assessments",
  auth: ["teacher", "student"],
  policy: anyOf(teachesCourse(), enrolledInCourse()),
  input: contracts.assessments.listQuery,
  output: contracts.assessments.list,
  handler: (actor, input) => assessmentsService.listForCourse(actor, input),
});
```

La política de la ruta es la **primera** barrera. La consulta con alcance del repositorio es la **segunda**. Las dos existen a propósito.

### 4.5 Reautenticación

Acciones que exigen haber confirmado la contraseña (o el TOTP) en los últimos 10 minutos:

- publicar notas, y editar una nota ya publicada;
- cambiar el correo o el código de un estudiante;
- borrar (lógicamente) un curso, grupo, banco o evaluación con intentos;
- importar la lista oficial de un grupo;
- todo el panel de administración;
- activar o desactivar el propio segundo factor.

La API responde `REAUTH_REQUIRED`. La web muestra un diálogo de contraseña **sin perder lo que se estaba haciendo** y reintenta.

### 4.6 Contraseñas, intentos y bloqueos

| | Estudiante | Docente / admin |
|---|---|---|
| Longitud mínima | 6 | **12** |
| Lista de prohibidas | Sí, **incluidas las cortas** (`123456`, `qwerty`, `password`…) | Sí |
| Secuencias de teclado o alfabeto | Rechazadas en cualquier tramo, en ambas direcciones | Igual |
| No puede contener | Su nombre, su correo, su código | Igual |
| Segundo factor (TOTP) | No | **Opcional**, con 10 códigos de respaldo |

**Intentos fallidos: retraso progresivo, no bloqueo.** Un bloqueo duro por cuenta permite que un compañero bloquee tu cuenta la noche antes del parcial.

```sql
CREATE TABLE login_throttle (
  key          TEXT PRIMARY KEY,     -- 'acct:<userId>:<ip>'  o  'ip:<ip>'
  failures     INTEGER NOT NULL DEFAULT 0,
  next_try_at  TIMESTAMPTZ,
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

- **Por cuenta + IP:** tras 5 fallos, espera de 2ⁿ⁻⁵ segundos (2, 4, 8…), con tope de 15 min. Un atacante desde su IP se frena; el dueño desde otra IP entra normal.
- **Por IP sola:** 300 intentos en 15 min. Es alto a propósito: el campus comparte una IP entre muchos estudiantes (lección del 8/09 en la v1).
- **Vive en Postgres, no en memoria.** Sobrevive a reinicios y sirve con varias instancias.
- Todo intento fallido va a `auth_events`.

La misma tabla limita los restablecimientos (§4.7).

### 4.7 Recuperar la cuenta — tres vías

| Vía | Requiere | Siempre disponible |
|---|---|---|
| *(Reemplazado por §1.1b A3.)* **Correo institucional + código de estudiante** | `student_code` cargado | Sí |
| **Código de un solo uso generado por la docente** | Que la docente lo entregue | Sí |
| **Enlace al correo** | `SMTP_HOST` configurado | **Se oculta sola si no hay SMTP** |

Reglas comunes:

- La respuesta es idéntica exista o no la cuenta.
- Restablecer **cierra todas las sesiones** del usuario.
- `users.password_reset_notice` guarda la vía y la fecha. En el siguiente ingreso el estudiante ve: *«Tu contraseña se cambió el 3 de octubre a las 22:14 usando tu código de estudiante. Si no fuiste tú, avisa a tu docente.»* Esto hace **visible** una toma de cuenta por un compañero que conoce su código.
- Cada uso queda en `audit_log`.
- La interfaz tiene **un solo** enlace, *¿Olvidaste tu contraseña?*, que lleva a una pantalla con las vías disponibles. No hay tres botones.

---

## 5. Bitácora de clase y calendario

### 5.1 Qué resuelve

La pregunta al abrir un curso es *«¿en qué quedamos la clase pasada y qué viene?»*. Hoy esa respuesta vive en un cuaderno. En la v2 es la primera pestaña del curso, y los estudiantes ven la misma línea de tiempo.

### 5.2 Modelo

Una **sesión de clase** es la unidad: en ella se pasa lista **y** se registra qué se vio. No son dos pantallas.

```sql
CREATE TABLE class_sessions (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  section_id     UUID NOT NULL REFERENCES sections(id),
  held_on        DATE NOT NULL,
  starts_at      TIME,
  summary        TEXT,          -- «Lo que vimos hoy». Visible a estudiantes. Markdown + LaTeX.
  teacher_notes  TEXT,          -- Solo docente. «El grupo se enredó con la racionalización».
  status         TEXT NOT NULL DEFAULT 'planned'
                 CHECK (status IN ('planned','held','cancelled')),
  created_by     UUID NOT NULL REFERENCES users(id),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (section_id, held_on, starts_at)
);

CREATE TABLE class_session_topics (
  session_id  UUID NOT NULL REFERENCES class_sessions(id) ON DELETE CASCADE,
  topic_id    UUID NOT NULL REFERENCES topics(id),
  PRIMARY KEY (session_id, topic_id)
);

CREATE TABLE class_session_materials (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id  UUID NOT NULL REFERENCES class_sessions(id) ON DELETE CASCADE,
  kind        TEXT NOT NULL CHECK (kind IN ('file','link')),
  title       TEXT NOT NULL,
  file_id     UUID REFERENCES files(id),
  url         TEXT,
  CHECK ((kind = 'file' AND file_id IS NOT NULL) OR (kind = 'link' AND url IS NOT NULL))
);

CREATE TABLE attendance (
  session_id  UUID NOT NULL REFERENCES class_sessions(id) ON DELETE CASCADE,
  student_id  UUID NOT NULL REFERENCES users(id),
  status      TEXT NOT NULL CHECK (status IN ('presente','ausente','tarde','excusa')),
  note        TEXT,
  marked_by   UUID NOT NULL REFERENCES users(id),
  marked_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (session_id, student_id)
);
```

**Pendientes y actividades:**

```sql
CREATE TABLE agenda_items (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id    UUID NOT NULL REFERENCES courses(id),
  section_id   UUID REFERENCES sections(id),      -- NULL = todo el curso
  session_id   UUID REFERENCES class_sessions(id), -- «quedó asignado en la clase del martes»
  kind         TEXT NOT NULL CHECK (kind IN ('actividad','pendiente')),
  visibility   TEXT NOT NULL CHECK (visibility IN ('teacher','students')),
  title        TEXT NOT NULL,
  detail       TEXT,
  due_at       TIMESTAMPTZ,
  done_at      TIMESTAMPTZ,                        -- solo tiene sentido para visibility='teacher'
  created_by   UUID NOT NULL REFERENCES users(id),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at   TIMESTAMPTZ
);
```

| Tipo | `visibility` | Ejemplo | Quién lo ve |
|---|---|---|---|
| Actividad | `students` | «Taller 3, entregar el viernes» | Docente y estudiantes del grupo o curso |
| Pendiente para estudiantes | `students` | «Traer calculadora el jueves» | Docente y estudiantes |
| Pendiente de la docente | `teacher` | «Subir notas del taller 2», «Reservar sala de cómputo» | **Solo docentes del curso** |

**Las evaluaciones no se duplican en la agenda.** El calendario es una consulta que une `class_sessions`, `agenda_items` y `assessments`, cada uno con su filtro de visibilidad. Si la fecha de una evaluación cambia, el calendario ya lo refleja.

🔒 **Filtro de visibilidad para estudiantes** (en el repositorio, no en la vista):

```sql
WHERE ai.visibility = 'students'
  AND ai.deleted_at IS NULL
  AND (ai.section_id IS NULL OR ai.section_id = :studentSectionId)
```

`teacher_notes` **nunca** sale en ninguna respuesta de estudiante: el esquema de salida del contrato para estudiantes simplemente no tiene ese campo.

### 5.3 Vistas

| Vista | Docente | Estudiante |
|---|---|---|
| **Próximo** (por defecto) | Última clase dictada, qué se vio, qué quedó asignado, y lo que viene en 14 días. Sus pendientes privados abiertos, arriba. | Lo mismo, sin notas ni pendientes privados. |
| **Semana** | Sesiones, entregas y evaluaciones en columnas | Igual |
| **Mes** | Cuadrícula con puntos por tipo | Igual |
| **Clase** (detalle) | Pasar lista + resumen + temas + material + asignado, **en una pantalla** | Resumen, temas, material y asignado |

**Pasar lista empieza con todos presentes** y solo se marcan las excepciones (heredado del plan anterior).

---

## 6. Modelo de datos

### 6.1 Convenciones

- Claves primarias `UUID` (`gen_random_uuid()`).
- `created_at` en todo. `deleted_at` para el borrado lógico donde aplique.
- **Nunca `ON DELETE CASCADE` desde `users`** hacia nada académico (intentos, notas, asistencia). Un usuario se desactiva (`deactivated_at`), no se borra.
- `CASCADE` solo hacia hijos puramente estructurales (páginas de un examen, temas de una sesión).
- Escala de notas 0,0–5,0: `NUMERIC(3,2)`.

### 6.2 Estructura académica

```
subjects (Cálculo Diferencial)
  └── courses (Cálculo Diferencial · 2026-2)  ── course_teachers (docentes)
        └── sections (Grupo A, Grupo B)
              └── enrollments (estudiantes)
terms (2026-2) ──┘
```

```sql
CREATE TABLE users (
  id                     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email                  CITEXT NOT NULL UNIQUE,
  full_name              TEXT   NOT NULL,
  role                   TEXT   NOT NULL CHECK (role IN ('student','teacher')),
  is_admin               BOOLEAN NOT NULL DEFAULT FALSE,
  student_code           TEXT,                -- solo dígitos
  national_id            TEXT,                -- cédula; única; con student_code sirve para recuperar (§1.1b A2, A3)
  student_code_verified  BOOLEAN NOT NULL DEFAULT FALSE,  -- TRUE si vino de la lista oficial
  password_hash          TEXT NOT NULL,
  password_changed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  password_reset_notice  JSONB,               -- {via, at} hasta que el usuario lo vea
  totp_secret_enc        BYTEA,               -- cifrado con TOTP_ENCRYPTION_KEY
  totp_enabled_at        TIMESTAMPTZ,
  leaderboard_opt_in     BOOLEAN NOT NULL DEFAULT FALSE,
  deactivated_at         TIMESTAMPTZ,
  created_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
  CHECK (NOT is_admin OR role = 'teacher')
);
CREATE UNIQUE INDEX uq_users_student_code ON users(student_code) WHERE student_code IS NOT NULL;

CREATE TABLE terms (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code       TEXT NOT NULL UNIQUE,        -- '2026-2'
  starts_on  DATE NOT NULL,
  ends_on    DATE NOT NULL
);

CREATE TABLE subjects (
  id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name  TEXT NOT NULL UNIQUE,
  code  TEXT UNIQUE                        -- código de la universidad, si existe
);

CREATE TABLE courses (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id   UUID NOT NULL REFERENCES subjects(id),
  term_id      UUID NOT NULL REFERENCES terms(id),
  title        TEXT NOT NULL,              -- por defecto «Materia · Periodo»
  archived_at  TIMESTAMPTZ,
  deleted_at   TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE course_teachers (
  course_id   UUID NOT NULL REFERENCES courses(id),
  user_id     UUID NOT NULL REFERENCES users(id),
  role        TEXT NOT NULL CHECK (role IN ('titular','auxiliar')),
  added_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  removed_at  TIMESTAMPTZ,
  PRIMARY KEY (course_id, user_id)
);

CREATE TABLE sections (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id    UUID NOT NULL REFERENCES courses(id),
  name         TEXT NOT NULL,              -- 'A', 'B'
  join_code    TEXT UNIQUE,                -- código de inscripción, rotable
  join_open    BOOLEAN NOT NULL DEFAULT TRUE,
  deleted_at   TIMESTAMPTZ,
  UNIQUE (course_id, name)
);

CREATE TABLE enrollments (
  section_id   UUID NOT NULL REFERENCES sections(id),
  student_id   UUID NOT NULL REFERENCES users(id),
  status       TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','withdrawn')),
  source       TEXT NOT NULL CHECK (source IN ('join_code','official_list','teacher')),
  enrolled_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (section_id, student_id)
);
-- Un estudiante está en un solo grupo por curso:
-- se garantiza en el servicio y con una restricción de exclusión o un trigger.
```

**Cómo se matricula un estudiante:**

1. **Código de inscripción** del grupo, que la docente muestra en clase. Se puede cerrar o rotar.
2. **Lista oficial** importada por la docente. Previsualización obligatoria, los cruces ambiguos se reportan y no se resuelven solos, y fija `student_code_verified = TRUE`.
3. **Manual**, desde la ficha del grupo.

**Quién crea qué:** *(ver §1.1b A1 y A5)*
- El administrador genera los códigos de invitación de docentes.
- El docente crea sus cursos y sus grupos, y gestiona todo lo demás del curso.

### 6.3 Temas, bancos y preguntas

```sql
CREATE TABLE topics (                     -- (ver §1.1b A20: pertenecen al banco)
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id  UUID NOT NULL REFERENCES subjects(id),  -- A20: se reemplaza por bank_id
  name        TEXT NOT NULL,               -- 'Límites por racionalización'
  parent_id   UUID REFERENCES topics(id),  -- 'Límites' > 'Límites por racionalización'
  UNIQUE (subject_id, name)
);

CREATE TABLE question_banks (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id     UUID NOT NULL REFERENCES users(id),   -- biblioteca del docente
  subject_id   UUID REFERENCES subjects(id),
  name         TEXT NOT NULL,
  description  TEXT,
  archived_at  TIMESTAMPTZ,
  deleted_at   TIMESTAMPTZ,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE questions (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  bank_id           UUID NOT NULL REFERENCES question_banks(id),
  topic_id          UUID REFERENCES topics(id),
  type              TEXT NOT NULL CHECK (type IN ('multiple_choice','numeric','true_false','open')),
  stem              TEXT NOT NULL,        -- Markdown + LaTeX
  options           JSONB,                -- para multiple_choice
  answer            JSONB,                -- esquema según type, validado con Zod
  generator_key     TEXT,                 -- pregunta paramétrica
  generator_params  JSONB,
  difficulty        SMALLINT,
  deleted_at        TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Los temas son de la **materia**, no del docente. Dos docentes de Cálculo comparten el vocabulario, y la bitácora, la práctica y las ayudas hablan de los mismos temas.

### 6.4 Evaluaciones, intentos y notas

```sql
CREATE TABLE assessments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id       UUID NOT NULL REFERENCES courses(id),
  section_id      UUID REFERENCES sections(id),      -- NULL = todo el curso
  title           TEXT NOT NULL,
  kind            TEXT NOT NULL CHECK (kind IN ('quiz','parcial','diagnostico','taller')),
  source          TEXT NOT NULL DEFAULT 'platform' CHECK (source IN ('platform','external')),
  modality        TEXT NOT NULL DEFAULT 'remote' CHECK (modality IN ('remote','in_person')),
  state           TEXT NOT NULL DEFAULT 'borrador'
                  CHECK (state IN ('borrador','programada','abierta','cerrada','publicada')),
  opens_at        TIMESTAMPTZ,
  closes_at       TIMESTAMPTZ,
  duration_min    INTEGER,
  weight          NUMERIC(5,2),                       -- % dentro del curso
  settings        JSONB NOT NULL DEFAULT '{}',        -- validado por contracts.assessmentSettings
  created_by      UUID NOT NULL REFERENCES users(id),
  deleted_at      TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Columnas contra `settings`:** lo que se consulta o filtra va en columna (estado, fechas, peso, alcance). Lo que solo configura la presentación va en `settings`, con un esquema Zod estricto y valores por defecto por plantilla: intentos, vidas, barajado, una pregunta a la vez, marca de agua, canarios, SEB, declaración y retroalimentación.

```sql
CREATE TABLE assessment_items (           -- qué se pregunta
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id   UUID NOT NULL REFERENCES assessments(id) ON DELETE CASCADE,
  position        INTEGER NOT NULL,
  question_id     UUID REFERENCES questions(id),   -- pregunta fija…
  bank_id         UUID REFERENCES question_banks(id),
  topic_id        UUID REFERENCES topics(id),      -- …o «N al azar de este banco/tema»
  pick_count      INTEGER,
  points          NUMERIC(5,2) NOT NULL DEFAULT 1
);

CREATE TABLE attempts (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id    UUID NOT NULL REFERENCES assessments(id),
  student_id       UUID NOT NULL REFERENCES users(id),
  number           SMALLINT NOT NULL,
  status           TEXT NOT NULL CHECK (status IN ('in_progress','submitted','auto_submitted','voided')),
  submit_reason    TEXT,              -- 'student','time','violations','closed'
  seed             BIGINT NOT NULL,   -- para preguntas paramétricas y barajado reproducibles
  started_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  deadline_at      TIMESTAMPTZ NOT NULL,
  submitted_at     TIMESTAMPTZ,
  lives_left       SMALLINT,
  frozen_at        TIMESTAMPTZ,
  frozen_reason    TEXT,
  frozen_evidence  JSONB,
  frozen_seconds   INTEGER NOT NULL DEFAULT 0,
  UNIQUE (assessment_id, student_id, number)
);

CREATE TABLE attempt_items (              -- la pregunta congelada tal como la vio el estudiante
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  attempt_id     UUID NOT NULL REFERENCES attempts(id),
  position       INTEGER NOT NULL,
  question_id    UUID NOT NULL REFERENCES questions(id),
  rendered       JSONB NOT NULL,          -- enunciado y opciones materializados
  answer_key     JSONB NOT NULL,          -- nunca sale al estudiante mientras el intento esté abierto
  points         NUMERIC(5,2) NOT NULL,
  served_at      TIMESTAMPTZ,
  answer         JSONB,
  answered_at    TIMESTAMPTZ,
  score          NUMERIC(5,2),
  UNIQUE (attempt_id, position)
);

CREATE TABLE attempt_events (             -- latido e integridad
  id           BIGSERIAL PRIMARY KEY,
  attempt_id   UUID NOT NULL REFERENCES attempts(id),
  type         TEXT NOT NULL,             -- window_blur, stem_copy_blocked, viewport_shrink, …
  payload      JSONB,
  occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE grades (                      -- libro de notas unificado
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id   UUID NOT NULL REFERENCES assessments(id),
  student_id      UUID NOT NULL REFERENCES users(id),
  attempt_id      UUID REFERENCES attempts(id),       -- NULL para externas y manuales
  value           NUMERIC(3,2) NOT NULL CHECK (value BETWEEN 0 AND 5),
  origin          TEXT NOT NULL CHECK (origin IN ('auto','manual','external','photo')),
  feedback        TEXT,
  published_at    TIMESTAMPTZ,
  UNIQUE (assessment_id, student_id)
);

CREATE TABLE grade_revisions (             -- cada cambio de nota, para siempre
  id          BIGSERIAL PRIMARY KEY,
  grade_id    UUID NOT NULL REFERENCES grades(id),
  old_value   NUMERIC(3,2),
  new_value   NUMERIC(3,2) NOT NULL,
  reason      TEXT,                        -- obligatorio si la nota ya estaba publicada
  changed_by  UUID NOT NULL REFERENCES users(id),
  changed_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Toda nota, venga de donde venga, termina en `grades`. El libro, la exportación y la nota ponderada (`weighting` portado de la v1) leen de un solo lugar.

### 6.5 Transversales

```sql
CREATE TABLE files (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  storage_key   TEXT NOT NULL UNIQUE,     -- ruta dentro de FILES_DIR; nunca la elige el cliente
  mime_type     TEXT NOT NULL,            -- detectado por contenido, no por extensión
  size_bytes    BIGINT NOT NULL,
  sha256        TEXT NOT NULL,
  uploaded_by   UUID NOT NULL REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE audit_log (
  id           BIGSERIAL PRIMARY KEY,
  actor_id     UUID REFERENCES users(id),
  action       TEXT NOT NULL,             -- 'grade.publish', 'user.email_change', 'auth.reset_with_code'…
  entity_type  TEXT,
  entity_id    TEXT,
  before       JSONB,
  after        JSONB,
  ip           INET,
  at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- El usuario de la aplicación solo tiene INSERT y SELECT sobre audit_log.
-- REVOKE UPDATE, DELETE ON audit_log FROM quiz_app;

CREATE TABLE auth_events (                -- ingresos, fallos, restablecimientos
  id          BIGSERIAL PRIMARY KEY,
  user_id     UUID REFERENCES users(id),
  email_tried CITEXT,
  type        TEXT NOT NULL,
  ip          INET,
  at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE password_reset_tokens (
  token_hash  TEXT PRIMARY KEY,
  user_id     UUID NOT NULL REFERENCES users(id),
  via         TEXT NOT NULL CHECK (via IN ('email','teacher_code')),
  expires_at  TIMESTAMPTZ NOT NULL,
  used_at     TIMESTAMPTZ
);
```

Las tablas de gamificación, ayudas entre compañeros y calificación por fotos se definen en sus etapas (§12). Su diseño de referencia sigue en `ARQUITECTURA_V2.md` §2.4b, d, f y la Fase 6, con estos ajustes: claves UUID, `topic_id` en vez de texto libre, insignias definidas en código y no como reglas en JSON, y el lote de exámenes en papel independiente de la evaluación.

---

## 7. Evaluaciones: ciclo de vida y variantes

### 7.0 Selección de preguntas: reglas *(9/10, §1.1b A9)*

**Lo que hace la v1** (`quiz_app/src/schemas/assessment.schema.js`, `public/js/pages/teacher-assessment.js`): cada regla es `{banco, tema | todos, dificultad | cualquiera, cantidad}`. Hay además un generador que crea de una vez una regla por cada tema marcado y por cada dificultad («2 difíciles + 2 medias + 1 fácil por tema»). Su límite: no permite «10 preguntas al azar entre *estos tres temas*, *medias o difíciles*» sin partirlo a mano.

**Regla de la v2** — un solo modelo que cubre todas las combinaciones:

```ts
// packages/contracts/assessmentRules.ts
rule = z.object({
  bankIds:      z.array(uuid).min(1),                 // uno o varios bancos del docente
  topicIds:     z.array(uuid).default([]),            // vacío = todos los temas
  difficulties: z.array(z.enum(["facil","medio","dificil"])).default([]),  // vacío = todas
  count:        z.number().int().min(1).max(100),
  spread:       z.enum(["random","even"]).default("random"),  // even = repartir parejo entre los temas marcados
  points:       z.number().positive().default(1),
})
```

| Lo que quiere la docente | Cómo queda |
|---|---|
| 10 al azar de todo el banco | `topicIds: []`, `difficulties: []`, `count: 10` |
| 8 difíciles de cualquier tema | `difficulties: ["dificil"]`, `count: 8` |
| 10 de Límites y Continuidad, medias o difíciles | `topicIds: [L, C]`, `difficulties: ["medio","dificil"]`, `count: 10` |
| Igual, pero 5 y 5 | lo mismo con `spread: "even"` |
| 2 fáciles + 2 medias + 1 difícil **por cada** tema | Botón **«Una regla por tema»** (el generador de la v1): crea una regla por tema y dificultad |

**Pantalla (paso *Qué evalúa*):**
1. Bancos (casillas, preselecciona el último usado).
2. **Tabla temas × dificultad** con la cantidad **disponible** en cada celda. Se marcan temas (filas) y dificultades (columnas) con casillas; *Todos / Ninguno* en cada eje.
3. Cantidad y reparto (*al azar* / *parejo por tema*) → **Añadir regla**.
4. Debajo, la lista de reglas, cada una con su resumen en una línea («10 · Límites, Continuidad · medias o difíciles · al azar») y su **disponibilidad real**.
5. Acceso a **Una regla por tema** y a **Preguntas fijas** (elegir preguntas una por una).

**Resolución sin repetir y sin falsos «insuficiente».** Cuando las reglas se solapan (p. ej. «3 de Límites» y «2 difíciles»), una pregunta difícil de Límites sirve a las dos, pero solo puede usarse una vez. Repartir con un método voraz puede declarar «insuficiente» algo que sí tiene solución. Por eso:

- `domain/assessment/ruleSolver.ts` modela las reglas como un **problema de flujo máximo**: fuente → regla (capacidad = `count`) → pregunta candidata (capacidad 1) → sumidero. Las preguntas fijas se descuentan primero. Con `spread: "even"`, la regla se divide en subreglas por tema con cuotas, y el sobrante se reparte al azar.
- **Al guardar** se comprueba la factibilidad y se informa por regla (*«Faltan 2 preguntas difíciles de Límites»*). No se puede programar una evaluación infactible.
- **Al iniciar cada intento** se ejecuta el mismo algoritmo con las candidatas **barajadas con la semilla del intento**: da una selección aleatoria, válida y reproducible.
- Las preguntas de bonificación (§7.3) se resuelven **después**, excluyendo las ya elegidas.
- Tests: solapamientos, factibilidad límite (exactamente justo), `even` con temas desiguales, misma semilla → misma selección.

### 7.1 Ciclo de vida

```
borrador ──programar──▶ programada ──(opens_at)──▶ abierta ──(closes_at o botón)──▶ cerrada ──publicar──▶ publicada
   ▲                        │                          │                              │
   └────────volver──────────┘                          └──────reabrir (con motivo)────┘
```

| Estado | Se puede editar | El estudiante ve | Botones del docente |
|---|---|---|---|
| **borrador** | Todo | Nada | Programar · Duplicar · Eliminar |
| **programada** | Todo menos lo que ya pasó | Título y fecha en el calendario | Abrir ahora · Volver a borrador |
| **abierta** | Solo ampliar el cierre | La evaluación, puede presentarla | Monitor · Cerrar ahora |
| **cerrada** | Calificación manual | «En revisión» | Calificar · Publicar · Reabrir |
| **publicada** | Notas, con motivo y revisión | Nota y retroalimentación | Editar nota · Exportar |

- Las transiciones viven en `domain/assessment/states.ts`, con una tabla y sus tests. Un servicio que intenta una transición inválida recibe `INVALID_STATE`.
- **Los botones del panel se derivan del estado.** Nunca hay un botón que haga dos cosas: *Cerrar* y *Publicar* son dos acciones.
- Reabrir una evaluación cerrada exige un motivo y queda en la auditoría.
- Una evaluación **externa** (`source = 'external'`) salta de borrador a cerrada: no se presenta en la plataforma, solo recibe notas importadas.

**Cierre y plazos individuales.** `closes_at` es la hora a partir de la cual **nadie puede empezar**. Cada intento tiene su propio `deadline_at`, que normalmente es `min(started_at + duración, closes_at)`, pero que una ampliación puede llevar más allá del cierre. La evaluación pasa a **cerrada** cuando se cumplen las dos cosas: pasó `closes_at` y no queda ningún intento en curso. Así ampliar el tiempo de un estudiante no reabre la evaluación para los demás.

### 7.2 Ampliar tiempo de una evaluación en curso

**Quién y desde dónde:**

| Actor | Dónde | Requisitos |
|---|---|---|
| Docente del curso | Monitor en vivo: botón **+ Tiempo** arriba (toda la evaluación) y en cada fila (un intento) | Motivo con un toque: *Falla técnica · Ajuste · Accesibilidad · Otro* |
| Administrador | Panel de administración → **Evaluaciones en curso** | Reautenticación + motivo escrito. Notifica a la docente en su Inicio. |

**Dos alcances:**

| Alcance | Qué cambia | Caso típico |
|---|---|---|
| **Toda la evaluación** | `closes_at += N` y `deadline_at += N` en **todos los intentos en curso**. Los ya entregados no se reabren. | Se cayó el servidor o la red del campus 5 minutos |
| **Un intento** | Solo `deadline_at += N` de ese intento. `closes_at` no cambia. | Un estudiante con un problema puntual, o un ajuste razonable de accesibilidad |

**Reglas:**

- Solo en estado **abierta**. Para una evaluación cerrada, se reabre con su motivo (§7.1).
- De 1 a 60 minutos por acción. Botones rápidos **+5 · +10 · +15**, y valor libre.
- No se puede quitar tiempo. Corregir una ampliación equivocada es otra decisión y no tiene botón.
- Funciona igual con intentos congelados: la pausa y la ampliación se suman.
- **El servidor manda.** El trabajo `attempt.expire` revisa `deadline_at` en el momento de ejecutarse, así que una ampliación posterior lo vuelve inofensivo; se reprograma al nuevo plazo. El latido devuelve el `deadline_at` vigente y la pantalla del estudiante muestra: *«Tu docente añadió 10 minutos.»*
- Una respuesta que llega después de `deadline_at` + 5 s de gracia se rechaza.

```sql
CREATE TABLE time_extensions (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id  UUID NOT NULL REFERENCES assessments(id),
  attempt_id     UUID REFERENCES attempts(id),     -- NULL = toda la evaluación
  minutes        SMALLINT NOT NULL CHECK (minutes BETWEEN 1 AND 60),
  reason         TEXT NOT NULL,
  granted_by     UUID NOT NULL REFERENCES users(id),
  granted_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

El monitor muestra el historial de ampliaciones de la evaluación. La exportación de resultados incluye una columna con los minutos añadidos a cada intento, para que una nota obtenida con tiempo extra no se lea como si no lo hubiera tenido.

### 7.3 Preguntas de bonificación

**Qué es:** preguntas opcionales al final de la evaluación. **Cada una bien respondida recupera una pregunta que el estudiante haya fallado.** No suman por encima de la nota máxima: sirven para reparar, no para inflar.

**Configuración** (paso *Qué evalúa* del constructor, interruptor *Preguntas de bonificación*):

```ts
// packages/contracts/assessmentSettings.ts
bonus: z.object({
  enabled:  z.boolean().default(false),
  count:    z.number().int().min(1).max(5),
  topicIds: z.array(z.string().uuid()).min(1),     // de qué temas salen
  bankIds:  z.array(z.string().uuid()).min(1),     // de qué bancos (de su biblioteca)
  match:    z.enum(["any", "same_topic"]).default("any"),
}).optional()
```

- Al elegir los temas, el constructor muestra cuántas preguntas hay disponibles (*«Límites: 14 disponibles»*). Si hay menos que `count`, no deja guardar.
- Solo se usan preguntas de **calificación automática** (opción múltiple, numérica, verdadero/falso). Una pregunta abierta no puede decidir de inmediato si recupera otra.
- Se sortean **al iniciar el intento**, con la semilla del intento y sin repetir preguntas ya incluidas. Admiten preguntas paramétricas.

**Presentación:**

1. El estudiante responde las preguntas normales.
2. Tras la última aparece una pantalla: *«Tienes 2 preguntas de bonificación opcionales. Cada una bien respondida reemplaza una que hayas fallado.»* → **Responder bonificación** / **Entregar ya**.
3. Si la evaluación permite volver atrás, al entrar a la bonificación se avisa: *«Ya no podrás cambiar tus respuestas anteriores.»* Así la bonificación no se usa como pista para corregir lo anterior.
4. Las preguntas de bonificación consumen el mismo reloj y las mismas vidas.

**Regla de reemplazo** — `domain/grading/bonus.ts`, función pura con tests:

```ts
export function applyBonus(regular: ScoredItem[], bonus: ScoredItem[], match: "any" | "same_topic"): BonusResult {
  // 1. Pendientes = preguntas normales con score < points (mal o en blanco),
  //    ordenadas por puntos perdidos (desc) y luego por posición.
  // 2. Por cada bonificación correcta (score === points), en orden de posición:
  //    - match "any": recupera la primera pendiente.
  //    - match "same_topic": recupera la primera pendiente de su mismo tema; si no hay, no recupera nada.
  //    La pendiente recuperada pasa a valer sus puntos completos y se marca recovered_by.
  // 3. Las bonificaciones correctas sin pendiente que recuperar no valen nada.
  // 4. La nota nunca supera el máximo de las preguntas normales.
}
```

- **Se recupera primero lo que más puntos perdió.** Es la opción más favorable al estudiante, y es determinista: la misma hoja da siempre la misma nota.
- Una bonificación recupera **el valor completo** de la pregunta reemplazada, valga lo que valga la bonificación.
- Si la docente corrige a mano una nota después, `applyBonus` se vuelve a ejecutar y queda en `grade_revisions`.

**Columnas que se añaden a `attempt_items`:**

```sql
is_bonus       BOOLEAN NOT NULL DEFAULT FALSE,
recovered_by   UUID REFERENCES attempt_items(id),   -- en la pregunta normal recuperada
final_score    NUMERIC(5,2)                         -- score tras aplicar la bonificación
```

**Lo que ve el estudiante** en su resultado: *«Pregunta 4 (Límites por racionalización): recuperada con la bonificación B2.»* La nota nunca cambia sin explicar de dónde salió el cambio.

### 7.4 Quices en pareja

**Configuración** (paso *Cómo se presenta*):

```ts
team: z.object({
  mode:      z.enum(["individual", "pairs"]).default("individual"),
  formation: z.enum(["assigned", "free"]).default("assigned"),
  allowTrio: z.boolean().default(true),   // para el número impar
}).default({ mode: "individual" })
```

**Cómo se forman las parejas:**

| Formación | Cómo funciona |
|---|---|
| *(Ver §1.1b A10.)* **Asignada** *(por defecto)* | La docente arma las parejas antes de abrir, por grupo: **Generar al azar** y luego ajustar arrastrando. Al iniciar, el estudiante ya sabe con quién presenta. |
| **Libre** | Al pulsar *Iniciar*, el primer estudiante recibe un código de 4 dígitos en pantalla. Su compañero lo escribe en su propio dispositivo para unirse. El intento empieza cuando la pareja está completa. |

En las dos formaciones:
- los dos tienen que estar **matriculados en el mismo grupo**, y ese grupo tiene que estar dentro del alcance de la evaluación;
- si la evaluación es presencial, los dos tienen que estar **presentes** en la lista de hoy;
- un estudiante pertenece a **una sola pareja** por evaluación;
- con número impar, si `allowTrio` lo permite, se forma un trío; si no, la docente marca a esa persona como individual desde el monitor.

**Un intento compartido, dos dispositivos vigilados:**

- **Responde un solo dispositivo**: el de quien inició, el *operador*.
- El dispositivo del compañero entra en **modo acompañante**: muestra la misma pregunta en solo lectura y **sigue vigilado**. Pérdida de foco, pantalla dividida y segunda pestaña cuentan en ese dispositivo también.

  Sin esto, la pareja tendría un teléfono libre para buscar respuestas mientras el otro contesta, y el quiz en pareja sería la forma más fácil de hacer trampa del sistema.
- Las **vidas son compartidas**: una infracción en cualquiera de los dos dispositivos descuenta al intento. `attempt_events` registra qué miembro la produjo.
- Si el operador pierde la conexión, la docente puede pasar el control al compañero desde el monitor.
- La **marca de agua visible** muestra los dos nombres.

**Modelo:**

```sql
CREATE TABLE assessment_teams (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  assessment_id  UUID NOT NULL REFERENCES assessments(id),
  section_id     UUID NOT NULL REFERENCES sections(id),
  join_code      TEXT,                  -- solo formación libre; caduca al completar la pareja
  created_by     UUID NOT NULL REFERENCES users(id),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
  team_id        UUID NOT NULL REFERENCES assessment_teams(id),
  assessment_id  UUID NOT NULL REFERENCES assessments(id),   -- duplicado a propósito, para la unicidad
  student_id     UUID NOT NULL REFERENCES users(id),
  joined_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (team_id, student_id),
  UNIQUE (assessment_id, student_id)                         -- una sola pareja por evaluación
);

-- En attempts:
team_id      UUID REFERENCES assessment_teams(id),   -- NULL en evaluaciones individuales
operator_id  UUID REFERENCES users(id)               -- quien responde; la docente puede cambiarlo
```

**Notas:**
- Al calificar se escribe **una fila en `grades` por cada miembro**, con el mismo `attempt_id` y el mismo valor.
- La docente puede ajustar la nota de un solo miembro (por ejemplo, si uno no participó). Queda en `grade_revisions` con motivo obligatorio.
- Los intentos se cuentan **por pareja**: si la evaluación permite 2 intentos, son 2 de la pareja, no 2 de cada uno.
- Una pareja que se arma no se puede cambiar una vez iniciado el intento.

🔒 **Acceso:** la regla «dueño del intento» de §10 pasa a ser **«miembro del intento»** (`attemptsRepo.findForMember`). Solo el operador puede responder; el acompañante que intenta responder recibe 403, porque el intento es suyo pero no le toca esa acción.

**Interacción con otras funciones:**
- Bonificación: la pareja la responde igual.
- Ampliar tiempo «a un intento» amplía el de la pareja.
- XP (etapa 8): ambos reciben el mismo; en los pelotones de velocidad la pareja cuenta una vez.
- Similitud entre exámenes: compara parejas, no individuos.

---

## 8. Integridad del examen (solo web)

### 8.1 Lo que se puede y lo que no, dicho de una vez

**No hay forma de impedir una captura de pantalla desde una página web.** Netflix protege video con DRM e Instagram usa una función del sistema operativo reservada a apps nativas; ninguna de las dos aplica al texto de una página. Esta versión **no** incluye app nativa (decisión §1.1). La estrategia es:

1. hacer que copiar sea incómodo;
2. hacer que toda copia sea **atribuible**;
3. hacer que copiar **rinda poco**.

### 8.2 Medidas

| Medida | Qué hace | Tipo |
|---|---|---|
| **Bloqueo de selección y copia** | `user-select: none` en el enunciado y las opciones; `copy`, `cut`, `contextmenu`, `dragstart` y `beforeprint` cancelados. Cada intento se registra como `stem_copy_blocked`. | Disuasión |
| **Impresión** | `@media print { .exam { display: none } }` | Disuasión |
| **Marca de agua visible** | Nombre + código + hora, en diagonal y semitransparente sobre la pregunta, repetida. Renderizada con CSS/SVG en capa superior, no como texto seleccionable. | **Atribución**: toda captura que circule dice de quién salió |
| **Marca de agua invisible** | Portada de `watermark.js` (v1): variantes invisibles del enunciado por intento, y `identificar-fuga` sobre un texto pegado | Atribución |
| **Una pregunta a la vez** | El examen completo nunca está en el navegador | Rendimiento de la trampa |
| **Barajado** de preguntas y opciones | El orden no es compartible | Rendimiento |
| **Preguntas paramétricas** | Números distintos por estudiante, con la semilla del intento | **La defensa más fuerte**: la respuesta del compañero no sirve |
| **Reloj por pregunta** (opcional por evaluación) | Límite desde `served_at`; consultar un chat cuesta más que responder | Rendimiento |
| **Detector de pantalla dividida** | Portado de `split-screen.js`, con congelación, espera de 4 s, pausa del reloj y liberación a los 10 min (etapa 6) | Detección |
| **Pérdida de foco y pantalla completa** | Descuentan vidas según la configuración | Detección |
| **Segunda pestaña** (`BroadcastChannel`) | Evento de alta señal | Detección |
| **Canarios** | Distinguen «copió la clave ajena» de «respondió rápido» | Detección |
| **SEB** | Opcional por evaluación, para sala de cómputo | Impedimento real (solo en escritorio) |

🔒 **`answer_key` nunca viaja al navegador** mientras el intento esté abierto. La corrección ocurre en el servidor.

La meta de viewport de la vista del examen incluye `interactive-widget=resizes-visual`. Sin ella, el teclado del móvil se confunde con una pantalla dividida.

---

## 9. Interfaz

### 9.1 Principios

- **Cada pantalla responde una pregunta.** Si responde dos, son dos pantallas o una tiene que irse.
- **Una acción por botón**, con un nombre que dice cuál. Lo irreversible confirma **nombrando la consecuencia**: *«Publicar notas de 32 estudiantes. Las verán de inmediato.»*
- **Lo común es corto, lo raro está plegado.** Configuración avanzada en un panel que se abre solo si se necesita.
- **Nada se ofrece si el servidor no puede cumplirlo** (`/features`).
- **Estados vacíos que enseñan**: un curso sin grupos dice «Crea el primer grupo» con el botón, no «No hay datos».

### 9.2 Panel docente

**Navegación global** (barra lateral en escritorio, inferior en móvil):

| Área | Qué responde |
|---|---|
| **Inicio** | *¿Qué tengo que hacer ahora?* |
| **Mis cursos** | *¿Qué cursos dicto este periodo?* (con acceso a periodos anteriores) |
| **Biblioteca** | *¿Qué preguntas y rúbricas tengo?* Bancos, temas, plantillas. Transversal a los cursos. |
| **Análisis** | *¿Cómo van mis grupos comparados?* Solo lo que cruza cursos o periodos. |

**Inicio es una bandeja, no un menú.** Muestra tarjetas con acción directa, ordenadas por urgencia:

- 🔴 Intentos congelados ahora mismo → *Revisar*
- 🟠 Evaluaciones que abren o cierran hoy → *Ir al monitor*
- 🟡 12 intentos por calificar en «Parcial 1» → *Calificar*
- 🟡 Notas listas sin publicar → *Publicar*
- ⚪ Tus pendientes privados que vencen esta semana → *Marcar hecho*
- ⚪ 5 estudiantes sin código verificado en Cálculo A → *Cargar lista oficial*
- ⚪ La última clase de Cálculo B no tiene resumen → *Escribirlo*

Si no hay nada, lo dice: *«Todo al día.»*

**Dentro de un curso**, pestañas:

| Pestaña | Contenido |
|---|---|
| **Próximo** *(por defecto)* | La bitácora: última clase, lo asignado, lo que viene (§5.3) |
| **Clases** | Lista de sesiones; cada una abre la pantalla única de lista + resumen + material |
| **Calendario** | Semana / mes |
| **Evaluaciones** | Agrupadas por estado. Botón principal: *Nueva evaluación* → plantilla |
| **Estudiantes** | Por grupo. Arriba, el **código de inscripción** con *Mostrar en pantalla* (está donde se necesita: cuando se habla de estudiantes). Filtros útiles: *sin código verificado*, *con bloqueo*, *con asistencia bajo el umbral* |
| **Notas** | Libro del curso, nota ponderada, importar externas, exportar |
| **Ajustes** | Grupos, docentes auxiliares, pesos. Lo que se toca una vez por semestre. |

Selector de grupo arriba: **«Todos los grupos» / A / B**. Filtra todas las pestañas a la vez.

Los cursos del periodo actual aparecen **directamente en la barra lateral**, bajo *Mis cursos*. Entrar a un curso es un clic, no dos.

**Botón global ＋ Crear** en la barra superior: *Evaluación · Clase · Pendiente · Banco de preguntas*. Pide el curso dentro del formulario. Crear algo nunca exige navegar primero hasta el lugar correcto.

**Evaluación cerrada o publicada → pestaña Integridad:** eventos por estudiante, similitud entre intentos e *Identificar fuga*. Todo lo forense de una evaluación vive junto a esa evaluación, no en un área aparte.

**Constructor de evaluaciones:** plantilla («Quiz rápido», «Parcial», «Diagnóstico», «Taller externo») → tres pasos (*Qué evalúa* · *Cómo se presenta* · *Cómo se califica*) → *Avanzado* plegado. En el primer paso se elige **a quién va**: todo el curso o un grupo.

**Monitor en vivo:** modo de pantalla completa. Congelados arriba con contador y aviso sonoro opcional; liberar o marcar presente con un toque, sin salir de la pantalla. Botón **+ Tiempo** arriba (toda la evaluación) y en cada fila (ese intento). En quices en pareja, cada fila es una pareja: muestra los dos nombres, quién opera, y el botón **Pasar el control**.

### 9.2b Reglas para que nada sobre y nada quede escondido

Son de cumplimiento obligatorio. Se revisan en cada PR que toque la interfaz, igual que la seguridad.

**1. Todo botón hace algo importante, y lo hace donde vive el objeto.**
- Cada botón corresponde a una acción del backend en §10, o a una navegación. No hay botones decorativos ni «por si acaso».
- La acción está **junto a lo que modifica**. *+ Tiempo* está en el monitor de esa evaluación. *Desbloquear* está en la ficha de ese estudiante. Nunca en un menú general de «herramientas».

**2. Una función, un lugar.**
- Cada función tiene **un solo** lugar en la interfaz, el que dice el mapa de §9.2c.
- Solo hay dos atajos permitidos que la repiten: las **tarjetas de Inicio** (cuando hay algo pendiente de esa función) y el **buscador de comandos** (regla 5). Cualquier otro duplicado es un defecto.

**3. Jerarquía por pantalla.**
- **Una** acción principal, visualmente destacada. Es lo que se viene a hacer a esa pantalla.
- Hasta **tres** secundarias visibles.
- El resto en un menú **⋯**, y solo si se usa poco **y** está en el buscador de comandos.
- Botones que no aplican en el estado actual **no se muestran**. No se muestran deshabilitados, salvo que explicar por qué no se puede sea útil, y entonces lo dicen: *«Disponible cuando la evaluación esté cerrada.»*

**4. Nada a más de dos clics.**
- Desde cualquier pantalla, toda función del mapa se alcanza en **dos clics o menos** (sin contar el clic de la acción misma).
- La configuración avanzada de una evaluación está **plegada en la misma pantalla**, con una línea que resume sus valores actuales: *«Avanzado: SEB no · Marca de agua sí · Canarios 2»*. Abrirla es un clic, nunca otra página. Plegar no es esconder: siempre se ve qué hay adentro.

**5. Buscador de comandos** (`Ctrl+K`, o la lupa en el móvil).
- Encuentra estudiantes, cursos y evaluaciones por nombre, y **acciones** por lo que hacen: *«ampliar tiempo»*, *«pasar lista»*, *«código de restablecimiento»*, *«exportar notas»*.
- Es la garantía de que ninguna función queda enterrada: si existe, se encuentra escribiendo su nombre.
- 🔒 Los resultados salen de la API, filtrados por la misma política de acceso. El buscador nunca sugiere un estudiante o un curso ajeno.

**6. La interfaz enseña lo que la app puede hacer.**
- **Estados vacíos con la acción**: *«Este curso no tiene grupos — Crear el primero»*, no *«Sin datos»*.
- **Lista de arranque del curso** en la pestaña Próximo, hasta completarla: *Grupos ✓ · Lista oficial ✗ · Primera clase ✗ · Primera evaluación ✗*. Desaparece sola cuando todo está hecho.
- Las funciones potentes (bonificación, parejas, preguntas paramétricas, marca de agua) aparecen **como opciones del constructor con una línea que dice para qué sirven**, no detrás de un manual.

**7. Lista de revisión para cada botón nuevo.** Antes de añadirlo:
1. ¿Hace algo importante, o se puede quitar?
2. ¿Está donde vive el objeto que modifica?
3. ¿Ya existe esa acción en otro lugar? Si sí, sobra uno de los dos.
4. ¿Aparece solo cuando se puede usar?
5. ¿Está en el mapa de §9.2c y en el buscador de comandos?

### 9.2c Mapa de funciones del docente

Fuente única de «dónde vive cada cosa». Una función que no está aquí no tiene botón. Los clics se cuentan desde Inicio y no incluyen el clic de la acción.

| Función | Dónde vive | Clics | Atajo en Inicio |
|---|---|---|---|
| Pasar lista de hoy | Curso → Clases → clase de hoy | 2 | Sí, «Clase de hoy» → 1 |
| Escribir el resumen de la clase | La misma pantalla de la clase | 2 | Sí, si la última no tiene resumen |
| Ver en qué quedó la clase pasada | Curso → Próximo | 1 | — |
| Crear pendiente (privado o para estudiantes) | ＋ Crear → Pendiente | 1 | — |
| Ver calendario | Curso → Calendario | 2 | — |
| Crear evaluación | ＋ Crear → Evaluación | 1 | — |
| Bonificación, parejas, alcance a grupo | Dentro del constructor de la evaluación | — | — |
| Armar parejas | Evaluación → Parejas *(solo si es en pareja)* | 2 | — |
| Abrir o cerrar ahora | Evaluación (según su estado) | 2 | Sí, si abre o cierra hoy |
| Monitor en vivo | Evaluación abierta → Monitor | 2 | Sí, mientras está abierta → 1 |
| Ampliar tiempo | Monitor (arriba o en la fila) | 2 | Desde la tarjeta del monitor → 1 |
| Liberar intento congelado | Monitor (arriba) | 2 | Sí → 1 |
| Pasar el control en pareja | Monitor (fila de la pareja) | 2 | — |
| Calificar | Evaluación cerrada → Calificar | 2 | Sí, si hay por calificar → 1 |
| Publicar notas | Evaluación cerrada | 2 | Sí, si hay listas → 1 |
| Integridad / identificar fuga | Evaluación → Integridad | 2 | — |
| Editar una nota | Curso → Notas → celda | 2 | — |
| Importar notas externas | Curso → Notas | 2 | — |
| Exportar notas | Curso → Notas | 2 | — |
| Mostrar código de inscripción | Curso → Estudiantes (arriba) | 2 | — |
| Cargar lista oficial | Curso → Estudiantes | 2 | Sí, si hay estudiantes sin código |
| Corregir correo, nombre o código | Ficha del estudiante | 3 → **1 con el buscador** | — |
| Desbloquear cuenta | Ficha del estudiante | 3 → **1 con el buscador** | — |
| Código de restablecimiento | Ficha del estudiante | 3 → **1 con el buscador** | — |
| Bancos y preguntas | Biblioteca | 1 | — |
| Importar preguntas | Biblioteca → banco | 2 | — |
| Grupos, auxiliares, pesos | Curso → Ajustes | 2 | — |
| Análisis entre cursos | Análisis | 1 | — |
| Sesiones activas, segundo factor | Menú de usuario → Seguridad | 2 | — |

La ficha del estudiante es la única función a tres clics por navegación. Se acepta porque buscar a un estudiante por nombre es el camino natural: **Ctrl+K → nombre → ficha**.

Este mapa se actualiza en la misma PR que añade, mueve o quita una función.

### 9.3 Estudiante

Barra inferior: **Inicio · Calendario · Notas · Práctica**.

- **Inicio**: evaluaciones abiertas ahora (arriba y grande), lo que viene esta semana, la última clase de cada curso.
- **Calendario**: la misma bitácora que ve la docente, sin lo privado.
- **Notas**: por curso, cada evaluación con su nota y retroalimentación, el acumulado ponderado (`current` y `projected`) y la asistencia.
- **Práctica**: marcador de posición hasta la etapa 8, pero con un mensaje honesto, no *«🚧 En construcción»*.

Aviso en el primer ingreso tras un restablecimiento (§4.7).

### 9.4 Administrador

Entrada propia en el menú, visible solo con `is_admin`. Requiere reautenticación.

- **Docentes**: crear, desactivar, asignar a cursos.
- **Materias y periodos**: catálogo.
- **Cursos**: crear el curso de un periodo y asignar titular.
- **Cuentas**: buscar cualquier usuario, ver sus bloqueos (por cuenta+IP y por IP, con la hora en que expiran) y liberarlos.
- **Evaluaciones en curso**: las abiertas ahora mismo en toda la plataforma, con curso, título, intentos en curso y hora de cierre. Solo permite **+ Tiempo** (§7.2). No muestra preguntas, respuestas ni notas.
- **Auditoría**: filtro por actor, acción y fecha. Solo lectura.

---

## 10. Mapa de rutas y su protección 🔒

Prefijo `/api/v1`. **P** = pública · **E** = estudiante · **D** = docente · **A** = admin · **R** = requiere reautenticación.

| Recurso | Método y ruta | Rol | Política |
|---|---|---|---|
| **Sesión** | `POST /auth/login` | P | Límite §4.6 |
| | `POST /auth/logout` | E D | Propia |
| | `POST /auth/register` | P | Correo institucional, código duplicado |
| | `POST /auth/reauth` | E D | Propia |
| | `POST /auth/reset-with-code` | P | Límite §4.6 |
| | `POST /auth/reset-with-token` | P | Token válido y no usado |
| | `POST /auth/forgot-password` | P | Solo si SMTP; respuesta genérica |
| | `GET /me` · `GET /me/sessions` · `DELETE /me/sessions/:id` | E D | Propia |
| | `POST /me/totp` · `DELETE /me/totp` | D **R** | Propia |
| **Features** | `GET /features` | P | — |
| **Cursos** | `GET /courses` | E D | Filtrado: los que dicta / en los que está matriculado |
| | `GET /courses/:courseId` | E D | `teachesCourse` ∨ `enrolledInCourse` |
| | `PATCH /courses/:courseId` | D | `teachesCourse` (titular) |
| **Grupos** | `POST /courses/:courseId/sections` | D | Titular |
| | `POST /sections/join` | E | Código válido y grupo abierto |
| | `POST /sections/:sectionId/official-list` | D **R** | `teachesSection` |
| | `GET /sections/:sectionId/students` | D | `teachesSection` |
| **Estudiantes** | `PATCH /students/:studentId` (correo, nombre, código) | D **R** | Estudiante matriculado en un grupo que dicta |
| | `POST /students/:studentId/reset-code` | D | Ídem |
| | `POST /students/:studentId/unlock` | D | Ídem |
| **Clases** | `GET /courses/:courseId/calendar` | E D | Ídem curso; filtro de visibilidad §5.2 |
| | `POST /sections/:sectionId/class-sessions` | D | `teachesSection` |
| | `GET /class-sessions/:id` | E D | Por su grupo; `teacher_notes` solo D |
| | `PATCH /class-sessions/:id` · `PUT /class-sessions/:id/attendance` | D | `teachesSection` |
| **Agenda** | `POST /courses/:courseId/agenda` · `PATCH /agenda/:id` | D | `teachesCourse` |
| **Biblioteca** | `GET/POST /banks` · `GET/PATCH /banks/:bankId` | D | `ownsBank` |
| | `POST /banks/:bankId/questions/import` | D | `ownsBank` |
| | `GET /topics?subjectId=` | D | Cualquier docente |
| **Evaluaciones** | `GET /courses/:courseId/assessments` | E D | Curso; para E, solo visibles y de su grupo |
| | `POST /courses/:courseId/assessments` | D | `teachesCourse`; bancos usados ⊂ `ownsBank` |
| | `GET /assessments/:id` | E D | `findForStudent` / `findForTeacher` |
| | `PATCH /assessments/:id` | D | `findForTeacher` + estado lo permite |
| | `POST /assessments/:id/transition` | D | `findForTeacher`; publicar = **R** |
| | `GET /assessments/:id/monitor` | D | `findForTeacher` |
| **Intentos** | `POST /assessments/:id/attempts` | E | `findForStudent` + abierta + intentos restantes + asistencia si presencial; en pareja, pareja completa y ambos presentes |
| | `GET /attempts/:id/current-item` | E | **Miembro del intento** + `in_progress` + no congelado |
| | `PUT /attempts/:id/items/:pos` · `POST /attempts/:id/submit` | E | Miembro del intento **y operador** |
| | `POST /attempts/:id/heartbeat` | E | Miembro del intento (los dos dispositivos en pareja) |
| | `POST /attempts/:id/unfreeze` · `POST /attempts/:id/void` | D | Dicta el curso de la evaluación |
| | `POST /attempts/:id/operator` | D | Dicta el curso; el nuevo operador es miembro |
| **Parejas** | `GET/PUT /assessments/:id/teams` | D | `findForTeacher` + estado borrador o programada |
| | `POST /assessments/:id/teams/join` | E | `findForStudent` + formación libre + mismo grupo + sin pareja |
| **Tiempo** | `POST /assessments/:id/time-extensions` | D · A **R** | D: `findForTeacher`. A: `is_admin`. Ambos: evaluación abierta, motivo obligatorio |
| | `POST /attempts/:id/time-extensions` | D · A **R** | Ídem, e intento en curso |
| | `GET /admin/live-assessments` | A **R** | `is_admin`; solo campos de §9.4 |
| **Notas** | `GET /me/grades` | E | Propias, solo publicadas |
| | `GET /courses/:courseId/gradebook` | D | `teachesCourse` |
| | `PATCH /grades/:id` | D (**R** si publicada) | Dicta el curso; motivo obligatorio si publicada |
| | `POST /assessments/:id/grades/import` | D | `findForTeacher` + `source='external'` |
| **Archivos** | `POST /files` | E D | Límite de tamaño y tipo por contexto |
| | `GET /files/:id` | E D | Política del dueño lógico (material, foto, adjunto) |
| **Admin** | `/admin/*` | A **R** | `is_admin` |

**Regla para rutas nuevas:** una ruta que no está en esta tabla no existe. Se añade aquí en la misma PR.

---

## 11. Pruebas

### 11.1 Registro de rutas (se ejecuta en cada build)

```ts
test("toda ruta declara auth y política", () => {
  for (const r of registry.all()) {
    expect(r.auth, `${r.method} ${r.path}`).toBeDefined();
    if (r.auth !== "public") expect(r.policy, `${r.method} ${r.path}`).toBeDefined();
  }
});

test("las rutas públicas son exactamente las esperadas", () => {
  expect(registry.public().map(key)).toEqual(PUBLIC_ROUTES); // lista cerrada
});
```

### 11.2 Matriz de acceso cruzado

La semilla de pruebas crea: docente **A** (curso X, grupos X1 y X2), docente **B** (curso Y), estudiante **e1** en X1, **e2** en X2, **e3** en Y, **e4** retirado de X1, y un administrador sin cursos.

Para **cada** ruta con parámetro de recurso se prueba, de forma automática a partir del registro:

| Actor | Recurso | Esperado |
|---|---|---|
| Docente B | Cualquier cosa del curso X | 404 |
| e3 | Evaluación del curso X | 404 |
| e2 | Evaluación asignada solo a X1 | 404 |
| e1 | Evaluación en borrador de X | 404 |
| e4 (retirado) | Evaluación abierta de X | 404 |
| e1 | Intento de e2 | 404 |
| e1 | `teacher_notes` de su clase | El campo no existe en la respuesta |
| e1 | Pendiente privado de la docente | No aparece en el calendario |
| Admin sin cursos | Notas del curso X | 404 |
| Docente A sin reauth | Publicar notas | `REAUTH_REQUIRED` |
| e2 (no es de la pareja) | Intento de la pareja e1 + e5 | 404 |
| e5 (acompañante) | Responder una pregunta del intento | 403 |
| e3 (otro curso) | Código de pareja de e1 | Rechazado, sin revelar si el código existe |
| Admin sin reauth | Ampliar tiempo | `REAUTH_REQUIRED` |
| Admin | Monitor, respuestas o eventos de una evaluación que amplió | 404 |
| Docente B | Ampliar tiempo en el curso X | 404 |
| Cualquiera | Ampliar tiempo de una evaluación cerrada | `INVALID_STATE` |

### 11.3 Resto

- **Dominio:** transiciones de estado, `applyBonus` (ningún caso supera el máximo, `same_topic` sin pareja de tema no recupera nada, mismo resultado en dos ejecuciones), recálculo de `deadline_at` con ampliación + congelación, nota ponderada, generadores paramétricos (portados con sus tests), contraseñas (incluidas las cortas prohibidas).
- **Integración:** cada servicio con una base real en Docker, no con mocks.
- **Interfaz:**
  - un test recorre el mapa de §9.2c y comprueba que cada función se alcanza con los clics declarados;
  - otro comprueba que cada acción de §10 con botón aparece en el buscador de comandos.
- **Prueba con personas, al cerrar cada etapa con interfaz:** un colega docente hace 5 tareas reales sin ayuda (*pasar lista, crear un quiz desde plantilla, ampliar tiempo, publicar notas, corregir el correo de un estudiante*). Si una tarea le toma más de un minuto o pregunta «¿dónde está…?», es un defecto de la etapa, no del usuario.
- **Punta a punta (Playwright):**
  - un estudiante se inscribe con código, presenta, entrega y ve su nota publicada;
  - una docente crea un curso desde cero y publica un quiz desde una plantilla en menos de dos minutos.
- **Integridad:** la matriz de seis casos del detector de pantalla dividida, con medidas reales de Android.

---

## 12. Plan de construcción

Cada etapa termina con algo que se puede usar. **La semana de arranque cubre las etapas 0 a 3**, y eso ya es un ciclo completo: cuentas, cursos, matrícula, quiz, nota. Lo demás se construye encima sin rehacer nada.

### Etapa 0 — Esqueleto · día 1

- Monorepo `pnpm`, TypeScript estricto, ESLint, Vitest.
- `docker-compose.yml`: postgres (volumen nombrado, healthcheck), api, worker, web con nginx en un solo origen.
- Kysely con la primera migración: `users`, `sessions`, `auth_events`, `audit_log`, `login_throttle`.
- Validación de configuración al arrancar. Logs con `pino`.
- `packages/contracts` con el primer esquema y OpenAPI generado.
- CI: tipos, lint, tests.

**Termina cuando:** `docker compose up` levanta todo, `GET /api/v1/health` responde, y la web muestra una página servida por nginx que llama a la API sin CORS.

### Etapa 1 — Identidad y acceso · días 2–3

- Registro de estudiante (correo institucional, código por duplicado sin pegar, política de contraseña).
- Login, logout, sesiones en Postgres, CSRF, retraso progresivo.
- Reautenticación. TOTP opcional para docentes.
- Administrador: crear docentes. Semilla inicial con **una** cuenta admin desde variables de entorno, la primera vez.
- **Registro de rutas con auth y política obligatorias**, y su test (§11.1).
- Recuperación por código de estudiante y por código de la docente, con el aviso de §4.7. La vía por correo queda oculta hasta tener SMTP.
- Web: login, registro, «¿Olvidaste tu contraseña?», layouts por rol con guardas de navegación.

**Termina cuando:** existen los tres roles, cada uno aterriza en su panel vacío, y la prueba del registro de rutas pasa.

### Etapa 2 — Estructura académica y biblioteca · días 3–4

- Materias, periodos y temas (admin).
- Cursos con titular; grupos; código de inscripción; matrícula por código y manual.
- Importar lista oficial con previsualización y cruces ambiguos reportados.
- Políticas `teachesCourse`, `teachesSection`, `enrolledInCourse`, `ownsBank`.
- Bancos de preguntas en la biblioteca del docente, e importación de preguntas **con tests** (el hueco de la v1).
- Portar los generadores paramétricos de la v1 con sus tests.
- **Matriz de acceso cruzado (§11.2)** para todo lo construido hasta aquí.

**Termina cuando:** una docente crea su curso con dos grupos, sus estudiantes se inscriben con el código, y la matriz de acceso pasa.

### Etapa 3 — Evaluaciones de punta a punta · días 5–7

- Máquina de estados con sus tests.
- Constructor con plantillas y los tres pasos; alcance a curso o grupo.
- Trabajos `assessment.open` / `assessment.close` en el worker.
- Presentar: una pregunta a la vez, barajado, semilla, reloj, vidas, latido, entrega. `answer_key` nunca en el cliente.
- Integridad web de §8.2: bloqueo de copia, marca de agua visible, pérdida de foco. **No** el detector de pantalla dividida todavía.
- Calificación automática, publicación, `grades` + `grade_revisions`.
- Web del estudiante: Inicio, presentar, Notas. Web del docente: Evaluaciones y un monitor básico.
- **Ampliar tiempo desde el monitor** (§7.2), a toda la evaluación y a un intento. Es parte del monitor básico, no un extra: sin esto, una caída en mitad de un quiz no tiene arreglo.
- Playwright: el flujo completo del estudiante.

**Termina cuando:** un quiz real se puede crear, presentar, cerrar y publicar, y el estudiante ve su nota. **Aquí la v2 puede reemplazar a la v1 para quices.**

### Etapa 3b — Variantes de evaluación · 4–5 días

- **Bonificación** (§7.3): configuración en el constructor con disponibilidad por tema, sorteo al iniciar, pantalla de transición, `applyBonus` con sus tests, explicación en el resultado.
- **Quices en pareja** (§7.4): parejas asignadas (con *Generar al azar*) y libres (código), operador y acompañante vigilado, vidas compartidas, una nota por miembro, *Pasar el control* en el monitor.
- Las filas nuevas de la matriz de acceso (§11.2).

**Termina cuando:** una pareja presenta un quiz con bonificación desde dos teléfonos, una infracción en el acompañante descuenta vida al intento, y los dos ven la misma nota con la pregunta recuperada explicada.

### Etapa 4 — Clases, bitácora y calendario · 1 semana

- Sesiones de clase con lista (cuatro estados, todos presentes por defecto), resumen, temas y material.
- Agenda con visibilidad; pendientes privados con `done_at`.
- Vistas Próximo, Semana y Mes, para docente y estudiante.
- Puerta de asistencia para evaluaciones presenciales, con *marcar presente* desde el monitor.
- Tarjetas de la bandeja de Inicio que dependen de esto.

### Etapa 5 — Libro de notas y gestión de cuentas · 1 semana

- Libro del curso con nota ponderada (`weighting` portado): `current` y `projected`.
- Evaluaciones externas e importación de notas (xlsx/csv) con previsualización.
- Exportación.
- Ficha del estudiante: corregir correo, nombre y código (con reautenticación y auditoría), ver y liberar bloqueos explicando cuál es cuál, generar código de restablecimiento.
- Filtro «sin código verificado».
- Panel de administración completo, con auditoría, incluida **Evaluaciones en curso** con *+ Tiempo* (§7.2, §9.4).

### Etapa 6 — Integridad avanzada · 1 semana

- Detector de pantalla dividida portado, meta `interactive-widget`, congelación con reloj en pausa, liberación automática a los 10 min, `rebaseline()`.
- Monitor con congelados arriba y aviso sonoro.
- Segunda pestaña, canarios, marca invisible con `identificar-fuga`, reloj por pregunta.
- SEB opcional.

### Etapa 7 — Calificación por fotos · 1–1,5 semanas

Se porta el módulo de la v1 (`rubrica.js`, `similitud.js`, `informe.js`, `prompt.js`, `esquema.js`, `claude.js`) con sus 163 tests. Se conservan sus reglas:

- el modelo percibe y el código decide;
- asíncrono con sondeo;
- prompt en caché;
- el par propuesta/decisión nunca se sobrescribe;
- el lote no depende de una evaluación existente.

Las notas entran a `grades` con `origin = 'photo'`.

### Etapa 8 — Gamificación, práctica y ayudas · 2 semanas

- XP como asientos, pelotones, insignias en código, rachas.
- Ranking con consentimiento.
- Modo práctica.
- Ayudas entre compañeros, bloqueadas durante un intento abierto.

Diseño de referencia en `ARQUITECTURA_V2.md` Fase 6 y `GAMIFICACION.md`.

---

## 13. Decisiones pendientes

| # | Pregunta | Cuándo hay que decidirla |
|---|---|---|
| **P1** | **¿Copias de seguridad?** Hoy no se adoptaron (§1.2). Un `pg_dump` diario comprimido fuera del servidor son unas 20 líneas en el compose y un trabajo programado. | **Antes de la primera nota real en la v2** |
| ~~P2~~ | Correo: **Resend** (§1.1b A3). Falta un dominio remitente verificado en Resend. | Etapa 1 |
| P3 | ¿El código de inscripción caduca solo (por ejemplo, a las 2 semanas de empezar el periodo)? | Etapa 2 |
| P4 | ¿Un docente puede compartir un banco con otro docente, o cada biblioteca es estrictamente personal? | Etapa 2 (afecta a `ownsBank`) |
| P5 | ¿Un estudiante puede estar en dos grupos del mismo curso? Este diseño asume que **no**. | Etapa 2 |
| P6 | ¿El administrador puede impersonar a un usuario para dar soporte? Este diseño asume que **no**. | Etapa 5 |
| P7 | ¿Una pregunta **en blanco** se puede recuperar con bonificación, o solo una mal respondida? Este diseño asume que **sí** (en blanco cuenta como fallada). Con «sí», un estudiante puede saltarse las difíciles y apostar a la bonificación. | Etapa 3b |
| P8 | ¿Se permiten parejas entre grupos distintos del mismo curso? Este diseño asume que **no**. | Etapa 3b |
| P9 | ¿El acompañante puede responder también (dos dispositivos activos)? Este diseño asume que **no**: un operador, para evitar conflictos de respuestas y el teléfono libre. | Etapa 3b |

---

## 14. Lecciones de la v1 que se heredan

1. **Una función que depende de configuración externa no se ofrece hasta que esa configuración existe.**
2. **Nunca una sola vía para recuperar la cuenta**, y menos si depende de algo externo.
3. **No confiar en un campo que escribe el usuario como prueba de identidad** (`document_id` contenía `000` en tres cuentas).
4. **Relajar un parámetro puede desactivar comprobaciones que dependían de él** (mínimo de 12 → 6 dejó pasar `123456`).
5. **Los datos que sirven para recuperar la cuenta se confirman por duplicado y sin pegar.**
6. **Cuando recuperar el acceso falla, la gente crea otra cuenta**, y el historial queda partido. El índice único de `student_code` lo impide.
7. **El límite por IP del campus bloquea a todo el salón a la vez**: el límite por IP tiene que ser alto, y el fino, por cuenta.
8. **Un contrato implícito entre servidor y cliente se rompe en silencio** (el botón «Revisar» con 841 tests en verde): contratos compartidos y TypeScript.
9. **Las peticiones largas las corta el navegador o el proxy**: trabajo largo = cola + sondeo.
10. **El modelo de lenguaje nunca pone la nota.**
11. **El WAF de la universidad bloquea expresiones algebraicas** y responde 200 con HTML: el cliente lo detecta y lo dice.
12. **Las importaciones masivas se previsualizan siempre** y reportan las ambigüedades en vez de resolverlas.

## 15. Lo que no hay que hacer

- No añadir una ruta sin declararla en §10 y sin política.
- No escribir un `findById` sin alcance accesible desde servicios de estudiante o docente.
- No devolver 403 para recursos ajenos.
- No enviar `answer_key` ni `teacher_notes` en ninguna respuesta que pueda ver un estudiante.
- No borrar notas ni usuarios. Revisar y desactivar.
- No dar al administrador acceso implícito a cursos. Ampliar tiempo es la única excepción, y no abre nada más.
- No dejar libre el dispositivo del compañero en un quiz en pareja.
- No dejar que la bonificación sume por encima del máximo, ni que cambie una nota sin explicar qué pregunta recuperó.
- No prometer en la interfaz que se bloquean capturas de pantalla. Lo que se hace es atribuirlas.
- No añadir Redis, microservicios ni réplicas. Postgres alcanza para esta escala y la cola ya vive ahí.
- No construir la etapa siguiente sin que la matriz de acceso cubra la actual.
- No copiar botones de la v1 por costumbre. Cada botón entra por la lista de revisión de §9.2b, regla 7.
- No poner la misma acción en dos lugares, fuera de Inicio y del buscador de comandos.
- No mandar una opción a otra página para «limpiar» una pantalla. Se pliega en la misma, mostrando su valor.
