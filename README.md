# Smart IT Helpdesk

API local de gestión de incidentes de IT que combina automatización con n8n, clasificación asistida por IA mediante Ollama y persistencia en PostgreSQL.

**Estado:** MVP funcional con pruebas manuales realizadas.

## 1. Objetivo del proyecto

Desarrollar una mesa de ayuda automatizada capaz de:

- Recibir incidentes mediante una API HTTP.
- Clasificar incidentes con un modelo de lenguaje local.
- Generar un resumen y acciones sugeridas.
- Persistir los tickets en PostgreSQL.
- Consultar tickets con filtros.
- Actualizar el estado y la prioridad de los tickets.
- Gestionar solicitudes repetidas mediante idempotencia.
- Devolver respuestas HTTP controladas ante errores de validación, recursos inexistentes y fallos de base de datos.

El objetivo es demostrar conocimientos de automatización, integración de APIs, procesamiento de datos, persistencia, validación y manejo de errores.

## 2. Tecnologías utilizadas

| Tecnología | Uso |
|---|---|
| n8n | Orquestación de workflows y endpoints HTTP |
| Ollama | Inferencia local con un modelo de lenguaje |
| Llama 3.2 3B | Clasificación y resumen de incidentes |
| PostgreSQL | Persistencia de tickets |
| Docker Desktop | Ejecución de servicios en contenedores |
| PowerShell | Pruebas manuales de la API |
| Windows | Entorno local de desarrollo |

El proyecto utiliza inferencia local con Ollama, sin depender de una API de modelos de pago.

## 3. Arquitectura

```text
                     ┌──────────────────┐
                     │ Cliente HTTP      │
                     │ PowerShell        │
                     └────────┬─────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │ n8n              │
                     │ Webhooks         │
                     └────────┬─────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                    ▼                   ▼
           ┌────────────────┐  ┌────────────────┐
           │ Ollama         │  │ PostgreSQL     │
           │ Llama 3.2 3B   │  │ Tickets        │
           └────────────────┘  └────────────────┘
```

### Workflows principales

1. **Crear tickets:** recibe el incidente, utiliza Ollama para clasificarlo y guarda el resultado.
2. **Consultar tickets:** devuelve tickets y permite filtrar por estado, prioridad, categoría y texto.
3. **Actualizar tickets:** permite modificar estado o prioridad y maneja validaciones y errores.
4. **Idempotencia:** reconoce reintentos de una operación y detecta la reutilización de una clave con contenido diferente.

## 4. Entorno local

Configuración utilizada durante el desarrollo:

- n8n: `http://localhost:5678`
- Ollama desde el contenedor n8n: `http://host.docker.internal:11434`
- Modelo local: `llama3.2:3b`
- Base de datos: `helpdesk`
- Tabla de tickets: `public.tickets`

Las URL de prueba utilizadas por n8n contienen `/webhook-test/`. Para ejecutarlas:

1. Abrir el workflow correspondiente.
2. Hacer clic en **Execute Workflow**.
3. Ejecutar la petición desde PowerShell.

La URL de prueba suele estar disponible para una ejecución. Para utilizar la API de forma persistente, activar el workflow y emplear la URL de producción que muestra n8n.

## 5. Modelo de datos

La tabla `tickets` contiene los siguientes campos:

| Campo | Descripción |
|---|---|
| `id` | Identificador del ticket |
| `title` | Título del incidente |
| `description` | Descripción original |
| `requester` | Solicitante |
| `category` | Categoría asignada |
| `priority` | Prioridad |
| `summary` | Resumen del incidente |
| `suggested_actions` | Acciones sugeridas por la IA |
| `requires_human_review` | Indica si requiere revisión humana |
| `status` | Estado actual |
| `created_at` | Fecha de creación, si está disponible en el esquema |

### Valores admitidos

**Estados:**
- `open`
- `in_progress`
- `resolved`
- `closed`

**Prioridades:**
- `low`
- `medium`
- `high`
- `critical`

Las categorías son propuestas por el modelo y pueden incluir valores como `network`, `security`, `software`, `access` y `hardware`.

La clasificación generada por IA debe considerarse una sugerencia. Los tickets que requieren revisión humana deben ser evaluados por una persona.

## 6. Endpoints de la API

### 6.1 Crear un ticket

**Método:** `POST`

**URL de prueba:**

`http://localhost:5678/webhook-test/chat-local`

#### Headers

```http
Content-Type: application/json
Idempotency-Key: helpdesk-idem-check-20261013-01
```

#### Request

```json
{
  "message": "Prueba de idempotencia: el monitor no enciende."
}
```

#### Respuesta de creación observada

```json
{
  "success": true,
  "TicketId": 13,
  "status": "open",
  "message": "Ticket creado correctamente"
}
```

