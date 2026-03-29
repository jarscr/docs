# Guía de integración para APIs cliente

Esta guía es para los desarrolladores de las APIs que consumen tokens del Auth Server de MiPOS. Se entrega junto con el archivo `public.pem` en un repositorio privado.

---

## Qué se entrega

Cada API cliente recibe acceso a un repositorio privado que contiene:

```
├── public.pem          # Clave pública RSA para verificar tokens
└── GUIA-CLIENTE.md     # Este documento
```

**La clave pública NO se descarga del servidor.** Se distribuye únicamente a través del repositorio. El endpoint JWKS (`/.well-known/jwks.json`) está disponible pero la forma oficial de obtener la clave es desde el repo.

---

## Credenciales

Al dar de alta un cliente se le entregan dos datos:

| Dato | Descripción | Ejemplo |
|------|-------------|---------|
| `base` | Identificador único del cliente | `restaurante_central` |
| `password` | Contraseña del cliente (bcrypt en BD) | Se entrega por canal seguro |

Estos datos se usan exclusivamente para obtener un token del Auth Server. **No se usan directamente en la API cliente.**

---

## Flujo de autenticación

```
1. Tu app → POST /login → Auth Server     (envías base + password)
2. Auth Server → tu app                    (recibes el JWT)
3. Tu app → API Cliente                    (envías el JWT en Authorization header)
4. API Cliente → verifica firma con public.pem (sin tocar BD del Auth Server)
```

La API cliente **nunca** se comunica con el Auth Server ni con su base de datos. Solo verifica la firma del token localmente usando `public.pem`.

---

## Paso 1 — Obtener un token

Hacer `POST` a `https://auth.mipos.co.cr/login`:

```bash
curl -X POST https://auth.mipos.co.cr/login \
  -H "Content-Type: application/json" \
  -d '{"base":"tu_base","password":"tu_password"}'
```

Respuesta:

```json
{
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 28800,
  "token_type": "Bearer"
}
```

- `token` — JWT firmado. Válido por 8 horas.
- `expires_in` — segundos hasta que expire (28800 = 8 horas).
- `token_type` — siempre `Bearer`.

---

## Paso 2 — Enviar el token a la API

Incluir el token en el header `Authorization` de cada request:

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

---

## Paso 3 — Verificar el token en tu API

### Ubicar la clave pública

Clonar el repositorio entregado y copiar `public.pem` a una ruta segura en el servidor de la API:

```bash
# Ejemplo: copiar a una ruta fuera del webroot
cp public.pem /etc/mipos/public.pem
chmod 644 /etc/mipos/public.pem
```

**Importante:** `public.pem` NO debe estar dentro del directorio público del servidor web (no en `public/`, `www/`, ni `htdocs/`).

### PHP

```bash
composer require firebase/php-jwt
```

```php
<?php

use Firebase\JWT\JWT;
use Firebase\JWT\Key;

// Ruta al archivo public.pem del repositorio
$publicKey = file_get_contents('/etc/mipos/public.pem');

// Extraer token del header Authorization
$authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
$token = str_replace('Bearer ', '', $authHeader);

if ($token === '') {
    http_response_code(401);
    exit(json_encode(['error' => 'Token no proporcionado']));
}

try {
    $decoded = JWT::decode($token, new Key($publicKey, 'RS256'));

    // Token válido — datos del cliente:
    $base = $decoded->base;   // Identificador del cliente
    $sub  = $decoded->sub;    // Mismo valor que base
    $iss  = $decoded->iss;    // https://auth.mipos.co.cr
    $exp  = $decoded->exp;    // Timestamp de expiración

} catch (\Firebase\JWT\ExpiredException $e) {
    http_response_code(401);
    exit(json_encode(['error' => 'Token expirado']));

} catch (\Exception $e) {
    http_response_code(401);
    exit(json_encode(['error' => 'Token inválido']));
}
```

### Node.js

```bash
npm install jsonwebtoken
```

```javascript
const jwt = require('jsonwebtoken');
const fs = require('fs');

// Ruta al archivo public.pem del repositorio
const publicKey = fs.readFileSync('/etc/mipos/public.pem', 'utf8');

function authMiddleware(req, res, next) {
  const token = (req.headers.authorization || '').replace('Bearer ', '');

  if (!token) {
    return res.status(401).json({ error: 'Token no proporcionado' });
  }

  try {
    const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256'] });

    // Token válido — datos del cliente:
    req.cliente = {
      base: decoded.base,   // Identificador del cliente
      sub:  decoded.sub,    // Mismo valor que base
      iss:  decoded.iss,    // https://auth.mipos.co.cr
      exp:  decoded.exp,    // Timestamp de expiración
    };
    next();

  } catch (err) {
    const message = err.name === 'TokenExpiredError'
      ? 'Token expirado'
      : 'Token inválido';
    res.status(401).json({ error: message });
  }
}

// Uso en Express
app.get('/ventas', authMiddleware, (req, res) => {
  const base = req.cliente.base;
  // ... lógica del endpoint
});
```

