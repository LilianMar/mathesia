# Mathesia (Quiz USB Cali v2) — Plan de trabajo

**Creado:** 9 de octubre de 2026
**Documento rector:** [`CONSTRUCCION_V2.md`](CONSTRUCCION_V2.md) (en adelante **CV2**). Este plan **no lo sustituye**: lo convierte en tareas ejecutables, en orden, con archivos, pruebas y criterios de salida.
**Versión anterior (solo consulta):** `/home/fac.ingenieria/thesis-manager/proyectos/quiz_app` (en adelante **v1**).

---

## 0. Cómo usar este plan

- Se trabaja **etapa por etapa**, en el orden de la §5. Una etapa no se da por cerrada hasta cumplir **todo** su *criterio de salida* y su fila de la matriz de acceso (§6).
- Cada tarea tiene un identificador (`E2.4`). Los commits y PR lo citan: `E2.4: importación de preguntas con previsualización`.
- Las casillas `[ ]` se marcan en este mismo archivo al terminar cada tarea. La §11 (*Bitácora*) registra lo que se decidió o se descubrió en el camino.
- Cuando una sección de CV2 ya dice **qué** hacer, aquí se cita (`CV2 §7.3`) y solo se añade el **cómo**. Si el plan y CV2 se contradicen, manda CV2, salvo en las correcciones explícitas de la §2, que se reflejan en CV2 en la misma PR.
- Las tareas marcadas **🔒** tocan invariantes de seguridad: requieren su prueba en `tests/security/` antes de cerrarse.

---

## 1. Resumen del análisis

### 1.1 Qué hay hoy

| Elemento | Estado |
|---|---|
| `mathesia/` | Solo `CONSTRUCCION_V2.md`. Carpeta sin repositorio propio (está dentro del repo `thesis-manager`, sin seguimiento). |
| v1 (`quiz_app/`) | Express 4 + SQLite (`better-sqlite3`) + HTML/JS sin framework. En producción en `https://lidis.usbcali.edu.co/quiz/`, contenedor `quiz-app`, puerto `9061`, con `src/` y `public/` montados. 21 migraciones, ~45 archivos de pruebas (`node --test`). |
| Documentación v1 útil | `ARQUITECTURA_V2.md` (gamificación, ayudas, fotos: §2.4b/d/f y Fase 6), `GAMIFICACION.md`, `EVALUACION_ANTITRAMPAS.md` (texto de la declaración §12.2, canarios §7.7), `MODULO_CALIFICACION_IA.md`, `Especificacion_Modulo_Calificacion.md`, `PLAN.md` (bitácora de hallazgos H-xxx). |
| Servidor | Node 20 en el host, **sin pnpm** (corepack 0.34 disponible). `docker compose` v5 (plugin) y `docker-compose` 1.29 (v1, roto al recrear). Imágenes locales: `node:22-alpine`, `node:22-slim`, `postgres:16-alpine`, `nginx:alpine`. |

### 1.2 Qué se porta de la v1 (y qué no)

| Módulo v1 | Destino v2 | Etapa | Notas del portado |
|---|---|---|---|
| `src/services/items/rng.js` | `apps/api/src/domain/items/rng.ts` | 2 | **Idéntico en comportamiento**: misma semilla → misma secuencia. Test de regresión con valores fijos sacados de la v1. |
| `src/services/items/registry.js` + `generators/{fundamentals,trigonometry,triples}.js` + `mathHelpers.js` | `apps/api/src/domain/items/` | 2 | Claves `fund.*` y `trig.*` **no se renombran**. Se portan los tests de `tests/unit/generators.test.js` y `tests/security/parametric.test.js`. La validación de contrato de `generateItem` se conserva. |
| `src/services/assessment/itemBuilder.js` | `domain/assessment/itemBuilder.ts` | 3 | Reutiliza `rng.ts` (la v1 tenía un segundo PRNG duplicado, `seededRandom`: se unifica). Reglas «N al azar de banco/tema». |
| `src/services/assessment/grading.js` | `domain/grading/autoGrade.ts` | 3 | Funciones puras. Tolerancia numérica. |
| `src/services/gradebook/weighting.js` + `stats.js` | `domain/grading/weighting.ts` | 5 | `current` y `projected`, `checkWeights`. Tests de `gradebook.test.js`. |
| `src/services/password.policy.js` + `constants.js` | `domain/auth/passwordPolicy.ts` | 1 | Lista de prohibidas **con las cortas**, secuencias en ambas direcciones, nombre/correo/código. Mínimo por rol (6 / 12). Tests de `password-policy.test.js`. |
| `src/services/assessment/watermark.js` | `domain/integrity/watermark.ts` | 6 | Marca invisible + `identificar-fuga`. |
| `public/js/split-screen.js` | `apps/web/src/features/exam/integrity/splitScreen.ts` | 6 | «Se porta tal cual» (lo dice su propio comentario). Solo depende de `window`. |
| `src/services/assessment/canary.js` | `domain/integrity/canary.ts` | 6 | |
| `src/services/assessment/seb.js` | `platform/seb.ts` + política | 6 | |
| `src/services/assessment/declaration.js` | `domain/assessment/declaration.ts` | 3 | Texto versionado por hash; **requisito legal** (Ley 1581): se incluye ya en Etapa 3 porque la Etapa 3 registra eventos de foco. |
| `src/lib/proctorEvents.js` | `domain/integrity/proctorEvents.ts` | 3 (básico) / 6 | Interpretación de eventos en causas posibles, para el monitor. |
| `src/services/analytics/similarity.js` | `domain/integrity/similarity.ts` | 6 | Comparar **parejas**, no individuos, en quices en pareja (CV2 §7.4). |
| `src/lib/nameMatch.js` | `domain/roster/nameMatch.ts` | 2 | Regla «ante la duda, no se decide». La usan lista oficial (E2) e importación de notas (E5). |
| `src/services/roster.service.js` (parser CSV) | `platform/csv.ts` | 2 | |
| `src/lib/xlsx.js` + lector de `scripts/import-notas-xlsx.js` | `platform/xlsx.ts` | 5 | Sin dependencias, como en la v1. |
| `src/lib/csvSafe.js` | `platform/csvSafe.ts` | 5 | Evita inyección de fórmulas al exportar. |
| `src/lib/markdown.js`, `mathText.js`, `docx.js` | `domain/photoGrading/` | 7 | |
| `src/services/grading/ai/*` (`rubrica`, `similitud`, `informe`, `prompt`, `esquema`, `claude`, `paginas`) | `domain/photoGrading/` + `jobs/grading.photo.ts` | 7 | Con sus tests (`ia-*.test.js`, `ia-grading`, `ia-panel`). |
| `src/middleware/methodOverride.js` | `platform/methodOverride.ts` | **0** | **Imprescindible** por el WAF (ver §2.2). |
| `public/js/math.js` + KaTeX vendorizado | `apps/web/src/shared/math/` (paquete `katex` empaquetado por Vite) | 3 | Convención `\( \)` y `\[ \]`, nunca `$`. Sin CDN. |
| `public/css/arcade.css` (tokens) + `fonts.css` | `apps/web/src/design/tokens.css` | 1 | Solo los tokens y la tipografía (Silkscreen, Space Mono, Outfit) **auto-alojada**; no el CSS de componentes. Tema oscuro por defecto + tema claro (líneas ~2590 de `arcade.css`). |
| `seeds/*.json` (bancos) | Fixture de importación y semilla de desarrollo | 2 | Formato `{questions:[{question, options, answerIndex, explanation, difficulty, topic}]}`. |
| `scripts/backup.js` | Referencia para el respaldo `pg_dump` (§7.4) | 0/3 | |
| **No se porta** | `legacy.*`, `diagnostic/*`, `defense/*`, `reconciliation`, `pendingGrades`, vistas HTML, `express-session`, `bcryptjs`, SQLite | — | Los datos no se migran (CV2 §1.1). Diagnóstico y defensa no están en CV2: si se quieren, entran como etapa nueva y se añaden a CV2 primero. |

---

## 2. Choques entre CV2 y el entorno real (resolver antes o durante la Etapa 0)

Estos puntos **no** están en CV2 y romperían la construcción o el despliegue si se ignoran.

### 2.1 La v1 vive bajo un prefijo, no en un dominio propio

La v1 está en `https://lidis.usbcali.edu.co/quiz/`, detrás de `nginx-ssl-test` (`/home/fac.ingenieria/lidis-growth-lab/nginx-projects/quiz.conf`), junto con otras ~30 aplicaciones del mismo dominio.

CV2 §4.2 pide la cookie `__Host-sid` con `Path=/`. Eso **exige** servir en la raíz de un origen. Bajo `lidis.usbcali.edu.co/mathesia/`:

- la cookie `Path=/` se enviaría a **todas** las aplicaciones del dominio (y sus logs);
- cualquier XSS en otra aplicación de `lidis` sería *mismo origen* para mathesia: podría llamar a la API con la sesión del usuario. `SameSite` y el CSRF ligado a sesión no protegen contra eso.

**Decisión D1 — resuelta el 9/10: prefijo `https://lidis.usbcali.edu.co/mathesia/`.**

- Cookie `__Secure-sid; HttpOnly; Secure; SameSite=Lax; Path=/mathesia/` (no `__Host-`, que exige `Path=/`).
- El riesgo de mismo origen con las demás apps de `lidis` queda **aceptado y escrito** en CV2 §1.2.
- Mitigaciones que sí se aplican: cookie limitada a `Path=/mathesia/`; CSRF ligado a sesión; CSP estricta y `frame-ancestors 'none'`; ninguna respuesta de la API se sirve como HTML; reautenticación en lo sensible.

**Cómo se construye:** la app sigue soportando raíz o prefijo (por si algún día hay subdominio), pero el valor por defecto de producción es `PUBLIC_URL=https://lidis.usbcali.edu.co/mathesia`.
- `PUBLIC_URL` es la única fuente del prefijo (lección `basePath.js` de la v1: un solo valor, nunca dos que deban coincidir).
- API: el enrutador se monta en `${BASE_PATH}/api/v1`. Cookie: `__Host-sid` si `BASE_PATH === ""`, si no `__Secure-sid` con `Path=${BASE_PATH}/`.
- Web: Vite `base` y `<BrowserRouter basename>` salen de la misma variable en tiempo de build (`VITE_BASE_PATH`, generada desde `PUBLIC_URL` por el script de build, no escrita a mano).
- nginx de dentro del stack sirve en la raíz; el prefijo lo maneja el nginx externo con `proxy_pass` **sin** barra final, como `quiz.conf`.

### 2.2 El WAF de la universidad bloquea PUT, PATCH y DELETE

El WAF delante de `lidis.usbcali.edu.co` bloquea PUT/DELETE (y PATCH según la v1) y responde **HTML con status 200**. También bloquea rutas de dos segmentos bajo `tests/` y, a veces, peticiones sin User-Agent de navegador. Además, la lección 11 de CV2: bloquea **expresiones algebraicas** en el cuerpo.

CV2 §10 usa `PATCH`, `PUT` y `DELETE`. Se conservan en el contrato (la API es REST), pero:

- [ ] El cliente web envía **todo lo que no es GET como POST** con `X-HTTP-Method-Override: PATCH|PUT|DELETE` (portado de `methodOverride.js`). El middleware corre **antes** del enrutador y del CSRF, así ningún otro componente sabe que existe.
- [ ] Ninguna ruta lleva el segmento `tests`.
- [ ] El cliente detecta «HTML donde esperaba JSON» y lo dice (CV2 §3.4), con un error tipado `WAF_BLOCKED` en la web.
- [ ] **Expresiones algebraicas — decisión D2, resuelta el 9/10: igual que la v1.** La v1 (en producción en `/quiz/`) **no codifica** nada: manda JSON plano y se apoya en dos cosas, que se portan tal cual desde `quiz_app/public/js/api.js` y `src/middleware/methodOverride.js`:
  1. POST + `X-HTTP-Method-Override` para PUT/PATCH/DELETE, con el CSRF pedido a `GET /auth/csrf`, cacheado y renovado una vez si el servidor lo rechaza;
  2. si la respuesta no es JSON (el WAF Radware contesta HTML con 200), error explícito. La v1 dice: «El servidor no devolvió datos válidos. Si el archivo es grande, intenta importarlo en lotes más pequeños…». La v2 lo tipa como `WAF_BLOCKED` y adapta el mensaje al contexto (respuesta de examen, importación, guardar pregunta).
  - El barrido de E0.9 sigue haciéndose, pero **solo para documentar** qué cuerpos bloquea el WAF (ya se vio con identidades como `a+b = b+a`). Si una respuesta de examen real resulta bloqueada, se reabre D2 con ese caso concreto; no se codifica de forma preventiva.
  - Las importaciones grandes se envían por lotes desde el cliente (la v1 ya chocó con el tamaño).
