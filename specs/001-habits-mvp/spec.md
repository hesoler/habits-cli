# Especificación — habits-cli MVP

## Contexto y objetivo

Muchas personas quieren construir y mantener hábitos de estudio, pero carecen de una herramienta simple y sin fricción para registrar su actividad diaria y monitorear su constancia. Las aplicaciones existentes suelen ser complejas, requieren registro de cuentas o dependen de conectividad a internet.

**Objetivo**: Desarrollar una interfaz de línea de comandos (CLI) que permita crear hábitos, marcarlos como completados (para hoy o para ayer), listar los hábitos con su racha actual de días consecutivos y eliminar hábitos, manteniendo persistencia local en formato JSON dentro de la raíz del proyecto.

## Usuarios

- **Estudiante / Desarrollador Junior**: Usuario que interactúa desde la terminal de comandos para gestionar sus hábitos de estudio diarios, exigiendo una herramienta rápida, predecible y con mensajes claros en español.

## Historias de usuario

- **HU-1 — Crear hábito**: Como estudiante, quiero dar de alta un nuevo hábito indicando su nombre para poder comenzar a registrar mi progreso diario.
- **HU-2 — Marcar hábito como completado**: Como estudiante, quiero marcar un hábito como hecho hoy o ayer (mediante `--yesterday`) para mantener mi historial actualizado incluso si olvidé registrarlo el mismo día.
- **HU-3 — Listar hábitos y rachas**: Como estudiante, quiero visualizar todos mis hábitos con su racha de días consecutivos y su estado de cumplimiento hoy para evaluar mi constancia y motivarme.
- **HU-4 — Eliminar hábito**: Como estudiante, quiero borrar un hábito y todo su historial cuando ya no forme parte de mis objetivos para mantener mi listado relevante y ordenado.

---

## Requisitos funcionales

### RF-1 — Crear hábito

- **RF-1.1**: CUANDO el usuario ejecute `habits add <nombre>` con un nombre válido, EL SISTEMA DEBE registrar el nuevo hábito y almacenar su fecha de creación en formato ISO 8601 (`YYYY-MM-DD`) (salida 0).
- **RF-1.2**: SI el nombre ya existe (comparación ignorando mayúsculas/minúsculas y espacios al inicio o final), ENTONCES EL SISTEMA DEBE rechazar la creación y mostrar un mensaje de error indicando que el hábito ya existe (salida 1).
- **RF-1.3**: SI el nombre proporcionado está vacío, contiene únicamente espacios en blanco o excede los 50 caracteres, ENTONCES EL SISTEMA DEBE rechazar la creación con un mensaje de error explicativo (salida 1).
- **RF-1.4**: SI el nombre contiene caracteres no permitidos (emojis, acentos, diéresis o la letra 'ñ'), ENTONCES EL SISTEMA DEBE rechazar la creación indicando que solo se permiten caracteres ASCII estándar sin tildes ni caracteres especiales (salida 1).

### RF-2 — Marcar hábito como completado

- **RF-2.1**: CUANDO el usuario ejecute `habits done <nombre>` para marcar un hábito sin flags adicionales y el hábito exista, EL SISTEMA DEBE registrar la fecha actual (hoy, formato ISO 8601 `YYYY-MM-DD`) en el historial del hábito (salida 0).
- **RF-2.2**: CUANDO el usuario ejecute el comando para marcar un hábito incluyendo el flag `--yesterday` al final, Ej: `habits done <nombre> --yesterday `, EL SISTEMA DEBE registrar la fecha del día inmediatamente anterior (ayer, formato ISO 8601 `YYYY-MM-DD`) en el historial del hábito (salida 0).
- **RF-2.3**: SI el hábito especificado no existe en el registro, ENTONCES EL SISTEMA DEBE rechazar la operación y mostrar un mensaje de error indicando que el hábito no existe, sugiriendo consultar `habits list` (salida 1).
- **RF-2.4**: SI el hábito ya fue marcado previamente para la fecha destino (hoy o ayer), ENTONCES EL SISTEMA DEBE operar de forma idempotente: no duplicará la fecha en el historial, no alterará la racha y emitirá un mensaje informativo indicando que ya estaba completado (salida 0).