El workflow utiliza Ollama para analizar el incidente y generar información como categoría, prioridad, resumen y acciones sugeridas antes de persistir el ticket.

#### Ejemplo con PowerShell

```powershell
$headers = @{
    "Idempotency-Key" = "helpdesk-idem-check-20261013-01"
}

$body = @{
    message = "Prueba de idempotencia: el monitor no enciende."
} | ConvertTo-Json

Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/chat-local" `
  -Method Post `
  -Headers $headers `
  -ContentType "application/json" `
  -Body $body
```

### 6.2 Idempotencia

La idempotencia permite reconocer reintentos de una misma operación para evitar crear tickets duplicados.

#### Caso A: misma clave y mismo contenido

Al repetir la solicitud con la misma clave y el mismo mensaje, el workflow devolvió el ticket original.

Respuesta observada:

```json
{
  "success": true,
  "ticketId": 13,
  "status": "open",
  "message": "Ticket ya existente; se devuelve el ticket original"
}
```

Resultado: se devuelve el ticket `13`, sin crear otro ticket en esta prueba.

#### Caso B: misma clave y contenido diferente

Al reutilizar la clave con otro mensaje, la API respondió con HTTP `409 Conflict`.

```json
{
  "success": false,
  "error": "IDEMPOTENCY_CONFLICT",
  "message": "La clave Idempotency-Key ya fue utilizada con un contenido diferente."
}
```

Resultado: la operación se rechaza porque la clave ya está asociada a un contenido diferente.

**Mejora pendiente:** unificar el nombre del identificador devuelto por el endpoint (`TicketId` y `ticketId`). También se debe verificar la unicidad de la clave a nivel de PostgreSQL y el comportamiento ante solicitudes simultáneas.

### 6.3 Consultar tickets

**Método:** `GET`

**URL de prueba:**

`http://localhost:5678/webhook-test/tickets`

#### Listar todos los tickets

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/tickets" `
  -Method Get
```

#### Filtrar por estado

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/tickets?status=open" `
  -Method Get
```

#### Filtrar por prioridad

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/tickets?priority=medium" `
  -Method Get
```

#### Filtrar por categoría

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/tickets?category=network" `
  -Method Get
```

#### Buscar por texto

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/tickets?q=VPN" `
  -Method Get
```

#### Combinar filtros

```powershell
Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/tickets?status=open&priority=medium" `
  -Method Get
```

La API aplica los filtros recibidos para limitar los resultados. La búsqueda por texto se realiza sobre los campos definidos en la consulta SQL del workflow.

#### Validación de parámetros

Las prioridades admitidas son `low`, `medium`, `high` y `critical`.

Ejemplo de prueba con un valor inválido:

```powershell
try {
    Invoke-RestMethod `
      -Uri "http://localhost:5678/webhook-test/tickets?priority=urgente" `
      -Method Get
}
catch {
    Write-Host "Código HTTP:" ([int]$_.Exception.Response.StatusCode)
}
```

Resultado observado: **HTTP 400 Bad Request**.

### 6.4 Actualizar un ticket

**Método:** `PATCH`

**Ruta lógica:** `/tickets/:id`

La URL de prueba observada durante el desarrollo fue:

`http://localhost:5678/webhook-test/1de157de-cdd5-41d7-bb9a-b7c5a6b605cd/tickets/:id`

El prefijo de prueba puede cambiar. Se debe utilizar la URL actual del nodo Webhook y reemplazar `:id` por el identificador real.

Solo se permite modificar `status` y `priority`.

#### Actualizar el estado

```powershell
$body = @{
    status = "resolved"
} | ConvertTo-Json

Invoke-RestMethod `
  -Uri "http://localhost:5678/webhook-test/1de157de-cdd5-41d7-bb9a-b7c5a6b605cd/tickets/2" `
  -Method Patch `
  -ContentType "application/json" `
  -Body $body
```

Respuesta exitosa observada:

```json
{
  "id": "2",
  "title": "No puedo conectarme a la VPN. Recibo un error de autenticación.",
  "category": "security",
  "priority": "medium",
  "status": "resolved",
  "summary": "Error de autenticación al conectarse a VPN",
  "requires_human_review": true
}
```

#### Validar un estado inválido

```powershell
$body = @{
    status = "urgent"
} | ConvertTo-Json

try {
    Invoke-RestMethod `
      -Uri "http://localhost:5678/webhook-test/1de157de-cdd5-41d7-bb9a-b7c5a6b605cd/tickets/2" `
      -Method Patch `
      -ContentType "application/json" `
      -Body $body
}
catch {
    Write-Host "Código HTTP:" ([int]$_.Exception.Response.StatusCode)
}
```

Resultado observado: **HTTP 400 Bad Request**.

#### Actualizar un ID inexistente

