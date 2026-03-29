# Deploy en aaPanel con Nginx

Guía para desplegar el Auth Server en un servidor con aaPanel y Nginx.

---

## 1. Crear el sitio en aaPanel

1. Ir a **Website** → **Add site**
2. Configurar:
   - **Domain:** `auth.mipos.co.cr`
   - **Root Directory:** `/www/wwwroot/auth.mipos.co.cr`
   - **PHP Version:** 8.2 (o superior)
   - **Database:** No crear (la BD `api` ya existe)

---

## 2. Subir archivos

Subir el proyecto al servidor. La estructura en el servidor debe quedar:

```
/www/wwwroot/auth.mipos.co.cr/
├── composer.json
├── composer.lock
├── .env
├── keys/
│   ├── private.pem
│   └── public.pem
├── public/
│   └── index.php
├── src/
│   └── ...
└── vendor/
    └── ...
```

```bash
# Desde tu máquina local
rsync -avz --exclude='.git' --exclude='.env' --exclude='keys/private.pem' \
  ./ usuario@servidor:/www/wwwroot/auth.mipos.co.cr/

# En el servidor
cd /www/wwwroot/auth.mipos.co.cr
composer install --optimize-autoloader --no-dev
```

---

## 3. Configurar .env en el servidor

```bash
cp .env.example .env
nano .env
```

```env
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=api
DB_USER=usuario_produccion
DB_PASS=contraseña_produccion

JWT_PRIVATE_KEY_PATH=/www/wwwroot/auth.mipos.co.cr/keys/private.pem
JWT_PUBLIC_KEY_PATH=/www/wwwroot/auth.mipos.co.cr/keys/public.pem

JWT_ISSUER=https://auth.mipos.co.cr
JWT_TTL_SECONDS=28800

JWT_KID=key-2024-01
```

---

## 4. Generar claves RSA (si no existen)

```bash
cd /www/wwwroot/auth.mipos.co.cr
mkdir -p keys
openssl genrsa -out keys/private.pem 2048
openssl rsa -in keys/private.pem -pubout -out keys/public.pem
```

---

## 5. Permisos

```bash
# Owner: www (usuario de Nginx/PHP en aaPanel)
chown -R www:www /www/wwwroot/auth.mipos.co.cr

# Directorio de claves
chmod 700 keys/
chmod 600 keys/private.pem
chmod 644 keys/public.pem

# .env no debe ser accesible desde web
chmod 600 .env

# vendor y src solo lectura para web
chmod -R 755 public/
chmod -R 750 src/
chmod -R 750 vendor/
```

---

## 6. Configuración Nginx en aaPanel

Ir a **Website** → `auth.mipos.co.cr` → **Config** (o **Settings** → **Nginx Configuration**).

Reemplazar el contenido del bloque `server` con:

```nginx
server {
    listen 80;
    listen 443 ssl http2;
    server_name auth.mipos.co.cr;

    # aaPanel SSL (se configura aparte en SSL → Let's Encrypt)
    # Las líneas ssl_certificate las agrega aaPanel automáticamente

    root /www/wwwroot/auth.mipos.co.cr/public;
    index index.php;

    # Logs
    access_log /www/wwwlogs/auth.mipos.co.cr.log;
    error_log  /www/wwwlogs/auth.mipos.co.cr.error.log;

    # Forzar HTTPS
    if ($server_port !~ 443) {
        rewrite ^(/.*)$ https://$host$1 permanent;
    }

    # Rewrite — todo pasa por index.php
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP-FPM
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/tmp/php-cgi-82.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;

        # Timeouts
        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout    60s;
        fastcgi_read_timeout    60s;
    }

    # ─── Seguridad: bloquear acceso a archivos sensibles ───

    # Bloquear .env, .git, .htaccess
    location ~ /\.(env|git|htaccess) {
        deny all;
        return 404;
    }

    # Bloquear acceso directo a claves, src, vendor, composer
    location ~ ^/(keys|src|vendor|composer\.(json|lock)) {
        deny all;
        return 404;
    }

    # Bloquear archivos .pem
    location ~* \.pem$ {
        deny all;
        return 404;
    }

    # ─── Headers de seguridad ───

    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # ─── CORS ───

    # Ajustar el dominio según las APIs cliente que consuman el Auth Server
    set $cors_origin "https://mipos.co.cr";

    location ~* ^/(login|\.well-known/jwks\.json)$ {
        # Preflight
        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' $cors_origin always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS' always;
            add_header 'Access-Control-Allow-Headers' 'Content-Type, Authorization' always;
            add_header 'Access-Control-Max-Age' 86400;
            add_header 'Content-Length' 0;
            return 204;
        }

        add_header 'Access-Control-Allow-Origin' $cors_origin always;

        # Pasar a PHP
        try_files $uri /index.php?$query_string;
    }

    # ─── Rate limiting (requiere definir zona en nginx.conf, ver paso 7) ───

    location = /login {
        limit_req zone=auth_limit burst=5 nodelay;
        try_files $uri /index.php?$query_string;
    }

    # ─── Cache para JWKS ───

    location = /.well-known/jwks.json {
        try_files $uri /index.php?$query_string;
        add_header Cache-Control "public, max-age=3600";
    }

    # Bloquear todo lo que no sea los endpoints definidos
    # (ya manejado por el router PHP con 404)
}
```

