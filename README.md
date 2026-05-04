# HANNI Obsidian Workflow Skill

Sistema de gestión de proyectos inteligente basado en **Obsidian Canvas** y un agente IA (Hanni) que procesa tareas de forma autónoma con polling automático.

---

## ¿Qué es?

HANNI es un workflow donde los proyectos viven como **canvas files** (`.canvas`) dentro de Obsidian. Cada canvas tiene nodos coloreados que representan estados de tarea. Un agente IA hace polling continuo, lee los canvas, procesa nodos verdes (Ready), ejecuta nodos naranja (Always Run), y notifica por Telegram cuando encuentra un nodo rojo (Human Intervention).

El resultado: gestión de proyectos semiautomatizada donde la IA trabaja por ti, y solo interviene cuando algo requiere un humano.

---

## Estructura del repo

```
hanni-obsidian-workflow/
├── SKILL.md                           ← Skill principal para Hermes Agent
├── README.md                          ← Este archivo
└── references/
    ├── hanni-cron-job-setup.md       ← Configuración del cron job de polling
    └── push-skill-to-github.md        ← Cómo publicar skills en GitHub
```

---

## Sistema de colores de nodos

Cada nodo en el canvas tiene un color que determina su comportamiento:

| Color | Estado | Significado |
|-------|--------|-------------|
| Sin color / blanco | **Backlog** | Pendiente, no procesable aún |
| Verde | **Ready** | La IA lo procesa ahora |
| Rojo | **Human Intervention** | **DETENER** — notificar por Telegram y esperar |
| Azul / Violeta | **Completado** | Procesado correctamente |
| Amarillo | **Contexto** | Léelo antes de procesar tareas relacionadas |
| Naranja | **Always Run** | Se ejecuta siempre al completar cualquier tarea |

### Mapeo interno de colores (Obsidian Canvas → HANNI)

| Valor `color` en JSON | Color visual |
|----------------------|--------------|
| `""` (vacío) | Sin color / Backlog |
| `"4"` | Verde / Ready |
| `"1"` | Rojo / Human Intervention |
| `"6"` | Azul-Violeta / Completado |
| `"3"` | Amarillo / Contexto |
| `"2"` | Naranja / Always Run |

---

## Estructura de un proyecto en la bóveda

```
HANNI/
└── WORKFLOW/
    ├── MI_PROYECTO/
    │   ├── MI_PROYECTO.canvas          ← Canvas con nodos y dependencias
    │   ├── MI_PROYECTO_PENDING.md      ← Estado: proyecto activo
    │   ├── MI_PROYECTO_COMPLETED.md    ← Estado: proyecto terminado
    │   ├── notes/                      ← Anotaciones generadas por la IA
    │   └── media/                      ← Medios del proyecto
    └── OTRO_PROYECTO/
        └── ...
```

**Importante:** los ficheros `_PENDING.md` y `_COMPLETED.md` **deben** tener extensión `.md` para que Obsidian los indexe en el explorador.

---

## Dependencias entre nodos

Las flechas entre nodos representan dependencias. Un nodo solo se procesa cuando **todos sus predecesores están en azul** (completado). La IA respeta siempre el orden topológico.

---

## Flujo de procesamiento

```
1. Cron job (cada 1 minuto) despierta al agente
2. Escanea HANNI/WORKFLOW/ buscando *_PENDING.md
3. Para cada proyecto pendiente:
   a. Lee el .canvas (JSON)
   b. Identifica nodos verdes (Ready) con dependencias satisfechas
   c. Si hay nodo rojo → DETENER y notificar por Telegram
   d. Procesa nodos verdes en orden
   e. Ejecuta nodos naranja (Always Run)
   f. Marca completados en azul
   g. Si todo listo → renombra _PENDING.md → _COMPLETED.md
4. Sin trabajo → "OK - sin trabajo pendiente"
```

---

## Instalación

### 1. Descarga el skill

Clona este repo:

```bash
git clone https://github.com/AlPeFe/hanni-obsidian-workflow.git
```

### 2. Crea la estructura en tu bóveda Obsidian

Crea la carpeta `HANNI/WORKFLOW/` en tu bóveda. Dentro, cada proyecto será una subcarpeta con su `.canvas` y `_PENDING.md`.

### 3. Configura el cron job

Usa el agent tool para crear el polling job:

```python
cronjob(
  action="create",
  name="HANNI Workflow Polling",
  prompt="""Eres Hanni. Lee tu skill `hanni-obsidian-workflow` para saber cómo funcionar.
1. Escanea tu bóveda Obsidian en HANNI/WORKFLOW/ buscando proyectos PENDING.
2. Para cada proyecto: lee canvas, procesa nodos verdes, ejecuta naranjas.
3. Si no hay trabajo: responde 'OK - sin trabajo pendiente'.
4. Nodos rojos = DETENER y notificar por Telegram.""",
  schedule="every 1m",
  skills=["hanni-obsidian-workflow"]
)
```

> **Formato:** usa `every Nm` — no `Ns` (ej: `every 1m`, no `60s`).

### 4. Crea tu primer proyecto

1. Crea la carpeta: `HANNI/WORKFLOW/MI_PROYECTO/`
2. Crea `MI_PROYECTO.canvas` con nodos coloreados
3. Crea `MI_PROYECTO_PENDING.md` (vacío o con notas)
4. La IA lo detectará en el siguiente polling

---

## Formato del Canvas (JSON)

El `.canvas` de Obsidian es JSON. Estructura de un nodo:

```json
{
  "id": "nodo-unico-123",
  "type": "text",        // "text" | "file" | "link" | "group"
  "text": "Contenido del nodo",
  "color": "4",          // ""=backlog, "1"=rojo, "2"=naranja, "3"=amarillo, "4"=verde, "6"=azul
  "x": 100, "y": 200,
  "width": 250,
  "height": 80,
  "file": "ruta/al/fichero.md"   // solo si type="file"
}
```

Las aristas (flechas) son:

```json
{
  "id": "edge-123",
  "fromNode": "nodo-origen",
  "toNode": "nodo-destino"
}
```

---

## Añadir este skill a tu Hermes Agent

```python
skill_manage(
  action="create",
  name="hanni-obsidian-workflow",
  category="knowledge",
  content=<contenido de SKILL.md>
)
```

---

## Recursos

- **Skill principal:** `SKILL.md`
- **Configuración del cron job:** `references/hanni-cron-job-setup.md`
- **Cómo publicar skills:** `references/push-skill-to-github.md`

---

## Autor

Alex / [AlPeFe](https://github.com/AlPeFe) — Sistema HANNI integrado con Hermes Agent.
