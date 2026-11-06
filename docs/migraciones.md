# Migraciones: modelo de datos en Laravel

Fuente: `spec-.net/data-model.md` (PostgreSQL 16, esquema `library`, UTC).
Este documento traduce ese modelo a migraciones de Laravel. Si algo contradice
al `data-model.md`, gana el `data-model.md`.

## 1. Reglas generales

- Esquema `library` (`search_path = library,public` en `config/database.php`).
- Seis tablas en inglés, singular: `category`, `user`, `member`, `book`, `loan`, `loan_item`.
- Claves primarias `uuid`, generadas por el dominio (no autoincrementales).
- Ninguna columna lleva `DEFAULT`.
- Todas las fechas son `timestamptz` (`timestampTz`).
- Sin `created_at` ni `updated_at` (`$timestamps = false` en los modelos).
- Restricciones que Laravel no expresa solas: `DB::statement` dentro de la migración.

## 2. Orden de migraciones

| # | Migración | Contenido |
|---|---|---|
| 1 | `create_library_schema` | `CREATE SCHEMA IF NOT EXISTS library` |
| 2 | `create_category_table` | Tabla + semilla de 5 categorías |
| 3 | `create_user_table` | Tabla |
| 4 | `create_member_table` | Tabla |
| 5 | `create_book_table` | Tabla + FK + CHECK + índices |
| 6 | `create_loan_table` | Tabla + FK + índice |
| 7 | `create_loan_item_table` | Tabla + FK + únicos + índices |
| 8 | `add_domain_checks` (T-18) | `ck_book_total_positive`, `ck_user_role` |

El orden respeta las dependencias entre claves foráneas.

## 3. Tablas

### category
| Columna | Tipo | Nulable | Nota |
|---|---|---|---|
| id | uuid | no | PK |
| name | varchar(120) | no | Único |

### user
| Columna | Tipo | Nulable | Nota |
|---|---|---|---|
| id | uuid | no | PK |
| username | varchar(120) | no | Único, minúsculas, recortado |
| password_hash | varchar(512) | no | Nunca se indexa ni se devuelve |
| role | varchar(40) | no | `admin` o `librarian` |

En SQL a mano se escribe `library."user"`.

### member
| Columna | Tipo | Nulable | Nota |
|---|---|---|---|
| id | uuid | no | PK |
| full_name | varchar(200) | no | |
| document_number | varchar(30) | no | Único |

### book
| Columna | Tipo | Nulable | Nota |
|---|---|---|---|
| id | uuid | no | PK |
| title | varchar(250) | no | Recortado por el dominio |
| author | varchar(200) | no | |
| isbn | char(13) | no | Solo dígitos; único entre activos |
| category_id | uuid | no | FK restrictiva a category |
| total_copies | integer | no | >= 1 |
| available_copies | integer | no | Entre 0 y total_copies |
| cover_key | varchar(512) | sí | Clave opaca; nunca cadena vacía |
| deleted_at | timestamptz | sí | Baja lógica; nulo = activo |

### loan
| Columna | Tipo | Nulable | Nota |
|---|---|---|---|
| id | uuid | no | PK |
| member_id | uuid | no | FK restrictiva a member |
| loaned_by_user_id | uuid | no | FK restrictiva a user |
| loaned_at | timestamptz | no | Instante del préstamo |

### loan_item
| Columna | Tipo | Nulable | Nota |
|---|---|---|---|
| id | uuid | no | PK |
| loan_id | uuid | no | FK con ON DELETE CASCADE |
| book_id | uuid | no | FK restrictiva a book |
| book_title | varchar(250) | no | Congelado |
| isbn | char(13) | no | Congelado |
| category_name | varchar(120) | no | Congelado; sin FK a propósito |
| due_at | timestamptz | no | loaned_at + 14 días |
| returned_at | timestamptz | sí | Nulo mientras no se devuelve |

## 4. Claves foráneas

| FK | ON DELETE | Motivo |
|---|---|---|
| book.category_id → category.id | RESTRICT | Una categoría con libros no se elimina |
| loan.member_id → member.id | RESTRICT | El socio de un préstamo no desaparece |
| loan.loaned_by_user_id → user.id | RESTRICT | La autoría es dato contable |
| loan_item.loan_id → loan.id | CASCADE | La línea no vive sin su préstamo |
| loan_item.book_id → book.id | RESTRICT | Protege el histórico |

