# Módulo 23 — Skills y Herramientas en Claude Code

> **Objetivo:** Entender cómo Claude Code usa skills (comandos `/`) para extender sus capacidades: crear skills personalizados, diseñar su estructura SKILL.md, combinarlos con hooks y comprender el flujo de ejecución completo.
>
> **Herramientas:** Python 3.11, json, pathlib, markdown
>
> **Prerequisito:** M21 (MCP Arquitectura), M22 (MCP Servidores)

---

## 1. Qué son los Skills en Claude Code

Los **skills** son comandos personalizados que extienden Claude Code sin modificar el modelo. Se definen como archivos `SKILL.md` con instrucciones en lenguaje natural más metadatos YAML frontmatter. Claude los lee como contexto adicional al ejecutar el comando `/nombre`.

```python
# modulo-23-skills-claude-code.py — Parte 1: Estructura de Skills

"""
Anatomía de un SKILL.md:

  ---
  description: "Texto que activa el skill automáticamente"
  allowed-tools: [Read, Write, Bash, Edit]
  ---

  # Nombre del Skill

  ## Contexto
  Qué problema resuelve y cuándo usarlo.

  ## Instrucciones
  Paso a paso de lo que Claude debe hacer.

  ## Formato de salida
  Cómo debe presentar los resultados.

Claude Code lo invoca cuando el usuario escribe: /nombre-skill
o cuando la descripción del skill coincide con el intent del usuario.
"""

import json
import re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Dict, List, Optional, Any
import os


# ── Modelo de datos para un Skill ────────────────────────────────────────────

@dataclass
class SkillFrontmatter:
    """Metadatos YAML del encabezado de un SKILL.md."""
    description: str
    allowed_tools: List[str] = field(default_factory=list)
    author: Optional[str] = None
    version: str = "1.0.0"
    tags: List[str] = field(default_factory=list)

    def to_yaml(self) -> str:
        lines = [
            "---",
            f'description: "{self.description}"',
        ]
        if self.allowed_tools:
            tools_str = ", ".join(self.allowed_tools)
            lines.append(f"allowed-tools: [{tools_str}]")
        if self.author:
            lines.append(f"author: {self.author}")
        lines.append(f"version: {self.version}")
        if self.tags:
            lines.append(f"tags: [{', '.join(self.tags)}]")
        lines.append("---")
        return "\n".join(lines)


@dataclass
class Skill:
    """
    Representación de un Skill de Claude Code.
    En disco: ~/.claude/skills/<nombre>/SKILL.md
    """
    nombre: str
    frontmatter: SkillFrontmatter
    contenido: str   # cuerpo Markdown con las instrucciones

    @property
    def ruta_skill(self) -> Path:
        return Path(".claude") / "skills" / self.nombre / "SKILL.md"

    def render(self) -> str:
        """Genera el SKILL.md completo."""
        return f"{self.frontmatter.to_yaml()}\n\n{self.contenido}"

    def guardar(self, base_dir: Path = Path(".")):
        """Escribe el SKILL.md en disco."""
        ruta = base_dir / self.ruta_skill
        ruta.parent.mkdir(parents=True, exist_ok=True)
        ruta.write_text(self.render(), encoding="utf-8")
        return ruta

    @classmethod
    def desde_archivo(cls, ruta: Path) -> "Skill":
        """Parsea un SKILL.md existente."""
        texto = ruta.read_text(encoding="utf-8")
        nombre = ruta.parent.name

        # Extraer frontmatter
        match = re.match(r"^---\n(.*?)\n---\n", texto, re.DOTALL)
        if not match:
            raise ValueError(f"Sin frontmatter en {ruta}")

        fm_texto = match.group(1)
        cuerpo = texto[match.end():]

        # Parsear campos del frontmatter
        description = re.search(r'description:\s*"([^"]+)"', fm_texto)
        tools_match = re.search(r'allowed-tools:\s*\[([^\]]+)\]', fm_texto)
        version_match = re.search(r'version:\s*(\S+)', fm_texto)

        fm = SkillFrontmatter(
            description=description.group(1) if description else "",
            allowed_tools=[t.strip() for t in tools_match.group(1).split(",")] if tools_match else [],
            version=version_match.group(1) if version_match else "1.0.0",
        )
        return cls(nombre=nombre, frontmatter=fm, contenido=cuerpo.strip())


# ── Habilidades predefinidas de ejemplo ──────────────────────────────────────

def crear_skill_review_pr() -> Skill:
    """Skill para revisar pull requests."""
    return Skill(
        nombre="review-pr",
        frontmatter=SkillFrontmatter(
            description="Use when the user asks to review a pull request, check PR quality, or analyze changes in a branch",
            allowed_tools=["Bash", "Read", "Glob", "Grep"],
            author="equipo-ia",
            version="1.2.0",
            tags=["git", "code-review", "pr"],
        ),
        contenido="""
# Review PR — Revisor de Pull Requests

## Contexto
Revisa los cambios de un pull request o rama y produce un informe estructurado
con feedback accionable para el autor.

## Instrucciones

1. **Obtener los cambios:**
   - Si se proporciona un número de PR: `gh pr diff <numero>`
   - Si es una rama: `git diff main...<rama> --stat` y luego `git diff main...<rama>`
   - Si no se especifica: revisar cambios staged (`git diff --cached`)

2. **Analizar el diff:**
   - Identificar archivos modificados y tipo de cambio (feature, fix, refactor, test, docs)
   - Detectar archivos críticos (auth, DB, APIs públicas)
   - Contar líneas añadidas/eliminadas

3. **Revisar cada archivo:**
   - Leer el contexto completo (no solo las líneas cambiadas)
   - Buscar: bugs potenciales, problemas de seguridad, código duplicado
   - Verificar: manejo de errores, tests, documentación

4. **Producir el informe** (ver formato de salida)

## Formato de salida

```
## Revisión de PR: <título/rama>

