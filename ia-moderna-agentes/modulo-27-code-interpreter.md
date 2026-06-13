# Módulo 27 — Agentes con Acceso a Código: Code Interpreter

> **Objetivo:** Construir agentes capaces de escribir, ejecutar y depurar código Python de forma segura: sandbox de ejecución, captura de outputs, manejo de errores y ciclo iterativo de mejora código-resultado.
>
> **Herramientas:** Python 3.11, subprocess, io, sys, contextlib
>
> **Prerequisito:** M16 (Agentes ReAct), M25 (Orquestación)

---

## 1. Sandbox de Ejecución Segura

```python
# modulo-27-code-interpreter.py — Parte 1: Sandbox seguro

"""
Un Code Interpreter agéntico necesita:
1. SEGURIDAD: aislar la ejecución del código del usuario
2. CAPTURA: interceptar stdout, stderr y valores retornados
3. ESTADO: mantener variables entre ejecuciones (REPL)
4. LÍMITES: timeout y límite de memoria/output
5. ITERACIÓN: bucle: escribir código → ejecutar → ver error → corregir
"""

import sys
import io
import ast
import time
import json
import uuid
import contextlib
import traceback
import subprocess
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Tuple
from enum import Enum


# ── Resultado de ejecución ────────────────────────────────────────────────────

@dataclass
class ResultadoEjecucion:
    """Captura todos los outputs de una ejecución de código."""
    codigo: str
    stdout: str = ""
    stderr: str = ""
    valor_retornado: Any = None      # resultado de la última expresión
    error: Optional[str] = None      # traceback si hay excepción
    tiempo_ms: int = 0
    exito: bool = True
    variables_nuevas: List[str] = field(default_factory=list)

    def to_dict(self) -> Dict:
        return {
            "exito": self.exito,
            "stdout": self.stdout[:2000],
            "stderr": self.stderr[:500],
            "error": self.error,
            "tiempo_ms": self.tiempo_ms,
            "variables_nuevas": self.variables_nuevas,
            "valor": str(self.valor_retornado)[:200] if self.valor_retornado is not None else None,
        }

    @property
    def texto_completo(self) -> str:
        """Output combinado para mostrar al usuario/LLM."""
        partes = []
        if self.stdout:
            partes.append(f"[stdout]\n{self.stdout}")
        if self.stderr:
            partes.append(f"[stderr]\n{self.stderr}")
        if self.error:
            partes.append(f"[error]\n{self.error}")
        if self.valor_retornado is not None and not self.stdout:
            partes.append(f"[resultado]\n{self.valor_retornado}")
        return "\n".join(partes) if partes else "(sin output)"


# ── Analizador de código estático (seguridad) ─────────────────────────────────

class AnalizadorSeguridad:
    """
    Analiza el AST del código para detectar operaciones peligrosas
    antes de ejecutarlo.
    """

    MODULOS_BLOQUEADOS = {
        "subprocess", "os.system", "shutil.rmtree",
        "socket", "urllib.request", "__import__",
    }

    FUNCIONES_BLOQUEADAS = {
        "exec", "eval", "compile", "open",  # open requiere permiso explícito
        "__import__", "globals", "locals",
    }

    MODULOS_PERMITIDOS = {
        "math", "statistics", "random", "json", "re", "datetime",
        "collections", "itertools", "functools", "string", "typing",
        "dataclasses", "enum", "pathlib", "io", "csv",
        # Data science (si están instalados)
        "numpy", "pandas", "matplotlib", "sklearn",
    }

    def __init__(self, permitir_archivos: bool = False, permitir_red: bool = False):
        self.permitir_archivos = permitir_archivos
        self.permitir_red = permitir_red

    def analizar(self, codigo: str) -> Tuple[bool, List[str]]:
        """
        Analiza el código y retorna (es_seguro, lista_de_advertencias).
        """
        advertencias = []

        try:
            arbol = ast.parse(codigo)
        except SyntaxError as e:
            return False, [f"Error de sintaxis: {e}"]

        for nodo in ast.walk(arbol):
            # Detectar importaciones de módulos bloqueados
            if isinstance(nodo, (ast.Import, ast.ImportFrom)):
                modulo = ""
                if isinstance(nodo, ast.Import):
                    for alias in nodo.names:
                        modulo = alias.name.split(".")[0]
                        if modulo not in self.MODULOS_PERMITIDOS:
                            advertencias.append(f"Importación no permitida: {alias.name}")
                elif isinstance(nodo, ast.ImportFrom) and nodo.module:
                    modulo = nodo.module.split(".")[0]
                    if modulo not in self.MODULOS_PERMITIDOS:
                        advertencias.append(f"Importación no permitida: {nodo.module}")

            # Detectar llamadas a funciones bloqueadas
            if isinstance(nodo, ast.Call):
                if isinstance(nodo.func, ast.Name):
                    if nodo.func.id in self.FUNCIONES_BLOQUEADAS:
                        if not (nodo.func.id == "open" and self.permitir_archivos):
                            advertencias.append(f"Función no permitida: {nodo.func.id}()")

        es_seguro = len(advertencias) == 0
        return es_seguro, advertencias


# ── Sandbox REPL ──────────────────────────────────────────────────────────────

class SandboxREPL:
    """
    REPL (Read-Eval-Print Loop) con estado persistente entre ejecuciones.
    Captura stdout/stderr y detecta errores.
    No usa aislamiento real de OS (para eso se necesita Docker/gVisor).
    """

    MAX_OUTPUT_CHARS = 10_000
    TIMEOUT_SEGUNDOS = 10.0

    def __init__(self, permitir_archivos: bool = False):
        self._namespace: Dict[str, Any] = {
            "__builtins__": __builtins__,
            "print": print,
        }
        self._historial: List[ResultadoEjecucion] = []
        self._analizador = AnalizadorSeguridad(permitir_archivos=permitir_archivos)

    def ejecutar(self, codigo: str, timeout: float = None) -> ResultadoEjecucion:
        """
        Ejecuta código en el namespace compartido.
        Captura stdout, stderr y el valor de la última expresión.
        """
        timeout = timeout or self.TIMEOUT_SEGUNDOS
        inicio = time.time()

        # 1. Análisis de seguridad
        es_seguro, advertencias = self._analizador.analizar(codigo)
        if not es_seguro:
            return ResultadoEjecucion(
                codigo=codigo,
                error=f"Código bloqueado por seguridad:\n" + "\n".join(f"  - {a}" for a in advertencias),
                exito=False,
                tiempo_ms=0,
            )

        # 2. Capturar stdout/stderr
        stdout_capturado = io.StringIO()
        stderr_capturado = io.StringIO()
        valor_retornado = None
        error_msg = None

        # Variables previas para detectar nuevas
        vars_previas = set(self._namespace.keys())

        try:
            with contextlib.redirect_stdout(stdout_capturado), \
                 contextlib.redirect_stderr(stderr_capturado):

                # Intentar evaluar como expresión primero (para mostrar valor)
                try:
                    arbol = ast.parse(codigo, mode="eval")
                    valor_retornado = eval(compile(arbol, "<sandbox>", "eval"), self._namespace)
                except SyntaxError:
                    # Si no es expresión, ejecutar como bloque
                    exec(compile(codigo, "<sandbox>", "exec"), self._namespace)

        except Exception as e:
            error_msg = traceback.format_exc()

        tiempo_ms = int((time.time() - inicio) * 1000)

        stdout_text = stdout_capturado.getvalue()[:self.MAX_OUTPUT_CHARS]
        stderr_text = stderr_capturado.getvalue()[:self.MAX_OUTPUT_CHARS]
        vars_nuevas = [v for v in self._namespace if v not in vars_previas and not v.startswith("_")]

        resultado = ResultadoEjecucion(
            codigo=codigo,
            stdout=stdout_text,
            stderr=stderr_text,
            valor_retornado=valor_retornado,
            error=error_msg,
            tiempo_ms=tiempo_ms,
            exito=error_msg is None,
            variables_nuevas=vars_nuevas,
        )
        self._historial.append(resultado)
        return resultado

    def obtener_variable(self, nombre: str) -> Any:
        return self._namespace.get(nombre)

    def limpiar(self):
        """Reinicia el namespace (nueva sesión)."""
        self._namespace = {"__builtins__": __builtins__, "print": print}

    @property
    def variables_disponibles(self) -> Dict[str, str]:
        """Retorna las variables definidas y sus tipos."""
        return {
            k: type(v).__name__
            for k, v in self._namespace.items()
            if not k.startswith("_") and k != "print"
        }

    @property
    def historial(self) -> List[ResultadoEjecucion]:
        return self._historial


# ── Demo 1: REPL básico ───────────────────────────────────────────────────────

if __name__ == "__main__":
    import os
    os.makedirs("outputs", exist_ok=True)

    print("=" * 60)
    print("DEMO 1 — Sandbox REPL")
    print("=" * 60)

    repl = SandboxREPL()

    snippets = [
        # Matemáticas simples
        "2 + 2 * 10",
        # Definir función
        "def factorial(n):\n    return 1 if n == 0 else n * factorial(n-1)",
        # Usar la función
        "factorial(10)",
        # Comprensión de lista
        "[x**2 for x in range(10)]",
        # Error intencional
        "1 / 0",
        # Código bloqueado (importación no permitida)
        "import subprocess\nsubprocess.run(['ls'])",
        # Numpy (si está disponible; si no, error manejado)
        "import math\n[math.sin(x/10) for x in range(5)]",
    ]

    resultados = []
    for snippet in snippets:
        r = repl.ejecutar(snippet)
        status = "✓" if r.exito else "✗"
        print(f"\n{status} Código: {snippet[:50].replace(chr(10), '↵')}")
        print(f"   Output: {r.texto_completo[:100]}")
        resultados.append(r.to_dict())

    print(f"\nVariables en namespace: {repl.variables_disponibles}")
    print(f"Ejecuciones: {len(repl.historial)}, Exitosas: {sum(1 for r in repl.historial if r.exito)}")

    with open("outputs/m27_repl_demo.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("Guardado: outputs/m27_repl_demo.json")
```

