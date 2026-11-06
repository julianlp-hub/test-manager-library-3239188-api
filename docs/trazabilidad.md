# Trazabilidad: del spec a la implementación en Laravel

Cada fila une un criterio de aceptación (spec.md) con el endpoint que lo expone
(api-contract.md), la tarea que lo construye (tasks.md) y la evidencia con la que
se comprobaría (Constitución, art. XII: nada se da por hecho sin evidencia).

Los códigos de error de la columna "Resultado" salen de api-contract.md §2.

## 1. Historias de usuario y criterios de aceptación

### HU-01 · Iniciar sesión
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-01.1 | POST /auth/login | T-07, T-14 | Login válido devuelve token con `expires_in` = 28800 | 200 |
| CA-01.2 | POST /auth/login | T-07, T-14 | Usuario inexistente y clave errónea dan el mismo mensaje | 401 INVALID_CREDENTIALS |
| CA-01.3 | Todos salvo login y /health | T-16 | Prueba de contrato: sin token 401, rol insuficiente 403 | 401 / 403 FORBIDDEN_ROLE |

### HU-02 · Dar de alta bibliotecarios (admin)
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-02.1 | POST /users | T-07 | Admin crea usuario con rol `librarian` | 201 |
| CA-02.2 | POST /users | T-06, T-12 | `"Ana "` queda `ana`; repetido falla | 409 USERNAME_TAKEN |
| CA-02.3 | POST /users | T-06, T-14 | Pedir rol `admin` falla; la respuesta no trae clave ni hash | 422 VALIDATION_ERROR |

### HU-03 · Gestionar libros (admin)
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-03.1 | POST /books | T-04, T-08 | Al crear, disponibles = total | 201 |
| CA-03.2 | POST /books, PUT /books/{id} | T-03, T-12 | ISBN de 13 dígitos; repetido entre activos falla | 409 ISBN_TAKEN |
| CA-03.3 | POST /books, PUT /books/{id} | T-03, T-04 | Total >= 1; cambiar total ajusta disponibles y nunca deja < 0 | 422 VALIDATION_ERROR |
| CA-03.4 | PUT /books/{id}/cover | T-15 | PNG de 2 MB ok; 3 MB falla; sin portada `cover_key` es null | 200 / 422 |
| CA-03.5 | DELETE /books/{id} | T-04, T-08 | Baja lógica; libro con préstamos abiertos no se da de baja | 204 / 409 BOOK_HAS_OPEN_LOANS |

### HU-04 · Buscar libros
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-04.1 | GET /books | T-08, T-12 | Texto parcial en título o autor, filtro por categoría, solo activos, orden por título, size máx. 50 | 200 |
| CA-04.2 | GET /books | T-08 | La respuesta incluye `total` | 200 |
| CA-04.3 | GET /books/{id} | T-08 | Id inexistente o dado de baja | 404 NOT_FOUND |

### HU-05 · Gestionar socios
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-05.1 | POST /members | T-06, T-12 | Documento repetido falla | 201 / 409 DOCUMENT_TAKEN |
| CA-05.2 | GET /members | T-12 | Búsqueda por nombre o documento, paginada | 200 |
| CA-05.3 | (sin ruta DELETE) | T-16 | No existe operación de borrado de socios | 404 / 405 |

### HU-06 · Registrar un préstamo
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-06.1 | POST /loans | T-05, T-09 | Un socio, usuario autenticado como registrador, 1 a 3 líneas | 201 |
| CA-06.2 | POST /loans | T-05, T-09 | Mismo libro dos veces en un préstamo | 422 DUPLICATE_BOOK_IN_LOAN |
| CA-06.3 | POST /loans | T-04, T-09, T-13 | Cada línea descuenta 1; sin ejemplares no se registra nada | 409 BOOK_NOT_AVAILABLE |
| CA-06.4 | POST /loans | T-05, T-09 | Socio con más de 3 sin devolver | 422 MEMBER_LIMIT_EXCEEDED |
| CA-06.5 | POST /loans | T-09 | Socio con algún vencido no pide otro | 409 MEMBER_HAS_OVERDUE |
| CA-06.6 | POST /loans | T-05, T-09 | `due_at` = `loaned_at` + 14 días; título, ISBN y categoría congelados | 201 |
| CA-06.7 | (sin PUT/DELETE) | T-16 | No hay operación de edición ni borrado de préstamos | 404 / 405 |

### HU-07 · Registrar una devolución
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-07.1 | POST /loans/{l}/items/{i}/return | T-05, T-10 | Pone `returned_at` y suma 1 a disponibles | 200 |
| CA-07.2 | POST /loans/{l}/items/{i}/return | T-10 | Devolver dos veces; no suma dos veces | 409 ITEM_ALREADY_RETURNED |
| CA-07.3 | (base de datos) | T-02 | `ck_book_available_range`: disponibles nunca superan el total | motor |
| CA-07.4 | GET /loans/{id} | T-05 | Con todas las líneas devueltas el estado es `closed` | 200 |

