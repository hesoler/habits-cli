# Plan de Implementación — habits-cli MVP

> **Para trabajadores agenticos:** HABILIDAD REQUERIDA: Usar `superpowers:subagent-driven-development` o `superpowers:executing-plans` para implementar este plan tarea por tarea.

**Objetivo:** Desarrollar una herramienta CLI en Python (estándar puro) para la gestión local de hábitos de estudio, registro de cumplimiento (hoy/ayer), cálculo de rachas de días consecutivos y almacenamiento en JSON.

**Arquitectura:** Separación estricta entre capa de presentación CLI (`habits/cli.py`), capa de persistencia atómica (`habits/storage.py`) y lógica pura del dominio (`habits/core.py`). Ninguna función del núcleo o almacenamiento realiza I/O directo por consola o maneja la interfaz de usuario.

**Tech Stack:** Python 3.12+ (exclusivamente biblioteca estándar: `argparse`, `json`, `datetime`, `pathlib`, `os`, `tempfile`, `sys`, `re`) + `pytest` para la suite de pruebas.

---

## 1. Estructura de Módulos

*Mapeo de requisitos y principios:* [Constitución Principios 1, 3, 5 y 6; RNF-1, RNF-3]

```text
habits-cli/
├── habits/
│   ├── __init__.py       # Marcador de paquete y versión (__version__ = "0.1.0")
│   ├── __main__.py       # Punto de entrada para invocación 'python -m habits'
│   ├── core.py           # Lógica pura del dominio (entidades, validaciones, algoritmo de racha)
│   ├── storage.py        # Persistencia en JSON, validación de esquema y escritura atómica
│   └── cli.py            # Parser CLI (argparse), orquestación y formateo de mensajes en español
├── tests/
│   ├── conftest.py       # Fixtures reutilizables de pytest (archivos temporales, estados de hábitos)
│   ├── test_core.py      # Tests unitarios de validaciones y cálculo de rachas
│   ├── test_storage.py   # Tests unitarios e integración de persistencia y archivos corruptos
│   └── test_cli.py       # Tests de integración de comandos CLI, mensajes y códigos de salida
├── specs/
│   └── 001-habits-mvp/
│       ├── spec.md       # Especificación funcional activa
│       └── plan.md       # Este documento de plan técnico
├── docs/
│   └── constitution.md   # Constitución del proyecto
├── .gitignore            # Exclusión de habits_data.json, pycache, etc.
└── AGENTS.md             # Instrucciones para agentes
```

### Responsabilidades por Módulo

- **`habits/core.py`**:
  - Definición de estructuras/dataclasses (`Habit`).
  - Validación de nombres (trim, longitud, caracteres ASCII imprimibles).
  - Sanitización y comparación case-insensitive de nombres.
  - Validación de reglas temporales (no marcar previo a creación, no marcar ayer tras hoy).
  - Algoritmo puro de cálculo de racha e indicador de estado ("hecho" / "pendiente").
  - *Cero dependencias de I/O de consola, sistema de archivos o `sys.exit`.*

- **`habits/storage.py`**:
  - Localización del archivo `habits_data.json` en la raíz del proyecto.
  - Lectura y parsing de datos JSON.
  - Validación estricta del esquema de datos JSON en memoria.
  - Guardado atómico utilizando archivos temporales (`tempfile.NamedTemporaryFile`) y reemplazo seguro (`os.replace`).
  - Manejo de excepciones de corrupción de archivo o formato inválido sin alterar el archivo original.

- **`habits/cli.py`**:
  - Definición de la estructura de subcomandos con `argparse`.
  - Captura de argumentos y orquestación con `storage` y `core`.
  - Formateo de salidas de texto y tablas dinámicas en español.
  - Retorno de códigos de salida del sistema (`0` para éxito, `1` para error) mediante `sys.exit`.

- **`habits/__main__.py`**:
  - Invoca la función principal de la CLI (`cli.main()`).

---

## 2. Modelo de Datos JSON y Esquema Estricto

*Mapeo de requisitos:* [RF-5.1, RF-5.2, RF-5.3, RF-5.4, RNF-4]

### Ubicación y Nombre del Archivo
- Path: `habits_data.json` ubicado en la raíz del repositorio Git o directorio de trabajo actual.
- Ignorado explícitamente en `.gitignore`.

### Especificación de Campos del Esquema

