# TurnosMed | API 3 de Integraciones Web

Daniel Oscar Michel · Versión 3.0.0 · 27 de septiembre de 2026

Refactorización de la entrega API 2 (TurnosRed) con controladores asincrónicos, validación previa, manejo uniforme de errores y documentación. Conserva las rutas `/medicos` y `/turnos`, los 30 profesionales y las 22 especialidades de API 2. Agrega `/profesionales`, `/especialidades` y la bienvenida `/`.

## Inicio rápido

Requisitos: Node.js 24 LTS (24.12 o posterior de la rama 24) y npm.
Abrir una terminal en esta carpeta, junto a `package.json`:

```powershell
npm.cmd install
Copy-Item .env.example .env
npm.cmd run build
npm.cmd start
```

Consultar `http://127.0.0.1:3000/`. Detener con Ctrl+C. En macOS/Linux usar `npm` y `cp .env.example .env`.

Se conserva `pnpm-lock.yaml` de la base porque no cambiaron las dependencias. Instalación reproducible alternativa: `pnpm install --frozen-lockfile`, `pnpm run build`, `pnpm start`. Con npm, conservar el `package-lock.json` que genera la instalación.

| Comando | Resultado |
|---|---|
| `npm.cmd run dev` | Ejecuta TypeScript con recarga; no comprueba tipos. |
| `npm.cmd run build` | Comprueba tipos y genera `dist/`. |
| `npm.cmd test` | Compila y ejecuta las ocho pruebas complementarias. |
| `npm.cmd run test:postman` | Ejecuta 65 solicitudes con Newman; compilar antes. |

`test:postman` inicia su propio servidor temporal en un puerto libre, carga los JSON y lo cierra al finalizar. No depende del servidor de desarrollo. El runner genera `reports/newman-results.json`. Cada ejecución empieza desde los datos originales. La colección requiere ejecutarse una sola vez y completa desde un estado inicial; ante una interrupción, reiniciar el servidor antes de volver a correrla.

## Rutas

| Recurso | Métodos y ruta | Contrato |
|---|---|---|
| Bienvenida | GET `/` | 200 y mensaje Hello World. |
| Profesionales | GET/POST `/profesionales`; GET/PUT/DELETE `/profesionales/:id` | Mismos datos y campos que médicos. |
| Especialidades | GET/POST `/especialidades`; GET/PUT/DELETE `/especialidades/:id` | Catálogo editable con integridad referencial. |
| Médicos | GET/POST `/medicos`; GET/PUT/DELETE `/medicos/:id` | Compatible con API 2. |
| Turnos | GET/POST `/turnos`; GET/PUT/DELETE `/turnos/:id` | Compatible con API 2. |
| Ruta no reconocida | Cualquier ruta sin handler | 404 mediante el controlador general. |

GET y PUT: 200. POST: 201 con `Location`. DELETE: 204 sin cuerpo (no se envía JSON con 204). Validación: 400. ID bien formado inexistente: 404. Fallo inesperado: 500. Un listado sin coincidencias devuelve 200 y `[]`.

## Cuerpos y filtros

Profesional/médico: `nombre` (string), `especialidad` (nombre exacto de una especialidad activa) y `disponible` (booleano), todos obligatorios. El servidor genera `medicoId` UUID; `/profesionales` conserva ese nombre para no romper el contrato heredado. Filtros: `especialidad` y `disponible=true|false`.

```json
{"nombre":"Profesional de prueba","especialidad":"Pediatría","disponible":true}
```

Especialidad: `nombreEspecialidad` (string no vacío) y `activa` (booleano), ambos obligatorios. El servidor genera `especialidadId` UUID. Filtros: `nombre` y `activa=true|false`.

```json
{"nombreEspecialidad":"Especialidad de prueba","activa":true}
```