### Resumen
- Archivos modificados: N
- Líneas añadidas: +X / eliminadas: -Y
- Tipo: [feature | fix | refactor | test | docs | mixed]

### 🔴 Crítico (bloquea merge)
- [ARCHIVO:LINEA] Descripción del problema

### 🟡 Sugerencias (mejoras recomendadas)
- [ARCHIVO] Descripción de la sugerencia

### 🟢 Positivo
- Qué está bien hecho

### Veredicto
[APROBADO | CAMBIOS REQUERIDOS | NECESITA DISCUSIÓN]
```

## Ejemplos de invocación
- `/review-pr 123` — revisa PR #123
- `/review-pr main..feature/auth` — compara rama con main
- `/review-pr` — revisa cambios staged actuales
""".strip(),
    )


def crear_skill_commit() -> Skill:
    """Skill para generar mensajes de commit."""
    return Skill(
        nombre="commit",
        frontmatter=SkillFrontmatter(
            description="Use when the user wants to create a git commit, generate a commit message, or stage and commit changes",
            allowed_tools=["Bash"],
            version="1.0.0",
            tags=["git", "commit"],
        ),
        contenido="""
# Commit — Generador de Commits

## Contexto
Analiza los cambios staged/unstaged y genera un commit con mensaje semántico.

## Instrucciones

1. Ejecutar `git status` para ver el estado
2. Ejecutar `git diff --cached` para ver cambios staged
3. Si no hay cambios staged, ejecutar `git diff` para ver unstaged
4. Analizar los cambios y determinar:
   - **Tipo**: feat | fix | docs | style | refactor | test | chore | perf
   - **Scope**: componente afectado (opcional)
   - **Descripción**: qué cambió y por qué (no cómo)
5. Generar mensaje siguiendo Conventional Commits:
   `<tipo>(<scope>): <descripción>`
6. Añadir body si los cambios son complejos (> 3 archivos o lógica no obvia)
7. Preguntar confirmación antes de hacer el commit
8. Ejecutar `git add -p` o `git add <archivos>` + `git commit -m "..."`

## Formato del mensaje

```
<tipo>(<scope>): <descripción corta (max 72 chars)>

[body opcional: qué y por qué, no cómo]