### HU-08 · Ver y listar préstamos
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-08.1 | GET /loans/{id} | T-05 | Trae líneas y estado calculado | 200 |
| CA-08.2 | GET /loans | T-12, T-13 | Filtros por rango, estado y socio; orden por fecha descendente; paginado | 200 |
| CA-08.3 | n/a | T-02, T-05 | No existe columna `status`; se calcula con el reloj | n/a |

### HU-09 · Libros vencidos
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-09.1 | GET /loans/overdue | T-10 | Líneas sin devolver con `due_at` anterior a ahora: libro, socio, días de retraso | 200 |
| CA-09.2 | GET /loans/overdue | T-10, T-16 | Orden por más días de retraso; solo admin y librarian | 200 / 403 |

### HU-10 · Reporte de libros más prestados
| CA | Endpoint | Tareas | Evidencia | Resultado |
|---|---|---|---|---|
| CA-10.1 | GET /reports/most-borrowed | T-11 | Una fila por libro y categoría congelada, de mayor a menor | 200 |
| CA-10.2 | GET /reports/most-borrowed | T-03, T-11 | Fin anterior al inicio falla; sin préstamos devuelve lista vacía | 422 INVALID_DATE_RANGE / 200 |
| CA-10.3 | GET /reports/most-borrowed | T-11 | Sin desglose por bibliotecario ni socio | 200 |
| CA-10.4 | GET /reports/most-borrowed | T-11 | Renombrar libro o categoría después del préstamo no cambia el reporte (ADR-004) | 200 |
| CA-10.5 | GET /reports/most-borrowed | T-16 | Solo admin | 403 FORBIDDEN_ROLE |

## 2. Reglas de negocio

| Regla | Cómo se cumple | Tareas |
|---|---|---|
| RN-1 Disponibles entre 0 y total | Dominio `Book` + `ck_book_available_range` | T-04, T-02 |
| RN-2 14 días, máx. 3 sin devolver | Dominio `Loan` | T-05 |
| RN-3 Vencidos bloquean nuevos préstamos | Caso de uso `LendBooksService` | T-09 |
| RN-4 Lo congelado no sigue al catálogo | `loan_item` guarda título, ISBN y categoría | T-02, T-09 |
| RN-5 Nada se borra; libros solo baja lógica | Sin rutas DELETE de préstamos; `deleted_at` en `book` | T-02, T-16 |
| RN-6 Fechas UTC; mensajes en español, código en inglés | Serialización ISO 8601 con `Z`; handler de errores | T-16 |

## 3. Requisitos no funcionales

| RNF | Cómo se mide | Tareas |
|---|---|---|
| RNF-1 Búsqueda < 300 ms con 5 000 libros | Prueba de carga con el sembrador de `tool` | T-12, T-21 |
| RNF-2 Último ejemplar: uno gana, otro 409 | Transacción con `lockForUpdate()` + `ck_book_available_range`; prueba concurrente | T-13 |
| RNF-3 Contraseña nunca en claro ni en logs | Revisión de logs y respuestas | T-14 |
| RNF-4 Mismo cuerpo de error siempre | Prueba de contrato sobre todas las rutas | T-16 |
| RNF-5 Un comando levanta todo; base vacía | `docker compose up` (repo `infra`) | T-20 |

## 4. Principios de la Constitución y su verificación

| Artículo | Principio | Verificación |
|---|---|---|
| I | El dominio no depende de nada | `arch()` de Pest: `App\Domain` no usa `Illuminate` |
| II | El núcleo declara sus puertos | `App\Application` no importa `App\Adapters` |
| III | Un único punto de composición | Solo `PortBindingsServiceProvider` instancia adaptadores |
| V | El esquema es de las migraciones | `infra` no contiene `CREATE TABLE` |
| VI | La base impone lo que puede | Consultas de `docs/migraciones.md` §9 |
| VII | Lo derivado se calcula | Sin columnas `status` ni `days_late` |
| VIII | Lo congelado no sigue al catálogo | Prueba de renombrar libro tras prestarlo |
| IX | Ningún secreto en el repo ni en logs | `.env` en `.gitignore`; solo `.env.example` |

## 5. Estado de las tareas

| Tarea | Entregable en este repositorio | Estado |
|---|---|---|
| T-01 a T-17 | Diseño documentado (DECISIONES, migraciones, endpoints, esta matriz) | Diseñado |
| T-18 | Migración `add_domain_checks` descrita en `docs/migraciones.md` §6 | Diseñado |
| T-19 | Pruebas de arquitectura descritas en §4 | Diseñado |
| T-20 | Repo `infra` | Ver su README |
| T-21 | Repo `tool` | Ver su README |
| T-22 | Repo `app` | Ver su README |
| T-23 | Repo `page` | Ver su README |