### RF-3 — Listar hábitos y cálculo de racha

- **RF-3.1**: CUANDO el usuario ejecute `habits list`, EL SISTEMA DEBE mostrar cada hábito con tres datos: nombre, racha actual de días consecutivos y estado de cumplimiento en el día de hoy ("hecho" o "pendiente"), ordenados descendentemente por racha y, a su vez, alfabéticamente por nombre (salida 0).
- **RF-3.2**: EL SISTEMA DEBE calcular la racha actual considerando únicamente la continuidad diaria ininterrumpida de fechas completadas:
  - Si un hábito no tiene fechas completadas: la racha es 0.
  - Si un hábito fue completado ayer u hoy como único registro: la racha toma valor 1.
  - Si existen fechas consecutivas terminando en hoy o en ayer: la racha es igual a la cantidad de días consecutivos en dicha secuencia ininterrumpida.
  - Si ni hoy ni ayer fueron completados: la racha es 0 (la continuidad se considera rota).
  - Días completados aislados con huecos entre sí no suman racha entre ellos.
- **RF-3.3**: SI no existen hábitos registrados al solicitar el listado, ENTONCES EL SISTEMA DEBE mostrar un mensaje informativo indicando que no hay hábitos registrados (salida 0).

### RF-4 — Eliminar hábito

- **RF-4.1**: CUANDO el usuario solicite eliminar un hábito existente mediante ejecute `habits remove <nombre>`, EL SISTEMA DEBE remover el hábito y la totalidad de su historial de fechas completadas.
- **RF-4.2**: SI el hábito a eliminar no existe en el registro, ENTONCES EL SISTEMA DEBE mostrar un mensaje de error indicando que el hábito no existe.

### RF-5 — Persistencia de datos

- **RF-5.1**: EL SISTEMA DEBE almacenar los hábitos en un archivo denominado `habits_data.json` ubicado en la raíz de este proyecto.
- **RF-5.2**: EL SISTEMA DEBE estructurar el archivo `habits_data.json` bajo el esquema:
  ```json
  {
    "habits": [
      {
        "name": "string",
        "created": "YYYY-MM-DD",
        "completed_dates": ["YYYY-MM-DD"]
      }
    ]
  }
  ```
- **RF-5.3**: CUANDO el archivo `habits_data.json` no exista al momento de consultar o registrar datos, EL SISTEMA DEBE crearlo automáticamente con una lista vacía de hábitos (`{"habits": []}`).
- **RF-5.4**: SI el archivo `habits_data.json` existe pero contiene un formato inválido o JSON corrupto, ENTONCES EL SISTEMA DEBE detener la operación y mostrar un mensaje de error explícito advirtiendo sobre la corrupción del archivo sin sobrescribir los datos dañados.

---

## Requisitos no funcionales

- **RNF-1 — Stack y dependencias**: La solución debe ejecutarse en Python 3.12+ utilizando exclusivamente la biblioteca estándar (con `pytest` reservado exclusivamente para la suite de pruebas).
- **RNF-2 — Idioma de interfaz**: Todos los mensajes visibles para el usuario (éxitos, errores, tablas y avisos) deben estar redactados en español claro y conciso.
- **RNF-3 — Idioma de desarrollo**: Los identificadores de código, nombres de funciones, variables, comentarios internos, tests y documentación técnica deben escribirse en inglés.
- **RNF-4 — Control de versiones**: El archivo de persistencia `habits_data.json` debe estar ignorado por el sistema de control de versiones Git (`.gitignore`).
- **RNF-5 — Rendimiento local**: La ejecución de cualquier comando de la CLI debe ser instantánea (< 150 ms en condiciones normales de hardware local).
- **RNF-6 — Multiplataforma**: La solución debe ser compatible para macOS, Linux y Windows.