| Campo                      | Tipo           | Requisitos / Restricciones                                                                                               |
|:---------------------------|:---------------|:-------------------------------------------------------------------------------------------------------------------------|
| `habits`                   | `list`         | Lista de objetos hábito. Si el archivo es nuevo, se inicializa como `[]`.                                                |
| `habits[].name`            | `string`       | Nombre del hábito. ASCII imprimible (caracteres `0x20` a `0x7E`), 1 a 50 caracteres (post-trim). Único case-insensitive. |
| `habits[].created`         | `string`       | Fecha de creación del hábito en formato ISO 8601 `YYYY-MM-DD`.                                                           |
| `habits[].completed_dates` | `list[string]` | Lista de fechas únicas completadas en formato ISO 8601 `YYYY-MM-DD`, almacenadas ordenadas cronológicamente.             |

### Ejemplo de `habits_data.json`

```json
{
  "habits": [
    {
      "name": "leer 20 paginas",
      "created": "2026-09-01",
      "completed_dates": [
        "2026-09-05",
        "2026-09-06",
        "2026-09-07"
      ]
    },
    {
      "name": "ejercicio",
      "created": "2026-09-03",
      "completed_dates": [
        "2026-09-06"
      ]
    },
    {
      "name": "meditar",
      "created": "2026-09-07",
      "completed_dates": []
    }
  ]
}
```

---

## 3. Algoritmo de Cálculo de Racha y Estado

*Mapeo de requisitos:* [RF-3.1, RF-3.2]

### Algoritmo en Pseudocódigo (`calculate_streak`)

```python
def calculate_streak(completed_dates_str: list[str], current_date: date) -> int:
    """
    Calcula la racha actual de días consecutivos completados.
    
    Reglas:
    1. Si no hay fechas completadas -> Racha = 0.
    2. Convierte strings ISO YYYY-MM-DD a objetos date, elimina duplicados y ordena de menor a mayor.
    3. Evalúa si la fecha más reciente consecutiva termina en current_date (hoy) o en current_date - 1 (ayer).
       Si ni hoy ni ayer están presentes en la lista -> Racha = 0 (racha rota).
    4. Recorre hacia atrás las fechas ordenadas desde el punto de anclaje (hoy o ayer):
       - Incrementa el contador por cada día estrictamente consecutivo (d_actual - d_previo == 1 día).
       - En cuanto se encuentra una brecha de 2 o más días -> interrumpe el conteo y retorna el acumulado.
    """
    if not completed_dates_str:
        return 0

    # Parsear a fechas únicas y ordenadas cronológicamente
    unique_dates = sorted({date.fromisoformat(d) for d in completed_dates_str})

    yesterday = current_date - timedelta(days=1)

    # Buscar si la racha está activa (anclada en hoy o ayer)
    if current_date in unique_dates:
        anchor_date = current_date
    elif yesterday in unique_dates:
        anchor_date = yesterday
    else:
        # Ni hoy ni ayer fueron completados: la racha se rompió
        return 0

    streak = 1
    target_previous = anchor_date - timedelta(days=1)

    # Filtrar fechas menores que el ancla y recorrer en orden descendente
    prior_dates = [d for d in unique_dates if d < anchor_date]
    
    for d in reversed(prior_dates):
        if d == target_previous:
            streak += 1
            target_previous -= timedelta(days=1)
        elif d < target_previous:
            # Hueco detectado en la secuencia diaria consecutiva
            break

    return streak


def calculate_status_today(completed_dates_str: list[str], current_date: date) -> str:
    """
    Retorna 'hecho' si la fecha actual está en completed_dates_str, o 'pendiente' en caso contrario.
    """
    today_str = current_date.isoformat()
    return "hecho" if today_str in completed_dates_str else "pendiente"
```

### Casos de Ejemplo del Algoritmo

| Fechas completadas (`completed_dates`)       | Fecha actual (`today`) | Racha Calculada | Estado Hoy  |
|:---------------------------------------------|:-----------------------|:----------------|:------------|
| `[]`                                         | `2026-09-07`           | `0`             | `pendiente` |
| `["2026-09-07"]`                             | `2026-09-07`           | `1`             | `hecho`     |
| `["2026-09-06"]`                             | `2026-09-07`           | `1`             | `pendiente` |
| `["2026-09-05"]`                             | `2026-09-07`           | `0`             | `pendiente` |
| `["2026-09-05", "2026-09-06", "2026-09-07"]` | `2026-09-07`           | `3`             | `hecho`     |
| `["2026-09-04", "2026-09-05", "2026-09-06"]` | `2026-09-07`           | `3`             | `pendiente` |
| `["2026-09-03", "2026-09-06", "2026-09-07"]` | `2026-09-07`           | `2`             | `hecho`     |