[footer opcional: referencias a issues/breaking changes]
```

## Tipos semánticos
- `feat`: nueva funcionalidad
- `fix`: corrección de bug
- `docs`: solo documentación
- `refactor`: refactoring sin cambio de comportamiento
- `test`: añadir o corregir tests
- `perf`: mejora de rendimiento
- `chore`: tareas de mantenimiento
""".strip(),
    )


def crear_skill_debug() -> Skill:
    """Skill para debugging."""
    return Skill(
        nombre="debug",
        frontmatter=SkillFrontmatter(
            description="Use when the user reports a bug, error, exception, or unexpected behavior in their code",
            allowed_tools=["Read", "Bash", "Grep", "Glob", "Edit"],
            version="1.1.0",
            tags=["debug", "error", "troubleshooting"],
        ),
        contenido="""
# Debug — Asistente de Depuración

## Contexto
Diagnostica y resuelve bugs, errores y comportamientos inesperados en el código.

## Proceso de debugging

### Fase 1: Reproducir
1. Identificar el error exacto (mensaje, stack trace, comportamiento)
2. Determinar en qué condiciones ocurre
3. Reproducir el error mínimamente

### Fase 2: Localizar
1. Leer el stack trace completo (de abajo hacia arriba)
2. Identificar el archivo y línea del error
3. Leer el código en contexto (±20 líneas)
4. Buscar código similar con `grep` si el error es genérico

### Fase 3: Diagnosticar
1. Formular hipótesis sobre la causa raíz
2. Verificar cada hipótesis con evidencia del código
3. Descartar hipótesis incorrectas

### Fase 4: Corregir
1. Implementar la corrección mínima
2. Asegurarse de que el fix no rompe otros casos
3. Añadir test si no existe

## Heurísticas comunes
- `KeyError`/`AttributeError`: objeto None o tipo incorrecto
- `IndexError`: lista vacía o índice fuera de rango
- `RecursionError`: falta caso base o ciclo infinito
- `MemoryError`: objeto demasiado grande o leak
- Comportamiento intermitente: race condition o estado mutable compartido

## Formato de salida
```
## Diagnóstico

**Error:** <descripción del error>
**Causa raíz:** <explicación>
**Archivo:** <ruta:línea>

## Corrección

[código corregido]

## Explicación
<por qué la corrección funciona>

## Prevención
<cómo evitar el mismo error en el futuro>
```
""".strip(),
    )


# ── Demo 1: Crear y guardar skills ───────────────────────────────────────────

if __name__ == "__main__":
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Crear Skills de Claude Code")
    print("=" * 60)

    skills = [
        crear_skill_review_pr(),
        crear_skill_commit(),
        crear_skill_debug(),
    ]

    base_dir = Path("outputs/demo_skills_workspace")
    rutas = []
    for skill in skills:
        ruta = skill.guardar(base_dir)
        rutas.append(str(ruta))
        print(f"✓ Creado: {ruta}")

    # Mostrar uno como ejemplo
    print(f"\nEjemplo de SKILL.md ({skills[0].nombre}):")
    print(skills[0].render()[:500] + "...\n")

    # Listar skills disponibles
    todos_los_skills = list(base_dir.glob(".claude/skills/*/SKILL.md"))
    print(f"Skills disponibles: {len(todos_los_skills)}")
    for s_path in todos_los_skills:
        s = Skill.desde_archivo(s_path)
        print(f"  /{s.nombre} — {s.frontmatter.description[:60]}...")

    with open("outputs/m23_skills_creados.json", "w", encoding="utf-8") as f:
        json.dump({
            "skills": [
                {
                    "nombre": s.nombre,
                    "description": s.frontmatter.description,
                    "tools": s.frontmatter.allowed_tools,
                    "version": s.frontmatter.version,
                }
                for s in skills
            ],
            "rutas": rutas,
        }, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m23_skills_creados.json")
```

---

## 2. Hooks en Claude Code

Los **hooks** son scripts que se ejecutan automáticamente en momentos específicos del ciclo de vida de Claude Code. Permiten validación, logging y automatización sin intervención manual.