### Python

```bash
pip install PyJWT cryptography
```

```python
import jwt

# Ruta al archivo public.pem del repositorio
with open('/etc/mipos/public.pem') as f:
    public_key = f.read()

def verificar_token(token: str) -> dict:
    try:
        decoded = jwt.decode(token, public_key, algorithms=['RS256'])

        # Token válido — datos del cliente:
        # decoded['base']  → Identificador del cliente
        # decoded['sub']   → Mismo valor que base
        # decoded['iss']   → https://auth.mipos.co.cr
        # decoded['exp']   → Timestamp de expiración
        return decoded

    except jwt.ExpiredSignatureError:
        raise Exception('Token expirado')
    except jwt.InvalidTokenError:
        raise Exception('Token inválido')
```

Middleware para Flask:

```python
from functools import wraps
from flask import request, jsonify

def requiere_auth(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = (request.headers.get('Authorization') or '').replace('Bearer ', '')

        if not token:
            return jsonify({'error': 'Token no proporcionado'}), 401

        try:
            request.cliente = verificar_token(token)
        except Exception as e:
            return jsonify({'error': str(e)}), 401

        return f(*args, **kwargs)
    return decorated

@app.route('/ventas')
@requiere_auth
def ventas():
    base = request.cliente['base']
    # ...
```

Middleware para FastAPI:

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer()

async def requiere_auth(cred: HTTPAuthorizationCredentials = Depends(security)):
    try:
        return verificar_token(cred.credentials)
    except Exception as e:
        raise HTTPException(status_code=401, detail=str(e))

@app.get('/ventas')
async def ventas(cliente: dict = Depends(requiere_auth)):
    base = cliente['base']
    # ...
```

---

## Contenido del JWT

Cada token contiene estos campos en su payload:

```json
{
  "iss": "https://auth.mipos.co.cr",
  "aud": "https://auth.mipos.co.cr",
  "iat": 1700000000,
  "exp": 1700028800,
  "jti": "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4",
  "sub": "nombre_cliente",
  "base": "nombre_cliente"
}
```

| Campo | Tipo | Para qué usarlo |
|-------|------|----------------|
| `base` | string | **Identificador del cliente.** Usar este campo para filtrar datos en tu API. |
| `sub` | string | Mismo valor que `base`. Estándar JWT. |
| `exp` | integer | Verificar si el token ya expiró (la librería lo hace automáticamente). |
| `jti` | string | ID único del token. Útil si se implementa revocación. |
| `iss` | string | Verificar que el token fue emitido por `https://auth.mipos.co.cr`. |
| `aud` | string | Audiencia del token. |
| `iat` | integer | Cuándo se emitió el token (Unix timestamp). |

---

## Errores al obtener token

| HTTP | `error` | `message` | Qué hacer |
|------|---------|-----------|-----------|
| 400 | `bad_request` | Los campos base y password son requeridos | Verificar que envías `base` y `password` en el body JSON |
| 401 | `invalid_credentials` | Base o contraseña incorrectos | Verificar credenciales. Contactar al administrador si persiste |
| 405 | `method_not_allowed` | Método no permitido | El endpoint `/login` solo acepta `POST` |
| 500 | `server_error` | Error interno | Reportar al administrador del Auth Server |

---

## Token expirado

El token dura **8 horas**. Cuando expire, hacer `POST /login` nuevamente para obtener uno nuevo. No hay endpoint de refresh.

Estrategia recomendada para tu aplicación:

```
1. Guardar el token y el timestamp de expiración en memoria/cache
2. Antes de cada request, verificar si faltan menos de 5 minutos para expirar
3. Si está por expirar, obtener un token nuevo antes de hacer el request
```

---

## Rotación de clave pública

Cuando se roten las claves RSA del Auth Server:

1. Se publicará un nuevo `public.pem` en el repositorio
2. Se notificará a los clientes por el canal acordado
3. Los tokens emitidos con la clave anterior seguirán siendo válidos hasta que expiren
4. Actualizar `public.pem` en tu servidor lo antes posible

---

## Reglas de seguridad

- **No exponer `public.pem` en endpoints públicos** de tu API
- **No loguear tokens completos** — si necesitas loguear, usar solo los últimos 8 caracteres
- **No decodificar tokens en herramientas online** (jwt.io, etc.) en producción
- **No cachear tokens en localStorage** del navegador — usar cookies httpOnly o memoria
- **No compartir credenciales** (`base`/`password`) entre ambientes (dev, staging, prod)
- **Verificar siempre el algoritmo** — usar `algorithms: ['RS256']` explícitamente para prevenir ataques de confusión de algoritmo