---

## 4. Contrato de la CLI

*Mapeo de requisitos:* [RF-1, RF-2, RF-3, RF-4, RF-5, RNF-2]

### Invocación General y Ayuda (`python -m habits` o `habits`)

- **Comando**: `python -m habits` (o `python -m habits --help`)
- **Código de salida**: `0`
- **Salida (stdout)**:
  ```text
  Uso: habits <comando> [opciones]

  Comandos disponibles:
    add <nombre>            Crea un nuevo hábito.
    done <nombre> [--yesterday]  Marca un hábito como completado (hoy o ayer).
    list                    Muestra la lista de hábitos, rachas y estado.
    delete <nombre>         Elimina un hábito y su historial.

  Opciones generales:
    -h, --help              Muestra este mensaje de ayuda.
  ```

---

### Comando 1: Crear Hábito (`add`)

- **Sintaxis**: `python -m habits add <nombre>`
- **Comportamiento**:
  - Aplica *trimming* de espacios en los extremos del argumento.
  - Valida que el nombre no esté vacío, tenga entre 1 y 50 caracteres ASCII imprimibles (`0x20` a `0x7E`).
  - Compara de forma *case-insensitive* contra hábitos existentes.
  - Guarda la fecha actual como fecha de creación (`created`).

- **Respuestas de Salida**:

| Caso                           | Salida Exacta                                                                                  | Código Exit | RF Cubierto |
|:-------------------------------|:-----------------------------------------------------------------------------------------------|:------------|:------------|
| **Éxito**                      | `Hábito '<nombre>' creado con éxito.`                                                          | `0`         | RF-1.1      |
| **Nombre ya existe**           | `Error: Ya existe un hábito con el nombre '<nombre_existente>'.`                               | `1`         | RF-1.2      |
| **Nombre vacío o >50 chars**   | `Error: El nombre del hábito debe tener entre 1 y 50 caracteres.`                              | `1`         | RF-1.3      |
| **Caracteres no ASCII**        | `Error: El nombre solo puede contener caracteres ASCII imprimibles (sin tildes, ñ ni emojis).` | `1`         | RF-1.4      |
| **Falta argumento `<nombre>`** | `Error: Debe proporcionar el nombre del hábito. Uso: habits add <nombre>`                      | `1`         | RF-1.5      |

---

### Comando 2: Marcar Completado (`done`)

- **Sintaxis**: `python -m habits done <nombre> [--yesterday]`
- **Comportamiento**:
  - Identifica el hábito objetivo por nombre (*case-insensitive*).
  - Determina la fecha objetivo (`hoy` o `ayer`).
  - Evalúa restricciones temporales antes de modificar:
    1. Si fecha objetivo < `created` -> rechazar.
    2. Si fecha objetivo es `ayer` y el hábito ya tiene registrada la fecha de `hoy` para la sesión del mismo día -> rechazar.
  - Registra la fecha en `completed_dates` (idempotente: si ya existe, no duplica).

- **Respuestas de Salida**:

| Caso                                  | Salida Exacta                                                                                                   | Código Exit | RF Cubierto |
|:--------------------------------------|:----------------------------------------------------------------------------------------------------------------|:------------|:------------|
| **Éxito (Hoy)**                       | `Hábito '<nombre>' marcado como completado para hoy.`                                                           | `0`         | RF-2.1      |
| **Éxito (Ayer)**                      | `Hábito '<nombre>' marcado como completado para ayer.`                                                          | `0`         | RF-2.2      |
| **Idempotente (Ya marcado hoy/ayer)** | `El hábito '<nombre>' ya estaba completado para [hoy\|ayer].`                                                   | `0`         | RF-2.6      |
| **Fecha anterior a creación**         | `Error: No se puede registrar cumplimiento anterior a la fecha de creación del hábito (<created>).`             | `1`         | RF-2.3      |
| **Ayer tras completar hoy**           | `Error: No se puede registrar ayer tras haber marcado el hábito como completado hoy.`                           | `1`         | RF-2.4      |
| **Hábito no existe**                  | `Error: No existe un hábito con el nombre '<nombre>'. Consulte 'habits list' para ver los hábitos registrados.` | `1`         | RF-2.5      |
| **Falta argumento `<nombre>`**        | `Error: Debe proporcionar el nombre del hábito. Uso: habits done <nombre> [--yesterday]`                        | `1`         | RF-2.7      |