```python
# Parte 2: Sistema de Hooks

"""
Hooks disponibles en Claude Code (settings.json):

{
  "hooks": {
    "PreToolUse":  [{"matcher": "Bash", "hooks": [{"type":"command","command":"..."}]}],
    "PostToolUse": [{"matcher": "*",    "hooks": [...]}],
    "Stop":        [{"hooks": [...]}],
    "SubagentStop":[{"hooks": [...]}]
  }
}

Cada hook recibe por stdin un JSON con contexto del evento.
El exit code controla si Claude continúa (0) o se bloquea (1).
El stdout del hook se muestra al usuario.
"""


@dataclass
class HookEvent:
    """Evento que dispara un hook."""
    tipo: str           # PreToolUse | PostToolUse | Stop | SubagentStop
    tool_name: str = ""
    tool_input: Dict = field(default_factory=dict)
    tool_output: str = ""
    session_id: str = ""
    timestamp: float = field(default_factory=lambda: __import__("time").time())

    def to_json(self) -> str:
        return json.dumps({
            "event": self.tipo,
            "tool": {"name": self.tool_name, "input": self.tool_input},
            "output": self.tool_output,
            "session_id": self.session_id,
            "timestamp": self.timestamp,
        }, ensure_ascii=False)


@dataclass
class HookResult:
    """Resultado que un hook retorna a Claude Code."""
    exit_code: int  # 0=continuar, 1=bloquear, 2=bloquear con mensaje
    stdout: str = ""
    stderr: str = ""

    @property
    def bloqueado(self) -> bool:
        return self.exit_code != 0


class HookHandler:
    """
    Implementa lógica de hooks.
    En producción: script shell/Python que lee stdin y escribe stdout.
    """

    def __init__(self, nombre: str):
        self.nombre = nombre
        self.log: List[Dict] = []

    def ejecutar(self, evento: HookEvent) -> HookResult:
        raise NotImplementedError


class BashSafetyHook(HookHandler):
    """
    PreToolUse para Bash: bloquea comandos peligrosos.
    """
    COMANDOS_BLOQUEADOS = [
        r"rm\s+-rf\s+/",        # rm -rf /
        r"git\s+push\s+--force", # force push
        r"chmod\s+777",          # permisos inseguros
        r"curl\s+.*\|\s*bash",  # pipe a bash (remote code exec)
        r"dd\s+if=",            # escribir a dispositivo
        r":(){:|:&};:",          # fork bomb
    ]

    def ejecutar(self, evento: HookEvent) -> HookResult:
        if evento.tipo != "PreToolUse" or evento.tool_name != "Bash":
            return HookResult(0)  # no aplica, continuar

        comando = evento.tool_input.get("command", "")
        self.log.append({"evento": "bash_check", "comando": comando[:80]})

        for patron in self.COMANDOS_BLOQUEADOS:
            if re.search(patron, comando):
                msg = f"⛔ Hook BashSafety bloqueó: patrón peligroso detectado ({patron!r})"
                self.log.append({"evento": "BLOCKED", "patron": patron})
                return HookResult(exit_code=2, stdout=msg)

        return HookResult(0, stdout=f"✓ Bash seguro: {comando[:50]}...")


class AuditLogHook(HookHandler):
    """
    PostToolUse: registra todas las llamadas a herramientas.
    """

    def __init__(self, log_path: str):
        super().__init__("AuditLog")
        self.log_path = log_path
        self._entries: List[Dict] = []

    def ejecutar(self, evento: HookEvent) -> HookResult:
        if evento.tipo != "PostToolUse":
            return HookResult(0)

        entrada = {
            "timestamp": evento.timestamp,
            "tool": evento.tool_name,
            "input_snippet": str(evento.tool_input)[:100],
            "output_snippet": evento.tool_output[:100],
        }
        self._entries.append(entrada)

        # Escribir al log
        try:
            with open(self.log_path, "a", encoding="utf-8") as f:
                f.write(json.dumps(entrada, ensure_ascii=False) + "\n")
        except IOError:
            pass

        return HookResult(0, stdout=f"📝 Auditado: {evento.tool_name}")


class NotificationHook(HookHandler):
    """
    Stop: notifica cuando Claude termina una sesión larga.
    """

    def __init__(self, threshold_tools: int = 5):
        super().__init__("Notification")
        self.threshold = threshold_tools
        self.tools_ejecutadas = 0

    def ejecutar(self, evento: HookEvent) -> HookResult:
        if evento.tipo == "PostToolUse":
            self.tools_ejecutadas += 1
            return HookResult(0)

        if evento.tipo == "Stop" and self.tools_ejecutadas >= self.threshold:
            msg = (
                f"🔔 Claude terminó: {self.tools_ejecutadas} herramientas ejecutadas. "
                "Sesión completada."
            )
            return HookResult(0, stdout=msg)

        return HookResult(0)


class HookOrchestrator:
    """
    Ejecuta la cadena de hooks en el orden correcto.
    En producción, Claude Code lo hace internamente; aquí lo simulamos.
    """

    def __init__(self):
        self._pre_tool_use: List[HookHandler] = []
        self._post_tool_use: List[HookHandler] = []
        self._stop: List[HookHandler] = []

    def registrar_pre(self, hook: HookHandler):
        self._pre_tool_use.append(hook)

    def registrar_post(self, hook: HookHandler):
        self._post_tool_use.append(hook)

    def registrar_stop(self, hook: HookHandler):
        self._stop.append(hook)

    def before_tool(self, tool_name: str, tool_input: Dict) -> HookResult:
        """Ejecuta PreToolUse hooks. Si alguno bloquea, se detiene."""
        evento = HookEvent("PreToolUse", tool_name=tool_name, tool_input=tool_input)
        for hook in self._pre_tool_use:
            resultado = hook.ejecutar(evento)
            if resultado.bloqueado:
                return resultado
        return HookResult(0)

    def after_tool(self, tool_name: str, tool_input: Dict, output: str) -> List[HookResult]:
        """Ejecuta PostToolUse hooks (todos, aunque alguno falle)."""
        evento = HookEvent("PostToolUse", tool_name=tool_name, tool_input=tool_input, tool_output=output)
        return [h.ejecutar(evento) for h in self._post_tool_use]

    def on_stop(self) -> List[HookResult]:
        """Ejecuta Stop hooks."""
        evento = HookEvent("Stop")
        return [h.ejecutar(evento) for h in self._stop]


def demo_hooks():
    print("\n" + "=" * 60)
    print("DEMO 2 — Sistema de Hooks")
    print("=" * 60)

    orquestador = HookOrchestrator()
    orquestador.registrar_pre(BashSafetyHook("BashSafety"))
    audit = AuditLogHook("outputs/m23_audit.log")
    notif = NotificationHook(threshold_tools=3)
    orquestador.registrar_post(audit)
    orquestador.registrar_post(notif)
    orquestador.registrar_stop(notif)

    # Simular uso de herramientas
    herramientas = [
        ("Bash", {"command": "ls -la /tmp"}),
        ("Bash", {"command": "git push --force origin main"}),  # bloqueado
        ("Read", {"file_path": "/home/user/config.py"}),
        ("Edit", {"file_path": "main.py", "old_string": "x", "new_string": "y"}),
        ("Bash", {"command": "python tests.py"}),
    ]

    for tool_name, tool_input in herramientas:
        print(f"\n[Tool: {tool_name}]")

        # PreToolUse
        pre_result = orquestador.before_tool(tool_name, tool_input)
        if pre_result.bloqueado:
            print(f"  BLOQUEADO: {pre_result.stdout}")
            continue
        if pre_result.stdout:
            print(f"  Pre: {pre_result.stdout}")

        # Simular ejecución
        output = f"Salida simulada de {tool_name}"
        print(f"  Ejecutado: {str(tool_input)[:50]}...")

        # PostToolUse
        post_results = orquestador.after_tool(tool_name, tool_input, output)
        for r in post_results:
            if r.stdout:
                print(f"  Post: {r.stdout}")

    # Stop
    print("\n[Stop]")
    stop_results = orquestador.on_stop()
    for r in stop_results:
        if r.stdout:
            print(f"  {r.stdout}")

    return {
        "tools_procesadas": len(herramientas),
        "audit_entries": len(audit._entries),
    }


if __name__ == "__main__":
    resultado_hooks = demo_hooks()
    print(f"\nAudit log: {resultado_hooks['audit_entries']} entradas")
    with open("outputs/m23_hooks_demo.json", "w", encoding="utf-8") as f:
        json.dump(resultado_hooks, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m23_hooks_demo.json")
```

