# Auth Server — Documentación de uso del API

Base URL: `https://auth.mipos.co.cr`

---

## Endpoints disponibles

| Método | Ruta | Descripción |
|--------|------|-------------|
| POST | `/login` | Autenticarse y obtener un JWT |
| GET | `/.well-known/jwks.json` | Obtener la clave pública en formato JWKS |

---

## 1. Uso desde Postman

### POST /login

1. Crear una nueva request en Postman
2. Configurar:
   - **Method:** `POST`
   - **URL:** `https://auth.mipos.co.cr/login`
   - **Headers:**
     - `Content-Type`: `application/json`
   - **Body** → raw → JSON:
     ```json
     {
       "base": "nombre_cliente",
       "password": "contraseña_del_cliente"
     }
     ```
3. Click **Send**

#### Respuesta exitosa (200 OK):

```json
{
  "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImtleS0yMDI0LTAxIn0.eyJpc3MiOiJodHRwczovL2F1dGgubWlwb3MuY28uY3IiLC...",
  "expires_in": 28800,
  "token_type": "Bearer"
}
```

#### Errores posibles:

| Código | error | message |
|--------|-------|---------|
| 400 | `bad_request` | Los campos base y password son requeridos |
| 401 | `invalid_credentials` | Base o contraseña incorrectos |
| 405 | `method_not_allowed` | Método no permitido |

### GET /.well-known/jwks.json

1. **Method:** `GET`
2. **URL:** `https://auth.mipos.co.cr/.well-known/jwks.json`
3. No requiere body ni autenticación

#### Respuesta (200 OK):

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "key-2024-01",
      "n": "base64url_del_modulo_RSA...",
      "e": "AQAB"
    }
  ]
}
```

### Usar el token en requests subsecuentes (Postman)

Una vez obtenido el token, enviarlo en el header `Authorization` hacia cualquier API cliente:

- **Header:** `Authorization`: `Bearer eyJhbGciOi...`

O bien en Postman: pestaña **Authorization** → Type: **Bearer Token** → pegar el token.

---

## 2. Integración por lenguaje

### PHP

#### Login (obtener token)

```php
<?php

$url  = 'https://auth.mipos.co.cr/login';
$data = json_encode([
    'base'     => 'nombre_cliente',
    'password' => 'contraseña_del_cliente',
]);

$ch = curl_init($url);
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_POSTFIELDS     => $data,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
]);

$response = curl_exec($ch);
$httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);

$result = json_decode($response, true);

if ($httpCode === 200) {
    $token     = $result['token'];
    $expiresIn = $result['expires_in'];
    // Usar $token en requests a las APIs cliente
} else {
    echo "Error: " . $result['message'];
}
```

#### Verificar token en una API cliente (PHP)

```php
<?php

// composer require firebase/php-jwt
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

$publicKey = file_get_contents('/ruta/segura/public.pem');
$authHeader = $_SERVER['HTTP_AUTHORIZATION'] ?? '';
$token = str_replace('Bearer ', '', $authHeader);

try {
    $decoded = JWT::decode($token, new Key($publicKey, 'RS256'));

    // Token válido — datos disponibles:
    $base = $decoded->base;  // Identificador del cliente
    $sub  = $decoded->sub;   // Mismo valor que base
    $exp  = $decoded->exp;   // Timestamp de expiración

} catch (\Exception $e) {
    http_response_code(401);
    echo json_encode(['error' => 'Token inválido o expirado']);
    exit;
}
```

---

### Node.js

#### Login (obtener token)

```javascript
// Con fetch nativo (Node 18+)
const response = await fetch('https://auth.mipos.co.cr/login', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    base: 'nombre_cliente',
    password: 'contraseña_del_cliente',
  }),
});

const result = await response.json();

if (response.ok) {
  const { token, expires_in, token_type } = result;
  // Usar token en requests a las APIs cliente
} else {
  console.error(`Error ${response.status}: ${result.message}`);
}
```

#### Verificar token en una API cliente (Node.js)

```javascript
// npm install jsonwebtoken
const jwt = require('jsonwebtoken');
const fs = require('fs');