- [ ] Los scripts de prueba contra el entorno público mandan User-Agent de navegador.

### 2.3 Docker no tiene subredes libres

`docker compose up` de un proyecto nuevo falla con `all predefined address pools have been fully subnetted`. CV2 §2.2 pide cuatro servicios que se hablan por nombre (postgres · api · worker · web), así que **sí** se necesita una red propia.

- [ ] La red del compose se declara con **subred explícita** (no toma del pool por defecto):

  ```yaml
  networks:
    mathesia:
      ipam:
        config:
          - subnet: 10.210.0.0/24   # comprobar antes que no se solapa: docker network inspect $(docker network ls -q) | grep Subnet
  ```

  Si Docker la rechaza igualmente, plan B: `postgres` en la red `bridge` publicada en `127.0.0.1:5433` (patrón de `evaluapp/scripts/db-dev.sh`) y api/worker con `network_mode: host`. La solución de fondo (`default-address-pools` en `daemon.json`) reinicia producción: **no** se hace sin ventana de mantenimiento.
- [ ] Se usa **`docker compose` (v2/v5)**, nunca `docker-compose` 1.29. Siempre con `-p mathesia` y **down antes de up** al redesplegar.

### 2.4 Puertos y procesos en el servidor

- Ocupados hoy: 3000, 3019, 5000, 80, 8000, 9000–9004, 9009, 9011, 9013, 9023, 9050–9056, 9059–9062, 9099, 9443 y `127.0.0.1:5432`.
- [ ] **Web (nginx del stack): `9063`** (comprobar de nuevo al desplegar). Publicado en `172.18.0.1:9063` (gateway de `lidis-growth-lab_default`) y no en `0.0.0.0`, para que no responda sin TLS (la v1 lo deja abierto y depende del cortafuegos).
- [ ] **Postgres no se publica** en producción. En desarrollo: `127.0.0.1:5433`.
- [ ] La API y el worker no publican puertos.

### 2.5 Herramientas

- [ ] **pnpm:** `corepack enable` puede requerir permisos sobre el directorio de Node. Alternativa sin sudo: `corepack enable --install-directory ~/.local/bin pnpm` (o `npm i -g pnpm --prefix ~/.local`). Se fija la versión en `package.json` → `"packageManager": "pnpm@<versión>"`.
- [ ] **Node:** el host tiene 20; las imágenes usan **22 LTS** (`node:22-alpine`, ya descargada). `.nvmrc` = `22`, `engines.node >= 22`. El desarrollo local en el host funciona con 20 mientras no se use nada exclusivo de 22; los tests de CI corren en el contenedor de 22.
- [ ] **Playwright:** el Chromium del host no arranca (faltan librerías, sin sudo). Se hornea una imagen `docker/playwright.Dockerfile` (`FROM node:22-slim` + librerías + `playwright-core`), como en la memoria de capturas móviles.

### 2.6 Huecos e inconsistencias internas de CV2

Se corrigen en CV2 **en la PR de la etapa donde aparecen**. Aquí queda la propuesta:

| # | Hueco | Propuesta | Etapa |
|---|---|---|---|
| H1 | El trabajo `attempt.expire` se usa en §7.2 pero no está en la tabla de §3.5. | Añadirlo: programado a `deadline_at`; al ejecutarse relee `deadline_at` y, si no venció, se reprograma. | 3 |
| H2 | `attempts.student_id NOT NULL` y `UNIQUE (assessment_id, student_id, number)` no encajan con intentos de pareja (se cuentan por pareja). | `student_id` = quien inició. Miembros = `student_id` (individual) o `team_members` (pareja). Índice único parcial `(assessment_id, team_id, number) WHERE team_id IS NOT NULL`. Función `attemptsRepo.findForMember`. | 3b |
| H3 | `attempt_events` no registra **qué miembro** produjo el evento (§7.4 lo exige). | Columna `student_id UUID REFERENCES users(id)`. Se añade ya en la Etapa 3 para no migrar después. | 3 |
| H4 | Los evaluaciones `in_person` exigen asistencia (§10) pero la asistencia llega en la Etapa 4. | En Etapa 3 el constructor solo ofrece `remote` (regla «nada se ofrece si no se puede cumplir»). `in_person` se activa en Etapa 4. | 3 → 4 |
| H5 | Faltan en §10 rutas que la interfaz necesita (cambiar contraseña, Inicio, buscador, ficha, previsualizaciones…). | Lista completa por etapa en la §4. Cada una se añade a §10 en la PR que la crea. | todas |
| H6 | `questions.stem NOT NULL`, pero una pregunta paramétrica genera su enunciado. | `stem` guarda una etiqueta descriptiva («Generador: Fracciones»); el enunciado real se materializa en `attempt_items.rendered`. | 2 |
| H7 | `difficulty SMALLINT` vs `facil/medio/dificil` de los bancos v1. | Mapeo 1/2/3 en la importación. | 2 |
| H8 | ¿Quién crea temas? CV2 dice admin (E2), pero importar preguntas trae temas nuevos. | **Decisión D3.** Recomendación: cualquier docente que dicte un curso de la materia puede crear temas; el admin puede fusionarlos. La importación **reporta** los temas desconocidos y propone crearlos; no los crea sola (lección 12). | 2 |
| H9 | Login, registro y restablecimiento son POST públicos: ¿cómo llevan `X-CSRF-Token` sin sesión? | Rutas públicas que cambian estado declaran `csrf: "origin"`: se exige `Origin`/`Sec-Fetch-Site: same-origin` y `Content-Type: application/json`. El registro de rutas lo hace explícito y el test de §11.1 comprueba que solo las públicas lo usan. | 1 |
| H10 | La semilla de §11.2 no crea `e5` (el compañero de pareja de `e1`). | Añadir `e5` en X1. | 3b |
| H11 | Configuración que falta en §2.3. | `INSTITUTIONAL_EMAIL_DOMAINS` (p. ej. `correo.usbcali.edu.co,usbcali.edu.co`), `TRUST_PROXY` (lección H-057 de la v1: si se queda corto, todo el campus comparte IP), `SEED_ADMIN_EMAIL`, `SEED_ADMIN_NAME`, `SEED_ADMIN_PASSWORD` (solo primera vez), `LOG_LEVEL`, `MAX_UPLOAD_MB`, `IA_MODELO`. | 0–1 |
| H12 | El nombre de §2.2 es `quiz-v2/` y el de la carpeta es `mathesia/`. | La raíz del monorepo **es** `mathesia/`. Paquetes `@mathesia/api`, `@mathesia/web`, `@mathesia/worker`, `@mathesia/contracts`. Base de datos `mathesia`, rol de la app `mathesia_app` (en vez de `quiz_app` del `REVOKE`). | 0 |
| H13 | Aviso al docente cuando un admin amplía tiempo (§4.1). | No requiere tabla: la bandeja de Inicio consulta `time_extensions` de sus cursos con `granted_by` admin en las últimas 24 h. | 5 |

### 2.7 Decisiones que necesito de la docente

| # | Pregunta | Supuesto con el que se avanza mientras tanto | Bloquea |
|---|---|---|---|
| ~~D1~~ | ✅ **Prefijo `/mathesia/`** en `lidis.usbcali.edu.co` (9/10). | — | — |
| ~~D2~~ | ✅ **Como la v1**: sin codificar; method override + detección de HTML (9/10). | — | — |
| **D3** | Quién crea temas (H8) | Docentes de la materia pueden crearlos. | E2 |
| ~~D4~~ | ✅ **Repo propio** `git@github.com:LilianMar/mathesia.git` (9/10). Se sube por SSH con la cuenta `cghidalgos` (la misma que firma los commits de quiz_app). | — | — |
| **P1** | Copias de seguridad (CV2 §13) | **Se implementan igual en E0** como servicio opcional del compose (20 líneas). Se puede apagar, pero no se puede improvisar el día que hagan falta. | Primera nota real |
| P2 | SMTP de la universidad | Sin SMTP: la vía por correo queda oculta por `/features`. | — |
| P3 | Caducidad automática del código de inscripción | No caduca solo; la docente lo cierra o rota. | — |
| P4 | Bancos compartidos entre docentes | Biblioteca personal (`ownsBank`). La política se escribe como función para cambiarla en un lugar. | — |
| P5 | Estudiante en dos grupos del mismo curso | No (trigger). | — |
| P6 | Impersonar usuarios | No. | — |
| P7 | Bonificación recupera preguntas en blanco | Sí (configurable en `applyBonus` con un parámetro, por si cambia). | — |
| P8 | Parejas entre grupos | No. | — |
| P9 | Acompañante responde | No. | — |

---

## 3. Convenciones de trabajo

### 3.1 Estructura final del repositorio

```
mathesia/
├── CONSTRUCCION_V2.md           documento rector
├── PLAN_DE_TRABAJO.md           este archivo
├── package.json                 workspaces, scripts raíz, packageManager
├── pnpm-workspace.yaml
├── tsconfig.base.json           strict, noUncheckedIndexedAccess, exactOptionalPropertyTypes
├── eslint.config.js             incluye reglas de capas (§3.3)
├── vitest.workspace.ts
├── .env.example
├── .nvmrc
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── main.ts          punto de entrada HTTP
│   │   │   ├── http/            route() + un archivo por recurso
│   │   │   ├── services/        casos de uso (un archivo por caso o por recurso)
│   │   │   ├── domain/          puro: auth/, assessment/, grading/, items/, integrity/, roster/, photoGrading/
│   │   │   ├── policies/
│   │   │   ├── repositories/
│   │   │   ├── jobs/            handlers + registro de colas
│   │   │   ├── db/              kysely.ts, types.ts (generado), migrations/NNNN_nombre.ts, seed/
│   │   │   └── platform/        config, logger, errors, session, csrf, throttle, methodOverride, files, mail, csv, xlsx, totp, crypto, clock
│   │   └── tests/
│   │       ├── security/        routeRegistry.test.ts, accessMatrix.test.ts, …
│   │       ├── integration/
│   │       ├── domain/
│   │       └── helpers/         testDb.ts, actors.ts, seedMatrix.ts, http.ts
│   ├── worker/
│   │   └── src/main.ts          importa jobs/ de api (mismo código, otro punto de entrada)
│   └── web/
│       ├── index.html           meta viewport con interactive-widget=resizes-visual en la vista de examen
│       ├── vite.config.ts
│       └── src/
│           ├── app/             router, layouts por rol, guardas, Ctrl+K
│           ├── features/        auth, home, courses, sections, students, classes, calendar, banks,
│           │                    assessments, monitor, exam, grades, admin, security
│           ├── shared/          api/ (cliente tipado), ui/ (componentes), hooks/, math/
│           └── design/          tokens.css, base.css, fonts/
├── packages/
│   └── contracts/
│       └── src/                 un archivo por recurso + assessmentSettings.ts + errors.ts + openapi.ts
├── docker/
│   ├── api.Dockerfile           multi-stage, node:22-alpine, usuario no root, tini
│   ├── web.Dockerfile           build Vite → nginx:alpine
│   ├── nginx.conf               SPA + /api → api:3000, CSP, cabeceras
│   ├── playwright.Dockerfile
│   └── backup/                  script pg_dump + rotación
├── scripts/                     db-dev.sh, check-subnet.sh, waf-sweep.mjs, gen-db-types.sh
└── docker-compose.yml           postgres · api · worker · web · (backup)
```