---

## 3. Diseño de Skills Avanzados con Argumentos

```python
# Parte 3: Skills con argumentos dinámicos y templates

class SkillEjecutor:
    """
    Simula cómo Claude Code procesa un skill invocado por el usuario.
    En producción: Claude lee el SKILL.md como contexto y actúa según sus instrucciones.
    """

    def __init__(self):
        self._skills: Dict[str, Skill] = {}
        self._ejecuciones: List[Dict] = []

    def registrar(self, skill: Skill):
        self._skills[skill.nombre] = skill

    def auto_detectar(self, mensaje_usuario: str) -> Optional[str]:
        """
        Encuentra el skill más relevante para un mensaje dado.
        En Claude Code: el LLM hace este matching usando las descripciones.
        """
        mensaje_l = mensaje_usuario.lower()
        scores: Dict[str, int] = {}

        for nombre, skill in self._skills.items():
            desc_words = skill.frontmatter.description.lower().split()
            score = sum(1 for word in desc_words if word in mensaje_l)
            if score > 0:
                scores[nombre] = score

        if not scores:
            return None
        return max(scores, key=lambda k: scores[k])

    def ejecutar(self, nombre_skill: str, args: Dict = None, contexto: str = "") -> Dict:
        """
        Simula la ejecución de un skill.
        En producción: Claude usa el SKILL.md + args para guiar su trabajo.
        """
        if nombre_skill not in self._skills:
            return {"error": f"Skill '{nombre_skill}' no encontrado"}

        skill = self._skills[nombre_skill]
        args = args or {}

        # Construir el prompt que Claude recibiría
        prompt_efectivo = (
            f"# Skill: /{nombre_skill}\n\n"
            f"{skill.contenido}\n\n"
            f"## Argumentos del usuario\n"
            + "\n".join(f"- {k}: {v}" for k, v in args.items())
            + (f"\n\n## Contexto adicional\n{contexto}" if contexto else "")
        )

        ejecucion = {
            "skill": nombre_skill,
            "args": args,
            "tools_disponibles": skill.frontmatter.allowed_tools,
            "prompt_length": len(prompt_efectivo),
            "status": "ejecutado",
        }
        self._ejecuciones.append(ejecucion)

        # Simular output basado en el tipo de skill
        outputs = {
            "review-pr": f"## Revisión de PR {args.get('pr', 'HEAD')}\n✓ Código analizado\n⚠ 2 sugerencias encontradas\n✅ Aprobado con cambios menores",
            "commit": f"feat({args.get('scope', 'core')}): {args.get('msg', 'actualización automática')}\n\nCommit creado exitosamente.",
            "debug": f"## Diagnóstico\nCausa raíz: {args.get('error', 'error desconocido')}\nCorreción aplicada en línea {args.get('linea', '42')}",
        }

        ejecucion["output"] = outputs.get(nombre_skill, f"Skill '{nombre_skill}' ejecutado con {len(args)} argumentos")
        return ejecucion


def demo_skills_avanzados():
    print("\n" + "=" * 60)
    print("DEMO 3 — Skills con argumentos y auto-detección")
    print("=" * 60)

    ejecutor = SkillEjecutor()
    ejecutor.registrar(crear_skill_review_pr())
    ejecutor.registrar(crear_skill_commit())
    ejecutor.registrar(crear_skill_debug())

    # Auto-detección de skills
    mensajes = [
        "revisa el pull request 42",
        "tengo un error en mi código de autenticación",
        "quiero hacer un commit de mis cambios",
        "qué hay de nuevo?",  # no detecta skill
    ]

    print("\n--- Auto-detección de Skills ---")
    for msg in mensajes:
        detectado = ejecutor.auto_detectar(msg)
        print(f"  '{msg[:45]}' → {('/' + detectado) if detectado else 'sin skill'}")

    # Ejecución directa con argumentos
    print("\n--- Ejecución de Skills ---")
    calls = [
        ("review-pr", {"pr": "123", "rama": "feature/login"}),
        ("commit", {"scope": "auth", "msg": "agregar validación de token JWT"}),
        ("debug", {"error": "AttributeError: 'NoneType' object", "linea": "87"}),
    ]

    resultados = []
    for nombre, args in calls:
        resultado = ejecutor.ejecutar(nombre, args)
        print(f"\n/{nombre} {args}")
        print(f"  → {resultado['output'][:100]}")
        resultados.append(resultado)

    return resultados


if __name__ == "__main__":
    resultados_skills = demo_skills_avanzados()
    with open("outputs/m23_skills_avanzados.json", "w", encoding="utf-8") as f:
        json.dump(resultados_skills, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m23_skills_avanzados.json")
    print("\n[M23 completado] Todos los outputs guardados en outputs/")
```

