# KanbanApi — Requisitos del proyecto

## Descripción general

KanbanApi es una API REST introductoria desarrollada con **.NET**, **Dapper** y **PostgreSQL**, que
modela un tablero de tareas estilo Kanban (similar a Trello). Los usuarios pueden crear tableros,
definir columnas/categorías personalizadas para mover tarjetas y designar una columna como el estado
final ("Hecho").

El proyecto está diseñado para abarcar estos conceptos técnicos fundamentales:

- Programación asíncrona (`async`/`await`)
- Transformación de datos (datos brutos → DTO agregados)
- Multiprocesamiento (generación de informes en segundo plano)

> **Nota:** Por el momento, la funcionalidad WebSockets (sincronización del tablero en tiempo real) se
> ha omitido intencionadamente. Se podrá añadir posteriormente, una vez que el núcleo REST esté
> consolidado.

---

## 1. Entidades principales

### Board (Tablero)

| Campo | Tipo | Notas |
| --- | --- | --- |
| `Id` | int | PK |
| `Name` | string | requerido |
| `Description` | string | opcional |
| `CreatedAt` | timestamp | por defecto `now()` |

Un tablero contiene varias columnas.

### Column (Columna / Categoría)

| Campo | Tipo | Notas |
| --- | --- | --- |
| `Id` | int | PK |
| `BoardId` | int | FK → boards |
| `Name` | string | p. ej. "Pendiente", "En progreso", "Revisión", "Hecho" |
| `Order` | int | posición en el tablero |
| `IsFinalColumn` | bool | marca el estado "Hecho" |

**Regla de negocio:** debe haber exactamente una columna marcada con `IsFinalColumn = true` por
tablero. Si un usuario intenta marcar una segunda como definitiva, la anterior se desmarcará
automáticamente **o** se rechazará la operación (decidir cuál de las dos opciones es la correcta).

Las columnas deben poder reordenarse (arrastrar y soltar en la interfaz de usuario → `PATCH` del
orden en el panel de administración).

### Card (Tarjeta)

| Campo | Tipo | Notas |
| --- | --- | --- |
| `Id` | int | PK |
| `ColumnId` | int | FK → columns (columna actual) |
| `Title` | string | requerido |
| `Description` | string | opcional |
| `CreatedAt` | timestamp | por defecto `now()` |
| `CompletedAt` | timestamp | nullable, se establece cuando la tarjeta entra en la columna final |

Las tarjetas solo deben moverse entre columnas dentro de su propio tablero (no entre tableros, a menos
que se permita explícitamente).

### CardMovement (Historial de movimientos)

| Campo | Tipo | Notas |
| --- | --- | --- |
| `Id` | int | PK |
| `CardId` | int | FK → cards |
| `FromColumnId` | int | FK → columns |
| `ToColumnId` | int | FK → columns |
| `Timestamp` | timestamp | por defecto `now()` |

Cada vez que una tarjeta cambia de columna, se inserta un registro aquí. Esta tabla es la fuente de
datos para el informe de "tiempo promedio por columna".

---

## 2. Endpoints REST

### Tableros

```
POST   /api/boards
GET    /api/boards
GET    /api/boards/{id}
DELETE /api/boards/{id}
```

### Columnas

```
POST   /api/boards/{boardId}/columns
GET    /api/boards/{boardId}/columns
PATCH  /api/columns/{id}              # renombrar, cambiar orden, marcar como final
DELETE /api/columns/{id}              # decidir: borrar las tarjetas, o moverlas a otra columna
```

### Tarjetas

```
POST   /api/columns/{columnId}/cards
GET    /api/boards/{boardId}/cards     # todas las tarjetas del tablero (render inicial)
PATCH  /api/cards/{id}/move            # body: { newColumnId } → dispara el registro en CardMovement
PATCH  /api/cards/{id}                 # editar título/descripción
DELETE /api/cards/{id}
```

### Informes (trabajo en segundo plano / asíncrono)

```
GET    /api/boards/{boardId}/reports/time-per-column
```

---

## 3. Regla clave de negocio: la columna "Hecho"

- Al crear un tablero, se puede generar automáticamente una columna marcada con
  `IsFinalColumn = true` por defecto (por ejemplo, "Hecho"), **o** el usuario puede designarla
  manualmente. Decidir qué método utilizar.
- Cuando una tarjeta entra en la columna final → `CompletedAt = now()`.
- Si una tarjeta abandona la columna final para ir a otra → `CompletedAt = null` (la tarea se
  "reabre").

---

## 4. Mapeo de conceptos técnicos

| Concepto | Dónde vive |
| --- | --- |
| **Asíncrono** | Todas las llamadas de Dapper usan `QueryAsync` / `ExecuteAsync`; sin E/S bloqueante |
| **Transformación de datos** | `Card` / `CardMovement` (bruto) → DTO de informe agregado (tiempo promedio, mínimo/máximo por columna) |
| **Multihilo** | El cálculo del "tiempo promedio por columna" se ejecuta en un `BackgroundService` o `Task.Run`, en un hilo separado del hilo de la solicitud. Relevante una vez que un tablero tiene un historial de movimientos significativo |

---

## 5. Stack tecnológico

- **.NET 8** (o LTS actual) — Web API (`ControllerBase`, sin vistas Razor)
- **Dapper** — SQL puro para todas las consultas, incluido el informe agregado (se prefiere `GROUP BY`
  en Postgres sobre la agregación en memoria)
- **PostgreSQL** — el esquema se gestiona mediante scripts de migración `.sql` manuales (Dapper no
  tiene migraciones integradas)
- **Npgsql** — driver de PostgreSQL para .NET
- Utilizar **`NpgsqlDataSource`** para la gestión de conexiones (patrón recomendado actualmente) en
  lugar de una `NpgsqlConnection` suelta por solicitud

---

## Estado

Repositorio recién inicializado. La implementación todavía no comenzó.
