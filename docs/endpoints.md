# Endpoints: contrato de la API en Laravel

Fuente: `spec-.net/api-contract.md`. El contrato HTTP es independiente del lenguaje:
en Laravel se respeta igual que en .NET o Python.

## 1. Convenciones

| Tema | Regla |
|---|---|
| Base | `/api/v1` (en Laravel: `apiPrefix: 'api/v1'` en `bootstrap/app.php`) |
| Formato | JSON, UTF-8, campos en `snake_case` |
| Identificadores | UUID v4 como texto |
| Fechas | ISO 8601 en UTC con `Z` (`2026-10-14T14:15:00Z`) |
| Autenticación | `Authorization: Bearer <token>` en todo salvo `POST /auth/login` y `GET /health` |
| Paginación | `?page=1&size=20`, `size` máximo 50. Respuesta `{ items, page, size, total }` |
| Idioma | Mensajes de error en español; campos y códigos en inglés |

## 2. Cuerpo de error único (RNF-4)

Todo error usa este formato, sin excepciones:

```json
{
  "code": "BOOK_NOT_AVAILABLE",
  "message": "No quedan ejemplares disponibles de «Cien años de soledad».",
  "details": [ { "field": "items[0].book_id", "issue": "sin ejemplares" } ]
}
```

`details` puede ir vacío. Nunca incluye trazas, SQL, rutas del servidor ni la contraseña.
En Laravel se logra con un handler global de excepciones (`bootstrap/app.php`, `withExceptions`).

## 3. Códigos de estado

| Estado | Cuándo |
|---|---|
| 200 / 201 / 204 | Éxito / creado / sin contenido |
| 401 | Sin token, token inválido o vencido, credenciales inválidas |
| 403 | Rol insuficiente |
| 404 | No existe o está dado de baja |
| 409 | Conflicto de estado: duplicado, sin ejemplares, ya devuelto, socio con vencidos, libro con préstamos abiertos |
| 422 | Validación: campo inválido, regla de entrada rota, rango de fechas invertido |

## 4. Códigos de error de negocio

| code | Estado | Origen |
|---|---|---|
| INVALID_CREDENTIALS | 401 | CA-01.2 |
| FORBIDDEN_ROLE | 403 | CA-01.3 |
| NOT_FOUND | 404 | Cualquier recurso |
| USERNAME_TAKEN | 409 | CA-02.2 |
| ISBN_TAKEN | 409 | CA-03.2 |
| DOCUMENT_TAKEN | 409 | CA-05.1 |
| BOOK_HAS_OPEN_LOANS | 409 | CA-03.5 |
| BOOK_NOT_AVAILABLE | 409 | CA-06.3 |
| MEMBER_HAS_OVERDUE | 409 | CA-06.5 |
| ITEM_ALREADY_RETURNED | 409 | CA-07.2 |
| MEMBER_LIMIT_EXCEEDED | 422 | CA-06.4 |
| DUPLICATE_BOOK_IN_LOAN | 422 | CA-06.2 |
| VALIDATION_ERROR | 422 | Cualquier campo inválido |
| INVALID_DATE_RANGE | 422 | CA-10.2 |

## 5. Rutas

### Autenticación y salud
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | /auth/login | público | Devuelve token (JWT, 8 h) |
| GET | /health | público | `{ "status": "ok" }` |

Respuesta de login: `{ access_token, token_type: "Bearer", expires_in: 28800, role }`.

### Usuarios
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | /users | admin | Crea un bibliotecario. Rol `admin` → 422 |

### Categorías
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | /categories | autenticado | Las cinco categorías, orden por nombre |

### Libros
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | /books | autenticado | Búsqueda: `q`, `category_id`, `page`, `size` |
| GET | /books/{id} | autenticado | Detalle |
| POST | /books | admin | Crea (`title`, `author`, `isbn`, `category_id`, `total_copies`) |
| PUT | /books/{id} | admin | Reemplaza título, autor, categoría y total |
| DELETE | /books/{id} | admin | Baja lógica |
| PUT | /books/{id}/cover | admin | Sube portada (`multipart/form-data`, campo `file`) |
| DELETE | /books/{id}/cover | admin | Quita la portada |
| GET | /books/{id}/cover | autenticado | Binario de la portada o 404 |

### Socios
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | /members | admin, librarian | Lista y busca: `q`, `page`, `size` |
| GET | /members/{id} | admin, librarian | Detalle con `active_loans` |
| POST | /members | admin, librarian | Crea (`full_name`, `document_number`) |

### Préstamos
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| POST | /loans | admin, librarian | Registra préstamo (1 a 3 líneas) |
| GET | /loans | admin, librarian | Lista: `from`, `to`, `status`, `member_id`, `page`, `size` |
| GET | /loans/overdue | admin, librarian | Líneas vencidas |
| GET | /loans/{id} | admin, librarian | Detalle con líneas y estado |
| POST | /loans/{loan_id}/items/{item_id}/return | admin, librarian | Devuelve una línea |

Importante en Laravel: declarar `GET /loans/overdue` antes de `GET /loans/{id}`,
o "overdue" se interpreta como un id.

### Reporte
| Método | Ruta | Rol | Descripción |
|---|---|---|---|
| GET | /reports/most-borrowed | admin | `from` y `to` obligatorios |

## 6. Autorización por rol

| Rol | Puede |
|---|---|
| Sin token | Solo `/auth/login` y `/health` |
| librarian | Categorías, consultar libros, socios (listar, ver, crear), préstamos, devoluciones y vencidos |
| admin | Todo lo anterior, más crear bibliotecarios, gestionar libros y portadas, y el reporte |

Un 403 no revela si el recurso existe. En Laravel: middleware de rol sobre grupos de rutas.

## 7. Estado calculado del préstamo

`status` no se almacena (Constitución, art. VII):
- `open`: hay líneas sin devolver y ninguna vencida.
- `overdue`: alguna sin devolver y vencida.
- `closed`: todas devueltas.

## 8. Seguridad

| Aspecto | Regla |
|---|---|
| Token | JWT firmado, 8 horas, con `sub` y `role`, sin datos personales |
| Contraseñas | Solo el hash; nunca sale en respuestas ni en logs |
| CORS | Solo el origen del frontend configurado por entorno |
| Portada | Solo `image/jpeg` y `image/png`, máx. 2 MB, tipo validado por contenido |