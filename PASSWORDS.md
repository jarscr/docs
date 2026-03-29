# Gestión de passwords

Documentación interna sobre cómo se cifran, almacenan y verifican los passwords de los clientes en el Auth Server.

---

## Algoritmo

Se usa `password_hash()` de PHP con `PASSWORD_BCRYPT`. Esto genera un hash de 60 caracteres que incluye el algoritmo, el cost y un salt aleatorio.

```
$2y$10$xN5Rz3kLm8vQ9wJp2aB1e.oYz4R7T6U8V0X2C4D6F8H0J2L4N6P8
 ^^^ ^^ ^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
 alg cost      salt (22 chars)          hash (31 chars)
```

- **Algoritmo:** `$2y$` (bcrypt)
- **Cost:** `10` (default — 2^10 = 1024 iteraciones)
- **Salt:** generado automáticamente por PHP — no se necesita generar uno manualmente

---

## Generar un hash

### Desde PHP (script o código)

```php
<?php
echo password_hash('la_contraseña', PASSWORD_BCRYPT);
// $2y$10$xN5Rz3k...
```

### Desde terminal (una sola línea)

```bash
php -r "echo password_hash(\$argv[1], PASSWORD_BCRYPT) . PHP_EOL;" "la_contraseña"
```

### Desde MySQL directamente

MySQL no tiene bcrypt nativo. Siempre generar el hash desde PHP y luego insertarlo:

```bash
# Generar
HASH=$(php -r "echo password_hash(\$argv[1], PASSWORD_BCRYPT);" "la_contraseña")

# Insertar
mysql -u usuario -p api -e "UPDATE clientes SET password = '$HASH' WHERE base = 'nombre_cliente';"
```

---

## Insertar o actualizar en la BD

```php
<?php
$pdo = new PDO('mysql:host=127.0.0.1;dbname=api', 'usuario', 'password');

$base     = 'nombre_cliente';
$password = 'contraseña_nueva';
$hash     = password_hash($password, PASSWORD_BCRYPT);

$stmt = $pdo->prepare('UPDATE clientes SET password = :hash WHERE base = :base');
$stmt->execute([':hash' => $hash, ':base' => $base]);

echo "Password actualizado para: $base\n";
```

---

## Cómo verifica el Auth Server

En `LoginRoute.php`, la verificación se hace con `password_verify()`:

```php
$cliente = $stmt->fetch(); // { base: '...', password: '$2y$10$...' }

if (!$cliente || !password_verify($passwordRecibido, $cliente['password'])) {
    // 401 — mismo mensaje para cliente inexistente o password incorrecto
}
```

`password_verify()` extrae el salt del hash almacenado, hashea el password recibido con el mismo salt, y compara. No se necesita lógica adicional.

---

## Requisito de la tabla

El campo `password` en la tabla `clientes` debe ser `VARCHAR(255)` para soportar el hash completo y posibles algoritmos futuros más largos.

```sql
-- Verificar el tipo actual
DESCRIBE clientes;

-- Si el campo es menor a 255, ajustar:
ALTER TABLE clientes MODIFY password VARCHAR(255) NOT NULL;
```

---

## Rehash automático

Si en el futuro se cambia el cost o el algoritmo, los hashes existentes siguen funcionando pero deberían actualizarse gradualmente:

```php
if (password_verify($password, $cliente['password'])) {
    // Login exitoso — verificar si el hash necesita actualización
    if (password_needs_rehash($cliente['password'], PASSWORD_BCRYPT)) {
        $nuevoHash = password_hash($password, PASSWORD_BCRYPT);
        $stmt = $pdo->prepare('UPDATE clientes SET password = :hash WHERE base = :base');
        $stmt->execute([':hash' => $nuevoHash, ':base' => $cliente['base']]);
    }
}
```

---

## Qué NO usar

| Método | Por qué no |
|--------|-----------|
| `md5()` | Roto — se pueden generar colisiones y tablas rainbow en segundos |
| `sha1()` | Roto — misma vulnerabilidad que md5 |
| `sha256()` / `hash('sha256', ...)` | Demasiado rápido para passwords — se puede hacer fuerza bruta a miles de millones por segundo con GPU |
| `base64_encode()` | No es cifrado, es codificación — se revierte trivialmente |
| Comparación directa (`===`) | Nunca comparar passwords en texto plano |
| Salt manual | `password_hash()` genera un salt criptográfico automáticamente — no agregar uno propio |