### 3.2 Ramas, commits y PR

- Rama por etapa (`etapa-0-esqueleto`, `etapa-1-identidad`…) y, dentro, ramas por tarea cuando la tarea sea grande.
- Mensajes en español, imperativo, con el identificador: `E3.7: entrega automática al vencer deadline_at`.
- **Lista de revisión de cada PR** (plantilla en `.github/pull_request_template.md` o en `docs/PR_CHECKLIST.md`):
  - [ ] `pnpm check` en verde (tipos, lint, tests).
  - [ ] Toda ruta nueva está en CV2 §10 con su rol y política.
  - [ ] Ninguna consulta sin alcance accesible desde servicios de estudiante/docente.
  - [ ] Lo ajeno responde 404; el 403 solo para «es tuyo pero no ahora».
  - [ ] Ningún contrato de salida de estudiante contiene `answer_key`, `teacher_notes` ni datos de otros estudiantes.
  - [ ] Filas nuevas de la matriz de acceso (§6) añadidas y en verde.
  - [ ] Si toca interfaz: lista de 5 preguntas de CV2 §9.2b regla 7 para cada botón nuevo; mapa CV2 §9.2c actualizado; acción en el buscador de comandos.
  - [ ] Migraciones nuevas son hacia adelante; nunca se edita una migración ya aplicada.
  - [ ] Auditoría en toda acción sensible (`audit_log`).

### 3.3 Reglas de capas, automatizadas

Se imponen con `eslint-plugin-boundaries` (o `no-restricted-imports` por carpeta) para que CV2 §3.1 no dependa de la memoria:

| Desde | Puede importar | No puede importar |
|---|---|---|
| `http/` | `services/`, `policies/`, `contracts`, `platform/{errors,session,csrf}` | `repositories/`, `db/` |
| `services/` | `repositories/`, `policies/`, `domain/`, `jobs/enqueue`, `platform/` | `express`, `http/` |
| `domain/` | `domain/`, `contracts` (solo tipos) | todo lo demás, incluido `Date.now()` (regla `no-restricted-syntax`: el reloj entra por parámetro) |
| `policies/` | `repositories/` | `services/`, escrituras |
| `repositories/` | `db/`, `kysely` | `services/`, `policies/` |

Además: regla que prohíbe exportar desde `repositories/` funciones `findById` sin sufijo de alcance (`ForStudent`, `ForTeacher`, `ForMember`, `ForAdmin`, `Internal`). Las `Internal` solo se pueden importar desde `jobs/`.

### 3.4 Pruebas y base de datos de pruebas

- `scripts/db-dev.sh` levanta `postgres:16-alpine` en la red `bridge`, `127.0.0.1:5433`, volumen nombrado `mathesia-dev-pg`.
- Las pruebas de integración crean **una base por proceso de Vitest** clonando una plantilla `mathesia_template` ya migrada (`CREATE DATABASE … TEMPLATE …`): rápido y sin interferencias.
- Cada test arma su escenario con fábricas (`tests/helpers/factories.ts`), no con la semilla de desarrollo.
- `pnpm check` = `tsc -b` + `eslint` + `vitest run` (dominio + integración + seguridad). Es la «CI» local. Si el repositorio tiene remoto en GitHub, el mismo comando en Actions con un servicio Postgres.

### 3.5 Definición de terminado (para cualquier tarea)

1. Contrato Zod en `packages/contracts` (entrada y salida).
2. Ruta declarada con `auth` + `policy` (+ `reauth` si aplica).
3. Servicio con su transacción; repositorio con alcance.
4. Prueba de dominio (si hay lógica), de integración (camino feliz + errores de estado) y de seguridad (fila de la matriz).
5. Si tiene interfaz: pantalla, estado vacío que enseña, error tipado, aparece en Ctrl+K.
6. Auditoría donde corresponde.
7. Casilla marcada aquí.

---

## 4. Rutas que faltan en CV2 §10 (se añaden en su etapa)

Sufijo **R** = reautenticación. Todas pasan por el registro y la matriz.

| Etapa | Ruta | Rol | Política |
|---|---|---|---|
| 0 | `GET /health` | P | — |
| 1 | `GET /auth/csrf` (devuelve el token ligado a la sesión) | E D | Propia |
| 1 | `POST /me/password` (cambiar la propia) | E D | Propia; cierra las demás sesiones |
| 1 | `POST /me/password-notice/ack` | E | Propia |
| 1 | `POST /me/totp/setup` · `POST /me/totp/confirm` · `POST /me/totp/backup-codes` | D **R** | Propia |
| 1 | `GET /admin/teachers` · `POST /admin/teachers` · `POST /admin/users/:id/deactivate` · `POST /admin/users/:id/admin` | A **R** | `is_admin` |
| 2 | `GET/POST /admin/subjects` · `GET/POST /admin/terms` · `PATCH` de ambos | A **R** | `is_admin` |
| 2 | `GET/POST /admin/courses` · `POST /admin/courses/:courseId/titular` | A **R** | `is_admin` |
| 2 | `POST /topics` · `PATCH /topics/:id` | D | Dicta un curso de la materia (D3) |
| 2 | `GET /courses/:courseId/sections` · `PATCH /sections/:sectionId` (nombre, `join_open`) · `POST /sections/:sectionId/join-code/rotate` | D | `teachesSection` / titular |
| 2 | `POST /courses/:courseId/teachers` · `DELETE /courses/:courseId/teachers/:userId` | D | Titular |
| 2 | `POST /sections/:sectionId/official-list/preview` (la existente pasa a ser el *commit*) | D | `teachesSection` |
| 2 | `POST /sections/:sectionId/enrollments` (manual) · `POST /sections/:sectionId/enrollments/:studentId/withdraw` | D | `teachesSection` |
| 2 | `GET /banks/:bankId/questions` · `POST /banks/:bankId/questions` · `PATCH /questions/:id` · `DELETE /questions/:id` | D | `ownsBank` (de la pregunta) |
| 2 | `POST /banks/:bankId/questions/import/preview` | D | `ownsBank` |
| 2 | `GET /generators` · `GET /generators/:key/preview` | D | Cualquier docente |
| 3 | `GET /assessment-templates` | D | — |
| 3 | `POST /assessments/:id/duplicate` · `DELETE /assessments/:id` (lógico; **R** si tiene intentos) | D | `findForTeacher` + estado |
| 3 | `GET /banks/availability?bankIds=&topicIds=&autoGradable=` (para el constructor y la bonificación) | D | `ownsBank` sobre cada banco |
| 3 | `GET /attempts/:id/result` | E | Miembro; evaluación publicada |
| 3 | `GET /assessments/:id/results` | D | `findForTeacher` |
| 3 | `GET /me/home` (Inicio del estudiante) | E | Propia |
| 3 | `GET /me/inbox` (bandeja de Inicio docente) | D | Solo sus cursos |
| 4 | `GET /sections/:sectionId/class-sessions` · `DELETE /class-sessions/:id` | D | `teachesSection` |
| 4 | `POST /class-sessions/:id/materials` · `DELETE /class-session-materials/:id` | D | `teachesSection` |
| 4 | `GET /courses/:courseId/agenda` · `DELETE /agenda/:id` · `POST /agenda/:id/done` | E D | Curso; visibilidad §5.2 |
| 4 | `POST /assessments/:id/attendance/:studentId/present` (marcar presente desde el monitor) | D | `findForTeacher` + evaluación presencial abierta |
| 5 | `GET /students/:studentId` (ficha) · `GET /students/:studentId/locks` | D | Estudiante en un grupo que dicta |
| 5 | `GET /courses/:courseId/gradebook/export` | D | `teachesCourse` |
| 5 | `POST /assessments/:id/grades/import/preview` | D | `findForTeacher` + externa |
| 5 | `GET /search?q=` | E D | Resultados filtrados por política (CV2 §9.2b regla 5) |
| 5 | `GET /admin/accounts?q=` · `GET /admin/accounts/:id/locks` · `POST /admin/locks/:key/release` · `GET /admin/audit` | A **R** | `is_admin` |
| 6 | `GET /assessments/:id/integrity` · `POST /assessments/:id/leak-identify` · `GET /assessments/:id/seb-config` | D | `findForTeacher` + cerrada/publicada (SEB: cualquier estado) |
| 7 | Rutas de lotes de fotos: `POST /photo-batches` · `GET /photo-batches/:id` · `POST /photo-batches/:id/pages` · `POST /photo-batches/:id/grade` · `POST /photo-batches/:id/decisions` · `POST /photo-batches/:id/export-grades` | D | Dueño del lote; exportar exige `findForTeacher` de la evaluación destino |
| 8 | Se definen al diseñar la etapa (ver E8.0). | | |

---

## 5. Etapas

> Duraciones: las de CV2 §12. Son orientativas; el criterio de salida manda.

---

### Etapa 0 — Esqueleto · día 1

**Objetivo:** `docker compose up` levanta todo, `GET /api/v1/health` responde y la web llama a la API sin CORS. Además, quedan resueltos los choques de entorno de la §2.

#### E0.1 Repositorio y herramientas
- [x] (D4) `git init` en `mathesia/` con remoto `git@github.com:LilianMar/mathesia.git`, rama `main`; `.gitignore` (node_modules, dist, .env, coverage, playwright-report).
- [ ] Instalar pnpm sin sudo (§2.5) y fijar `packageManager`.
- [ ] `pnpm-workspace.yaml` con `apps/*` y `packages/*`.
- [ ] `tsconfig.base.json` estricto (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `noImplicitOverride`, `verbatimModuleSyntax`), `module: NodeNext` en api/worker/contracts, `bundler` en web. Project references.
- [ ] ESLint (flat config) con `typescript-eslint` estricto + reglas de capas (§3.3) + `no-floating-promises`.
- [ ] Vitest workspace (proyectos: `contracts`, `api-domain`, `api-integration`, `api-security`, `web`).
- [ ] Scripts raíz: `dev`, `build`, `check`, `test`, `db:migrate`, `db:types`, `db:seed`, `lint`, `typecheck`.
- [ ] `.nvmrc` (22) y `engines`.

#### E0.2 Contratos
- [ ] `packages/contracts`: Zod (fijar versión mayor compatible con `@asteasolutions/zod-to-openapi` en el momento; anotar en §11).
- [ ] `errors.ts`: enumeración estable de códigos (`NOT_FOUND`, `FORBIDDEN_ACTION`, `INVALID_STATE`, `VALIDATION`, `REAUTH_REQUIRED`, `RATE_LIMITED`, `UNAUTHENTICATED`, `CSRF`, `CONFLICT`) y la forma de la respuesta de error `{ error: { code, message, details? } }`.
- [ ] `health.ts` como primer contrato.
- [ ] `openapi.ts`: registro que genera `openapi.json` en build (`pnpm --filter contracts openapi`). Se sirve en `GET /api/v1/openapi.json` solo en desarrollo.

#### E0.3 Plataforma de la API
- [ ] `platform/config.ts`: esquema Zod de todas las variables (CV2 §2.3 + H11). Falla al arrancar nombrando la que falta. Deriva `basePath`, `cookieName`, `cookiePath` y `isSecure` desde `PUBLIC_URL` (§2.1).
- [ ] `platform/logger.ts`: `pino` en JSON; `pino-http` con `requestId` (UUID por petición, devuelto en `X-Request-Id`) y `actorId` cuando exista. Redacción de `authorization`, `cookie`, `password*`, `totp*`.
- [ ] `platform/errors.ts`: `AppError` + fábricas (`notFound()`, `invalidState()`, …) + manejador global que **nunca** devuelve trazas.
- [ ] `platform/methodOverride.ts` (portado; §2.2) montado antes del enrutador.
- [ ] `platform/clock.ts`: `now()` inyectable (los servicios lo reciben; los tests lo fijan).
- [ ] `http/route.ts` **esqueleto del registro**: `route({ method, path, auth, policy, input, output, reauth?, csrf?, handler })`. Ya en E0 lanza error si falta `auth` (la política se exige desde E1). Valida `input` con Zod y **también la salida** en desarrollo/test (en producción se recorta con `output.parse` para no filtrar campos).
- [ ] `helmet` con CSP estricta (la CSP de la web la pone nginx; la de la API: `default-src 'none'`).
- [ ] `trust proxy` según `TRUST_PROXY`.
- [ ] `GET /api/v1/health`: comprueba la base (`select 1`) y devuelve `{ ok, version, db }`.

