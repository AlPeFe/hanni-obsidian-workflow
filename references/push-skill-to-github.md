# Subir un Skill a GitHub y añadirlo al Skillset

Guía para publicar un skill en GitHub y añadirlo a tu Hermes Agent local.

## Paso a paso

### 1. Localizar el skill

El skill puede venir de Windows y ser un ZIP:

```bash
python3 -c "
import zipfile
with zipfile.ZipFile('/ruta/al/archivo.skill', 'r') as z:
    z.extractall('/tmp/skill-extract/')
"
```

### 2. Crear repo en GitHub

```bash
# Público
curl -s -X POST "https://api.github.com/user/repos" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"repo-name","description":"...","private":false}'

# Privado: añadir "private":true
```

GitHub token de AlPeFe: usar variable de entorno `GITHUB_TOKEN` o el token guardado en memoria.

### 3. Push a GitHub

```bash
cd /tmp/skill-extract/hanni-workflow
git init
git config user.email "alex@polaper.com"
git config user.name "Alex"
git add .
git commit -m "Initial commit"
git branch -M main
git remote add origin "https://${GITHUB_TOKEN}@github.com/AlPeFe/repo-name.git"
git push -u origin main
```

### 4. Añadir al skillset local

```python
skill_manage(
  action="create",
  name="hanni-obsidian-workflow",
  category="knowledge",
  content=<contenido completo del SKILL.md>
)
```

El skill se guarda en:
`~/.hermes/skills/<category>/<skill-name>/SKILL.md`

### 5. Verificar

```bash
ls ~/.hermes/skills/<category>/<skill-name>/
# Debe tener: SKILL.md + references/ + templates/ + scripts/ (opcional)
```

## Pitfall conocido

- **No usar `unzip`** — no está disponible en este entorno. Usar siempre `python3 -c "import zipfile; ..."`.
