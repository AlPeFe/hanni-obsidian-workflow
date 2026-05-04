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
    │   ├── PROYECTO_1_PENDING         ← Fichero de estado (cambia a _COMPLETED)
    │   ├── notes/                     ← Notas y anotaciones generadas
    │   └── media/                     ← Medios del proyecto
    ├── PROYECTO_2/
    │   ├── PROYECTO_2.canvas
    │   ├── PROYECTO_2_COMPLETED
    │   └── notes/
    └── ...
```

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

Buscar ficheros cuyo nombre termina en `_PENDING` (sin extensión, o `.md`) dentro de cada subcarpeta de `HANNI/WORKFLOW/`:

```bash
find HANNI/WORKFLOW -name "*_PENDING*" -type f
```

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

Tipos comunes de ficheros referenciados:
- `agents/agent_name.md` — Perfil del agente (rol, instrucciones, restricciones)
- `context/proyecto_context.md` — Contexto del proyecto
- `credentials/service_creds.md` — Credenciales (leer con cuidado)
- `references/*.md` — Documentación de referencia

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

Enviar notificación Telegram ANTES de cualquier acción:

```python
def notify_telegram_human_required(node, project_name):
    message = f"""🔴 HANNI WORKFLOW — Intervención Humana Requerida

📁 Proyecto: {project_name}
📌 Nodo: {node.get('text', node['id'])}

Por favor, revisa y confirma antes de continuar."""
    send_telegram(message)
    # DETENER procesamiento de este nodo hasta confirmación
```

### Al completar un nodo

1. Cambiar `color` del nodo a `"6"` (azul/completado) en el JSON del canvas
2. Añadir anotación si corresponde (ver sección Anotaciones)
3. Ejecutar todos los nodos naranja (siempre ejecutar)
4. Guardar el canvas actualizado
5. Verificar si el proyecto está completo

```python
def mark_node_completed(canvas, node_id, annotation=None):
    for node in canvas['nodes']:
        if node['id'] == node_id:
            node['color'] = '6'
            break
    
    if annotation:
        add_annotation(canvas, node_id, annotation)
    
    save_canvas(canvas)
```

---

## Anotaciones

El LLM puede añadir anotaciones a nodos completados. Se crean como ficheros `.md` dentro de `HANNI/WORKFLOW/PROYECTO_X/notes/` y se referencian desde el canvas como nodos tipo `file`.

### Crear una anotación

```python
def add_annotation(canvas, related_node_id, content, project_path):
    # 1. Crear el fichero de nota
    note_filename = f"notes/annotation_{related_node_id}_{timestamp()}.md"
    note_path = f"{project_path}/{note_filename}"
    
    note_content = f"""# Anotación — {timestamp()}

{content}

---
*Generado automáticamente por HANNI Workflow*
"""
    write_file(note_path, note_content)
    
    # 2. Añadir nodo tipo file al canvas referenciando la nota
    new_node = {
        "id": generate_id(),
        "type": "file",
        "file": note_path,
        "x": related_node_x + 300,
        "y": related_node_y,
        "width": 400,
        "height": 200,
        "color": "5"  # cian para distinguir anotaciones
    }
    canvas['nodes'].append(new_node)
    
    # 3. Añadir edge desde el nodo original a la anotación
    canvas['edges'].append({
        "id": generate_id(),
        "fromNode": related_node_id,
        "toNode": new_node['id']
    })
```

**Ejemplos de anotaciones útiles:**
- Puerto donde está corriendo un servicio: `"Servicio desplegado en puerto 3001 (http://localhost:3001)"`
- URL de un PR creado: `"PR #42 creado: https://github.com/org/repo/pull/42"`
- Resultado de un comando: `"Build exitoso — artefacto en dist/app.tar.gz"`
- Credenciales temporales generadas
- Errores encontrados y cómo se resolvieron

---

## Gestión del Estado del Proyecto

### Verificar si el proyecto está completo

```python
def is_project_complete(nodes):
    for node in nodes:
        color = node.get('color', '')
        # Ignorar nodos de contexto (amarillo) y siempre-ejecutar (naranja)
        if color in ('3', '2'):
            continue
        # Si queda algún nodo no completado (no azul/violeta)
        if color != '6':
            return False
    return True
```

### Marcar proyecto como COMPLETED

```python
def complete_project(project_path, project_name):
    # Renombrar o crear fichero _COMPLETED, eliminar _PENDING
    pending_file = f"{project_path}/{project_name}_PENDING"
    completed_file = f"{project_path}/{project_name}_COMPLETED"
    
    os.rename(pending_file, completed_file)
    
    # Notificar por Telegram
    notify_telegram_project_completed(project_name)
```

### Notificación Telegram — Proyecto Completado

```python
def notify_telegram_project_completed(project_name):
    message = f"""✅ HANNI WORKFLOW — Proyecto Completado

📁 Proyecto: {project_name}
🕐 Completado: {datetime.now().strftime('%Y-%m-%d %H:%M')}

Todas las tareas han sido procesadas correctamente."""
    send_telegram(message)
```

---

## Creación de Nuevos Proyectos

Cuando se pide planificar un proyecto nuevo, seguir SIEMPRE esta estructura:

### 1. Crear la estructura de carpetas

```bash
HANNI/WORKFLOW/NOMBRE_PROYECTO/
├── NOMBRE_PROYECTO.canvas
├── NOMBRE_PROYECTO_PENDING    ← fichero vacío o con descripción
└── notes/
```

### 2. Generar el canvas inicial

El canvas debe tener:
- Un nodo de **contexto** (amarillo) con la descripción del proyecto
- Nodos de tareas en orden lógico con flechas de dependencia
- Al menos un nodo naranja (siempre ejecutar) si aplica (PR, notificación, registro)
- Nodos rojos para pasos que requieren intervención humana

```python
def create_project_canvas(project_name, tasks):
    """
    tasks: lista de dicts con {name, description, color, dependencies: [idx], always_run: bool}
    """
    canvas = {"nodes": [], "edges": []}
    
    # Nodo de contexto del proyecto
    context_node = {
        "id": "ctx_0",
        "type": "text",
        "text": f"# {project_name}\n\nContexto del proyecto",
        "color": "3",  # amarillo
        "x": -200, "y": 0, "width": 300, "height": 100
    }
    canvas['nodes'].append(context_node)
    
    # Añadir tareas y dependencias
    node_ids = {}
    for i, task in enumerate(tasks):
        node_id = f"task_{i}"
        node = {
            "id": node_id,
            "type": "text",
            "text": f"**{task['name']}**\n{task.get('description', '')}",
            "color": "2" if task.get('always_run') else "4",  # naranja o verde
            "x": i * 350, "y": 200,
            "width": 300, "height": 100
        }
        canvas['nodes'].append(node)
        node_ids[i] = node_id
        
        for dep_idx in task.get('dependencies', []):
            canvas['edges'].append({
                "id": f"edge_{dep_idx}_{i}",
                "fromNode": node_ids[dep_idx],
                "toNode": node_id
            })
    
    return canvas
```

### 3. Añadir referencias a ficheros de la bóveda

Si el proyecto necesita un agente o contexto externo, añadir nodo tipo `file`:

```python
reference_node = {
    "id": "ref_agent",
    "type": "file",
    "file": "agents/mi_agente.md",
    "color": "3",  # amarillo - contexto
    "x": -200, "y": 200,
    "width": 250, "height": 80
}
canvas['nodes'].append(reference_node)
```

---

## Adición de Contenido al Canvas

### Añadir enlace web

```python
def add_web_link(canvas, url, title, x, y):
    node = {
        "id": generate_id(),
        "type": "link",
        "url": url,
        "x": x, "y": y,
        "width": 400, "height": 200
    }
    canvas['nodes'].append(node)
```

### Añadir medio desde la bóveda

Los medios deben guardarse en `HANNI/WORKFLOW/PROYECTO_X/media/` y referenciarse como nodo `file`:

```python
def add_media(canvas, media_path, x, y):
    # media_path relativo a la raíz de la bóveda
    node = {
        "id": generate_id(),
        "type": "file",
        "file": media_path,
        "x": x, "y": y,
        "width": 400, "height": 300
    }
    canvas['nodes'].append(node)
```

---

## Integración con Telegram

Implementar `send_telegram()` usando la API de Telegram Bot:

```python
import requests

def send_telegram(message, chat_id=None, bot_token=None):
    """
    bot_token y chat_id pueden leerse de un nodo de credenciales
    referenciado en el canvas (tipo file, color amarillo).
    """
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    payload = {
        "chat_id": chat_id,
        "text": message,
        "parse_mode": "Markdown"
    }
    response = requests.post(url, json=payload)
    return response.json()
```

Las credenciales de Telegram se leen desde el fichero referenciado en el canvas (nodo amarillo de contexto con `file: credentials/telegram.md`).

---

## Polling Automático — Cron Job cada 60 segundos

El LLM debe instalar un cron job que escanee `HANNI/WORKFLOW/` cada 60 segundos, detecte proyectos PENDING y actúe sobre ellos automáticamente.

### Script del worker: `hanni_worker.py`

Crear en la raíz de la bóveda o en un directorio de scripts accesible:

```python
#!/usr/bin/env python3
"""
HANNI Workflow Worker — escanea HANNI/WORKFLOW/ cada 60 segundos
y procesa automáticamente los proyectos PENDING.
"""

import json
import os
import sys
import time
import logging
from datetime import datetime
from pathlib import Path

# ── Configuración ──────────────────────────────────────────────
VAULT_ROOT    = Path(os.environ.get("HANNI_VAULT", "/ruta/a/tu/boveda"))
WORKFLOW_DIR  = VAULT_ROOT / "HANNI" / "WORKFLOW"
LOG_FILE      = VAULT_ROOT / "HANNI" / "hanni_worker.log"
LOCK_FILE     = VAULT_ROOT / "HANNI" / ".hanni_worker.lock"
POLL_INTERVAL = 60  # segundos

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler(sys.stdout),
    ],
)
log = logging.getLogger("hanni")

# ── Colores Obsidian → Estado ──────────────────────────────────
COLOR_BACKLOG    = ""
COLOR_READY      = "4"   # verde
COLOR_HUMAN      = "1"   # rojo
COLOR_DONE       = "6"   # azul/violeta
COLOR_CONTEXT    = "3"   # amarillo
COLOR_ALWAYS_RUN = "2"   # naranja


def find_pending_projects():
    """Devuelve lista de (project_dir, project_name) con fichero _PENDING."""
    pending = []
    if not WORKFLOW_DIR.exists():
        return pending
    for project_dir in sorted(WORKFLOW_DIR.iterdir()):
        if not project_dir.is_dir():
            continue
        for f in project_dir.iterdir():
            if f.name.endswith("_PENDING"):
                project_name = f.name.replace("_PENDING", "")
                pending.append((project_dir, project_name))
                break
    return pending


def load_canvas(project_dir, project_name):
    canvas_path = project_dir / f"{project_name}.canvas"
    if not canvas_path.exists():
        log.error(f"Canvas no encontrado: {canvas_path}")
        return None
    with open(canvas_path, "r", encoding="utf-8") as fh:
        return json.load(fh)


def save_canvas(canvas, project_dir, project_name):
    canvas_path = project_dir / f"{project_name}.canvas"
    with open(canvas_path, "w", encoding="utf-8") as fh:
        json.dump(canvas, fh, ensure_ascii=False, indent=2)
    log.info(f"Canvas guardado: {canvas_path}")


def get_processable_nodes(nodes, edges):
    completed_ids = {n["id"] for n in nodes if n.get("color") == COLOR_DONE}
    result = []
    for node in nodes:
        if node.get("color") != COLOR_READY:
            continue
        preds = [e["fromNode"] for e in edges if e["toNode"] == node["id"]]
        if all(p in completed_ids for p in preds):
            result.append(node)
    return result


def get_context_nodes(nodes):
    return [n for n in nodes if n.get("color") == COLOR_CONTEXT]


def get_always_run_nodes(nodes):
    return [n for n in nodes if n.get("color") == COLOR_ALWAYS_RUN]


def is_project_complete(nodes):
    skip = {COLOR_CONTEXT, COLOR_ALWAYS_RUN}
    for node in nodes:
        color = node.get("color", "")
        if color in skip:
            continue
        if color != COLOR_DONE:
            return False
    return True


def read_file_node(node):
    """Lee el contenido de un nodo tipo 'file' desde la bóveda."""
    rel_path = node.get("file", "")
    abs_path = VAULT_ROOT / rel_path
    if abs_path.exists():
        return abs_path.read_text(encoding="utf-8")
    log.warning(f"Fichero de nodo no encontrado: {abs_path}")
    return ""


def send_telegram(message):
    """Lee credenciales de HANNI/credentials/telegram.json y envía mensaje."""
    import requests
    creds_path = VAULT_ROOT / "HANNI" / "credentials" / "telegram.json"
    if not creds_path.exists():
        log.warning("No se encontraron credenciales de Telegram.")
        return
    creds = json.loads(creds_path.read_text())
    bot_token = creds["bot_token"]
    chat_id   = creds["chat_id"]
    url = f"https://api.telegram.org/bot{bot_token}/sendMessage"
    try:
        r = requests.post(url, json={
            "chat_id": chat_id,
            "text": message,
            "parse_mode": "Markdown"
        }, timeout=10)
        r.raise_for_status()
        log.info("Telegram: mensaje enviado.")
    except Exception as e:
        log.error(f"Telegram error: {e}")


def add_annotation(canvas, project_dir, related_node_id, content):
    """Crea un .md de anotación y lo añade al canvas como nodo file."""
    import uuid
    notes_dir = project_dir / "notes"
    notes_dir.mkdir(exist_ok=True)
    ts = datetime.now().strftime("%Y%m%d_%H%M%S")
    note_file = notes_dir / f"annotation_{related_node_id[:8]}_{ts}.md"
    note_file.write_text(
        f"# Anotación — {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n"
        f"{content}\n\n---\n*HANNI Worker*\n",
        encoding="utf-8"
    )
    # Posición: junto al nodo original
    orig = next((n for n in canvas["nodes"] if n["id"] == related_node_id), {})
    new_node = {
        "id": str(uuid.uuid4())[:8],
        "type": "file",
        "file": str(note_file.relative_to(VAULT_ROOT)),
        "x": orig.get("x", 0) + orig.get("width", 250) + 40,
        "y": orig.get("y", 0),
        "width": 400, "height": 200,
        "color": "5"
    }
    canvas["nodes"].append(new_node)
    canvas["edges"].append({
        "id": str(uuid.uuid4())[:8],
        "fromNode": related_node_id,
        "toNode": new_node["id"]
    })
    log.info(f"Anotación creada: {note_file}")


def mark_completed(canvas, node_id):
    for n in canvas["nodes"]:
        if n["id"] == node_id:
            n["color"] = COLOR_DONE
            break


def complete_project(project_dir, project_name):
    pending = project_dir / f"{project_name}_PENDING"
    completed = project_dir / f"{project_name}_COMPLETED"
    if pending.exists():
        pending.rename(completed)
    msg = (
        f"✅ *HANNI WORKFLOW — Proyecto Completado*\n\n"
        f"📁 Proyecto: `{project_name}`\n"
        f"🕐 {datetime.now().strftime('%Y-%m-%d %H:%M')}\n\n"
        f"Todas las tareas han sido procesadas correctamente."
    )
    send_telegram(msg)
    log.info(f"Proyecto completado: {project_name}")


def process_project(project_dir, project_name):
    log.info(f"── Procesando: {project_name} ──")
    canvas = load_canvas(project_dir, project_name)
    if canvas is None:
        return

    nodes = canvas.get("nodes", [])
    edges = canvas.get("edges", [])

    # 1. Leer nodos de contexto primero
    for ctx in get_context_nodes(nodes):
        if ctx.get("type") == "file":
            content = read_file_node(ctx)
            log.info(f"Contexto leído: {ctx.get('file', ctx['id'])}")

    # 2. Obtener nodos procesables
    processable = get_processable_nodes(nodes, edges)
    if not processable:
        log.info(f"{project_name}: sin nodos procesables en este ciclo.")
        return

    changed = False
    for node in processable:
        node_text = node.get("text", node["id"])
        color     = node.get("color", "")

        # Nodo rojo → intervención humana, no procesar
        if color == COLOR_HUMAN:
            msg = (
                f"🔴 *HANNI — Intervención Humana*\n\n"
                f"📁 `{project_name}`\n"
                f"📌 `{node_text[:200]}`\n\n"
                f"Revisa y actualiza el nodo a verde cuando esté listo."
            )
            send_telegram(msg)
            log.warning(f"Nodo rojo detectado, esperando intervención: {node_text[:60]}")
            continue

        # ──── AQUÍ el LLM ejecuta la tarea real del nodo ────
        # En producción: llamar al agente/herramienta correspondiente.
        # Este script marca el nodo y delega al orquestador LLM.
        log.info(f"Ejecutando nodo: {node_text[:80]}")
        annotation_content = f"Procesado por HANNI Worker en {datetime.now().isoformat()}"

        # Marcar completado
        mark_completed(canvas, node["id"])
        add_annotation(canvas, project_dir, node["id"], annotation_content)
        changed = True

        # Ejecutar nodos naranja tras cada tarea
        for always_node in get_always_run_nodes(nodes):
            log.info(f"Ejecutando siempre: {always_node.get('text','')[:60]}")
            # Aquí va la lógica del nodo naranja (PR, registro, etc.)

    if changed:
        save_canvas(canvas, project_dir, project_name)

    # 3. Comprobar si el proyecto está completo
    updated_nodes = canvas.get("nodes", [])
    if is_project_complete(updated_nodes):
        complete_project(project_dir, project_name)


def scan_and_process():
    pending = find_pending_projects()
    log.info(f"Proyectos PENDING encontrados: {len(pending)}")
    for project_dir, project_name in pending:
        try:
            process_project(project_dir, project_name)
        except Exception as e:
            log.error(f"Error procesando {project_name}: {e}", exc_info=True)


def main():
    log.info("HANNI Worker iniciado.")
    while True:
        scan_and_process()
        time.sleep(POLL_INTERVAL)


if __name__ == "__main__":
    main()
```

---

### Instalación del Cron Job

Cuando el usuario pide activar el worker o iniciar el sistema automático, ejecutar los siguientes pasos:

#### Opción A — Cron clásico (Linux/macOS)

```bash
# 1. Copiar el worker a un directorio estable
cp hanni_worker.py /ruta/a/tu/boveda/HANNI/hanni_worker.py
chmod +x /ruta/a/tu/boveda/HANNI/hanni_worker.py

# 2. Añadir entrada al crontab (cada minuto)
(crontab -l 2>/dev/null; echo "* * * * * /usr/bin/python3 /ruta/a/tu/boveda/HANNI/hanni_worker.py >> /ruta/a/tu/boveda/HANNI/hanni_cron.log 2>&1") | crontab -

# 3. Verificar que quedó instalado
crontab -l | grep hanni_worker
```

> ⚠️ `* * * * *` ejecuta cada minuto (mínimo de cron). El script hace `time.sleep(60)` internamente si se lanza como daemon. Para modo cron puro, quitar el `while True` del `main()` y dejar solo `scan_and_process()`.

#### Opción B — Systemd service (recomendado en Linux)

```ini
# /etc/systemd/system/hanni-worker.service
[Unit]
Description=HANNI Obsidian Workflow Worker
After=network.target

[Service]
Type=simple
User=TU_USUARIO
WorkingDirectory=/ruta/a/tu/boveda/HANNI
Environment=HANNI_VAULT=/ruta/a/tu/boveda
ExecStart=/usr/bin/python3 /ruta/a/tu/boveda/HANNI/hanni_worker.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
# Instalar y activar
sudo systemctl daemon-reload
sudo systemctl enable hanni-worker
sudo systemctl start hanni-worker

# Comprobar estado
sudo systemctl status hanni-worker
journalctl -u hanni-worker -f
```

#### Opción C — launchd (macOS)

```xml
<!-- ~/Library/LaunchAgents/com.hanni.worker.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>             <string>com.hanni.worker</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/python3</string>
    <string>/ruta/a/tu/boveda/HANNI/hanni_worker.py</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>HANNI_VAULT</key>    <string>/ruta/a/tu/boveda</string>
  </dict>
  <key>StartInterval</key>    <integer>60</integer>
  <key>StandardOutPath</key>  <string>/ruta/a/tu/boveda/HANNI/hanni_worker.log</string>
  <key>StandardErrorPath</key><string>/ruta/a/tu/boveda/HANNI/hanni_worker_err.log</string>
  <key>RunAtLoad</key>        <true/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.hanni.worker.plist
launchctl start com.hanni.worker
```

---

### Comandos de gestión del worker

```bash
# Ver log en tiempo real
tail -f /ruta/boveda/HANNI/hanni_worker.log

# Detener (systemd)
sudo systemctl stop hanni-worker

# Reiniciar tras cambios en el worker
sudo systemctl restart hanni-worker

# Desinstalar cron
crontab -l | grep -v hanni_worker | crontab -
```

---

### Notificación de resumen periódico (opcional)

El worker puede enviar por Telegram un resumen cada vez que arranca el ciclo con proyectos pendientes:

```python
def notify_scan_summary(pending_projects):
    if not pending_projects:
        return  # No notificar si no hay nada pendiente
    lines = "\n".join(f"  • `{name}`" for _, name in pending_projects)
    msg = (
        f"🔍 *HANNI — Ciclo de revisión*\n\n"
        f"📋 Proyectos PENDING: {len(pending_projects)}\n"
        f"{lines}\n\n"
        f"🕐 {datetime.now().strftime('%H:%M:%S')}"
    )
    send_telegram(msg)
```

---

## Checklist de Procesamiento

Antes de procesar cualquier workflow, verificar en orden:

- [ ] ¿Existe el fichero `_PENDING`? → Continuar
- [ ] ¿Existe el `.canvas`? → Leer y parsear
- [ ] ¿Hay nodos amarillos? → **Leer primero**
- [ ] ¿Hay nodos de tipo `file` con perfiles de agente? → **Leer antes de actuar**
- [ ] ¿Hay nodos rojos en el camino? → **Notificar Telegram y detener**
- [ ] Procesar nodos verdes en orden topológico
- [ ] Al completar cada nodo → ejecutar todos los nodos naranja
- [ ] Actualizar el canvas con el nuevo estado (azul) tras cada tarea
- [ ] ¿Todos los nodos (excepto amarillo y naranja) están en azul? → Completar proyecto
- [ ] Renombrar `_PENDING` → `_COMPLETED`
- [ ] Notificar Telegram que el proyecto finalizó

---

## Notas Importantes

1. **Guardar siempre** el canvas tras cada cambio de estado — no acumular cambios.
2. **Los nodos naranja** se ejecutan tras CADA tarea completada, no solo al final.
3. **Los nodos amarillo** nunca cambian de color — son solo contexto.
4. **Las anotaciones** siempre van dentro de la carpeta del proyecto, nunca en la raíz de la bóveda.
5. **Respetar la estructura de carpetas** — cada proyecto es una subcarpeta independiente.
6. El LLM puede **añadir nuevos nodos** al canvas si se le pide planificar o si identifica pasos adicionales necesarios.
7. Siempre **avisar por Telegram** al completar un proyecto, sin excepción.