#### E0.4 Base de datos
- [ ] `db/kysely.ts`: `Kysely<DB>` sobre `pg.Pool`. Tipos generados con `kysely-codegen` (`pnpm db:types`) y versionados en `db/types.ts`.
- [ ] Migrador de Kysely con archivos `0001_…ts`. Las migraciones corren **al arrancar la API** (como la v1) con bloqueo consultivo (`pg_advisory_lock`) para que api y worker no migren a la vez; el worker espera a que estén aplicadas.
- [ ] **Migración 0001**: extensiones `pgcrypto`, `citext`; tablas `users`, `sessions`, `auth_events`, `audit_log`, `login_throttle` tal como CV2 §4.2, §4.6, §6.2, §6.5.
- [ ] Rol `mathesia_app` sin permisos de `UPDATE/DELETE` sobre `audit_log` (migración con `REVOKE`; el propietario del esquema es otro rol `mathesia_owner` que solo usa el migrador). Documentar en `.env.example` las dos URLs (`DATABASE_URL` y `DATABASE_MIGRATION_URL`).
- [ ] `repositories/audit.ts` con `log(tx, actor, action, entity, before?, after?)`.

#### E0.5 Cola
- [ ] `pg-boss` con su propio esquema (`pgboss`). Arranque en el worker; en la API solo como emisor.
- [ ] **Prueba de concepto crítica:** `jobs/enqueue.ts` → `enqueue(tx, name, data, opts)` que envía el trabajo **con la conexión de la transacción de Kysely** (adaptador `{ executeSql: (text, values) => tx.executeQuery(CompiledQuery.raw(text, values)) }` en la opción `db` de `boss.send`). Test de integración: si la transacción hace *rollback*, el trabajo no existe; si hace *commit*, sí.
- [ ] Trabajo `sessions.cleanup` programado (diario) como primer manejador idempotente.

#### E0.6 Worker
- [ ] `apps/worker/src/main.ts`: carga config, espera migraciones, registra manejadores de `apps/api/src/jobs/`, apagado limpio con `SIGTERM`.

#### E0.7 Web
- [ ] Vite + React 19 + TypeScript + React Router + TanStack Query.
- [ ] `shared/api/client.ts`: `fetch` con `credentials: "same-origin"`, base `${BASE_PATH}/api/v1`, POST + `X-HTTP-Method-Override` para PATCH/PUT/DELETE, `X-CSRF-Token` en todo lo que no es GET, y **detección de HTML** (`content-type` no JSON → `WAF_BLOCKED` con el mensaje «La red de la universidad bloqueó la petición…»). Tipado desde `contracts`.
- [ ] Página de inicio que llama a `/health` y muestra el estado.
- [ ] `design/tokens.css` provisional (los tokens reales en E1.12).

#### E0.8 Docker
- [ ] `docker/api.Dockerfile` (multi-stage, `pnpm deploy --filter api --prod`), usuario no root, `tini`, `HEALTHCHECK` contra `/api/v1/health`. El worker usa la misma imagen con otro `CMD`.
- [ ] `docker/web.Dockerfile`: build de Vite y `nginx:alpine`.
- [ ] `docker/nginx.conf`: `location /api/ { proxy_pass http://api:3000; }`, SPA con `try_files … /index.html`, `client_max_body_size 12m`, `proxy_read_timeout 300s`, cabeceras de seguridad y CSP (`default-src 'self'; img-src 'self' data: blob:; style-src 'self' 'unsafe-inline'; font-src 'self'; connect-src 'self'; frame-ancestors 'none'`).
- [ ] `docker-compose.yml`: `postgres:16-alpine` (volumen nombrado `mathesia-pg`, healthcheck `pg_isready`), `api`, `worker`, `web` (publica `172.18.0.1:9063:80` en prod, `127.0.0.1:9063:80` en dev), red con subred explícita (§2.3), `depends_on: condition: service_healthy`, límites de memoria (512M api, 256M worker).
- [ ] `docker-compose.override.yml` para desarrollo (montajes de código, `tsx watch`, Vite con HMR detrás de nginx o directo con proxy).
- [ ] `scripts/check-subnet.sh`: lista las subredes en uso y avisa si la elegida se solapa.

#### E0.9 Verificación del WAF (adelantada a propósito)
- [ ] Desplegar **solo** el esqueleto en el destino real (`/mathesia/` en `lidis`, con `mathesia.conf` en `nginx-projects`), recargar `nginx-ssl-test`.
- [ ] `scripts/waf-sweep.mjs`: con User-Agent de navegador, envía a la URL pública y al puerto local: GET, POST, POST con override PATCH/PUT/DELETE, y cuerpos con `x^2-4`, `\frac{1}{2}`, `sqrt(x)`, `<`, `'`, `--`, `/*`. Compara `content-type`. Resultado en §11 (documenta qué bloquea el WAF; D2 ya está decidida).
- [ ] Retirar el despliegue de prueba o dejarlo detrás de autenticación básica de nginx hasta la Etapa 3.

#### E0.10 Respaldo (P1)
- [ ] Servicio `backup` en el compose (perfil `backup`, activable): `pg_dump -Fc` diario, rotación 14 diarios + 8 semanales, a un volumen montado en `./backups` (como la v1) para sacarlo de la máquina.
- [ ] `scripts/restore-test.sh`: restaura el último volcado en una base temporal y cuenta filas de `users`. Documentado; se ejecuta antes de la primera nota real.

**Criterio de salida E0**
- [ ] `docker compose -p mathesia up -d` levanta los cuatro servicios en sanos.
- [ ] `curl http://127.0.0.1:9063/api/v1/health` → `{"ok":true,…}`.
- [ ] La web servida por nginx muestra el estado llamando a la API, sin CORS.
- [ ] `pnpm check` en verde, incluido el test de *rollback* de la cola.
- [ ] Barrido del WAF hecho y registrado.

---

### Etapa 1 — Identidad y acceso · días 2–3

**Objetivo:** existen los tres roles, cada uno aterriza en su panel vacío, y el test del registro de rutas pasa.

#### E1.1 Registro de rutas completo 🔒
- [ ] `auth: "public" | Role[]` obligatorio; `policy` obligatoria salvo en `public`; `reauth?: true`; `csrf?: "session" (defecto) | "origin"` (solo permitido en públicas, H9).
- [ ] `registry.all()` y `registry.public()`; `PUBLIC_ROUTES` como lista cerrada en `tests/security/routeRegistry.test.ts` (CV2 §11.1).
- [ ] El registro **lanza al iniciar** si una ruta no cumple; el test lo verifica construyendo una ruta inválida.
- [ ] Middleware por ruta en este orden: método real → sesión → `auth` → CSRF → límite → validación de entrada → política (dentro de transacción de solo lectura o la del servicio) → `reauth` → handler → validación de salida.

#### E1.2 Sesiones 🔒
- [ ] `platform/session.ts`: token aleatorio de 32 bytes en la cookie; en la base `id = sha256(token)`. Construye el `Actor` (CV2 §3.2) una sola vez por petición.
- [ ] Tiempos de CV2 §4.2 por rol (inactividad y absoluto). Actualizar `last_seen_at` como máximo cada 60 s para no escribir en cada petición.
- [ ] Regla: con un intento `in_progress` la sesión del estudiante no vence por inactividad (el latido renueva) — se implementa la condición ya; el latido llega en E3.
- [ ] Cookie según §2.1 (`__Host-sid` o `__Secure-sid` con `Path` del prefijo). `HttpOnly; Secure; SameSite=Lax`.
- [ ] Rotación del token al iniciar sesión y al reautenticar (fijación de sesión).

#### E1.3 CSRF 🔒
- [ ] Token = HMAC(`SESSION_SECRET`, `sessionId`). `GET /auth/csrf` lo entrega; también viene en la respuesta de login y de `/me`.
- [ ] Comparación en tiempo constante. Rutas `csrf: "origin"` validan `Origin` contra `PUBLIC_URL` y `Sec-Fetch-Site`.

#### E1.4 Contraseñas
- [ ] `domain/auth/passwordPolicy.ts` portado (§1.2) con mínimo por rol (6 estudiante, 12 docente/admin), lista de prohibidas incluidas las cortas, secuencias de teclado/alfabeto en cualquier tramo y en ambas direcciones, sin nombre/correo/código. Mensajes en español, uno por regla.
- [ ] Hash `argon2id` (parámetros OWASP de la v1: 19 MiB, t=2, p=1) en `platform/crypto.ts`. **Fuera** del dominio (efecto).
- [ ] Tests portados + casos nuevos: `123456`, `qwerty`, `abcdef`, `fedcba`, `987654`, nombre dentro de la contraseña.

#### E1.5 Retraso progresivo 🔒
- [ ] `platform/throttle.ts` sobre `login_throttle` (CV2 §4.6): clave `acct:<userId>:<ip>` (2ⁿ⁻⁵ s tras 5 fallos, tope 15 min) y `ip:<ip>` (300 / 15 min). Operación atómica con `INSERT … ON CONFLICT … RETURNING`.
- [ ] Para correos inexistentes se usa `acct:email:<sha256(email)>:<ip>` para no revelar existencia y frenar igual.
- [ ] Todo fallo a `auth_events`. Respuesta `RATE_LIMITED` con `retryAfterSeconds`.
- [ ] Tiempo de respuesta constante (hash *dummy*) cuando el usuario no existe.

#### E1.6 Registro de estudiante
- [ ] `POST /auth/register`: correo de dominio institucional (`INSTITUTIONAL_EMAIL_DOMAINS`), nombre, código de estudiante (solo dígitos) **por duplicado**, contraseña por duplicado. Índice único de `student_code` → error claro: «Ese código ya tiene cuenta. Si es tuyo, recupera la contraseña» (lección 6).
- [ ] La web bloquea pegar en los campos de confirmación (`onPaste` cancelado) — lección 5.

#### E1.7 Login, logout, `/me`, sesiones
- [ ] `POST /auth/login` (si docente con TOTP: segundo paso `POST /auth/login/totp` con token intermedio de 5 min, o campo `totp` en el mismo cuerpo — elegir uno y añadirlo a §10).
- [ ] `POST /auth/logout`, `GET /me`, `GET /me/sessions`, `DELETE /me/sessions/:id` (solo propias; ajena → 404).
- [ ] `POST /me/password`: exige la actual, cierra las demás sesiones, `auth_events`.

#### E1.8 Reautenticación 🔒
- [ ] `POST /auth/reauth` (contraseña o TOTP) fija `sessions.reauth_at`. Ventana 10 min.
- [ ] `reauth: true` en el registro → `REAUTH_REQUIRED` si vencida.
- [ ] Web: interceptor global que ante `REAUTH_REQUIRED` abre el diálogo de contraseña **sin desmontar** la pantalla y **reintenta** la petición original (una vez).

#### E1.9 TOTP para docentes
- [ ] `otplib`; secreto cifrado AES-256-GCM con `TOTP_ENCRYPTION_KEY` (`platform/totp.ts`).
- [ ] Flujo: `setup` (devuelve URI `otpauth://` y QR generado en el cliente) → `confirm` (código válido activa) → 10 códigos de respaldo (hash argon2, de un solo uso) → `DELETE /me/totp`. Todo con **R**.
- [ ] Ventana de ±1 paso; rechazo de reutilización del mismo código en la misma ventana.

