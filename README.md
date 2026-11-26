# Bóveda IA — Backend (API)

API REST de **Bóveda IA**, un sistema para almacenar, organizar y proteger documentos digitales. Cada archivo
se cifra antes de guardarse; el acceso se controla por roles, permisos por documento y nivel de sensibilidad,
y la inteligencia artificial clasifica los documentos, detecta información sensible, permite **buscar dentro
de los archivos cifrados** y **detecta accesos sospechosos**.

**Python + FastAPI · AES-256-GCM · scikit-learn · PostgreSQL**

La interfaz web está en el repositorio `boveda-ia-frontend`.

```
boveda-ia-backend/
├── app/
│   ├── ai/         clasificador, datos sensibles, índice ciego, detector de anomalías
│   ├── crypto/     cifrado AES-256-GCM y llavero con rotación
│   ├── routers/    endpoints (documentos, búsqueda, alertas, papelera, administración…)
│   └── services/   permisos, políticas, alertas, bitácora, almacenamiento
├── ejemplos/       documentos ficticios para la demostración y las pruebas
├── scripts/simulate_attacks.py   demostración de ataques reales contra la API
├── tests/          22 pruebas automáticas
├── render.yaml     Blueprint de Render (API + PostgreSQL)
└── run.py          arranque local
```

---

## 1. Funcionalidades

| Módulo | Detalle |
|---|---|
| Autenticación | JWT, contraseñas con bcrypt, política de contraseñas, bloqueo tras 5 intentos fallidos |
| Roles (RBAC) | `admin` (todo), `editor` (sube, comparte, elimina lo suyo), `viewer`/lector (solo lo que le comparten) |
| Políticas por sensibilidad | Confidencial: no se comparte con todo el rol Lector. Restringido: solo con usuarios con nombre propio. Se aplican al instante si la sensibilidad cambia |
| Cifrado | Cifrado de sobre AES-256-GCM: llave aleatoria por documento y por versión; integridad SHA-256; datos asociados que impiden intercambiar archivos cifrados |
| Llavero y rotación | Llave raíz (entorno) → llave maestra versionada (BD) → llave de cada documento. Rotar re-envuelve todas las llaves sin tocar el contenido cifrado |
| Gestión documental | Subir, listar, descargar, compartir, revocar |
| Versiones | Historial cifrado de cada documento; descargar o restaurar cualquier versión |
| Papelera | Borrado suave con restauración durante 30 días y purga automática |
| IA: clasificación | TF-IDF de palabras y caracteres + regresión logística en 7 categorías (exactitud por validación cruzada ≈ 71 %) |
| IA: aprendizaje con correcciones | El modelo se reentrena con las correcciones de categoría (guardando solo palabras comunes, sin nombres ni números) |
| IA: datos sensibles | Cédula, NIT, tarjetas (Luhn), cuentas, correos, teléfonos, direcciones, contraseñas, salud, Ley 1581 de 2012; valores enmascarados |
| IA: búsqueda inteligente | Busca dentro del contenido cifrado con un índice ciego HMAC; tolera plurales y errores; entiende la intención |
| IA: accesos sospechosos | 7 reglas explicables + modelo Isolation Forest sobre la bitácora, con alertas y respuestas automáticas |
| Bitácora | Todas las acciones con usuario, IP, resultado y detalle |

## 2. Detección de accesos sospechosos

| Regla | Condición | Severidad | Respuesta automática |
|---|---|---|---|
| Fuerza bruta | ≥ 3 inicios fallidos del mismo usuario en 10 min | alta / crítica (≥ 5) | Cuenta bloqueada 15 min |
| Ataque desde una IP | Una IP falla con ≥ 3 usuarios distintos en 10 min | crítica | IP bloqueada para iniciar sesión 15 min |
| Descarga masiva | ≥ 10 descargas en 10 min | alta / crítica | Descargas del usuario pausadas 15 min |
| Accesos denegados | ≥ 3 intentos sin permiso en 10 min | media / alta | — |
| Fuera de horario | Documento confidencial o restringido entre 22:00 y 06:00 | media | — |
| IP nueva | Inicio de sesión desde una IP nunca usada | baja | — |
| Exploración masiva | ≥ 25 búsquedas o consultas en 5 min | media | — |

**Modelo de IA (Isolation Forest, 200 árboles):** convierte cada evento en 12 variables de comportamiento y
marca como anómalo lo que se aleja del comportamiento normal (puntaje ≥ 60/100), explicando qué variables la
causaron. Arranca con una línea base sintética y se reentrena con la bitácora real cada 200 eventos.

## 3. Búsqueda inteligente (índice ciego)

El texto de los documentos nunca se guarda. Cada palabra relevante se normaliza (`facturas → factur`) y se
transforma con HMAC-SHA256 usando una llave derivada de la llave raíz. La relevancia combina contenido cifrado
(55 %), metadatos con tolerancia a errores (35 %) e intención detectada por el clasificador (10 %).

## 4. Ejecutar en tu computador

Necesitas **Python 3.10+** (Anaconda sirve).

```bash
conda create -n boveda python=3.11 -y     # solo la primera vez
conda activate boveda
pip install -r requirements-dev.txt        # solo la primera vez
python run.py
```

API en <http://127.0.0.1:8000> y documentación interactiva en <http://127.0.0.1:8000/docs>.
Usuario inicial en desarrollo: **admin** / **Admin123\*** (en producción la contraseña la genera Render).

Para usar la interfaz web, en otra terminal ejecuta el repositorio `boveda-ia-frontend` (`npm run dev`).

## 5. Pruebas y demostración

```bash
python -m pytest -v
```