---

### Comando 3: Listar Hábitos (`list`)

- **Sintaxis**: `python -m habits list`
- **Comportamiento**:
  - Calcula racha y estado de hoy para cada hábito.
  - Ordena hábitos descendentemente por racha (`racha` desc), y ante igual racha, alfabéticamente por nombre (`name` asc, case-insensitive).
  - Formatea la tabla dinámicamente según el ancho del hábito más largo (mínimo de cabecera: "Habito" -> 6 caracteres).

- **Respuestas de Salida**:

| Caso                        | Salida Exacta                           | Código Exit | RF Cubierto    |
|:----------------------------|:----------------------------------------|:------------|:---------------|
| **Éxito con hábitos**       | *(Ver formato de tabla a continuación)* | `0`         | RF-3.1, RF-3.2 |
| **Sin hábitos registrados** | `No hay hábitos registrados.`           | `0`         | RF-3.3         |

#### Formato Visual de Tabla

Ancho de columna `Habito`: `max(6, max(len(h.name) for h in habits))`.

```text
Habito               | Racha | Hoy
---------------------+-------+----------
leer 20 paginas      | 3     | hecho
ejercicio            | 1     | pendiente
meditar              | 0     | pendiente
```

---

### Comando 4: Eliminar Hábito (`delete`)

- **Sintaxis**: `python -m habits delete <nombre>`
- **Comportamiento**:
  - Busca el hábito por nombre (*case-insensitive*).
  - Elimina el objeto hábito y su historial.
  - Guarda los cambios atómicamente.

- **Respuestas de Salida**:

| Caso                           | Salida Exacta                                                                | Código Exit | RF Cubierto |
|:-------------------------------|:-----------------------------------------------------------------------------|:------------|:------------|
| **Éxito**                      | `Hábito '<nombre>' eliminado con éxito.`                                     | `0`         | RF-4.1      |
| **Hábito no existe**           | `Error: No existe un hábito con el nombre '<nombre>'.`                       | `1`         | RF-4.2      |
| **Falta argumento `<nombre>`** | `Error: Debe proporcionar el nombre del hábito. Uso: habits delete <nombre>` | `1`         | RF-4.3      |

---

### Salidas de Error Generales (Archivos / Argumentos Inválidos)

| Condición                            | Salida Exacta                                                                                                                       | Código Exit | RF Cubierto  |
|:-------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------|:------------|:-------------|
| **JSON corrupto o esquema alterado** | `Error: El archivo 'habits_data.json' contiene datos corruptos o un formato inválido. Operación cancelada para proteger los datos.` | `1`         | RF-5.4       |
| **Comando / Flag desconocido**       | `Error: Comando o flag no reconocido '<input>'. Use 'habits --help' para ver la ayuda.`                                             | `1`         | Casos Límite |

---

## 5. Decisiones Técnicas Justificadas

### Decisión 1: Módulo CLI — `argparse` (Stdlib) frente a Parser Manual con `sys.argv` o `click`/`typer`
- **Elección**: Utilizar `argparse` de la biblioteca estándar de Python.
- **Justificación**:
  - Cumple estrictamente con el **Principio 1 de la Constitución** (Stack mínimo, cero dependencias de terceros fuera de stdlib).
  - Ofrece manejo robusto de subcomandos (`add_subparsers`), flags como `--yesterday` e `--help` nativos, y parsing estandarizado de argumentos.
  - Se configurará con formateo de mensajes personalizado para sobreescribir las salidas por defecto en inglés de `argparse` y garantizar mensajes 100% en español (Principio 6 y RNF-2).
- **Alternativas descartadas**:
  - `click` / `typer`: Descartadas por requerir dependencias externas.
  - Parsing manual con `sys.argv`: Descartado por ser propenso a errores en la gestión de flags opcionales (`--yesterday`, `--help`) y combinación de argumentos.

---