const publicKey = fs.readFileSync('/ruta/segura/public.pem', 'utf8');

function verificarToken(req, res, next) {
  const authHeader = req.headers.authorization || '';
  const token = authHeader.replace('Bearer ', '');

  try {
    const decoded = jwt.verify(token, publicKey, { algorithms: ['RS256'] });

    // Token válido — datos disponibles:
    req.cliente = {
      base: decoded.base,  // Identificador del cliente
      sub: decoded.sub,    // Mismo valor que base
      exp: decoded.exp,    // Timestamp de expiración
    };
    next();

  } catch (err) {
    res.status(401).json({ error: 'Token inválido o expirado' });
  }
}
```

---

### Python

#### Login (obtener token)

```python
import requests

url = "https://auth.mipos.co.cr/login"
payload = {
    "base": "nombre_cliente",
    "password": "contraseña_del_cliente"
}

response = requests.post(url, json=payload)
result = response.json()

if response.status_code == 200:
    token = result["token"]
    expires_in = result["expires_in"]
    # Usar token en requests a las APIs cliente
else:
    print(f"Error {response.status_code}: {result['message']}")
```

#### Verificar token en una API cliente (Python)

```python
# pip install PyJWT cryptography
import jwt

with open("/ruta/segura/public.pem", "r") as f:
    public_key = f.read()

def verificar_token(token: str) -> dict:
    try:
        decoded = jwt.decode(token, public_key, algorithms=["RS256"])

        # Token válido — datos disponibles:
        # decoded["base"]  -> Identificador del cliente
        # decoded["sub"]   -> Mismo valor que base
        # decoded["exp"]   -> Timestamp de expiración
        return decoded

    except jwt.ExpiredSignatureError:
        raise Exception("Token expirado")
    except jwt.InvalidTokenError:
        raise Exception("Token inválido")
```

---

## 3. Estructura de la respuesta

### POST /login — Respuesta exitosa (200)

```json
{
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "expires_in": 28800,
  "token_type": "Bearer"
}
```

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `token` | string | JWT firmado con RS256. Enviar como `Authorization: Bearer <token>` |
| `expires_in` | integer | Segundos hasta que el token expire (28800 = 8 horas) |
| `token_type` | string | Siempre `"Bearer"` |

### Payload del JWT (decodificado)

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

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `iss` | string | Emisor del token (URL del Auth Server) |
| `aud` | string | Audiencia del token |
| `iat` | integer | Timestamp Unix de emisión |
| `exp` | integer | Timestamp Unix de expiración |
| `jti` | string | ID único del token (32 caracteres hex) |
| `sub` | string | Identificador del cliente (mismo que `base`) |
| `base` | string | Nombre del cliente en la base de datos |

### POST /login — Respuestas de error

Todas las respuestas de error tienen esta estructura:

```json
{
  "error": "codigo_de_error",
  "message": "Descripción legible del error"
}
```

| HTTP | `error` | `message` | Causa |
|------|---------|-----------|-------|
| 400 | `bad_request` | Los campos base y password son requeridos | Falta `base` o `password` en el body |
| 401 | `invalid_credentials` | Base o contraseña incorrectos | Credenciales no válidas |
| 405 | `method_not_allowed` | Método no permitido | Se usó GET u otro método en vez de POST |
| 404 | `not_found` | Endpoint no existe | Ruta no reconocida |
| 500 | `server_error` | Error interno | Error del servidor (sin detalles expuestos) |

---

## Notas importantes

- El token tiene una validez de **8 horas** (28800 segundos) por defecto.
- Las APIs cliente **nunca** consultan la base de datos del Auth Server. Solo verifican la firma del JWT usando `public.pem` o consultando el endpoint JWKS.
- El campo `base` en el JWT identifica al cliente. Usarlo para filtrar datos en cada API.
- Si el token expira, el cliente debe volver a hacer `POST /login` para obtener uno nuevo.