`ON UPDATE NO ACTION` en todas: las claves UUID no cambian.

## 5. Restricciones e índices

| Regla | Objeto | Dónde vive |
|---|---|---|
| PK de las seis tablas | PK_* | Motor |
| category.name, member.document_number, user.username únicos | Índices únicos | Motor |
| isbn único entre activos | Índice único parcial `WHERE deleted_at IS NULL` | Motor |
| 0 <= available_copies <= total_copies | `ck_book_available_range` | Motor (última barrera de concurrencia) |
| (loan_id, book_id) único | Índice único | Motor |
| total_copies >= 1 | `ck_book_total_positive` | Pendiente T-18 (hoy solo dominio) |
| role en (admin, librarian) | `ck_user_role` | Pendiente T-18 (hoy solo dominio) |
| Máximo 3 sin devolver por socio | n/a | Solo dominio |
| Socio con vencidos no pide | n/a | Solo dominio |
| Descontar ejemplar y crear línea = una operación | n/a | Solo dominio (transacción) |
| Un préstamo tiene 1 a 3 líneas | n/a | Solo dominio |

Índices de acceso:
- `book (category_id, title)` parcial sobre activos.
- `loan (loaned_at)`.
- `loan_item (book_id)`.
- `loan_item (due_at) WHERE returned_at IS NULL` (vencidos).

## 6. Fragmentos SQL dentro de las migraciones

```php
// create_book_table
DB::statement('ALTER TABLE library.book ADD CONSTRAINT ck_book_available_range
    CHECK (available_copies >= 0 AND available_copies <= total_copies)');
DB::statement('CREATE UNIQUE INDEX uq_book_isbn_active ON library.book (isbn)
    WHERE deleted_at IS NULL');
DB::statement('CREATE INDEX ix_book_category_title ON library.book (category_id, title)
    WHERE deleted_at IS NULL');

// create_loan_item_table
DB::statement('CREATE INDEX ix_loan_item_due_open ON library.loan_item (due_at)
    WHERE returned_at IS NULL');

// add_domain_checks (T-18)
DB::statement('ALTER TABLE library.book ADD CONSTRAINT ck_book_total_positive
    CHECK (total_copies >= 1)');
DB::statement("ALTER TABLE library.\"user\" ADD CONSTRAINT ck_user_role
    CHECK (role IN ('admin','librarian'))");
```

## 7. Semilla (migración de category)

| id | name |
|---|---|
| 11111111-1111-4111-8111-111111111111 | Novela |
| 22222222-2222-4222-8222-222222222222 | Ciencia |
| 33333333-3333-4333-8333-333333333333 | Tecnología |
| 44444444-4444-4444-8444-444444444444 | Historia |
| 55555555-5555-4555-8555-555555555555 | Infantil |

El administrador inicial no lo siembra la base: lo crea el arranque de la
aplicación con credenciales del entorno (el hash lo produce el puerto de hash).
El sembrador de demostración (repo `tool`) usa la API, nunca SQL directo.

## 8. Privacidad

- `password_hash`: secreto; nunca en logs ni respuestas.
- `member.full_name` y `document_number`: datos personales; solo admin y librarian; nunca en el reporte.
- Préstamos y líneas nunca se borran; libros solo baja lógica.
- La portada es el único dato que se borra físicamente: primero se anula `cover_key` y se confirma la transacción; después se borra el binario.

## 9. Evidencia de T-02

Estas consultas deben devolver lo documentado aquí, sin ningún DEFAULT:

```sql
SELECT table_name, ordinal_position, column_name, data_type, is_nullable, column_default
FROM information_schema.columns WHERE table_schema = 'library' ORDER BY 1, 2;

SELECT c.conrelid::regclass, c.conname, c.contype, pg_get_constraintdef(c.oid)
FROM pg_constraint c JOIN pg_namespace n ON n.oid = c.connamespace
WHERE n.nspname = 'library' ORDER BY 1, 3, 2;

SELECT tablename, indexname, indexdef FROM pg_indexes WHERE schemaname = 'library' ORDER BY 1, 2;
```

Si una regla marcada como Motor no aparece, alguien la borró. Gana el motor: el documento se corrige.