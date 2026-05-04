# Cron Job Setup — HANNI Workflow Polling

Configuración del job de polling que despierta al agente cada minuto para procesar proyectos pendientes.

## Comando de creación

```python
cronjob(
  action="create",
  name="HANNI Workflow Polling",
  prompt="""Eres Hanni. Lee tu skill `hanni-obsidian-workflow` para saber cómo funciona el sistema.

1. Escanea tu bóveda Obsidian en `/mnt/c/Users/alexlocal/Documents/obsidian/HANNI/WORKFLOW/` buscando proyectos PENDING.
2. Para cada proyecto: lee canvas, procesa nodos verdes, ejecuta naranjas, renombra a _COMPLETED si completo.
3. Si no hay trabajo: responde "OK - sin trabajo pendiente".
4. Nodos rojos (rojo) = DETENER y notificar por Telegram.""",
  schedule="every 1m",
  skills=["hanni-obsidian-workflow"]
)
```

## Formato del schedule

| Input | Resultado |
|-------|-----------|
| `every 1m` | ✅ OK — cada 1 minuto |
| `every 60s` | ❌ Invalid duration — rechazado |
| `every 2h` | ✅ OK — cada 2 horas |

## Ruta de la bóveda

La bóveda Obsidian está en:
- **WSL path:** `/mnt/c/Users/alexlocal/Documents/obsidian/`
- **Windows path:** `C:\Users\alexlocal\Documents\obsidian\`

## Verificación

Ver jobs activos:
```python
cronjob(action="list")
```

Detener un job:
```python
cronjob(action="remove", job_id="<job_id>")
```

## Notas

- El job se ejecuta en local (sin delivery externo)
- Skills cargadas automáticamente en cada ejecución
- El agente responde "OK - sin trabajo pendiente" cuando no hay nada que hacer