---

## Checkpoint

```python
# checkpoint_m23.py — 6 tests para Skills y Hooks

import json, re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Dict, List, Optional
import os


@dataclass
class SkillFrontmatter:
    description: str; allowed_tools: List[str] = field(default_factory=list)
    version: str = "1.0.0"; tags: List[str] = field(default_factory=list)
    author: Optional[str] = None
    def to_yaml(self):
        lines = ["---", f'description: "{self.description}"']
        if self.allowed_tools: lines.append(f"allowed-tools: [{', '.join(self.allowed_tools)}]")
        lines.append(f"version: {self.version}")
        lines.append("---")
        return "\n".join(lines)

@dataclass
class Skill:
    nombre: str; frontmatter: SkillFrontmatter; contenido: str
    def render(self): return f"{self.frontmatter.to_yaml()}\n\n{self.contenido}"
    def guardar(self, base_dir=Path(".")):
        ruta = base_dir / ".claude" / "skills" / self.nombre / "SKILL.md"
        ruta.parent.mkdir(parents=True, exist_ok=True)
        ruta.write_text(self.render(), encoding="utf-8")
        return ruta
    @classmethod
    def desde_archivo(cls, ruta):
        texto = ruta.read_text(encoding="utf-8")
        nombre = ruta.parent.name
        m = re.match(r"^---\n(.*?)\n---\n", texto, re.DOTALL)
        if not m: raise ValueError("Sin frontmatter")
        fm_texto = m.group(1); cuerpo = texto[m.end():]
        desc = re.search(r'description:\s*"([^"]+)"', fm_texto)
        tools = re.search(r'allowed-tools:\s*\[([^\]]+)\]', fm_texto)
        fm = SkillFrontmatter(
            description=desc.group(1) if desc else "",
            allowed_tools=[t.strip() for t in tools.group(1).split(",")] if tools else [],
        )
        return cls(nombre=nombre, frontmatter=fm, contenido=cuerpo.strip())

@dataclass
class HookEvent:
    tipo: str; tool_name: str = ""; tool_input: Dict = field(default_factory=dict)
    tool_output: str = ""; timestamp: float = 0.0

@dataclass
class HookResult:
    exit_code: int; stdout: str = ""; stderr: str = ""
    @property
    def bloqueado(self): return self.exit_code != 0

COMANDOS_BLOQUEADOS = [r"rm\s+-rf\s+/", r"git\s+push\s+--force", r"chmod\s+777"]
def bash_safety_check(comando: str) -> HookResult:
    for patron in COMANDOS_BLOQUEADOS:
        if re.search(patron, comando):
            return HookResult(2, stdout=f"Bloqueado: {patron}")
    return HookResult(0)


def test_01_skill_frontmatter_render():
    """SkillFrontmatter genera YAML correcto."""
    fm = SkillFrontmatter(
        description="Review a pull request",
        allowed_tools=["Read", "Bash"],
        version="2.0.0",
    )
    yaml = fm.to_yaml()
    assert "---" in yaml
    assert 'description: "Review a pull request"' in yaml
    assert "allowed-tools: [Read, Bash]" in yaml
    assert "version: 2.0.0" in yaml
    print("✓ test_01 — SkillFrontmatter YAML OK")

def test_02_skill_render_and_save():
    """Skill se renderiza y guarda correctamente en disco."""
    skill = Skill(
        nombre="test-skill",
        frontmatter=SkillFrontmatter("Test skill description", ["Read"]),
        contenido="## Instrucciones\n\nHaz algo útil.",
    )
    rendered = skill.render()
    assert "---" in rendered
    assert "Test skill description" in rendered
    assert "## Instrucciones" in rendered

    # Guardar y verificar
    ruta = skill.guardar(Path("outputs/test_m23"))
    assert ruta.exists()
    assert ruta.name == "SKILL.md"
    print("✓ test_02 — Skill render y save OK")

def test_03_skill_desde_archivo():
    """Skill.desde_archivo() parsea correctamente un SKILL.md."""
    ruta = Path("outputs/test_m23/.claude/skills/test-skill/SKILL.md")
    if not ruta.exists():
        # Crear si no existe
        Skill(
            nombre="test-skill",
            frontmatter=SkillFrontmatter("Test skill desc", ["Bash"]),
            contenido="## Body\nContenido.",
        ).guardar(Path("outputs/test_m23"))

    skill = Skill.desde_archivo(ruta)
    assert skill.nombre == "test-skill"
    assert skill.frontmatter.description != ""
    assert len(skill.contenido) > 0
    print("✓ test_03 — Skill.desde_archivo() OK")

def test_04_bash_safety_hook_bloquea():
    """El hook de seguridad bloquea comandos peligrosos."""
    result_peligroso = bash_safety_check("rm -rf /var/data")
    assert result_peligroso.bloqueado, "rm -rf / debe bloquearse"
    assert result_peligroso.exit_code == 2

    result_fuerza = bash_safety_check("git push --force origin main")
    assert result_fuerza.bloqueado, "git push --force debe bloquearse"
    print("✓ test_04 — BashSafety bloqueo OK")

def test_05_bash_safety_hook_permite():
    """El hook permite comandos seguros."""
    comandos_seguros = ["ls -la", "python tests.py", "git status", "cat README.md"]
    for cmd in comandos_seguros:
        result = bash_safety_check(cmd)
        assert not result.bloqueado, f"'{cmd}' no debe bloquearse"
    print("✓ test_05 — BashSafety permisos OK")

def test_06_skill_auto_detection():
    """El sistema detecta el skill más relevante para un mensaje."""
    skills_desc = {
        "review-pr": "review pull request check PR quality analyze changes",
        "commit": "create git commit generate commit message stage changes",
        "debug": "bug error exception unexpected behavior troubleshooting",
    }

    def detectar(mensaje: str) -> Optional[str]:
        msg_l = mensaje.lower()
        scores = {}
        for nombre, desc in skills_desc.items():
            words = desc.lower().split()
            score = sum(1 for w in words if w in msg_l)
            if score > 0: scores[nombre] = score
        return max(scores, key=lambda k: scores[k]) if scores else None

    assert detectar("review my pull request") == "review-pr"
    assert detectar("I have a bug in my code") == "debug"
    assert detectar("make a commit of my changes") == "commit"
    assert detectar("hello world") is None
    print("✓ test_06 — Auto-detección de skills OK")


if __name__ == "__main__":
    os.makedirs("outputs/test_m23", exist_ok=True)
    tests = [
        test_01_skill_frontmatter_render,
        test_02_skill_render_and_save,
        test_03_skill_desde_archivo,
        test_04_bash_safety_hook_bloquea,
        test_05_bash_safety_hook_permite,
        test_06_skill_auto_detection,
    ]
    aprobados = 0
    for test in tests:
        try:
            test()
            aprobados += 1
        except AssertionError as e:
            print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e:
            print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")

    print(f"\n{'='*40}")
    print(f"Resultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M23 completado")
```