### Decisión 2: Persistencia Atómica — `tempfile.NamedTemporaryFile` + `os.replace` frente a `open('w')` directo
- **Elección**: Usar escritura en archivo temporal en el mismo directorio y reemplazo atómico mediante `os.replace`.
- **Justificación**:
  - Garantiza **RF-5.5** (Persistencia atómica). Evita que un cierre abrupto del sistema, fallo de energía o cancelación de proceso (`Ctrl+C`) deje el archivo `habits_data.json` truncado o en estado intermedio corrupto.
  - `os.replace` es una operación atómica a nivel de sistema operativo en POSIX y Windows (cuando ambos archivos están en el mismo sistema de archivos/volumen).
- **Alternativas descartadas**:
  - `open("habits_data.json", "w")` directo: Descartado porque trunca el archivo a 0 bytes al abrirlo antes de escribir los datos, dejando el sistema vulnerable a corrupción de datos ante interrupciones.

---

### Decisión 3: Formateo de Tabla — Formateador propio con `ljust()` y `f-strings` frente a `tabulate` / `rich`
- **Elección**: Implementar una función formateadora de tablas utilizando manipulación de strings nativa de Python (`str.ljust()`, `f-strings`).
- **Justificación**:
  - Cumple con **Principio 1** (stdlib pura) y **RNF-5** (Rendimiento local < 150 ms).
  - La tabla requerida para el MVP es sencilla (3 columnas fijas: `Habito`, `Racha`, `Hoy`). Un formateador propio requiere menos de 15 líneas de código y ejecuta en < 1 ms.
- **Alternativas descartadas**:
  - `tabulate` / `rich` / `texttable`: Descartadas por ser librerías de terceros.

---

### Decisión 4: Validaciones del Esquema JSON en Memoria frente a `jsonschema`
- **Elección**: Implementar una función de validación de esquema nativa en `habits/storage.py` usando `isinstance()` y comprobaciones de tipos/formatos ISO.
- **Justificación**:
  - Cumple con **Principio 1** (stdlib pura).
  - Permite emitir el mensaje de error exacto exigido en **RF-5.4** sin exponer trazas internas ni depender de paquetes externos.
- **Alternativas descartadas**:
  - `jsonschema`: Descartada por ser una dependencia de terceros.

---

### Decisión 5: Inyección de Fecha Actual (`today_provider`) para Pruebas frente a `freezegun`
- **Elección**: Diseñar las funciones del núcleo (`core.py`) aceptando un parámetro explícito `today: date | None = None` (que toma `date.today()` por defecto si es `None`). Para los tests de la CLI, utilizar `unittest.mock.patch` / `monkeypatch` sobre el punto de obtención de fecha.
- **Justificación**:
  - Mantiene el núcleo puro y determinista sin depender de estados globales ni parches durante las pruebas unitarias de la lógica.
  - No añade dependencias de pruebas fuera de `pytest` y la stdlib (`unittest.mock`).
- **Alternativas descartadas**:
  - `freezegun` / `time-machine`: Descartadas por requerir dependencias de terceros.

---

## 6. Estrategia de Tests y Matriz de Trazabilidad

*Mapeo de requisitos:* [Principio 4 de la Constitución, RNF-1]

### Capas de Pruebas

1. **Pruebas Unitarias del Núcleo (`tests/test_core.py`)**:
   - Validaciones de nombres de hábitos (límites de longitud, caracteres permitidos, espacios).
   - Sanitización y equivalencia case-insensitive.
   - Algoritmo de cálculo de racha en todos los escenarios temporales (0, 1, N días, rachas rotas, fechas desordenadas).
   - Reglas de negocio para registro de fechas (no previo a creación, no ayer tras hoy).

2. **Pruebas Unitarias de Persistencia (`tests/test_storage.py`)**:
   - Creación automática de `habits_data.json` si no existe (`{"habits": []}`).
   - Lectura y guardado correcto de hábitos.
   - Verificación de atomiciencia en el guardado.
   - Detección y manejo adecuado de archivos JSON con sintaxis corrupta o esquemas alterados (retorno de error sin modificar archivo).

3. **Pruebas de Integración de la CLI (`tests/test_cli.py`)**:
   - Ejecución de subcomandos (`add`, `done`, `list`, `delete`) mediante invocación directa a la CLI capturando `stdout`, `stderr` y códigos de salida (`sys.exit`).
   - Verificación de mensajes en español exactos.
   - Manejo de argumentos faltantes o comandos no reconocidos.