```powershell
$body = @{
    status = "resolved"
} | ConvertTo-Json

try {
    Invoke-RestMethod `
      -Uri "http://localhost:5678/webhook-test/1de157de-cdd5-41d7-bb9a-b7c5a6b605cd/tickets/9999" `
      -Method Patch `
      -ContentType "application/json" `
      -Body $body
}
catch {
    Write-Host "Código HTTP:" ([int]$_.Exception.Response.StatusCode)
}
```

Resultado observado: **HTTP 404 Not Found**.

Respuesta configurada:

```json
{
  "success": false,
  "error": "TICKET_NOT_FOUND",
  "message": "No existe un ticket con el ID solicitado."
}
```

#### Error de base de datos

Se probó temporalmente una consulta SQL inválida para comprobar el manejo de errores. El workflow devolvió HTTP `500` y el body configurado:

```json
{
  "success": false,
  "error": "DATABASE_ERROR",
  "message": "No se pudo actualizar el ticket. Inténtalo nuevamente."
}
```

La consulta SQL original fue restaurada después de la prueba.

## 7. Matriz de pruebas manuales

| ID | Área | Prueba | Resultado esperado | Resultado observado |
|---|---|---|---|---|
| Q-01 | Consulta | Listado general | Lista de tickets | Correcto |
| Q-02 | Consulta | `status=open` | Solo tickets abiertos | Correcto |
| Q-03 | Consulta | `priority=medium` | Solo prioridad media | Correcto |
| Q-04 | Consulta | `category=network` | Solo categoría network | Correcto |
| Q-05 | Consulta | `q=VPN` | Tickets coincidentes | Correcto |
| Q-06 | Consulta | Búsqueda sin coincidencias | Sin tickets coincidentes | Correcto |
| Q-07 | Consulta | Estado y prioridad combinados | Cumplir ambos filtros | Correcto |
| Q-08 | Consulta | Prioridad inválida | HTTP 400 | Correcto |
| U-01 | Actualización | Ticket 2 a `resolved` | HTTP 200 y estado actualizado | Correcto |
| U-02 | Actualización | Estado `urgent` | HTTP 400 | Correcto |
| U-03 | Actualización | Ticket `9999` | HTTP 404 | Correcto |
| U-04 | Actualización | Error SQL temporal | HTTP 500 controlado | Correcto |
| I-01 | Idempotencia | Primera solicitud con clave nueva | Crear ticket | Creó ticket 13 |
| I-02 | Idempotencia | Misma clave y contenido | Devolver ticket 13 | Correcto |
| I-03 | Idempotencia | Misma clave y contenido diferente | HTTP 409 | Correcto |
| DB-01 | Persistencia | Consultar ticket 13 | Datos guardados correctamente | Correcto |

## 8. Verificación de persistencia

Consulta ejecutada en PostgreSQL:

```sql
SELECT
    id,
    title,
    description,
    category,
    priority,
    status,
    created_at
FROM tickets
WHERE id = 13;
```

Resultado observado:

| Campo | Valor |
|---|---|
| `id` | `13` |
| `title` | `Prueba de idempotencia: el monitor no enciende.` |
| `description` | `Prueba de idempotencia: el monitor no enciende.` |
| `category` | `hardware` |
| `priority` | `high` |
| `status` | `open` |
| `created_at` | `2026-10-13T15:48:41.510Z` |

La consulta confirmó que el ticket original existe y que sus datos se guardaron correctamente. La verificación de unicidad de la clave en PostgreSQL sigue pendiente.

## 9. Códigos HTTP

| Código | Significado en el proyecto |
|---|---|
| `200 OK` | Consulta o actualización exitosa; reintento idempotente según el workflow |
| `400 Bad Request` | Parámetros o datos inválidos |
| `404 Not Found` | Ticket inexistente |
| `409 Conflict` | Reutilización de una clave de idempotencia con contenido diferente |
| `500 Internal Server Error` | Error interno, como un fallo de consulta a PostgreSQL |

## 10. Mejoras futuras


1. Incorporar autenticación y autorización antes de exponer la API fuera del entorno local.
2. Añadir logs y métricas de ejecución.
3. Automatizar las pruebas para que sean repetibles.

## 11. Ejecución de las pruebas

1. Abrir el workflow correspondiente en n8n.
2. Hacer clic en **Execute Workflow**.
3. Ejecutar el comando de PowerShell.
4. Revisar la respuesta y la ejecución del workflow.
5. Para operaciones que modifican datos, comprobar el resultado en PostgreSQL.
6. Restaurar cualquier consulta o configuración temporal utilizada para provocar errores.

---

**Proyecto:** Smart IT Helpdesk  
**Stack:** n8n, Ollama, PostgreSQL, Docker Desktop y PowerShell  
**Tipo de pruebas:** manuales en entorno local
