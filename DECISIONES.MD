# Decisiones de traducción: spec .NET → Laravel

Fuente: spec-.net (constitution, spec, data-model, api-contract, tasks).
Lo de negocio se respeta tal cual. Lo de tecnología se traduce aquí.

## 1. Arquitectura (Constitución, arts. I a IV)

| Spec (.NET)                         | Laravel                                   |
|-------------------------------------|-------------------------------------------|
| src/Library.Domain                  | app/Domain (PHP puro, sin `Illuminate\`)  |
| src/Library.Application + puertos   | app/Application (+ interfaces en Ports/)  |
| src/Library.Adapters                | app/Adapters (Eloquent, JWT, Storage, HTTP)|
| Bootstrap/PortBindings.cs (art. III)| app/Providers/PortBindingsServiceProvider |
| NetArchTest                         | `arch()` de Pest                          |

Puertos: Clock, PasswordHasher, TokenIssuer, CoverStorage y los repositorios.
Verificación: tests de arquitectura que fallan si Domain importa Illuminate
o si Application importa Adapters.

## 2. Datos (data-model.md)

- PostgreSQL 16, esquema `library`; en config/database.php `search_path = library`.
- Seis tablas, nombres singulares: category, book, member, loan, loan_item, user.
  Eloquent: `protected $table` explícito, `$incrementing = false`,
  `$keyType = 'string'`, `$timestamps = false` (el spec no lleva created_at/updated_at).
- Sin DEFAULT en ninguna columna: los valores los pone el dominio.
- CHECK, índice parcial y FK se crean con `DB::statement` dentro de la migración.
- Semilla: las 5 categorías con los UUID fijos de §9, dentro de la migración.
- El admin NO se siembra en la migración: lo crea el arranque desde .env.

## 3. Concurrencia (RNF-2)

Prestar = una transacción: traer los libros por lote con `lockForUpdate()`
(ordenados por id para evitar deadlocks), validar, descontar, crear préstamo y líneas.
Última barrera: `ck_book_available_range`. Si salta, se traduce a 409 BOOK_NOT_AVAILABLE.

## 4. Seguridad

- Hash: `Hash::make` detrás del puerto PasswordHasher.
- Token: JWT 8 h con `sub` y `role` (firebase/php-jwt). Sanctum no emite JWT.
- Mensaje de login único (INVALID_CREDENTIALS), sin distinguir usuario/clave.
- La clave y el hash nunca salen en respuestas ni en logs.

## 5. Errores (RNF-4)

Un handler global convierte toda excepción al cuerpo {code, message, details}.
Nunca trazas, SQL ni rutas. 401/403/404/409/422 según api-contract §1 y §2.

## 6. Pruebas

Pest. `Carbon::setTestNow()` hace de Clock falso. Los casos de uso se prueban con fakes.

## 7. Trazabilidad
(tabla HU → CA → endpoint → tarea → evidencia; ver abajo)