22 pruebas: cifrado y manipulación, validadores, enmascaramiento, roles y permisos, políticas por
sensibilidad, fuerza bruta, versiones, papelera, correcciones y reentrenamiento, rotación de llaves, índice
ciego y búsqueda, panel, migraciones, reglas de detección, respuestas automáticas, Isolation Forest y alertas.

Con la API encendida, `python scripts/simulate_attacks.py` lanza ataques reales (fuerza bruta, descarga
masiva, accesos denegados y exploración) y muestra las alertas generadas.

## 6. API principal

| Método | Ruta | Rol |
|---|---|---|
| POST | `/api/auth/login` | público |
| GET | `/api/stats` · `/api/search` | autenticado |
| GET / POST | `/api/documents` | autenticado / admin, editor |
| GET | `/api/documents/{id}/download` | permiso de lectura |
| GET / POST | `/api/documents/{id}/versions` | lectura / gestión |
| GET · POST | `/api/documents/{id}/versions/{n}/download` · `/restore` | lectura · gestión |
| PATCH | `/api/documents/{id}/classification` | gestión (admin, editor) |
| POST · DELETE | `/api/documents/{id}/share` · `/permissions/{pid}` | gestión |
| DELETE | `/api/documents/{id}` (a la papelera) | gestión |
| GET · POST · DELETE | `/api/trash` · `/api/trash/{id}/restore` · `/api/trash/{id}` | propietario, admin |
| GET · PATCH | `/api/alerts` · `/api/alerts/summary` · `/api/alerts/{id}` | admin |
| POST | `/api/alerts/model/retrain` | admin |
| GET · POST | `/api/admin/security` · `/api/admin/rotate-key` · `/api/admin/classifier/retrain` | admin |
| GET / POST / PATCH | `/api/users` · `/api/users/{id}` · `/api/users/{id}/unblock` | admin |
| GET | `/api/audit` | admin |

## 7. Configuración (variables de entorno)

| Variable | Uso | Por defecto |
|---|---|---|
| `VAULT_MASTER_KEY` | Llave raíz en base64 (32 bytes) | se genera en `data/master.key` |
| `VAULT_JWT_SECRET` | Secreto para firmar tokens | se genera en `data/jwt.secret` |
| `VAULT_ADMIN_USER` / `VAULT_ADMIN_PASSWORD` | Administrador inicial | `admin` / `Admin123*` |
| `VAULT_ENV` | `production` exige llaves y PostgreSQL definidos | `development` |
| `DATABASE_URL` | Conexión PostgreSQL (la pone Render) | SQLite local |
| `VAULT_STORAGE_BACKEND` | `disk` o `database` | `disk` local / `database` producción |
| `VAULT_CORS_ORIGINS` · `VAULT_CORS_ORIGIN_REGEX` | Orígenes del frontend permitidos | — |
| `VAULT_TIMEZONE` | Zona horaria para la regla de horario | `America/Bogota` |
| `VAULT_OFF_HOURS_START` · `VAULT_OFF_HOURS_END` | Horario no laboral | 22 · 6 |
| `VAULT_TRASH_DAYS` | Días en la papelera | 30 |
| `VAULT_DOWNLOAD_BLOCK_MINUTES` | Pausa de descargas tras descarga masiva | 15 |
| `VAULT_TOKEN_MINUTES` · `VAULT_MAX_UPLOAD_MB` | Duración de sesión · tamaño máximo | 60 · 10 |

> ⚠️ Si se pierde la llave raíz (`VAULT_MASTER_KEY`), los documentos **no se pueden recuperar**.
> La carpeta `data/` (llave, secreto y base local) está en `.gitignore`: nunca la subas al repositorio.

## 8. Despliegue en Render

**Si ya tienes la API publicada** (servicio `boveda-ia-api` conectado al repositorio anterior `boveda-ia`):
no crees un Blueprint nuevo, porque generaría otra base de datos y otra llave maestra y los documentos
existentes dejarían de abrirse. En su lugar:

1. Render → **boveda-ia-api → Settings → Build & Deploy → Repository** → elige `boveda-ia-backend`.
2. En el mismo lugar, deja **Root Directory** vacío (antes era `backend`).
3. **Manual Deploy → Deploy latest commit** y prueba
   <https://boveda-ia-api.onrender.com/api/health> → `{"status":"ok","version":"1.0.0"}`.

**Despliegue desde cero:** Render → **New → Blueprint** → selecciona este repositorio. El `render.yaml` crea
la API y la base PostgreSQL, y genera la llave maestra, el secreto JWT y la contraseña del admin (en
**Environment**, junto a `VAULT_ADMIN_PASSWORD`). Respalda `VAULT_MASTER_KEY` en un lugar seguro.

| Síntoma | Causa probable |
|---|---|
| "Could not open requirements file" | El servicio todavía tiene Root Directory `backend`: déjalo vacío |
| "Faltan variables de entorno obligatorias" | Falta alguna variable del Blueprint |
| `blocked by CORS policy` en el navegador | La URL del frontend no coincide con `VAULT_CORS_ORIGINS` (https, sin `/` final) |
| Error 409 "Falla de integridad" al descargar | `VAULT_MASTER_KEY` cambió respecto a la que cifró el documento |

El plan gratuito duerme la API tras 15 minutos sin tráfico (el primer acceso tarda ~1 minuto) y la base
PostgreSQL gratuita vence a los 30 días.

## 9. Limitaciones conocidas

- El clasificador parte de 210 ejemplos semilla; mejora con las correcciones de los usuarios.
- La detección de datos sensibles se basa en patrones; no reconoce nombres propios.
- Los PDF escaneados no tienen texto extraíble sin OCR.
- El modelo de anomalías vive en memoria: se reentrena al arrancar y periódicamente desde la bitácora.
- El índice ciego revela si dos documentos comparten una palabra (no cuál) a quien acceda a la base de datos.