---

## 2. Agente Code Interpreter con Ciclo Iterativo

```python
# Parte 2: Agente Code Interpreter completo

class LLMCodeGenerator:
    """
    LLM simulado especializado en generar código Python.
    Analiza el error anterior y genera código corregido.
    """

    def generar_codigo(self, tarea: str, error_anterior: Optional[str] = None, contexto: str = "") -> str:
        """Genera código Python para resolver una tarea."""
        tarea_l = tarea.lower()

        if "fibonacci" in tarea_l or "sucesión" in tarea_l:
            if error_anterior and "recursion" in error_anterior.lower():
                # Corrección: usar versión iterativa tras error de recursión
                return (
                    "def fibonacci_iterativo(n):\n"
                    "    a, b = 0, 1\n"
                    "    for _ in range(n):\n"
                    "        a, b = b, a + b\n"
                    "    return a\n\n"
                    "resultados = [fibonacci_iterativo(i) for i in range(15)]\n"
                    "print('Fibonacci:', resultados)"
                )
            return (
                "def fibonacci(n):\n"
                "    if n <= 1:\n"
                "        return n\n"
                "    return fibonacci(n-1) + fibonacci(n-2)\n\n"
                "print([fibonacci(i) for i in range(10)])"
            )

        if "estadística" in tarea_l or "estadisticas" in tarea_l or "promedio" in tarea_l:
            return (
                "import math\n\n"
                "datos = [23, 45, 12, 67, 34, 89, 56, 78, 9, 44]\n\n"
                "n = len(datos)\n"
                "media = sum(datos) / n\n"
                "datos_ord = sorted(datos)\n"
                "mediana = datos_ord[n//2] if n % 2 == 1 else (datos_ord[n//2-1] + datos_ord[n//2]) / 2\n"
                "varianza = sum((x - media)**2 for x in datos) / n\n"
                "std = math.sqrt(varianza)\n\n"
                "print(f'Media: {media:.2f}')\n"
                "print(f'Mediana: {mediana}')\n"
                "print(f'Desv. estándar: {std:.2f}')\n"
                "print(f'Mín: {min(datos)}, Máx: {max(datos)}')"
            )

        if "ordenar" in tarea_l or "sort" in tarea_l or "algoritmo" in tarea_l:
            return (
                "def merge_sort(arr):\n"
                "    if len(arr) <= 1:\n"
                "        return arr\n"
                "    mid = len(arr) // 2\n"
                "    izq = merge_sort(arr[:mid])\n"
                "    der = merge_sort(arr[mid:])\n"
                "    return merge(izq, der)\n\n"
                "def merge(izq, der):\n"
                "    result = []\n"
                "    i = j = 0\n"
                "    while i < len(izq) and j < len(der):\n"
                "        if izq[i] <= der[j]:\n"
                "            result.append(izq[i]); i += 1\n"
                "        else:\n"
                "            result.append(der[j]); j += 1\n"
                "    return result + izq[i:] + der[j:]\n\n"
                "datos = [64, 34, 25, 12, 22, 11, 90]\n"
                "ordenado = merge_sort(datos)\n"
                "print(f'Original: {datos}')\n"
                "print(f'Ordenado: {ordenado}')"
            )

        # Código genérico
        return (
            f"# Solución para: {tarea[:50]}\n"
            f"resultado = 'Tarea procesada: {tarea[:30]}'\n"
            f"print(resultado)\n"
            f"resultado  # retornar valor"
        )

    def analizar_error(self, codigo: str, error: str) -> str:
        """Analiza un error y sugiere la corrección."""
        if "ZeroDivisionError" in error:
            return "Agrega validación: if denominador == 0: return None"
        if "NameError" in error:
            return "Variable no definida. Verifica el nombre o defínela antes de usarla."
        if "RecursionError" in error:
            return "Stack overflow. Convierte la recursión a versión iterativa."
        if "TypeError" in error:
            return "Tipo de datos incorrecto. Verifica los tipos antes de operar."
        if "IndexError" in error:
            return "Índice fuera de rango. Verifica len() antes de acceder."
        return f"Error: {error[:100]}. Revisa la lógica del código."


class CodeInterpreterAgent:
    """
    Agente con capacidad de ejecutar código Python.
    Ciclo: entender tarea → generar código → ejecutar → revisar → iterar.
    """

    def __init__(self, llm: LLMCodeGenerator, max_intentos: int = 3):
        self.llm = llm
        self.max_intentos = max_intentos
        self.repl = SandboxREPL()
        self._sesiones: List[Dict] = []

    def resolver(self, tarea: str, verbose: bool = True) -> Dict:
        """
        Resuelve una tarea mediante código:
        1. Genera código
        2. Ejecuta en sandbox
        3. Si hay error → analiza → genera código corregido
        4. Repite hasta max_intentos
        """
        if verbose:
            print(f"\n[CodeInterpreter] Tarea: {tarea}")

        historial_intentos = []
        error_anterior = None
        codigo_anterior = None

        for intento in range(1, self.max_intentos + 1):
            if verbose:
                print(f"  Intento {intento}/{self.max_intentos}")

            # Generar código
            codigo = self.llm.generar_codigo(tarea, error_anterior)
            if verbose:
                print(f"  Código ({len(codigo)} chars):\n    " + codigo[:100].replace("\n", "\n    "))

            # Ejecutar
            resultado = self.repl.ejecutar(codigo)

            intento_info = {
                "intento": intento,
                "codigo": codigo,
                "resultado": resultado.to_dict(),
            }
            historial_intentos.append(intento_info)

            if resultado.exito:
                if verbose:
                    print(f"  ✓ Éxito en {resultado.tiempo_ms}ms")
                    print(f"  Output: {resultado.texto_completo[:200]}")

                sesion = {
                    "tarea": tarea,
                    "exito": True,
                    "intentos": intento,
                    "resultado_final": resultado.to_dict(),
                    "historial": historial_intentos,
                }
                self._sesiones.append(sesion)
                return sesion

            else:
                error_anterior = resultado.error
                if verbose:
                    print(f"  ✗ Error: {resultado.error[:80] if resultado.error else 'desconocido'}")
                if intento < self.max_intentos:
                    analisis = self.llm.analizar_error(codigo, error_anterior or "")
                    if verbose:
                        print(f"  Análisis: {analisis}")

        # Agotó intentos
        sesion = {
            "tarea": tarea,
            "exito": False,
            "intentos": self.max_intentos,
            "error_final": error_anterior,
            "historial": historial_intentos,
        }
        self._sesiones.append(sesion)
        if verbose:
            print(f"  ✗ Agotados {self.max_intentos} intentos")
        return sesion

    def ejecutar_directo(self, codigo: str) -> ResultadoEjecucion:
        """Ejecuta código directamente sin loop de corrección."""
        return self.repl.ejecutar(codigo)

    @property
    def estado_namespace(self) -> Dict[str, str]:
        return self.repl.variables_disponibles


def demo_code_interpreter():
    print("\n" + "=" * 60)
    print("DEMO 2 — Code Interpreter Agéntico")
    print("=" * 60)

    agente = CodeInterpreterAgent(LLMCodeGenerator(), max_intentos=3)

    tareas = [
        "Calcula la sucesión Fibonacci hasta el término 14",
        "Calcula estadísticas descriptivas de un dataset de números",
        "Implementa y prueba el algoritmo merge sort",
    ]

    resultados = []
    for tarea in tareas:
        r = agente.resolver(tarea, verbose=True)
        resultados.append({
            "tarea": tarea,
            "exito": r["exito"],
            "intentos": r["intentos"],
        })

    print(f"\n--- Resumen ---")
    for r in resultados:
        estado = "✓" if r["exito"] else "✗"
        print(f"  {estado} {r['tarea'][:50]} ({r['intentos']} intento(s))")

    print(f"\nVariables en namespace: {agente.estado_namespace}")

    with open("outputs/m27_agent_demo.json", "w", encoding="utf-8") as f:
        json.dump(resultados, f, ensure_ascii=False, indent=2)
    print("\nGuardado: outputs/m27_agent_demo.json")
    return resultados


if __name__ == "__main__":
    demo_code_interpreter()
    print("\n[M27 completado]")
```