---

## Casos límite

| Caso | Condición / Entrada | Comportamiento esperado |
|------|---------------------|------------------------|
| **Nombre vacío o espacios** | `""` o `"   "` | Error en español: rechazo por nombre inválido. |
| **Longitud excedida** | Nombre > 50 caracteres | Error en español: excede límite de 50 caracteres. |
| **Caracteres inválidos** | Tildes (`matemáticas`), ñ (`año`), emojis (`📚`) | Error en español: solo caracteres ASCII permitidos. |
| **Sensibilidad a mayúsculas** | Crear "Estudiar" existiendo "estudiar" | Permitido: son tratados como hábitos diferentes. |
| **Duplicado exacto** | Crear "Estudiar" existiendo "Estudiar" | Error en español: "Ya existe un hábito con ese nombre". |
| **Marcar hábito inexistente** | Nombre no registrado | Error en español: "No existe un hábito con ese nombre". |
| **Idempotencia hoy** | Marcar hoy dos o más veces | Operación exitosa, no duplica fecha en JSON, confirma estado. |
| **Idempotencia ayer** | Marcar `--yesterday` dos o más veces | Operación exitosa, no duplica fecha en JSON, confirma estado. |
| **Hábito creado ayer y completado ayer** | Creado ayer, marcado ayer, hoy pendiente | Racha = 1; hoy figura como pendiente. |
| **Racha con dos días consecutivos** | Completado ayer y hoy | Racha = 2; hoy figura como hecho. |
| **Racha rota por hueco de un día** | Completado anteayer, ayer no, hoy no | Racha = 0; continuidad rota. |
| **Racha con hueco pero retomado hoy** | Completado hace 3 días, ayer no, hoy sí | Racha = 1 (comienza nueva continuidad a partir de hoy). |
| **Eliminar hábito inexistente** | Nombre no registrado | Error en español: "No existe un hábito con ese nombre". |
| **Listar con registro vacío** | `habits_data.json` sin hábitos | Mensaje en español: "No hay hábitos registrados". |
| **JSON corrupto o truncado** | Archivo no parseable por parser JSON | Error en español sin sobrescribir ni borrar archivo dañado. |

---

## Fuera de alcance (MVP)

- Modificación o renombrado de hábitos existentes.
- Registro retroactivo de fechas previas a ayer (más de 1 día hacia atrás).
- Registro de fechas futuras.
- Frecuencias personalizadas de hábitos (semanales, días laborables, etc.). Solo seguimiento diario.
- Métricas avanzadas, gráficas, porcentajes de adherencia o visualizaciones complejas.
- Soporte para múltiples perfiles de usuario o autenticación.
- Sincronización en la nube o almacenamiento en bases de datos relacionales/NoSQL.
- Notificaciones del sistema o recordatorios en segundo plano.
- Exportación o importación masiva de datos (CSV, Markdown, etc.).

---

## Criterios de finalización

- [ ] Los 4 comandos de la CLI (`add`, `done` con o sin `--yesterday`, `list`, `remove`) funcionan según la especificación.
- [ ] La persistencia respeta la ubicación en la raíz del proyecto y el esquema JSON establecido en `habits_data.json`.
- [ ] El archivo `habits_data.json` está explícitamente ignorado en `.gitignore`.
- [ ] El cálculo de racha cumple estrictamente las reglas de continuidad diaria (0, 1 y N días consecutivos).
- [ ] Todos los mensajes dirigidos al usuario están en español y manejan validaciones de entrada (longitud, caracteres ASCII, duplicados, idempotencia).
- [ ] Cada requisito funcional cuenta con una suite de pruebas automatizadas con `pytest` que lo verifique.
- [ ] La constitución del proyecto se cumple rigurosamente (stdlib pura, sin dependencias externas fuera de pytest, código en inglés).

---

## Dudas abiertas

*No hay dudas abiertas pendientes. Todas las decisiones funcionales y de formato han sido resueltas para este MVP.*
