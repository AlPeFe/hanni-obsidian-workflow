---
name: hanni-obsidian-workflow
description: >
  Use this skill whenever the user wants to interact with their Obsidian HANNI/WORKFLOW system.
  Triggers: managing projects in Obsidian canvas, reading/updating workflow nodes, processing tasks, planning projects, marking nodes as completed, sending Telegram notifications, reading agent profiles, annotating completed tasks, creating new projects. Use this skill when the user mentions HANNI, workflow nodes, obsidian canvas projects, pending tasks, node colors, telegram notifications for tasks, or wants to plan/execute a project following the HANNI structure. Also triggers for "revisar workflows", "procesar tareas", "planificar proyecto", "actualizar nodo", or any reference to the canvas-based project system.
---

# HANNI Obsidian Workflow Skill

Sistema de gestión de proyectos basado en Obsidian Canvas. Los proyectos viven en `HANNI/WORKFLOW/` dentro de la bóveda de Obsidian como archivos `.canvas` con un sistema de nodos coloreados y dependencias.

---

## Estructura de la Bóveda

```
HANNI/
└── WORKFLOW/
    ├── PROYECTO_1/
    │   ├── PROYECTO_1.canvas          ← Canvas principal del proyecto
    │   ├── PROYECTO_1_PENDING.md      ← Fichero de estado (cambia a _COMPLETED.md)
    │   ├── notes/                     ← Notas y anotaciones generadas
    │   └── media/                     ← Medios del proyecto
    ├── PROYECTO_2/
    │   ├── PROYECTO_2.canvas
    │   ├── PROYECTO_2_COMPLETED.md
    │   └── notes/
    └── ...
```

**IMPORTANTE**: Los ficheros de estado (`_PENDING` y `_COMPLETED`) deben tener extensión `.md` para que Obsidian los indexe y los muestre en el explorador de archivos. Sin la extensión, Obsidian los oculta.

**Regla crítica**: Todas las anotaciones, medios y notas nuevas se crean SIEMPRE dentro de la carpeta del proyecto correspondiente (`HANNI/WORKFLOW/PROYECTO_X/notes/` o `media/`), y se referencian desde el canvas mediante el sistema de bóveda de Obsidian.

---

## Sistema de Nodos — Colores y Significado

| Color    | Estado           | Comportamiento |
|----------|------------------|----------------|
| Sin color / blanco | **Backlog / Pendiente** | Aún no procesable |
| 🟢 Verde  | **Ready**        | Puede/debe procesarse ahora |
| 🔴 Rojo   | **Human Intervention** | DETENER — notificar por Telegram antes de procesar |
| 🔵 Azul   | **Completado**   | Procesado correctamente |
| 🟡 Amarillo | **Contexto**   | Solo aporta contexto, no se procesa; léelo antes de la tarea relacionada |
| 🟠 Naranja | **Siempre Ejecutar** | Se ejecuta siempre al completar cualquier tarea, independientemente de dependencias (ej: crear PR, notificar, registrar) |

---

## Dependencias entre Nodos

- Las **flechas** entre nodos representan dependencias: el nodo destino solo puede procesarse cuando el origen está en 🔵 Azul (completado).
- Un nodo puede tener **subnodos** (nodos anidados dentro de grupos en el canvas).
- Los subnodos heredan las dependencias del nodo padre y deben completarse antes de marcar el padre como completado.
- Procesar siempre en orden topológico respetando las flechas.

---

## Flujo de Trabajo Principal

### 1. Leer el estado del workflow

```python
# Leer el fichero .canvas (es JSON)
import json

canvas_path = "HANNI/WORKFLOW/PROYECTO_X/PROYECTO_X.canvas"
with open(canvas_path, 'r', encoding='utf-8') as f:
    canvas = json.load(f)

nodes = canvas.get('nodes', [])
edges = canvas.get('edges', [])
```

**Estructura de un nodo canvas de Obsidian:**
```json
{
  "id": "node_id_único",
  "type": "text",          // "text" | "file" | "link" | "group"
  "text": "Contenido del nodo",
  "color": "1",            // "" | "1"=rojo | "2"=naranja | "3"=amarillo | "4"=verde | "5"=cian | "6"=violeta
  "x": 0, "y": 0, "width": 250, "height": 60,
  "file": "ruta/al/fichero.md"  // si type="file"
}
```

**Tabla de colores Obsidian Canvas:**
| Valor `color` | Color visual | Estado HANNI |
|---------------|--------------|--------------|
| `""` o ausente | Sin color | Backlog |
| `"4"` | Verde | Ready |
| `"1"` | Rojo | Human Intervention |
| `"6"` | Azul / Violeta | Completado |
| `"3"` | Amarillo | Contexto |
| `"2"` | Naranja | Siempre Ejecutar |