No se duplican nombres de especialidades aunque cambien mayúsculas o tildes. No se permite borrar, renombrar ni desactivar una especialidad vinculada a profesionales o turnos. Primero deben resolverse sus relaciones. Los PUT son reemplazos completos; se rechazan campos extra, IDs inválidos, valores de tipo incorrecto y parámetros de consulta repetidos. Los filtros por nombre toleran mayúsculas/minúsculas y ausencia de tildes.

Turno: `paciente`, `documento`, `medicoId`, `especialidad`, `fecha` y `hora`, todos strings obligatorios. `documento` conserva letras, puntos y ceros iniciales. Fecha acepta `AAAA-MM-DD` o `DD/MM/AAAA` y se devuelve `AAAA-MM-DD`. Hora: `HH:MM`. Filtros: `especialidad`, `fecha`, `medicoId`; se combinan con AND. Agenda de lunes a viernes, de 07:00 a 12:30 cada 30 minutos, hasta el 30 de diciembre del año de ejecución. Se permiten fechas pasadas como en API 2. No se permite superponer un médico o paciente en la misma fecha/hora, reservar con un médico no disponible o con una especialidad distinta.

## Arquitectura

```text
Solicitud HTTP
  -> rutas (asocian método y controlador)
  -> controlador async (valida, espera servicio, decide estado y responde)
  -> servicio (reglas de negocio)
  -> arrays en memoria (cargados de src/data/*.json)

Error de validación o negocio -> catch del controlador -> JSON uniforme
JSON mal formado o cuerpo >100 KB -> middleware de errores -> JSON uniforme
Ruta desconocida -> general.controller.notFound -> 404
```

Cada método exportado de `src/controllers/` es `async`, contiene `try/catch`, usa una variable local `status` y termina cada respuesta con `return res.status(status)...`. Las validaciones se ejecutan antes de consultas o mutaciones. `parse` lanza `new AppError(...)`, una subclase de `Error`; los servicios también lanzan errores de dominio. El catch transforma esos errores en la respuesta pactada. Los errores inesperados generan 500 y ocultan los detalles internos al cliente. Se usa `await` al invocar los servicios para aceptar luego implementaciones asincrónicas; hoy los servicios trabajan con arrays y no simulan una base de datos.

La separación implementa la etapa de controladores que pide la actividad. No se afirma que sea una Clean Architecture completa: servicios todavía dependen del almacén concreto. `documentation/arquitectura.md` describe el alcance y la evolución posible.

```json
{"status":404,"message":"Ruta no encontrada","code":"ROUTE_NOT_FOUND","details":[]}
```

## Postman y evidencias

Importar `turnos-med.postman_collection.json` y `turnos-med.postman_environment.json`. Elegir **TurnosMed API3 Local**, comprobar `baseUrl=http://127.0.0.1:3000` y ejecutar la colección en orden con Collection Runner. Los IDs se capturan automáticamente. Autorización: No Auth; no se requiere token.

Resultado verificado: 65 solicitudes, 212 aserciones, cero fallas; ocho pruebas complementarias aprobadas y compilación TypeScript correcta. Los informes de ejecución están en `reports/`. Los ejemplos de respuestas guardados en la colección proceden de la ejecución real de esta versión; contienen UUID ilustrativos de esa ejecución, no IDs persistentes.

## Documentación incluida

- `documentation/openapi.json`: contrato OpenAPI 3.0.3 de las rutas.
- `documentation/arquitectura.md`: flujo, límites y diagrama Mermaid.
- `documentation/ADR-001-controladores.md`: decisión y consecuencias.
- `CHANGELOG.md`: cambios respecto de API 2.
- `documentation/contrato-heredado-api2.md`: referencia histórica, no guía principal de esta versión.
- `turnos-red.postman_collection.json`: colección API 2 conservada para regresión.

## Entorno y datos

PORT=3000 y HOST=127.0.0.1 por defecto. Solo escucha local. Los cambios duran mientras vive el proceso; al reiniciar se restauran los JSON y se vacían los turnos. Conservar `src/data/` junto a `dist/`. No incluye base de datos ni autenticación. Los datos son ficticios.

