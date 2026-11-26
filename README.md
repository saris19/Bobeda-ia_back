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