#### E1.10 Recuperación de cuenta (CV2 §4.7)
- [ ] `POST /auth/reset-with-code`: correo + código de estudiante + nueva contraseña. Respuesta idéntica exista o no. Throttle por la misma tabla.
- [ ] Código de un solo uso generado por la docente: el endpoint de generación llega en E5 (ficha), pero `POST /auth/reset-with-token` y la tabla `password_reset_tokens` (migración 0002) se hacen ahora, con un script de admin para generar uno mientras tanto.
- [ ] `POST /auth/forgot-password` solo si SMTP (se registra la ruta siempre, pero responde 404 si la función está apagada; la web no la muestra por `/features`).
- [ ] Restablecer cierra todas las sesiones; guarda `password_reset_notice {via, at}`; `audit_log`.
- [ ] Aviso en el siguiente ingreso con el texto de CV2 §4.7 y `POST /me/password-notice/ack`.

#### E1.11 Administración mínima y semilla
- [ ] Al arrancar, si no hay ningún admin y existen `SEED_ADMIN_*`, crea la cuenta (docente + `is_admin`) y lo registra en `audit_log`. Si ya hay admin, ignora las variables y lo avisa en el log.
- [ ] Rutas `admin/teachers` (§4). Contraseña inicial del docente: temporal, mostrada una sola vez, con cambio obligatorio en el primer ingreso (`must_change_password` en `users`, migración 0002 — añadir a CV2 §6.2).
- [ ] `GET /features` (P): `{ emailReset, photoGrading, … }` según configuración.

#### E1.12 Web: identidad y diseño
- [ ] `design/tokens.css` desde `arcade.css` (colores, radios, sombras, gradientes, tipografías; temas oscuro y claro) y fuentes auto-alojadas desde `quiz_app/public/css/fonts.css` y sus archivos.
- [ ] Componentes base en `shared/ui/`: Button (primaria/secundaria/peligro), Field, Dialog, ConfirmDialog (nombra la consecuencia), EmptyState (con acción), Toast, Tabs, Menu ⋯, Skeleton.
- [ ] Pantallas: Login, Registro, «¿Olvidaste tu contraseña?» (una sola pantalla con las vías disponibles), Restablecer, Cambiar contraseña, Seguridad (sesiones + TOTP).
- [ ] Layouts por rol: estudiante (barra inferior **Inicio · Calendario · Notas · Práctica**), docente (barra lateral **Inicio · Mis cursos · Biblioteca · Análisis**, inferior en móvil, botón **＋ Crear**), admin (entrada propia). Guardas de navegación por rol; estados vacíos honestos.
- [ ] Paneles vacíos que enseñan: docente «Aún no tienes cursos este periodo. Un administrador te asigna como titular».

**Pruebas E1**
- [ ] `routeRegistry.test.ts` (CV2 §11.1).
- [ ] Integración: registro, login, fallo con retraso, logout, sesiones (cerrar una ajena → 404), cambiar contraseña cierra las demás, reauth, TOTP completo, reset por código con aviso, reset con token usado → rechazo.
- [ ] Seguridad: CSRF ausente/incorrecto → `CSRF`; cookie con atributos correctos; sesión revocada no sirve; admin sin reauth → `REAUTH_REQUIRED`.

**Criterio de salida E1**
- [ ] Los tres roles existen y aterrizan en su panel vacío.
- [ ] El test del registro de rutas pasa.
- [ ] Una persona hace: registrarse, salir, olvidar la contraseña y recuperarla con su código, y ve el aviso.

---

### Etapa 2 — Estructura académica y biblioteca · días 3–4

**Objetivo:** una docente crea su curso con dos grupos, sus estudiantes se inscriben con el código, y la matriz de acceso pasa.

#### E2.1 Migración 0003 — estructura académica
- [ ] `terms`, `subjects`, `courses`, `course_teachers`, `sections`, `enrollments` (CV2 §6.2).
- [ ] Trigger **un estudiante, un grupo activo por curso** (P5): `BEFORE INSERT OR UPDATE ON enrollments` que busca otro `active` del mismo `course_id`. Test de concurrencia (dos inserciones simultáneas → una falla).
- [ ] `join_code`: 6 caracteres sin ambiguos (sin 0/O/1/I), único, generado en el servicio con reintento ante colisión.

#### E2.2 Migración 0004 — temas, bancos y preguntas
- [ ] `topics`, `question_banks`, `questions` (CV2 §6.3) con H6 y H7.
- [ ] Esquema Zod de `answer` y `options` **por tipo** (`multiple_choice`: opciones + índice/ID correcto; `numeric`: valor + tolerancia absoluta/relativa; `true_false`: booleano; `open`: rúbrica opcional). Las opciones llevan ID estable (`o1`, `o2`…) para que el barajado no dependa del índice.

#### E2.3 Repositorios con alcance y políticas 🔒
- [ ] `courseTeachersRepo.isActive`, `isTitular`; `enrollmentsRepo.isActiveInCourse`, `activeSectionOf(studentId, courseId)`; `sectionsRepo.findForTeacher`; `banksRepo.findForOwner`.
- [ ] Políticas `teachesCourse`, `teachesSection`, `isTitular`, `enrolledInCourse`, `ownsBank`, `teachesStudent` (estudiante con matrícula activa en un grupo que el actor dicta), combinadores `anyOf` / `allOf`.
- [ ] `GET /courses` filtrado por actor; `GET /courses/:courseId` con `anyOf(teachesCourse(), enrolledInCourse())`.

#### E2.4 Administración académica
- [ ] CRUD de materias, periodos, cursos y asignación de titular (rutas de §4). Título por defecto «Materia · Periodo».
- [ ] Temas (según D3): árbol de dos niveles por materia; edición de nombre; sin borrado si tiene referencias.
- [ ] Web admin: pantallas Docentes, Materias y periodos, Cursos (todas tras reautenticación al entrar).

#### E2.5 Grupos y matrícula
- [ ] Titular crea grupos (`POST /courses/:courseId/sections`), los renombra, abre/cierra inscripción y rota el código.
- [ ] `POST /sections/join`: código válido y grupo abierto. Respuesta genérica si el código no existe o está cerrado (no revelar cuál). Throttle por IP y por cuenta con la tabla de E1.5 (clave `join:<userId>`).
- [ ] Matrícula manual y retiro (retiro conserva notas publicadas en solo lectura — se prueba en E3).
- [ ] Auxiliares: invitar por correo de docente existente; quitar.

#### E2.6 Lista oficial 🔒 (lección 12)
- [ ] `platform/csv.ts` (portado) y lectura de `.xlsx` (portar el lector de `scripts/import-notas-xlsx.js` a `platform/xlsx.ts` ya aquí; la escritura llega en E5).
- [ ] `preview`: cruza por código, luego por correo, luego por nombre (`nameMatch.ts`, «ante la duda, no se decide»). Devuelve `matched`, `newEnrollments`, `ambiguous` (con candidatos), `conflicts` (código ya usado por otra cuenta), `notFound`.
- [ ] `commit` (**R**): aplica solo lo inequívoco que la docente confirmó, en una transacción; fija `student_code_verified = TRUE`; `audit_log` con el resumen.

#### E2.7 Biblioteca
- [ ] Bancos: crear, editar, archivar, borrado lógico (**R** si alguna evaluación los usa).
- [ ] Preguntas: crear/editar/borrar lógico con previsualización de LaTeX.
- [ ] **Importación con tests** (el hueco de la v1): formato JSON de la v1 (`seeds/*.json`) y CSV. `preview` valida cada fila con el contrato, mapea dificultad, resuelve temas por nombre (exacto, luego normalizado sin tildes) y **reporta** los desconocidos; `import` aplica solo las válidas confirmadas. Fixtures: los cuatro bancos de `quiz_app/seeds/`.
- [ ] Generadores: `domain/items/` portado (§1.2). `GET /generators` agrupado por familia; `preview` con 3 muestras. Crear pregunta paramétrica = elegir generador + tema + dificultad.
- [ ] Tests de regresión de semilla: para 5 claves y 3 semillas fijas, el `stem` y la respuesta coinciden **exactamente** con los de la v1 (se generan una vez ejecutando la v1 y se guardan como fixture).

#### E2.8 Web docente: cursos y biblioteca
- [ ] **Mis cursos** (periodo actual en la barra lateral; periodos anteriores plegados).
- [ ] Curso → pestañas visibles según etapa: **Estudiantes** (código de inscripción arriba con *Mostrar en pantalla* a pantalla completa, por grupo), **Ajustes** (grupos, auxiliares). Las demás pestañas aparecen cuando existan (no se muestran vacías «en construcción»).
- [ ] Selector de grupo **Todos / A / B** en la cabecera del curso, en la URL (`?section=`), compartido por las pestañas.
- [ ] **Biblioteca**: bancos, preguntas, importar (con la pantalla de previsualización), generadores.
- [ ] Estudiante: pantalla «Inscribirme con un código» desde Inicio cuando no tiene cursos.

#### E2.9 Matriz de acceso cruzado — versión 1 🔒
- [ ] `tests/helpers/seedMatrix.ts`: docente A (curso X, grupos X1 y X2), docente B (curso Y), e1 en X1, e2 en X2, e3 en Y, e4 retirado de X1, admin sin cursos (CV2 §11.2).
- [ ] `accessMatrix.test.ts` **generado desde el registro**: para cada ruta con parámetro de recurso, una tabla `resourceFixtures` dice cómo obtener un ID «de X» y un ID «de Y». El test recorre todas las rutas × actores y verifica 404 donde corresponde. Una ruta con parámetro **sin** entrada en `resourceFixtures` hace fallar el test (así nadie la olvida).
- [ ] Filas de esta etapa: §6.

**Criterio de salida E2**
- [ ] Una docente crea su curso (asignado por admin) con dos grupos; tres estudiantes se inscriben con el código; uno se importa por lista oficial con un cruce ambiguo reportado.
- [ ] Importar `banco_limites_continuidad.json` funciona y reporta sus temas.
- [ ] Matriz v1 en verde.

---

### Etapa 3 — Evaluaciones de punta a punta · días 5–7

**Objetivo:** un quiz real se crea, presenta, cierra y publica, y el estudiante ve su nota. **Aquí la v2 puede reemplazar a la v1 para quices.**

#### E3.1 Migración 0005 — evaluaciones
- [ ] `assessments`, `assessment_items`, `attempts`, `attempt_items` (ya con `is_bonus`, `recovered_by`, `final_score` para no migrar en 3b), `attempt_events` (con `student_id`, H3), `grades`, `grade_revisions`, `time_extensions` (CV2 §6.4, §7.2).
- [ ] Columnas de pareja en `attempts` (`team_id`, `operator_id`) **nulas** desde ya; las tablas de equipos llegan en 3b.
- [ ] Índices: `attempts(assessment_id, status)`, `attempt_items(attempt_id, position)`, `attempt_events(attempt_id, occurred_at)`, `grades(assessment_id)`, parcial `assessments(state) WHERE deleted_at IS NULL`.

#### E3.2 Máquina de estados
- [ ] `domain/assessment/states.ts`: tabla de transiciones de CV2 §7.1 (incluidas *volver a borrador*, *reabrir con motivo*, y el salto `borrador → cerrada` para `source = 'external'`).
- [ ] `assertTransition(from, to, ctx)` → `INVALID_STATE`. `allowedActions(state, ctx)` → lista de botones (la web **solo** pinta lo que devuelve esto: «los botones se derivan del estado»).
- [ ] Regla de cierre: pasa a `cerrada` cuando `now ≥ closes_at` **y** no quedan intentos `in_progress`.
- [ ] Tests: tabla completa de transiciones válidas e inválidas.

#### E3.3 Configuración (`settings`)
- [ ] `packages/contracts/assessmentSettings.ts`: esquema estricto con valores por defecto **por plantilla** (`quiz_rapido`, `parcial`, `diagnostico`, `taller_externo`): intentos, vidas, barajado de preguntas y opciones, una a la vez (siempre sí en E3), volver atrás, mostrar retroalimentación, marca de agua visible, declaración, (E6: canarios, SEB, reloj por pregunta), (3b: `bonus`, `team`).
- [ ] Las opciones de etapas futuras **no** se aceptan aún (el esquema las rechaza) ni aparecen en la interfaz.