### Nota sobre el socket PHP-FPM

El path del socket depende de la versión de PHP instalada en aaPanel:

| PHP Version | Socket path |
|-------------|-------------|
| 8.2 | `/tmp/php-cgi-82.sock` |
| 8.3 | `/tmp/php-cgi-83.sock` |
| 8.4 | `/tmp/php-cgi-84.sock` |

Verificar con:

```bash
ls -la /tmp/php-cgi-*.sock
```

---

## 7. Rate limiting (nginx.conf global)

En aaPanel ir a **Nginx** → **Configuration** → **nginx.conf** y agregar dentro del bloque `http {}`:

```nginx
# Rate limit para auth server — 10 requests por segundo por IP
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/s;
```

Reiniciar Nginx después de este cambio.

---

## 8. SSL con Let's Encrypt

En aaPanel:

1. Ir a **Website** → `auth.mipos.co.cr` → **SSL**
2. Seleccionar **Let's Encrypt**
3. Marcar `auth.mipos.co.cr`
4. Click **Apply**
5. Activar **Force HTTPS**

---

## 9. Configurar PHP en aaPanel

Ir a **App Store** → **PHP 8.2** → **Settings**:

### Extensiones requeridas

Verificar que estén instaladas:
- `openssl` (para RSA)
- `pdo_mysql` (para la BD)
- `mbstring`
- `json` (incluido por defecto en 8.2+)

### Funciones desbloqueadas

Ir a **Disabled Functions** y asegurarse de que estas NO estén bloqueadas:
- `file_get_contents`
- `openssl_pkey_get_public`
- `openssl_pkey_get_details`

aaPanel bloquea algunas funciones por defecto. Si `file_get_contents` está en la lista de disabled, removerla.

### php.ini recomendado

```ini
; Seguridad
expose_php = Off
display_errors = Off
log_errors = On
error_log = /www/wwwlogs/php_errors.log

; Timezone
date.timezone = America/Costa_Rica
```

---

## 10. Verificar el deploy

Desde tu máquina local o desde el servidor:

```bash
# Health check (si lo tienes) o probar JWKS
curl -s https://auth.mipos.co.cr/.well-known/jwks.json | python3 -m json.tool

# Probar login
curl -s -X POST https://auth.mipos.co.cr/login \
  -H "Content-Type: application/json" \
  -d '{"base":"cliente_prueba","password":"su_password"}' | python3 -m json.tool

# Verificar que archivos sensibles NO sean accesibles
curl -I https://auth.mipos.co.cr/.env
# Esperado: 404 o 403

curl -I https://auth.mipos.co.cr/keys/private.pem
# Esperado: 404 o 403

curl -I https://auth.mipos.co.cr/composer.json
# Esperado: 404 o 403
```

---

## 11. Checklist post-deploy

- [ ] `curl /.well-known/jwks.json` retorna el JWKS correctamente
- [ ] `curl -X POST /login` con credenciales válidas retorna un JWT
- [ ] `curl /.env` retorna 404 (no expuesto)
- [ ] `curl /keys/private.pem` retorna 404 (no expuesto)
- [ ] `curl /composer.json` retorna 404 (no expuesto)
- [ ] SSL activo y forzando HTTPS
- [ ] Permisos: `keys/` 700, `private.pem` 600, `.env` 600
- [ ] PHP extensions: `openssl`, `pdo_mysql`, `mbstring` instaladas
- [ ] `file_get_contents` NO está en disabled functions de PHP
- [ ] Rate limiting activo en `/login`
- [ ] Logs funcionando en `/www/wwwlogs/`

---

## Troubleshooting

### 502 Bad Gateway

El socket PHP-FPM no está corriendo o el path es incorrecto.

```bash
# Verificar que PHP-FPM está corriendo
ps aux | grep php-fpm

# Verificar el socket
ls -la /tmp/php-cgi-82.sock

# Reiniciar PHP-FPM desde aaPanel o:
/etc/init.d/php-fpm-82 restart
```

### 500 Internal Server Error

Revisar los logs:

```bash
tail -50 /www/wwwlogs/auth.mipos.co.cr.error.log
tail -50 /www/wwwlogs/php_errors.log
```

Causas comunes:
- `.env` no existe o tiene valores vacíos
- `vendor/` no existe (falta `composer install`)
- `keys/private.pem` no existe o no tiene permisos de lectura para `www`
- `file_get_contents` bloqueada en PHP

### Token no se genera

```bash
# Verificar que openssl funciona en PHP
php -r "echo openssl_pkey_get_details(openssl_pkey_get_public(file_get_contents('keys/public.pem')))['bits'];"
# Esperado: 2048

# Verificar conexión a BD
php -r "\$p=new PDO('mysql:host=127.0.0.1;dbname=api','usuario','password'); echo 'OK';"
```

### CORS errors en el navegador

Verificar que el dominio en `$cors_origin` coincide exactamente con el origen del request (incluyendo `https://` y sin trailing slash).

Si necesitas múltiples orígenes:

```nginx
# Reemplazar la línea set $cors_origin con:
map $http_origin $cors_origin {
    default "";
    "https://mipos.co.cr"     $http_origin;
    "https://app.mipos.co.cr" $http_origin;
    "https://admin.mipos.co.cr" $http_origin;
}
```

El bloque `map` va **fuera** del bloque `server`, dentro de `http {}` en nginx.conf.