---

### Matriz de Trazabilidad de Requisitos

| Requisito Funcional / Casos Límite                 | Módulo Responsable                | Función / Componente Clave        | Archivo de Test                  |
|:---------------------------------------------------|:----------------------------------|:----------------------------------|:---------------------------------|
| **RF-1.1**: Crear hábito válido                    | `core.py`, `storage.py`, `cli.py` | `Habit.create()`, `save_habits()` | `test_core.py`, `test_cli.py`    |
| **RF-1.2**: Rechazar duplicados (case-insensitive) | `core.py`                         | `is_duplicate_name()`             | `test_core.py`, `test_cli.py`    |
| **RF-1.3**: Nombre vacío / >50 chars               | `core.py`                         | `validate_habit_name()`           | `test_core.py`, `test_cli.py`    |
| **RF-1.4**: Rechazar caracteres no ASCII           | `core.py`                         | `validate_habit_name()`           | `test_core.py`, `test_cli.py`    |
| **RF-1.5**: Falta argumento `<nombre>` en `add`    | `cli.py`                          | `parse_args()`                    | `test_cli.py`                    |
| **RF-2.1**: Marcar completado hoy                  | `core.py`, `cli.py`               | `mark_completed()`                | `test_core.py`, `test_cli.py`    |
| **RF-2.2**: Marcar completado ayer                 | `core.py`, `cli.py`               | `mark_completed(yesterday=True)`  | `test_core.py`, `test_cli.py`    |
| **RF-2.3**: Marcar antes de creación               | `core.py`                         | `validate_completion_date()`      | `test_core.py`, `test_cli.py`    |
| **RF-2.4**: Marcar ayer tras hoy                   | `core.py`                         | `validate_completion_date()`      | `test_core.py`, `test_cli.py`    |
| **RF-2.5**: Marcar hábito inexistente              | `cli.py`, `storage.py`            | `get_habit_by_name()`             | `test_cli.py`                    |
| **RF-2.6**: Idempotencia al marcar                 | `core.py`                         | `mark_completed()`                | `test_core.py`, `test_cli.py`    |
| **RF-2.7**: Falta argumento `<nombre>` en `done`   | `cli.py`                          | `parse_args()`                    | `test_cli.py`                    |
| **RF-3.1**: Tabla formateada y ordenada            | `cli.py`                          | `format_habits_table()`           | `test_cli.py`                    |
| **RF-3.2**: Algoritmo de cálculo de racha          | `core.py`                         | `calculate_streak()`              | `test_core.py`                   |
| **RF-3.3**: Listado sin hábitos                    | `cli.py`                          | `cmd_list()`                      | `test_cli.py`                    |
| **RF-4.1**: Eliminar hábito existente              | `storage.py`, `cli.py`            | `delete_habit()`                  | `test_storage.py`, `test_cli.py` |
| **RF-4.2**: Eliminar hábito inexistente            | `cli.py`                          | `cmd_delete()`                    | `test_cli.py`                    |
| **RF-4.3**: Falta argumento `<nombre>` en `delete` | `cli.py`                          | `parse_args()`                    | `test_cli.py`                    |
| **RF-5.1**: Ubicación `habits_data.json`           | `storage.py`                      | `get_storage_path()`              | `test_storage.py`                |
| **RF-5.2**: Esquema JSON estricto                  | `storage.py`                      | `validate_schema()`               | `test_storage.py`                |
| **RF-5.3**: Creación automática JSON vacío         | `storage.py`                      | `load_habits()`                   | `test_storage.py`                |
| **RF-5.4**: Manejo de JSON corrupto                | `storage.py`                      | `load_habits()`                   | `test_storage.py`                |
| **RF-5.5**: Escritura atómica                      | `storage.py`                      | `save_habits_atomically()`        | `test_storage.py`                |
| **RNF-1**: Stdlib + pytest exclusivamente          | *Todo el proyecto*                | N/A                               | Auditoría de código / tests      |
| **RNF-2**: Mensajes de usuario en español          | `cli.py`                          | Formato de mensajes               | `test_cli.py`                    |
| **RNF-3**: Identificadores y tests en inglés       | *Todo el código*                  | N/A                               | Respetado en desarrollo          |
| **RNF-4**: `.gitignore` incluye JSON               | `.gitignore`                      | Configuración Git                 | `test_storage.py`                |