#### E3.4 Constructor (API)
- [ ] `POST /courses/:courseId/assessments` desde plantilla: alcance (curso o grupo del curso), título, tipo, fechas, duración, peso.
- [ ] Ítems: preguntas fijas y reglas «N al azar de banco/tema». Validación: los bancos son del actor (`ownsBank`), hay suficientes preguntas (mensaje de la v1: «La regla pide 5 … pero solo hay 3»).
- [ ] `PATCH` según estado (borrador: todo; programada: todo menos lo pasado; abierta: nada salvo ampliar).
- [ ] `duplicate`, borrado lógico.
- [ ] `POST /assessments/:id/transition` `{ to, reason? }`: programar (encola `assessment.open` a `opens_at` y `assessment.close` a `closes_at`, **dentro** de la transacción), abrir ahora, volver a borrador (cancela trabajos), cerrar ahora, reabrir (motivo + auditoría), publicar (**R**).
- [ ] Cambiar fechas de una programada reprograma sus trabajos (los manejadores igual revalidan estado y hora: idempotentes).

#### E3.5 Trabajos del worker
- [ ] `assessment.open`: si `state = programada` y `now ≥ opens_at` → `abierta`; si no, nada.
- [ ] `assessment.close`: si `abierta` y `now ≥ closes_at` → no admite nuevos intentos; si quedan intentos `in_progress` con `deadline_at` futuro, **no** cierra (los vence `attempt.expire`); el último `attempt.expire` dispara el cierre.
- [ ] `attempt.expire` (H1): relee `deadline_at` (+ congelación); si venció → `auto_submitted` con `submit_reason = 'time'`; si no, se reprograma al nuevo plazo.
- [ ] `assessment.closed`: calcula notas automáticas (E3.8) para todos los intentos; idempotente (recalcular da lo mismo).
- [ ] Tests de idempotencia: ejecutar cada manejador dos veces → mismo estado.

#### E3.6 Presentar 🔒
- [ ] `POST /assessments/:id/attempts`: `findForStudent` + abierta + `now < closes_at` + intentos restantes + aceptó la declaración (hash guardado) + (presencial: deshabilitado hasta E4, H4). Si hay un intento `in_progress`, lo **reanuda** (no crea otro).
- [ ] Materialización (`itemBuilder.ts`) con semilla del intento: resuelve reglas, baraja preguntas y opciones, genera paramétricas, guarda `rendered` y `answer_key` en `attempt_items`. `deadline_at = min(started_at + duración, closes_at)`. Encola `attempt.expire`.
- [ ] `GET /attempts/:id/current-item`: miembro + `in_progress` + no congelado. Devuelve **solo** la pregunta actual (`rendered` sin clave), posición, total, `deadline_at`, vidas. Marca `served_at`.
- [ ] `PUT /attempts/:id/items/:pos`: valida la forma de la respuesta según tipo; rechaza tras `deadline_at + 5 s`; rechaza posiciones no servidas aún; si no se permite volver atrás, rechaza posiciones anteriores ya respondidas.
- [ ] `POST /attempts/:id/heartbeat` cada 15 s: recibe lote de eventos de integridad (máx. 50), devuelve `deadline_at` vigente, vidas, estado y mensajes («Tu docente añadió 10 minutos»). Descuento de vidas según `settings` → `auto_submitted` con `submit_reason = 'violations'` al llegar a 0. Renueva la sesión (E1.2).
- [ ] `POST /attempts/:id/submit`.
- [ ] Contrato de salida del estudiante **sin** `answer_key` (test de seguridad que inspecciona el esquema Zod de salida y además la respuesta real).

#### E3.7 Integridad básica (CV2 §8.2, sin pantalla dividida)
- [ ] Vista de examen: `user-select: none`; cancelar `copy`, `cut`, `contextmenu`, `dragstart`, `beforeprint` y registrar `stem_copy_blocked`; `@media print { .exam { display:none } }`.
- [ ] Marca de agua visible: SVG en capa superior (`pointer-events:none`), nombre + código + hora, diagonal, repetida, semitransparente; no seleccionable; se regenera cada minuto (la hora).
- [ ] Eventos `window_blur`, `visibility_hidden`, `fullscreen_exit` con duración; descuento de vidas según configuración.
- [ ] Meta `interactive-widget=resizes-visual` en la vista de examen.
- [ ] Declaración de integridad (texto de `EVALUACION_ANTITRAMPAS.md` §12.2, versionado por hash) antes de iniciar.
- [ ] La interfaz **nunca** promete bloquear capturas (CV2 §15).

#### E3.8 Calificación y publicación
- [ ] `domain/grading/autoGrade.ts` (portado): puntaje por ítem, tolerancia numérica, escala a 0,0–5,0 con `NUMERIC(3,2)` y redondeo estable.
- [ ] `open` queda `score = NULL` → «por calificar»; `PATCH` de puntaje por ítem para la docente (calificación manual simple).
- [ ] `grades` se escribe en `assessment.closed`; `grade_revisions` en todo cambio; motivo obligatorio si ya publicada (**R**).
- [ ] Publicar (**R**): confirma nombrando la consecuencia («Publicar notas de 32 estudiantes. Las verán de inmediato.»), fija `published_at`, `audit_log`.
- [ ] `GET /me/grades` (solo publicadas) y `GET /attempts/:id/result` (pregunta por pregunta con retroalimentación si la configuración lo permite).
- [ ] Retirado: conserva en solo lectura sus notas publicadas, no accede a evaluaciones abiertas.

#### E3.9 Monitor básico y ampliar tiempo 🔒
- [ ] `GET /assessments/:id/monitor` (sondeo cada 5 s con TanStack Query): intentos con estado, posición, vidas, último latido, eventos recientes interpretados (`proctorEvents.ts`), minutos añadidos.
- [ ] `POST /assessments/:id/time-extensions` y `POST /attempts/:id/time-extensions` (CV2 §7.2): solo `abierta`; 1–60 min; motivo con un toque (*Falla técnica · Ajuste · Accesibilidad · Otro*); toda la evaluación = `closes_at += N` y `deadline_at += N` de los `in_progress`; un intento = solo su `deadline_at`; reprograma `attempt.expire`; `audit_log`. Admin: la ruta acepta `is_admin` con **R** y motivo escrito (la pantalla de admin llega en E5).
- [ ] `POST /attempts/:id/void` (anular con motivo).
- [ ] Web: pantalla completa, botón **+ Tiempo** arriba y en cada fila, con +5 · +10 · +15 y valor libre; historial de ampliaciones.

#### E3.10 Web del estudiante
- [ ] **Inicio** (`/me/home`): evaluaciones abiertas ahora arriba y grandes; lo que viene esta semana.
- [ ] **Presentar**: una pregunta a la vez, reloj del servidor (se corrige con cada latido; el cliente nunca decide el vencimiento), vidas, KaTeX, guardado al responder con reintento, pantalla de entrega, aviso si se pierde la red.
- [ ] **Notas**: por curso, evaluación, nota y retroalimentación.
- [ ] **Práctica**: mensaje honesto («La práctica libre llega más adelante en el semestre»), sin «🚧».

#### E3.11 Web del docente
- [ ] Pestaña **Evaluaciones** agrupada por estado; botón principal **Nueva evaluación** → plantilla.
- [ ] Constructor: plantilla → *Qué evalúa* (alcance, ítems) · *Cómo se presenta* · *Cómo se califica* → *Avanzado* plegado con la línea resumen («Avanzado: Marca de agua sí · Vidas 3»).
- [ ] Ficha de evaluación con botones de `allowedActions`; *Calificar* (abiertas por ítem), *Publicar*, *Resultados*.
- [ ] **Inicio** docente (`/me/inbox`) con las tarjetas que ya tienen datos: evaluaciones que abren/cierran hoy, por calificar, notas listas sin publicar. «Todo al día» si no hay nada.
- [ ] **＋ Crear → Evaluación** (pide el curso en el formulario).

#### E3.12 Pruebas de punta a punta
- [ ] Imagen `docker/playwright.Dockerfile`.
- [ ] Playwright: estudiante se inscribe con código, presenta, entrega, la docente cierra y publica, el estudiante ve su nota (CV2 §11.3).
- [ ] Playwright: ampliar tiempo de un intento mientras el estudiante presenta; su reloj se actualiza con el mensaje.

#### E3.13 Despliegue de producción
- [ ] `mathesia.conf` en `nginx-projects`, `docker compose -p mathesia up -d`, admin semilla, barrido del WAF (`waf-sweep.mjs`) sobre **todas** las rutas del registro (genera la lista desde `registry.all()`).
- [ ] Respaldo activado y `restore-test.sh` ejecutado (**P1: antes de la primera nota real**).
- [ ] Prueba con una persona (CV2 §11.3): pasar lista queda para E4; aquí: crear un quiz desde plantilla, ampliar tiempo, publicar notas.

**Criterio de salida E3**
- [ ] Un quiz real se crea, se presenta (en móvil y escritorio), se cierra y se publica; el estudiante ve su nota.
- [ ] Matriz v2 (§6) en verde. Playwright en verde.
- [ ] Desplegado y con respaldo probado.

---

### Etapa 3b — Variantes de evaluación · 4–5 días

**Objetivo:** una pareja presenta un quiz con bonificación desde dos teléfonos, una infracción en el acompañante descuenta vida al intento, y los dos ven la misma nota con la pregunta recuperada explicada.

#### E3b.1 Bonificación (CV2 §7.3)
- [ ] `settings.bonus` en el contrato (`enabled`, `count` 1–5, `topicIds`, `bankIds`, `match`).
- [ ] `GET /banks/availability`: por tema, cuántas preguntas **autocalificables** hay en los bancos elegidos («Límites: 14 disponibles»). El constructor no deja guardar si hay menos que `count`.
- [ ] Sorteo al iniciar el intento (misma semilla), sin repetir preguntas del intento, admite paramétricas; `attempt_items.is_bonus = TRUE` al final.
- [ ] Pantalla de transición con el texto de CV2 §7.3; si se permitía volver atrás, aviso «Ya no podrás cambiar tus respuestas anteriores» y bloqueo en servidor de las posiciones regulares al servir la primera bonificación.
- [ ] `domain/grading/bonus.ts` → `applyBonus(regular, bonus, match, { blankCounts: true /* P7 */ })`. Tests: nunca supera el máximo; `same_topic` sin pendiente del tema no recupera; orden por puntos perdidos y luego posición; determinismo (dos ejecuciones iguales); bonificación correcta sin pendientes no vale nada; recupera el valor completo de la reemplazada.
- [ ] Recalcular al editar a mano una nota → nueva `grade_revisions`.
- [ ] Resultado del estudiante: «Pregunta 4 (Límites por racionalización): recuperada con la bonificación B2».

#### E3b.2 Migración 0006 — parejas
- [ ] `assessment_teams`, `team_members` (CV2 §7.4); índice único parcial de H2; FK de `attempts.team_id` y `operator_id`.
- [ ] `attemptsRepo.findForMember(tx, studentId, attemptId)` 🔒 sustituye a «dueño del intento» en todas las rutas de intentos.

#### E3b.3 Formación de parejas
- [ ] `settings.team` (`mode`, `formation`, `allowTrio`).
- [ ] **Asignada:** `GET/PUT /assessments/:id/teams` (borrador o programada). *Generar al azar* por grupo (con la semilla de la evaluación, reproducible) y ajuste; validación: mismo grupo, una sola pareja por estudiante, trío solo si `allowTrio`.
- [ ] **Libre:** al iniciar, el primero recibe un código de 4 dígitos (único entre los equipos abiertos de esa evaluación, caduca al completar); `POST /assessments/:id/teams/join` valida mismo grupo y sin pareja; respuesta genérica si el código no existe o es de otro grupo/curso (fila e3 de la matriz). Throttle por cuenta.
- [ ] El intento empieza cuando la pareja está completa; antes, pantalla de espera con sondeo.
- [ ] Web docente: pestaña **Parejas** en la evaluación (solo si es en pareja), con arrastrar y soltar y alternativa por teclado/selector (accesible en móvil).