### 2. Detectar proyectos PENDING

Buscar ficheros cuyo nombre termina en `_PENDING.md` (o `.md` para el caso de `PROYECTO_PENDING.md`) dentro de cada subcarpeta de `HANNI/WORKFLOW/`:

```bash
find HANNI/WORKFLOW -name "*_PENDING*" -type f
```

**Importante**: usar `*_PENDING*` para capturar tanto `PROYECTO_PENDING.md` como `PROYECTO_PENDING` (sin extensión, por compatibilidad con Obsidian).

Para cada proyecto PENDING:
1. Leer el canvas correspondiente
2. Identificar nodos en estado Ready (verde, `color: "4"`)
3. Verificar dependencias satisfechas (todos los predecesores en Azul/Completado)
4. Procesar en orden

### 3. Leer notas referenciadas en nodos

Los nodos de tipo `"file"` referencian ficheros de la bóveda:

```python
for node in nodes:
    if node.get('type') == 'file':
        file_path = node.get('file', '')
        # Leer el fichero para obtener contexto, perfil de agente, credenciales, etc.
        with open(f"vault_root/{file_path}", 'r') as f:
            content = f.read()
```

**SIEMPRE leer los nodos amarillos (contexto) antes de procesar cualquier tarea del mismo grupo.**

---

## Procesamiento de Tareas

### Algoritmo de procesamiento

```python
def get_processable_nodes(nodes, edges):
    completed_ids = {n['id'] for n in nodes if n.get('color') == '6'}  # azul

    processable = []
    for node in nodes:
        if node.get('color') != '4':  # solo Ready/verde
            continue
        # Verificar que todos los predecesores están completados
        predecessors = [e['fromNode'] for e in edges if e['toNode'] == node['id']]
        if all(p in completed_ids for p in predecessors):
            processable.append(node)
    return processable

def get_always_run_nodes(nodes):
    # Nodos naranja: ejecutar siempre al completar cualquier tarea
    return [n for n in nodes if n.get('color') == '2']

def requires_human_intervention(node):
    return node.get('color') == '1'  # rojo
```

### Antes de procesar un nodo rojo

Enviar notificación Telegram ANTES de cualquier acción. DETENER procesamiento hasta confirmación.

### Al completar un nodo

1. Cambiar `color` del nodo a `"6"` (azul/completado) en el JSON del canvas
2. Añadir anotación si corresponde
3. Ejecutar todos los nodos naranja (siempre ejecutar)
4. Guardar el canvas actualizado
5. Verificar si el proyecto está completo

---

## Anotaciones

El LLM puede añadir anotaciones a nodos completados. Se crean como ficheros `.md` dentro de `HANNI/WORKFLOW/PROYECTO_X/notes/` y se referencian desde el canvas como nodos tipo `file`.

---

## Gestión del Estado del Proyecto

### Marcar proyecto como COMPLETED

```python
def complete_project(project_path, project_name):
    pending_file = f"{project_path}/{project_name}_PENDING.md"
    completed_file = f"{project_path}/{project_name}_COMPLETED.md"
    os.rename(pending_file, completed_file)
    notify_telegram_project_completed(project_name)
```

**Importante**: usar siempre extensión `.md` para que Obsidian indexe los ficheros de estado.

---

## Creación de Nuevos Proyectos

Cuando se pide planificar un proyecto nuevo, seguir SIEMPRE esta estructura:

### 1. Crear la estructura de carpetas
```
HANNI/WORKFLOW/NOMBRE_PROYECTO/
├── NOMBRE_PROYECTO.canvas
├── NOMBRE_PROYECTO_PENDING.md  ← fichero de estado (SIEMPRE con extensión .md)
└── notes/
```

### 2. Generar el canvas inicial
El canvas debe tener:
- Un nodo de **contexto** (amarillo) con la descripción del proyecto
- Nodos de tareas en orden lógico con flechas de dependencia
- Al menos un nodo naranja (siempre ejecutar) si aplica
- Nodos rojos para pasos que requieren intervención humana

---

## Integración con Telegram

Implementar `send_telegram()` usando la API de Telegram Bot. Las credenciales se leen desde el fichero referenciado en el canvas (nodo amarillo de contexto con `file: credentials/telegram.md`).

---

## Polling Automático — Cron Job

### Formato de schedule
Usa el cron tool con formato `every Nm` (ej: `every 1m`). NO uses `60s` — el tool rechaza ese formato con "Invalid duration".

### 安装 el cron job
Ver `references/hanni-cron-job-setup.md` para la configuración completa del job de polling.

Repo: https://github.com/AlPeFe/hanni-obsidian-workflow
