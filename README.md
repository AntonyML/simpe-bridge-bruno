# SINPE Bridge API — Colección Bruno

Colección completa de requests HTTP para la API SINPE Bridge, lista para usar en
[Bruno](https://www.usebruno.com/) (alternativa open-source a Postman/Insomnia).

---

## Estructura de la colección

```
simpe-bridge-bruno/
│
├── README.md                    ← Este archivo
├── bruno.json                   ← Configuración de la colección
│
├── environments/
│   ├── production.bru           ← api.tonyml.com (vía Worker Cloudflare)
│   ├── github-codespace.bru     ← Backend directo (GitHub Codespace)
│   └── local-fastapi.bru        ← Backend local (localhost:8000)
│
├── Health/
│   ├── 01-root.bru              ← GET /
│   ├── 02-health-check.bru      ← GET /health
│   └── 03-readiness-check.bru   ← GET /ready
│
├── Payments/
│   ├── 01-post-payment.bru      ← POST /api/v1/payments
│   ├── 02-get-payment-by-id.bru ← GET  /api/v1/payments/{message_id}
│   └── 03-list-payments.bru     ← GET  /api/v1/payments?id_pos=...
│
├── Orders/
│   ├── 01-create-order.bru      ← POST /api/v1/orders
│   ├── 02-get-order.bru         ← GET  /api/v1/orders/{order_number}
│   └── 03-list-orders.bru       ← GET  /api/v1/orders?id_pos=...
│
├── Uploads/
│   ├── 01-upload-receipt.bru    ← POST /api/v1/uploads/receipts (multipart)
│   ├── 02-upload-qr.bru         ← POST /api/v1/uploads/qr (multipart)
│   ├── 03-get-upload.bru        ← GET  /api/v1/uploads/{upload_id}
│   ├── 04-delete-upload.bru     ← DELETE /api/v1/uploads/{upload_id}
│   └── assets/
│       ├── dummy-receipt.jpg    ← Imagen de prueba para uploads
│       └── dummy-qr.png         ← QR de prueba para uploads
│
├── Docs/
│   ├── 01-swagger-ui.bru        ← GET /api/v1/docs (Swagger HTML)
│   └── 02-openapi-json.bru      ← GET /api/v1/openapi.json
│
└── Worker/
    ├── 01-proxy-health.bru      ← Verifica que el Worker responde
    ├── 02-proxy-cors-preflight.bru ← OPTIONS preflight CORS
    └── 03-proxy-invalid-apikey.bru ← Test de rechazo de API Key inválida
```

---

## Cómo abrir la colección en Bruno

1. Descarga e instala [Bruno](https://www.usebruno.com/downloads)
2. Abre Bruno → **Open Collection**
3. Navega a: `C:\DEV\Simpe-bridge\simpe-bridge-bruno`
4. Selecciona la carpeta → **OK**
5. En la barra lateral aparecerán todas las carpetas y requests

---

## Seleccionar el entorno

Antes de ejecutar cualquier request, selecciona el environment correcto:

| Environment | Cuándo usarlo |
|-------------|---------------|
| `production` | Llamadas reales vía `https://api.tonyml.com` (Worker Cloudflare) |
| `github-codespace` | Llamadas directas al backend en GitHub Codespace |
| `local-fastapi` | Desarrollo local con `uvicorn app.main:app --port 8000` |

Para cambiar: esquina superior derecha de Bruno → dropdown de entornos.

---

## Variables de entorno disponibles

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `baseUrl` | URL base de la API | `https://api.tonyml.com` |
| `apiKey` | API Key para `x-api-key` header | `468bc1becd1b92ff7a6cdeafa31e891d` |
| `correlationId` | UUID para rastreo de requests | `a7a74c7c-c310-4bbb-...` |
| `messageId` | UUID de un mensaje SINPE | `89d05111-5134-4e81-...` |
| `deviceId` | Identificador del dispositivo | `android-device-001` |
| `deviceHash` | Hash anonimizado del device | `6a40cd4b48a023e5...` |
| `idPos` | Identificador del terminal POS | `pos-terminal-001` |
| `orderNumber` | Número de orden del POS | `ORD-2025-001` |
| `uploadId` | UUID de un upload existente | `upload-abc-123` |

---

## Headers comunes (incluidos en todos los requests)

```
x-api-key: {{apiKey}}
x-correlation-id: {{correlationId}}
x-request-id: req-<unique-id>
x-trace-id: trace-<unique-id>
x-device-id: {{deviceId}}
```

> El header `x-api-key` es validado por el Cloudflare Worker (en `worker_backup.js`)
> y también puede ser validado por el backend FastAPI según configuración.

---

## Arquitectura del sistema

```
Cliente / Bruno
      ↓  (HTTPS)
api.tonyml.com/api/*
      ↓
Cloudflare Worker (proxy-apy-bridgesimpe)
  ├─ Valida x-api-key
  ├─ Valida x-signature (HMAC SHA-256) — solo para JSON
  ├─ Valida Origin / CORS
  ├─ Filtra User-Agents maliciosos
  ├─ Agrega headers de tracing
  └─ Reenvía al backend
      ↓
GitHub Codespace FastAPI (puerto 8000)
  ├─ TraceMiddleware
  ├─ CORSMiddleware
  ├─ /api/v1/payments   — Mensajes SINPE
  ├─ /api/v1/orders     — Órdenes de compra
  └─ /api/v1/uploads/*  — Subida de imágenes
```

---

## Flujo completo: Pago SINPE

```
1. POS crea orden:        POST /api/v1/orders
2. Android detecta SMS:   POST /api/v1/payments (con body del SMS)
3. Backend concilia:      El mensaje SINPE se asocia a la orden
4. POS consulta estado:   GET  /api/v1/orders/{order_number}
5. Android sube imagen:   POST /api/v1/uploads/receipts (multipart)
```

---

## Probar uploads (multipart/form-data)

Los requests de uploads apuntan a imágenes dummy en `Uploads/assets/`.
Para probar con imágenes reales:

1. Abre el request `01-upload-receipt.bru` en Bruno
2. En el body, campo `file`, cambia la ruta a tu imagen local
3. Ejecuta el request

Imágenes dummy incluidas (archivos mínimos válidos para testing):
- `Uploads/assets/dummy-receipt.jpg` — JPEG de prueba
- `Uploads/assets/dummy-qr.png` — PNG de prueba

---

## Swagger / OpenAPI

La documentación interactiva está disponible **directamente en el backend**
(no pasa por el proxy Worker):

- **Swagger UI**: `{backend_url}/api/v1/docs`
- **OpenAPI JSON**: `{backend_url}/api/v1/openapi.json`

Con el environment `github-codespace`:
```
https://super-duper-space-broccoli-gg49xpjx57jfp774-8000.app.github.dev/api/v1/docs
```

---

## ⚠️ Bugs e inconsistencias detectadas

### BUG #1 — Worker.js producción: path stripping incorrecto

**Archivo**: `proxy-apy-bridgesimpe/src/worker.js`

**Problema**: El worker actual strip `/api` del path:
```javascript
target.pathname = incomingUrl.pathname.replace(/^\/api/, "") || "/";
```

Esto causa:
- Cliente envía: `api.tonyml.com/api/v1/payments`
- Path resultante en backend: `/v1/payments`
- Pero FastAPI espera: `/api/v1/payments`

**Resultado**: Todos los endpoints API retornan `404 Not Found` en producción.

**Fix sugerido**: No strip el prefijo, o ajustar la regex para que el backend
reciba el path correcto:
```javascript
// Opción A: No strip
target.pathname = incomingUrl.pathname;

// Opción B: Strip solo si el backend tiene rutas sin /api
// (requeriría cambiar el prefix en FastAPI)
```

> El `worker_backup.js` tiene el mismo problema en `buildTargetURL()`.
> La solución más simple es cambiar a opción A en ambos workers.

---

### BUG #2 — Uploads router: prefijo duplicado

**Archivo**: `sinpe-bridge-api/app/api/v1/endpoints/uploads.py`

**Problema**: El router de uploads define su propio prefix:
```python
router = APIRouter(prefix="/api/v1/uploads", tags=["uploads"])
```

Pero en `router.py` (y eventualmente en `main.py`) se monta con prefix `/api/v1`:
```python
# router.py NO incluye uploads actualmente — pero si se incluyera:
api_router.include_router(uploads.router, ...)
# app.include_router(api_router, prefix="/api/v1")
```

**Resultado potencial**: Rutas duplicadas `/api/v1/api/v1/uploads/receipts`.

**Fix**: Cambiar el router en `uploads.py` a:
```python
router = APIRouter(tags=["uploads"])
```
Y agregar el prefijo en `router.py`:
```python
api_router.include_router(uploads.router, prefix="/uploads", tags=["Uploads"])
```

---

### INCONSISTENCIA #1 — Worker actual vs README

El `README.md` del Worker describe un pipeline de 9 fases de seguridad
(HMAC, API Key, Origin check, UA filter, etc.), pero el `worker.js` en
producción es un proxy dumb sin ninguna validación. Toda la lógica de
seguridad está en `worker_backup.js`.

**Acción recomendada**: Hacer deploy de `worker_backup.js` como `worker.js`
para activar la seguridad descrita en el README.

---

### INCONSISTENCIA #2 — Health endpoints no expuestos vía proxy

El proxy solo intercepta `api.tonyml.com/api/*`. Los endpoints `/health`
y `/ready` están en la raíz del backend, fuera del patrón del Worker.

**Para producción**, si se quieren exponer estos endpoints públicamente:
- Agregar una ruta adicional en `wrangler.toml` para `/health` y `/ready`, o
- Crear endpoints equivalentes dentro del prefijo `/api/v1/health`

---

### NOTA — Módulos vacíos

Los siguientes endpoints están declarados en `app/api/v1/` pero tienen
implementación vacía (archivos Python en blanco):

- `administration.py` — Sin implementar
- `monitoring.py` — Sin implementar
- `notifications.py` — Sin implementar
- `reconciliation.py` — Sin implementar

Tampoco están incluidos en `router.py`. Son placeholders para funcionalidad futura.

---

## Seguridad — API Key y HMAC

### API Key

El `x-api-key` debe coincidir con el valor configurado en el Worker:

```
API_KEY=468bc1becd1b92ff7a6cdeafa31e891d  (development/staging)
```

> ⚠️ Esta key está en el `.dev.vars` del Worker y en el `.env.example`.
> En producción real, usar `wrangler secret put API_KEY` para no exponerla.

### HMAC SHA-256 (x-signature)

Para requests JSON, la firma se calcula así (desde `worker_backup.js`):

```javascript
const body = JSON.stringify(requestBody);
const key = await crypto.subtle.importKey("raw", encode(apiKey), {name:"HMAC", hash:"SHA-256"}, false, ["sign"]);
const sig = await crypto.subtle.sign("HMAC", key, encode(body));
const hex = Array.from(new Uint8Array(sig)).map(b => b.toString(16).padStart(2,"0")).join("");
// hex → x-signature header
```

El header `x-signature` es **opcional** en el Worker. Si no se envía, se omite la validación.

---

## Comandos útiles

```bash
# Levantar backend local
cd sinpe-bridge-api
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Levantar Worker local
cd proxy-apy-bridgesimpe
npm run dev  # → http://localhost:8787

# Ver logs del Worker en producción
cd proxy-apy-bridgesimpe
npm run logs  # wrangler tail --env production

# Deploy del Worker
cd proxy-apy-bridgesimpe
npm run deploy

# Ver Swagger local
open http://localhost:8000/api/v1/docs
```

---

## Contacto / Contexto

- **Proyecto**: SINPE Bridge — Integración de pagos SINPE Móvil con sistemas POS en Costa Rica
- **Backend**: FastAPI (Python) en GitHub Codespaces
- **Proxy**: Cloudflare Worker en `api.tonyml.com`
- **App móvil**: Android (Kotlin) — detecta SMS de bancos y los envía al backend
- **Colección generada**: Análisis automático del proyecto con MCP Filesystem + Claude
#   s i m p e - b r i d g e - b r u n o  
 