#### E3b.4 Operador y acompañante 🔒
- [ ] Solo el operador puede `PUT items` y `submit`; el acompañante → **403** `FORBIDDEN_ACTION`.
- [ ] El acompañante hace `current-item` (solo lectura) y `heartbeat` (sus eventos cuentan, con su `student_id`). Vidas compartidas.
- [ ] `POST /attempts/:id/operator` (docente): el nuevo operador debe ser miembro; `audit_log`.
- [ ] Marca de agua con los dos (o tres) nombres.
- [ ] Notas: una fila en `grades` por miembro, mismo `attempt_id` y valor; ajuste individual con motivo obligatorio. Intentos contados por pareja.
- [ ] Monitor: cada fila es una pareja, muestra quién opera y **Pasar el control**; ampliar tiempo a la fila amplía el de la pareja; marcar individual a quien quedó sin pareja (si no hay trío).

#### E3b.5 Matriz y E2E
- [ ] Seed con `e5` (H10). Filas de §6 de esta etapa.
- [ ] Playwright con **dos contextos** de navegador (dos teléfonos simulados): pareja libre por código, bonificación, infracción del acompañante descuenta vida, mismo resultado para ambos con la recuperación explicada.

**Criterio de salida E3b:** el de CV2 §12 (arriba) + matriz en verde.

---

### Etapa 4 — Clases, bitácora y calendario · 1 semana

#### E4.1 Migración 0007
- [ ] `files` (CV2 §6.5), `class_sessions`, `class_session_topics`, `class_session_materials`, `attendance`, `agenda_items` (CV2 §5.2).

#### E4.2 Archivos 🔒
- [ ] `platform/files.ts` como **único** acceso a disco (mitigación de CV2 §1.2): `storage_key` aleatoria, tipo MIME detectado por contenido (`file-type`), `sha256`, tamaño máximo por contexto, lista blanca de tipos por contexto.
- [ ] `POST /files` (multipart en streaming, sin cargar todo en memoria) y `GET /files/:id` que resuelve el **dueño lógico** (material de una clase → política de la sesión; foto → E7) y aplica su política; ajeno → 404. `Content-Disposition` y `X-Content-Type-Options: nosniff`.
- [ ] Volumen `mathesia-files` en el compose (`FILES_DIR`).

#### E4.3 Clases y asistencia
- [ ] Crear sesión (grupo, fecha, hora), estados `planned/held/cancelled`.
- [ ] Pantalla única: **pasar lista** (todos presentes por defecto, se marcan excepciones: ausente/tarde/excusa con nota) + **resumen** (Markdown + LaTeX, visible a estudiantes) + **notas privadas** + **temas** + **material** + **asignado** en esta clase.
- [ ] `PUT /class-sessions/:id/attendance` reemplaza el conjunto en una transacción.
- [ ] 🔒 `teacher_notes` no existe en el contrato de salida del estudiante (test de esquema + test de respuesta).

#### E4.4 Agenda y calendario
- [ ] Agenda con `kind` y `visibility`; pendientes privados con `done_at`.
- [ ] 🔒 Filtro de visibilidad en el **repositorio** (CV2 §5.2).
- [ ] `GET /courses/:courseId/calendar?from&to&section` = unión de sesiones, agenda y evaluaciones (sin duplicar evaluaciones en la agenda), cada parte con su filtro.
- [ ] Vistas **Próximo** (última clase, asignado, próximos 14 días; pendientes privados arriba para docente), **Semana**, **Mes**, **Clase**.
- [ ] Lista de arranque del curso en Próximo (*Grupos · Lista oficial · Primera clase · Primera evaluación*), desaparece al completarse.

#### E4.5 Evaluaciones presenciales (H4)
- [ ] Se habilita `modality: in_person` en el constructor.
- [ ] Iniciar intento exige `attendance.status IN ('presente','tarde')` en la sesión de hoy de su grupo; en pareja, ambos.
- [ ] Monitor: **marcar presente** con un toque (crea la sesión de hoy si no existe y marca).

#### E4.6 Inicio y estudiante
- [ ] Tarjetas: pendientes privados que vencen esta semana (*Marcar hecho*), «la última clase de X no tiene resumen» (*Escribirlo*), «Clase de hoy» (*Pasar lista*).
- [ ] Estudiante: Calendario (misma bitácora sin lo privado), Inicio con la última clase de cada curso.
- [ ] **＋ Crear → Clase · Pendiente**.

**Criterio de salida E4:** una docente pasa lista y escribe el resumen en una pantalla; el estudiante ve el resumen y lo asignado, nunca las notas privadas; un parcial presencial solo deja entrar a los presentes; matriz en verde; prueba con una persona (pasar lista).

---

### Etapa 5 — Libro de notas y gestión de cuentas · 1 semana

#### E5.1 Libro de notas
- [ ] `domain/grading/weighting.ts` portado: `current`, `projected`, `checkWeights` con el aviso de pesos ≠ 100 %.
- [ ] `GET /courses/:courseId/gradebook` (por grupo), edición por celda (`PATCH /grades/:id`, **R** si publicada, motivo), historial de revisiones visible.
- [ ] Estudiante: acumulado `current` y `projected` con su explicación, y asistencia.

#### E5.2 Externas e importación
- [ ] Evaluación externa (`source = 'external'`, salta a cerrada).
- [ ] Importar notas xlsx/csv: `preview` (cruce por código → correo → nombre con `nameMatch`, ambiguos reportados, valores fuera de 0–5 rechazados) y commit.

#### E5.3 Exportación
- [ ] xlsx (`platform/xlsx.ts` portado) y csv con `csvSafe`; columna de minutos añadidos por intento (CV2 §7.2).

#### E5.4 Ficha del estudiante
- [ ] `GET /students/:studentId`: datos, matrícula, notas, asistencia, bloqueos.
- [ ] Corregir correo, nombre, código (**R**, auditoría con antes/después).
- [ ] Ver bloqueos explicando cuál es cuál («por cuenta desde la IP x, expira a las…» / «por IP del campus») y liberar los de cuenta.
- [ ] `POST /students/:studentId/reset-code`: código de un solo uso (vía `teacher_code`) que la docente entrega; muestra una sola vez.
- [ ] Filtros en Estudiantes: *sin código verificado*, *con bloqueo*, *con asistencia bajo el umbral*.
- [ ] Tarjeta de Inicio «5 estudiantes sin código verificado → Cargar lista oficial».

#### E5.5 Buscador de comandos 🔒
- [ ] `GET /search?q=`: estudiantes, cursos y evaluaciones **filtrados por política**; para estudiantes, solo sus cursos y evaluaciones visibles.
- [ ] Catálogo de **acciones** en la web (fuente única en `app/commands.ts`), cada una con palabras clave («ampliar tiempo», «pasar lista», «código de restablecimiento», «exportar notas»…) y la condición en la que aplica.
- [ ] `Ctrl+K` / lupa en móvil.
- [ ] Test: cada acción de §10 con botón aparece en el catálogo (CV2 §11.3).

#### E5.6 Panel de administración completo
- [ ] Cuentas (buscar, bloqueos, liberar), Docentes, Materias y periodos, Cursos, **Evaluaciones en curso** (solo curso, título, intentos en curso, cierre; **+ Tiempo** con R y motivo escrito) y **Auditoría** (filtros actor/acción/fecha, solo lectura).
- [ ] Inicio docente: tarjeta «El administrador añadió 10 min a Parcial 1 (motivo…)» (H13).
- [ ] 🔒 Matriz: admin no ve monitor/respuestas/eventos/notas de lo que amplió (404).

#### E5.7 Prueba de interfaz
- [ ] Test que recorre el mapa de CV2 §9.2c y comprueba los clics declarados (Playwright: desde Inicio, cuenta navegaciones hasta la pantalla de la función).
- [ ] Prueba con una persona: las 5 tareas de CV2 §11.3 en menos de un minuto cada una.

---

### Etapa 6 — Integridad avanzada · 1 semana

- [ ] **E6.1** Pantalla dividida: portar `split-screen.js` a `splitScreen.ts` (área vs. alto, ignorar con foco en campo, espera de 4 s, `rebaseline()`). Congelación en servidor (`frozen_at`, `frozen_reason`, `frozen_evidence`), reloj en pausa (`frozen_seconds` suma al `deadline_at` efectivo), trabajo `attempt.unfreeze` a los 10 min, `POST /attempts/:id/unfreeze`.
- [ ] **E6.2** Monitor: congelados arriba con contador y aviso sonoro opcional; tarjeta roja en Inicio.
- [ ] **E6.3** Segunda pestaña (`BroadcastChannel`), evento de alta señal.
- [ ] **E6.4** Canarios (`canary.ts`): ítems que no puntúan, variantes por estudiante; informe en Integridad.
- [ ] **E6.5** Marca invisible (`watermark.ts`) en `rendered` y `POST /assessments/:id/leak-identify` sobre un texto pegado.
- [ ] **E6.6** Reloj por pregunta (límite desde `served_at`, validado en servidor).
- [ ] **E6.7** SEB opcional: Config Key por evaluación, verificación del hash en las rutas de intento, descarga del `.seb`.
- [ ] **E6.8** Pestaña **Integridad** de la evaluación (cerrada/publicada): eventos por estudiante interpretados, similitud (por pareja en quices en pareja), identificar fuga.
- [ ] **E6.9** Matriz de seis casos del detector con medidas reales de Android (CV2 §11.3): fixtures de dimensiones medidas y test de unidad; verificación manual en dos teléfonos.

---

### Etapa 7 — Calificación por fotos · 1–1,5 semanas

- [ ] **E7.0** Releer `MODULO_CALIFICACION_IA.md`, `Especificacion_Modulo_Calificacion.md`, `Prompt_Calificacion_Reutilizable.md` y la migración 013–019 de la v1. Diseñar tablas (`photo_batches`, `photo_pages`, `photo_proposals`, `photo_decisions`) con UUID y el **par propuesta/decisión que nunca se sobrescribe**; el lote **no** depende de una evaluación existente. Añadirlas a CV2 §6.
- [ ] **E7.1** Portar `rubrica`, `similitud`, `informe`, `prompt`, `esquema`, `paginas`, `claude` y sus tests. Antes de tocar `claude.ts`, revisar la referencia actual de la API de Claude (modelo por defecto, caché de *prompt*, salida estructurada) — la v1 usa `IA_MODELO=claude-opus-5`.
- [ ] **E7.2** Trabajo `grading.photo` (cola + sondeo, nunca petición larga; lección 9), con reintentos acotados, un examen que falla no tumba el lote, resultados guardados y no repedidos.
- [ ] **E7.3** Interfaz: capturar/subir, agrupar páginas, revisar propuestas, decidir. «El modelo percibe y el código decide» (lección 10): la nota sale de la rúbrica, no del modelo.
- [ ] **E7.4** Exportar decisiones a `grades` con `origin = 'photo'` en la evaluación destino (`findForTeacher`).
- [ ] **E7.5** Solo visible si `ANTHROPIC_API_KEY` (`/features`).

---

### Etapa 8 — Gamificación, práctica y ayudas · 2 semanas

- [ ] **E8.0** Diseño: releer `ARQUITECTURA_V2.md` §2.4b, §2.4d, §2.4f y Fase 6, y `GAMIFICACION.md`. Ajustes de CV2 §6.5: UUID, `topic_id`, insignias en código, lote de papel independiente. Escribir las tablas y rutas en CV2 §6 y §10 **antes** de programar.
- [ ] **E8.1** XP como asientos (libro mayor, nunca un contador editable), reparto en `assessment.closed`; en pareja, ambos reciben lo mismo.
- [ ] **E8.2** Pelotones (la pareja cuenta una vez), insignias definidas en código, rachas.
- [ ] **E8.3** Ranking solo con `leaderboard_opt_in`.
- [ ] **E8.4** Modo práctica con generadores paramétricos y bancos marcados para práctica.
- [ ] **E8.5** Ayudas entre compañeros, **bloqueadas mientras el estudiante tenga un intento abierto** (comprobación en servidor).