---

## Checkpoint

```python
# checkpoint_m27.py — 6 tests para Code Interpreter

import io, ast, sys, contextlib, time, json
from dataclasses import dataclass, field
from typing import Any, Dict, List, Optional, Tuple


@dataclass
class ResultadoEjecucion:
    codigo: str; stdout: str = ""; stderr: str = ""; valor_retornado: Any = None
    error: Optional[str] = None; tiempo_ms: int = 0; exito: bool = True
    variables_nuevas: List[str] = field(default_factory=list)
    def to_dict(self): return {"exito":self.exito,"stdout":self.stdout,"error":self.error,"tiempo_ms":self.tiempo_ms}
    @property
    def texto_completo(self):
        partes = []
        if self.stdout: partes.append(f"[stdout]\n{self.stdout}")
        if self.error: partes.append(f"[error]\n{self.error}")
        if self.valor_retornado is not None and not self.stdout: partes.append(f"[resultado]\n{self.valor_retornado}")
        return "\n".join(partes) if partes else "(sin output)"

class AnalizadorSeguridad:
    MODULOS_PERMITIDOS = {"math","statistics","random","json","re","datetime","collections","itertools","functools","string","typing","dataclasses","enum","pathlib","io","csv"}
    FUNCIONES_BLOQUEADAS = {"exec","eval","compile","__import__","globals","locals"}
    def __init__(self, permitir_archivos=False): self.permitir_archivos = permitir_archivos
    def analizar(self, codigo):
        advertencias = []
        try: arbol = ast.parse(codigo)
        except SyntaxError as e: return False, [f"Error de sintaxis: {e}"]
        for nodo in ast.walk(arbol):
            if isinstance(nodo, (ast.Import, ast.ImportFrom)):
                modulo = ""
                if isinstance(nodo, ast.Import):
                    for alias in nodo.names: modulo = alias.name.split(".")[0]
                elif nodo.module: modulo = nodo.module.split(".")[0]
                if modulo and modulo not in self.MODULOS_PERMITIDOS:
                    advertencias.append(f"Importación no permitida: {modulo}")
            if isinstance(nodo, ast.Call) and isinstance(nodo.func, ast.Name):
                if nodo.func.id in self.FUNCIONES_BLOQUEADAS:
                    if not (nodo.func.id == "open" and self.permitir_archivos):
                        advertencias.append(f"Función bloqueada: {nodo.func.id}")
        return len(advertencias) == 0, advertencias

class SandboxREPL:
    MAX_OUTPUT_CHARS = 10000; TIMEOUT_SEGUNDOS = 10.0
    def __init__(self, permitir_archivos=False):
        self._namespace = {"__builtins__": __builtins__, "print": print}
        self._historial = []; self._analizador = AnalizadorSeguridad(permitir_archivos)
    def ejecutar(self, codigo, timeout=None):
        inicio = time.time()
        es_seguro, advertencias = self._analizador.analizar(codigo)
        if not es_seguro:
            return ResultadoEjecucion(codigo=codigo, error="\n".join(advertencias), exito=False)
        stdout_c = io.StringIO(); stderr_c = io.StringIO(); valor = None; error_msg = None
        vars_previas = set(self._namespace.keys())
        try:
            with contextlib.redirect_stdout(stdout_c), contextlib.redirect_stderr(stderr_c):
                try: valor = eval(compile(ast.parse(codigo, mode="eval"), "<s>", "eval"), self._namespace)
                except SyntaxError: exec(compile(codigo, "<s>", "exec"), self._namespace)
        except Exception:
            import traceback; error_msg = traceback.format_exc()
        t = int((time.time()-inicio)*1000)
        vars_nuevas = [v for v in self._namespace if v not in vars_previas and not v.startswith("_")]
        r = ResultadoEjecucion(codigo=codigo, stdout=stdout_c.getvalue()[:self.MAX_OUTPUT_CHARS],
                               stderr=stderr_c.getvalue()[:self.MAX_OUTPUT_CHARS], valor_retornado=valor,
                               error=error_msg, tiempo_ms=t, exito=error_msg is None, variables_nuevas=vars_nuevas)
        self._historial.append(r); return r
    @property
    def historial(self): return self._historial
    @property
    def variables_disponibles(self): return {k:type(v).__name__ for k,v in self._namespace.items() if not k.startswith("_") and k != "print"}


def test_01_sandbox_ejecuta_codigo_simple():
    """Sandbox ejecuta código Python y captura stdout."""
    repl = SandboxREPL()
    r = repl.ejecutar("print('hola mundo')")
    assert r.exito, "Debe ejecutar exitosamente"
    assert "hola mundo" in r.stdout
    assert r.error is None
    print("✓ test_01 — Sandbox ejecución simple OK")

def test_02_sandbox_captura_valor_retornado():
    """Sandbox captura el valor de la última expresión."""
    repl = SandboxREPL()
    r = repl.ejecutar("2 + 2 * 10")
    assert r.exito
    assert r.valor_retornado == 22
    print("✓ test_02 — Captura valor retornado OK")

def test_03_sandbox_captura_error():
    """Sandbox captura errores sin crashear."""
    repl = SandboxREPL()
    r = repl.ejecutar("1 / 0")
    assert not r.exito
    assert "ZeroDivisionError" in (r.error or "")
    assert r.stdout == ""
    print("✓ test_03 — Captura de error OK")

def test_04_sandbox_estado_persistente():
    """El namespace persiste entre ejecuciones."""
    repl = SandboxREPL()
    repl.ejecutar("x = 42")
    r = repl.ejecutar("x * 2")
    assert r.exito
    assert r.valor_retornado == 84
    print("✓ test_04 — Estado persistente OK")

def test_05_analizador_bloquea_import_peligroso():
    """AnalizadorSeguridad bloquea imports de módulos no permitidos."""
    analizador = AnalizadorSeguridad()
    es_seguro, advertencias = analizador.analizar("import subprocess\nsubprocess.run(['ls'])")
    assert not es_seguro
    assert any("subprocess" in a for a in advertencias)
    print("✓ test_05 — Bloqueo de import peligroso OK")

def test_06_sandbox_registra_variables_nuevas():
    """Sandbox registra las nuevas variables definidas."""
    repl = SandboxREPL()
    r = repl.ejecutar("mi_variable = [1, 2, 3]\notra = 'texto'")
    assert "mi_variable" in r.variables_nuevas or "mi_variable" in repl.variables_disponibles
    assert repl.variables_disponibles.get("mi_variable") == "list"
    print("✓ test_06 — Registro de variables nuevas OK")


if __name__ == "__main__":
    tests = [
        test_01_sandbox_ejecuta_codigo_simple,
        test_02_sandbox_captura_valor_retornado,
        test_03_sandbox_captura_error,
        test_04_sandbox_estado_persistente,
        test_05_analizador_bloquea_import_peligroso,
        test_06_sandbox_registra_variables_nuevas,
    ]
    aprobados = 0
    for test in tests:
        try: test(); aprobados += 1
        except AssertionError as e: print(f"✗ {test.__name__} FALLÓ: {e}")
        except Exception as e: print(f"✗ {test.__name__} ERROR: {type(e).__name__}: {e}")
    print(f"\n{'='*40}\nResultado: {aprobados}/{len(tests)} tests aprobados")
    assert aprobados == len(tests)
    print("✓ Checkpoint M27 completado")
```

---

## Resumen

| Componente | Función | Detalle clave |
|---|---|---|
| `AnalizadorSeguridad` | AST estático | Bloquea antes de ejecutar; whitelist de módulos |
| `SandboxREPL` | Ejecución con captura | `redirect_stdout` + `redirect_stderr` + try/except |
| `ResultadoEjecucion` | Resultado estructurado | stdout, stderr, valor, error, tiempo, variables |
| `CodeInterpreterAgent` | Ciclo iterativo | generar → ejecutar → analizar error → corregir |
| `LLMCodeGenerator` | Generación de código | Usa error anterior para dirigir la corrección |

```python
# Ciclo Code Interpreter:
for intento in range(max_intentos):
    codigo = llm.generar_codigo(tarea, error_anterior)
    resultado = repl.ejecutar(codigo)
    if resultado.exito:
        break                              # ← éxito
    error_anterior = resultado.error       # ← retroalimentación
    analisis = llm.analizar_error(codigo, error_anterior)
    # El LLM usa analisis para generar mejor código en siguiente intento
```