---

## Resumen

| Concepto | Descripción |
|---|---|
| **SKILL.md** | Archivo Markdown con frontmatter YAML + instrucciones en lenguaje natural |
| **description** | Texto que activa el skill automáticamente cuando coincide con el intent del usuario |
| **allowed-tools** | Lista de herramientas que el skill puede usar (Read, Write, Bash, Edit…) |
| **Hook PreToolUse** | Se ejecuta antes de cada herramienta; puede bloquear (exit 1/2) o permitir (exit 0) |
| **Hook PostToolUse** | Se ejecuta después de cada herramienta; útil para auditoría y logging |
| **Hook Stop** | Se ejecuta al final de la sesión de Claude |
| **stdin/stdout del hook** | Hook recibe contexto JSON por stdin; escribe output por stdout |
| **exit code 0** | Continuar normalmente |
| **exit code 1/2** | Bloquear la acción (Claude no ejecuta la herramienta) |

### Estructura de directorios

```
.claude/
├── settings.json          ← configuración global (hooks, permisos)
├── skills/
│   ├── review-pr/
│   │   └── SKILL.md       ← /review-pr
│   ├── commit/
│   │   └── SKILL.md       ← /commit
│   └── debug/
│       └── SKILL.md       ← /debug
└── hooks/
    ├── bash_safety.py     ← hook PreToolUse para Bash
    └── audit_log.py       ← hook PostToolUse para auditoría
```