---

## 6. Matriz de acceso cruzado por etapa 🔒

Las filas se acumulan: cada etapa ejecuta las suyas y todas las anteriores. «No construir la etapa siguiente sin que la matriz cubra la actual» (CV2 §15).

| Etapa | Actor | Recurso | Esperado |
|---|---|---|---|
| 1 | Cualquiera | `DELETE /me/sessions/:id` de otro usuario | 404 |
| 1 | Docente sin reauth | `POST /me/totp` | `REAUTH_REQUIRED` |
| 1 | Estudiante / docente no admin | `/admin/*` | 404 |
| 1 | Sin CSRF | Cualquier ruta que cambia estado | `CSRF` |
| 2 | Docente B | Curso X, sus grupos, su lista, sus estudiantes | 404 |
| 2 | e3 | `GET /courses/:X` | 404 |
| 2 | Docente B | Banco de A, preguntas de A, importar en banco de A | 404 |
| 2 | Auxiliar de X | Crear grupo en X (solo titular) | 403 `FORBIDDEN_ACTION` |
| 2 | Admin sin cursos | Estudiantes del curso X | 404 |
| 2 | e1 | `POST /sections/join` con código cerrado / inexistente | Misma respuesta genérica |
| 3 | e3 | Evaluación del curso X | 404 |
| 3 | e2 | Evaluación asignada solo a X1 | 404 |
| 3 | e1 | Evaluación en borrador de X | 404 |
| 3 | e4 (retirado) | Evaluación abierta de X | 404 |
| 3 | e4 (retirado) | Su nota ya publicada de X | 200, solo lectura |
| 3 | e1 | Intento de e2 (cualquier ruta de intento) | 404 |
| 3 | e1 | Respuesta de `current-item` y `result` antes de publicar | Sin `answer_key` |
| 3 | Admin sin cursos | Notas, monitor, resultados del curso X | 404 |
| 3 | Docente A sin reauth | Publicar notas | `REAUTH_REQUIRED` |
| 3 | Docente B | Ampliar tiempo en el curso X | 404 |
| 3 | Cualquiera | Ampliar tiempo de una evaluación cerrada | `INVALID_STATE` |
| 3 | Admin sin reauth | Ampliar tiempo | `REAUTH_REQUIRED` |
| 3 | Admin | Monitor, respuestas o eventos de la evaluación que amplió | 404 |
| 3 | e1 | Responder después de `deadline_at + 5 s` | Rechazado |
| 3b | e2 (no es de la pareja) | Intento de la pareja e1 + e5 | 404 |
| 3b | e5 (acompañante) | `PUT items` / `submit` | 403 |
| 3b | e3 (otro curso) | Código de pareja de e1 | Rechazo genérico |
| 3b | Docente | Cambiar operador a alguien que no es miembro | `VALIDATION` |
| 4 | e1 | `teacher_notes` de su clase | El campo no existe |
| 4 | e1 | Pendiente privado de la docente | No aparece en el calendario |
| 4 | e2 | Clase y material de X1 | 404 |
| 4 | e3 | `GET /files/:id` de un material de X | 404 |
| 4 | e1 ausente | Iniciar parcial presencial | 403 con motivo |
| 5 | Docente B | Ficha de e1, sus bloqueos, su código de restablecimiento | 404 |
| 5 | Docente A sin reauth | Cambiar correo de e1 | `REAUTH_REQUIRED` |
| 5 | e1 / Docente B | `GET /search?q=<nombre de e2 / curso X>` | Sin resultados ajenos |
| 6 | Docente B | Integridad / identificar fuga de X | 404 |
| 7 | Docente B | Lote de fotos de A, sus archivos | 404 |

---

## 7. Despliegue y operación

### 7.1 Entornos

| Entorno | Cómo | URL |
|---|---|---|
| Desarrollo | `scripts/db-dev.sh` + `pnpm dev` (api con `tsx watch`, worker, Vite) | `http://localhost:5173` (Vite con proxy a la API) |
| Stack local | `docker compose -p mathesia up -d --build` | `http://127.0.0.1:9063` |
| Producción | Mismo compose con `.env` de producción; publicación en `172.18.0.1:9063` | `https://lidis.usbcali.edu.co/mathesia/` |

### 7.2 Redesplegar (siempre así)

```bash
cd ~/thesis-manager/proyectos/mathesia
docker compose -p mathesia exec postgres sh -c 'pg_dump -U $POSTGRES_USER -Fc $POSTGRES_DB' > backups/pre-deploy-$(date +%F-%H%M).dump
# comprobar que no hay intentos en curso (o que el despliegue puede esperar):
docker compose -p mathesia exec postgres psql -U mathesia -c "select count(*) from attempts where status='in_progress'"
docker compose -p mathesia build api worker web
docker compose -p mathesia rm -sf api worker web    # down antes de up
docker compose -p mathesia up -d api worker web
curl -fsS http://127.0.0.1:9063/api/v1/health
```

- Nunca `docker-compose` (v1). Nunca sin `-p mathesia`. Nunca `down -v`.
- Las migraciones corren al arrancar la API (E0.4). Son siempre hacia adelante y compatibles con el código anterior durante el reinicio (añadir columnas nulas primero, usar después).

### 7.3 nginx externo

`/home/fac.ingenieria/lidis-growth-lab/nginx-projects/mathesia.conf`, copiando el patrón de `quiz.conf`: `location = /mathesia { return 308 …/mathesia/; }`, `location ^~ /mathesia/ { proxy_pass http://172.18.0.1:9063; … }` sin barra final, `client_max_body_size 12m`, `proxy_read_timeout 300s`, `X-Forwarded-Proto https`. El nginx interno del stack debe entonces aceptar el prefijo (o reescribirlo). Recargar `nginx-ssl-test` con `nginx -t` antes.

### 7.4 Respaldos (P1)

- Servicio `backup` activo en producción desde E3 (E0.10).
- `restore-test.sh` mensual y antes de cada inicio de periodo. Resultado anotado en la §11.
- Copia fuera de la máquina: documentar el destino (pendiente de decidir con la docente).

### 7.5 Barrido del WAF

`node scripts/waf-sweep.mjs --base https://<público> --local http://127.0.0.1:9063` recorre `registry.all()` (exportado como JSON en el build) con User-Agent de navegador y marca toda respuesta `text/html` en una ruta de API. Se ejecuta en cada despliegue que añada rutas.

### 7.6 Convivencia con la v1

- La v1 sigue en `/quiz/` y **no se toca** salvo urgencias. Sus datos no se migran.
- Cuando E3 cumpla su criterio, los quices nuevos se crean en la v2; la v1 queda para consulta de notas del periodo en curso hasta su cierre.
- Una función que la v1 tiene y la v2 aún no (calificación IA, diagnóstico) se sigue usando en la v1 hasta su etapa.

---

## 8. Riesgos

| Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|
| Sin respaldo y pérdida del volumen | Media | **Crítico** (notas) | E0.10 + `restore-test.sh` antes de la primera nota real (P1). |
| El WAF bloquea respuestas con álgebra | Media (la v1 funciona sin codificar) | Alto | Detección de HTML con mensaje claro; barrido E0.9; reabrir D2 con el caso concreto si aparece. |
| Subredes de Docker agotadas | Segura | Bloquea E0 | Subred explícita; plan B en §2.3. |
| Mismo origen con las demás apps de `lidis` (D1 = prefijo) | Media | Alto | Aceptado en CV2 §1.2; cookie con `Path=/mathesia/`, CSRF, CSP, reautenticación. |
| Encolar dentro de la transacción no funciona como se espera con pg-boss + Kysely | Media | Alto | Prueba de concepto en E0.5 antes de construir encima. |
| Semana de arranque (E0–E3) demasiado ambiciosa | Alta | Medio | El criterio de salida manda sobre la fecha; recortar *Avanzado* del constructor antes que la matriz o el respaldo. |
| Falsos positivos de integridad en móvil (teclado) | Alta | Alto | `interactive-widget`, ignorar con foco en campo, matriz de Android en E6. Hasta E6, la pérdida de foco descuenta vidas solo si la plantilla lo activa. |
| Reloj del cliente desfasado | Media | Medio | El servidor manda (`deadline_at` en cada latido); el cliente solo muestra. |
| Deriva entre CV2 §10 y el código | Media | Alto | Test del registro + matriz generada desde el registro + barrido WAF desde el registro. |
| Dependencias nuevas con versiones mayores recientes (Express 5, Zod, React 19) | Media | Bajo | Fijar versiones exactas en E0 y anotar en la §11. |

---

## 9. Orden de lectura de la v1 cuando toque portar

| Al empezar | Leer en `quiz_app/` |
|---|---|
| E1 | `src/services/password.policy.js`, `src/config/constants.js`, `src/middleware/{auth,csrf,rateLimit,methodOverride}.js`, `src/db/sessionStore.js`, `src/services/account.service.js`, `src/lib/basePath.js`, `tests/security/{session,reset,csrf}.test.js`, `PLAN.md` (H-057 y bitácora del WAF) |
| E2 | `src/services/items/**`, `src/services/roster.service.js`, `src/lib/nameMatch.js`, `scripts/importar-listas.js`, `seeds/*.json`, `tests/unit/generators.test.js`, `tests/security/{idor,authorization,parametric}.test.js` |
| E3 | `src/services/assessment/{attempt.service,itemBuilder,grading,declaration}.js`, `src/repositories/attempt.repo.js`, `public/js/pages/{quiz,teacher-monitor,teacher-assessment}.js`, `public/js/math.js`, `src/lib/proctorEvents.js`, `EVALUACION_ANTITRAMPAS.md` §12.2, `tests/security/attempt-integrity.test.js`, `tests/integration/{expiry,concurrency}.test.js` |
| E5 | `src/services/gradebook/*`, `src/lib/{xlsx,csvSafe}.js`, `scripts/import-notas-xlsx.js`, `scripts/desbloquear.js`, `tests/unit/{gradebook,xlsx,csvSafe,csvParser}.test.js` |
| E6 | `public/js/split-screen.js`, `src/services/assessment/{watermark,canary,seb}.js`, `src/services/analytics/*`, `EVALUACION_ANTITRAMPAS.md` |
| E7 | `src/services/grading/**`, `src/routes/ia-grading.routes.js`, `public/js/pages/teacher-calificacion-ia.js`, migraciones 013–019, `MODULO_CALIFICACION_IA.md`, `tests/unit/ia-*.test.js`, `tests/integration/ia-*.test.js` |
| E8 | `ARQUITECTURA_V2.md` §2.4b/d/f y Fase 6, `GAMIFICACION.md`, `diseno/` (iconos de XP, racha, reto) |

---

## 10. Primeros pasos concretos (próxima sesión)

1. Resolver **D3** (temas) antes de la Etapa 2; no bloquea la Etapa 0.
2. E0.1: `git init`, pnpm, workspaces, tsconfig, ESLint, Vitest.
3. `scripts/check-subnet.sh` y probar `docker network create --subnet 10.210.0.0/24 mathesia-test` (y borrarla) para validar §2.3 antes de escribir el compose.
4. E0.4 + E0.5: migración 0001 y la prueba de concepto de encolado transaccional.
5. E0.7–E0.8: web mínima y compose; criterio de salida E0.
6. E0.9: barrido del WAF (documental).

---

## 11. Bitácora

> Fecha · etapa/tarea · qué se decidió o descubrió · enlace a la PR o al commit.

| Fecha | Tarea | Nota |
|---|---|---|
| 2026-10-09 | — | Plan creado tras revisar CV2 y la v1. Pendientes: D1, D2, D3, D4, P1. |
| 2026-10-09 | D1, D2, D4 | D1 = prefijo `/mathesia/`. D2 = como la v1 (sin codificar; method override + detección de HTML). D4 = repo `LilianMar/mathesia`, push por SSH como `cghidalgos`. Riesgo de mismo origen añadido a CV2 §1.2. Siguen pendientes D3 y P1. |
