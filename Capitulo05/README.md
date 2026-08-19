# 1. Práctica 1. Implementar un agente con Function Calling que utilice herramientas externas para consultar información, ejecutar operaciones y colaborar con otros componentes de una solución GenAI.

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 75 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica implementará un agente de soporte basado en Azure OpenAI y Function Calling. El agente decidirá si responde directamente o si debe usar una de tres herramientas controladas: búsqueda en un catálogo FAQ local, cálculo aritmético seguro y consulta simulada del estado de un ticket.

La implementación conservará el control en Python: validará los argumentos con Pydantic, restringirá las expresiones matemáticas mediante AST, limitará el número de iteraciones y registrará cada ejecución en formato JSONL. El archivo de trazas resultante será reutilizado por los evaluadores especializados del laboratorio `06-00-01`.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Implementar un ciclo de orquestación con Azure OpenAI Function Calling.
- [ ] Definir contratos JSON Schema estrictos para herramientas autorizadas.
- [ ] Validar argumentos de herramientas con modelos Pydantic y rechazar propiedades no permitidas.
- [ ] Implementar una calculadora segura sin usar `eval`, `exec` ni comandos del sistema.
- [ ] Registrar conversaciones, llamadas a herramientas, errores y respuestas finales en `runs/function_calling_runs.jsonl`.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de Python, `venv`, JSON y `pytest`.
- Conceptos de APIs REST, mensajes de chat y Function Calling.
- Comprensión del ciclo agentivo: modelo, herramientas, estado, validación y criterio de finalización.
- Laboratorio `04-00-01` completado, con el archivo `data/processed/faq_catalog.json`.

### Acceso y configuración requeridos

Debe contar con:

- Una suscripción de Azure con acceso a Azure OpenAI.
- Un despliegue de modelo de chat compatible con Function Calling.
- Cuota disponible para el modelo configurado.
- Las siguientes variables definidas en `~/genai-agent-labs/.env`:

```dotenv
AZURE_OPENAI_ENDPOINT=https://<su-recurso>.openai.azure.com/
AZURE_OPENAI_API_KEY=<su-clave>
AZURE_OPENAI_DEPLOYMENT=<nombre-del-despliegue>
```

> **Importante:** no agregue valores reales de secretos al repositorio Git. El archivo `.env` debe estar incluido en `.gitignore`.

## Entorno del laboratorio

### Rutas obligatorias

| Recurso | Ruta |
|---|---|
| Directorio de trabajo | `~/genai-agent-labs` |
| Entorno virtual | `~/genai-agent-labs/.venv` |
| Código fuente | `~/genai-agent-labs/src` |
| Datos procesados | `~/genai-agent-labs/data/processed` |
| Datos simulados | `~/genai-agent-labs/data/mock` |
| Escenarios de evaluación | `~/genai-agent-labs/data/evaluation` |
| Resultados de ejecución | `~/genai-agent-labs/runs` |
| Pruebas | `~/genai-agent-labs/tests` |

### Software utilizado

| Componente | Versión objetivo |
|---|---:|
| Python | 3.12.x |
| OpenAI Python SDK | 1.58.1 |
| Pydantic | 2.10.4 |
| pytest | 8.3.4 |
| Azure OpenAI REST API | `2024-10-21` |

### Preparación inicial

1. Abra una terminal y active el entorno virtual compartido del curso:

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate
   ```

2. Confirme las versiones principales:

   ```bash
   python --version
   pip show openai pydantic pytest
   ```

3. Instale o actualice las dependencias requeridas para este laboratorio:

   ```bash
   pip install "openai==1.58.1" "pydantic==2.10.4" "pytest==8.3.4" "python-dotenv==1.0.1"
   ```

4. Cree los directorios necesarios:

   ```bash
   mkdir -p src/agent data/mock data/evaluation runs tests
   touch src/__init__.py src/agent/__init__.py
   ```

5. Verifique que `.env` no será versionado:

   ```bash
   grep -qxF ".env" .gitignore || echo ".env" >> .gitignore
   ```

**Salida esperada**

Los directorios `src/agent`, `data/mock`, `data/evaluation`, `runs` y `tests` existen. El entorno virtual está activo.

**Verificación**

Ejecute:

```bash
test -f .env && echo ".env encontrado"
test -f data/processed/faq_catalog.json && echo "Catálogo FAQ encontrado"
```

Si el segundo comando no confirma la existencia del catálogo, complete primero el laboratorio `04-00-01`.

## Paso a paso

### Paso 1. Inspeccionar y preparar los datos controlados

**Objetivo:** confirmar la estructura del catálogo FAQ y crear la fuente local autorizada para los tickets simulados.

**Instrucciones**

1. Inspeccione las primeras entradas del catálogo generado en el laboratorio anterior:

   ```bash
   python -m json.tool data/processed/faq_catalog.json | head -n 80
   ```

2. El catálogo puede contener objetos con campos como `question`, `answer`, `category`, `id` o variantes equivalentes. La herramienta que implementará buscará de forma tolerante en los campos textuales disponibles, pero solo leerá este archivo local.

3. Cree el archivo controlado de tickets:

   ```bash
   cat > data/mock/tickets.json <<'JSON'
   {
     "TKT-1001": {
       "status": "abierto",
       "priority": "media",
       "summary": "No puedo acceder al portal de soporte."
     },
     "TKT-1002": {
       "status": "en_progreso",
       "priority": "alta",
       "summary": "Error al procesar una solicitud de facturación."
     },
     "TKT-1003": {
       "status": "resuelto",
       "priority": "baja",
       "summary": "Consulta sobre actualización de contraseña."
     }
   }
   JSON
   ```

4. Cree escenarios iniciales de evaluación. Si ya existe un archivo generado por el instructor, revíselo antes de reemplazarlo. Cada escenario debe incluir un identificador y un mensaje de usuario.

   ```bash
   cat > data/evaluation/scenarios.json <<'JSON'
   [
     {
       "id": "faq-001",
       "user_message": "¿Cómo puedo restablecer mi contraseña?",
       "expected_tool": "search_faq"
     },
     {
       "id": "calc-001",
       "user_message": "Calcula (125 * 4) - 30.",
       "expected_tool": "calculate"
     },
     {
       "id": "ticket-001",
       "user_message": "¿Cuál es el estado del ticket TKT-1002?",
       "expected_tool": "get_ticket_status"
     },
     {
       "id": "ticket-002",
       "user_message": "Consulta el estado del ticket TKT-9999.",
       "expected_tool": "get_ticket_status"
     }
   ]
   JSON
   ```

5. Valide ambos archivos JSON:

   ```bash
   python -m json.tool data/mock/tickets.json > /dev/null
   python -m json.tool data/evaluation/scenarios.json > /dev/null
   ```

**Salida esperada**

No se muestran errores de JSON. El archivo `tickets.json` contiene únicamente tickets con identificadores `TKT-` seguidos de cuatro dígitos.

**Verificación**

Ejecute:

```bash
cat data/mock/tickets.json
```

Confirme que no existen rutas, comandos, credenciales ni información externa en este archivo. La herramienta de tickets debe depender exclusivamente de esta fuente local controlada.

---

### Paso 2. Implementar los contratos, validadores y herramientas seguras

**Objetivo:** crear herramientas de mínimo privilegio con JSON Schema estricto, validación Pydantic y cálculo aritmético basado en AST restringido.

**Instrucciones**

1. Cree el archivo `src/agent/function_calling_agent.py`:

   ```bash
   cat > src/agent/function_calling_agent.py <<'PY'
   """Agente de soporte con Azure OpenAI Function Calling y herramientas controladas."""

   from __future__ import annotations

   import ast
   import json
   import os
   import re
   import time
   import uuid
   from datetime import datetime, timezone
   from pathlib import Path
   from typing import Any, Literal

   from openai import AzureOpenAI
   from pydantic import BaseModel, ConfigDict, Field, ValidationError, field_validator


   PROJECT_ROOT = Path(__file__).resolve().parents[2]
   FAQ_PATH = PROJECT_ROOT / "data" / "processed" / "faq_catalog.json"
   TICKETS_PATH = PROJECT_ROOT / "data" / "mock" / "tickets.json"
   RUNS_PATH = PROJECT_ROOT / "runs" / "function_calling_runs.jsonl"

   MAX_TOOL_ITERATIONS = 5
   MAX_EXPRESSION_LENGTH = 120
   TICKET_PATTERN = re.compile(r"^TKT-\d{4}$")


   class StrictModel(BaseModel):
       """Modelo base que rechaza propiedades adicionales."""

       model_config = ConfigDict(extra="forbid", str_strip_whitespace=True)


   class SearchFaqArguments(StrictModel):
       query: str = Field(
           min_length=2,
           max_length=200,
           description="Texto de búsqueda para el catálogo FAQ local.",
       )
       category: str | None = Field(
           default=None,
           min_length=2,
           max_length=80,
           description="Categoría opcional para limitar la búsqueda.",
       )


   class CalculateArguments(StrictModel):
       expression: str = Field(
           min_length=1,
           max_length=MAX_EXPRESSION_LENGTH,
           description="Expresión aritmética con números y operadores permitidos.",
       )

       @field_validator("expression")
       @classmethod
       def reject_unsafe_characters(cls, value: str) -> str:
           if not re.fullmatch(r"[\d\s\+\-\*\/%\(\)\.\*]+", value):
               raise ValueError(
                   "La expresión contiene caracteres no permitidos."
               )
           return value


   class TicketStatusArguments(StrictModel):
       ticket_id: str = Field(
           description="Identificador de ticket con formato TKT-0000."
       )

       @field_validator("ticket_id")
       @classmethod
       def validate_ticket_id(cls, value: str) -> str:
           value = value.upper()
           if not TICKET_PATTERN.fullmatch(value):
               raise ValueError(
                   "ticket_id debe tener exactamente el formato TKT-0000."
               )
           return value


   TOOL_DEFINITIONS: list[dict[str, Any]] = [
       {
           "type": "function",
           "function": {
               "name": "search_faq",
               "description": (
                   "Busca respuestas de soporte únicamente en el catálogo FAQ "
                   "local validado. Úsala para preguntas frecuentes de soporte."
               ),
               "parameters": {
                   "type": "object",
                   "properties": {
                       "query": {
                           "type": "string",
                           "minLength": 2,
                           "maxLength": 200,
                           "description": "Consulta de soporte.",
                       },
                       "category": {
                           "type": ["string", "null"],
                           "minLength": 2,
                           "maxLength": 80,
                           "description": "Categoría opcional.",
                       },
                   },
                   "required": ["query"],
                   "additionalProperties": False,
               },
           },
       },
       {
           "type": "function",
           "function": {
               "name": "calculate",
               "description": (
                   "Calcula una expresión aritmética simple. "
                   "No acepta variables, funciones, nombres ni código."
               ),
               "parameters": {
                   "type": "object",
                   "properties": {
                       "expression": {
                           "type": "string",
                           "minLength": 1,
                           "maxLength": MAX_EXPRESSION_LENGTH,
                           "description": "Expresión como (125 * 4) - 30.",
                       }
                   },
                   "required": ["expression"],
                   "additionalProperties": False,
               },
           },
       },
       {
           "type": "function",
           "function": {
               "name": "get_ticket_status",
               "description": (
                   "Consulta el estado de un ticket en el archivo local "
                   "controlado. Solo admite identificadores TKT-0000."
               ),
               "parameters": {
                   "type": "object",
                   "properties": {
                       "ticket_id": {
                           "type": "string",
                           "pattern": "^TKT-\\d{4}$",
                           "description": "Identificador del ticket.",
                       }
                   },
                   "required": ["ticket_id"],
                   "additionalProperties": False,
               },
           },
       },
   ]


   def load_json(path: Path) -> Any:
       """Carga un archivo JSON local y genera un error controlado si falla."""
       try:
           with path.open("r", encoding="utf-8") as file:
               return json.load(file)
       except FileNotFoundError as error:
           raise RuntimeError(f"No se encontró el archivo requerido: {path}") from error
       except json.JSONDecodeError as error:
           raise RuntimeError(f"JSON inválido en {path}: {error.msg}") from error


   def normalize_text(value: Any) -> str:
       return str(value or "").casefold().strip()


   def search_faq(query: str, category: str | None = None) -> dict[str, Any]:
       """Busca coincidencias simples exclusivamente en el catálogo FAQ local."""
       catalog = load_json(FAQ_PATH)
       entries = catalog.get("items", catalog) if isinstance(catalog, dict) else catalog

       if not isinstance(entries, list):
           return {
               "ok": False,
               "error": "faq_catalog_invalid",
               "message": "El catálogo FAQ no contiene una lista de entradas.",
           }

       query_terms = set(re.findall(r"\w+", normalize_text(query)))
       requested_category = normalize_text(category)
       matches: list[dict[str, Any]] = []

       for index, entry in enumerate(entries):
           if not isinstance(entry, dict):
               continue

           entry_category = normalize_text(entry.get("category"))
           if requested_category and entry_category != requested_category:
               continue

           searchable = " ".join(
               normalize_text(entry.get(key))
               for key in ("question", "answer", "title", "content", "text", "category")
           )
           score = sum(term in searchable for term in query_terms)

           if score > 0:
               matches.append(
                   {
                       "id": entry.get("id", f"faq-{index + 1}"),
                       "question": entry.get("question", entry.get("title", "")),
                       "answer": entry.get("answer", entry.get("content", entry.get("text", ""))),
                       "category": entry.get("category"),
                       "score": score,
                   }
               )

       matches.sort(key=lambda item: item["score"], reverse=True)
       return {
           "ok": True,
           "source": "data/processed/faq_catalog.json",
           "query": query,
           "category": category,
           "results": matches[:3],
           "result_count": len(matches[:3]),
       }


   _ALLOWED_BINARY_OPERATORS = {
       ast.Add: lambda left, right: left + right,
       ast.Sub: lambda left, right: left - right,
       ast.Mult: lambda left, right: left * right,
       ast.Div: lambda left, right: left / right,
       ast.Mod: lambda left, right: left % right,
       ast.Pow: lambda left, right: left**right,
   }

   _ALLOWED_UNARY_OPERATORS = {
       ast.UAdd: lambda value: +value,
       ast.USub: lambda value: -value,
   }


   def evaluate_arithmetic(node: ast.AST) -> int | float:
       """Evalúa exclusivamente nodos aritméticos permitidos."""
       if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
           if isinstance(node.value, bool):
               raise ValueError("Los valores booleanos no están permitidos.")
           return node.value

       if isinstance(node, ast.BinOp) and type(node.op) in _ALLOWED_BINARY_OPERATORS:
           left = evaluate_arithmetic(node.left)
           right = evaluate_arithmetic(node.right)

           if isinstance(node.op, ast.Pow) and abs(right) > 10:
               raise ValueError("El exponente no puede ser mayor que 10.")
           return _ALLOWED_BINARY_OPERATORS[type(node.op)](left, right)

       if isinstance(node, ast.UnaryOp) and type(node.op) in _ALLOWED_UNARY_OPERATORS:
           return _ALLOWED_UNARY_OPERATORS[type(node.op)](evaluate_arithmetic(node.operand))

       raise ValueError("La expresión contiene una operación no permitida.")


   def calculate(expression: str) -> dict[str, Any]:
       """Calcula una expresión mediante AST restringido; nunca usa eval."""
       try:
           parsed = ast.parse(expression, mode="eval")
           result = evaluate_arithmetic(parsed.body)
           if not isinstance(result, (int, float)) or abs(result) > 1_000_000_000:
               raise ValueError("El resultado está fuera del rango permitido.")
           return {"ok": True, "expression": expression, "result": result}
       except ZeroDivisionError:
           return {
               "ok": False,
               "error": "division_by_zero",
               "message": "No es posible dividir entre cero.",
           }
       except (SyntaxError, ValueError, OverflowError) as error:
           return {
               "ok": False,
               "error": "invalid_expression",
               "message": str(error),
           }


   def get_ticket_status(ticket_id: str) -> dict[str, Any]:
       """Consulta un ticket únicamente en el archivo local controlado."""
       tickets = load_json(TICKETS_PATH)

       if not isinstance(tickets, dict):
           return {
               "ok": False,
               "error": "tickets_invalid",
               "message": "El archivo de tickets tiene una estructura inválida.",
           }

       ticket = tickets.get(ticket_id)
       if ticket is None:
           return {
               "ok": False,
               "error": "ticket_not_found",
               "ticket_id": ticket_id,
               "message": "No se encontró un ticket con ese identificador.",
           }

       return {
           "ok": True,
           "ticket_id": ticket_id,
           "status": ticket.get("status"),
           "priority": ticket.get("priority"),
           "summary": ticket.get("summary"),
       }


   TOOL_REGISTRY = {
       "search_faq": (SearchFaqArguments, search_faq),
       "calculate": (CalculateArguments, calculate),
       "get_ticket_status": (TicketStatusArguments, get_ticket_status),
   }


   def execute_tool(name: str, raw_arguments: str) -> dict[str, Any]:
       """Valida y ejecuta una herramienta registrada; nunca ejecuta código arbitrario."""
       if name not in TOOL_REGISTRY:
           return {
               "ok": False,
               "error": "tool_not_authorized",
               "message": f"La herramienta '{name}' no está autorizada.",
           }

       try:
           arguments = json.loads(raw_arguments)
       except json.JSONDecodeError:
           return {
               "ok": False,
               "error": "invalid_tool_arguments",
               "message": "Los argumentos de la herramienta no son JSON válido.",
           }

       model_class, function = TOOL_REGISTRY[name]

       try:
           validated = model_class.model_validate(arguments)
       except ValidationError as error:
           return {
               "ok": False,
               "error": "tool_argument_validation_failed",
               "message": "Los argumentos no cumplen el contrato de la herramienta.",
               "details": error.errors(include_url=False),
           }

       try:
           return function(**validated.model_dump())
       except RuntimeError as error:
           return {
               "ok": False,
               "error": "tool_data_unavailable",
               "message": str(error),
           }
       except Exception:
           return {
               "ok": False,
               "error": "tool_execution_failed",
               "message": "La herramienta no pudo completar la operación.",
           }


   def get_client() -> tuple[AzureOpenAI, str]:
       endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
       api_key = os.getenv("AZURE_OPENAI_API_KEY")
       deployment = os.getenv("AZURE_OPENAI_DEPLOYMENT")

       if not endpoint or not api_key or not deployment:
           raise RuntimeError(
               "Faltan AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_API_KEY o "
               "AZURE_OPENAI_DEPLOYMENT en el entorno."
           )

       return (
           AzureOpenAI(
               azure_endpoint=endpoint,
               api_key=api_key,
               api_version="2024-10-21",
           ),
           deployment,
       )


   def append_run(record: dict[str, Any]) -> None:
       RUNS_PATH.parent.mkdir(parents=True, exist_ok=True)
       with RUNS_PATH.open("a", encoding="utf-8") as file:
           file.write(json.dumps(record, ensure_ascii=False) + "\n")


   def run_agent(user_message: str, scenario_id: str | None = None) -> dict[str, Any]:
       """Ejecuta el ciclo de orquestación con un máximo de cinco iteraciones."""
       client, deployment = get_client()
       run_id = str(uuid.uuid4())
       started = time.perf_counter()
       tool_calls_log: list[dict[str, Any]] = []

       messages: list[dict[str, Any]] = [
           {
               "role": "system",
               "content": (
                   "Eres un asistente de soporte técnico. Usa herramientas cuando "
                   "necesites consultar FAQ, calcular o revisar un ticket. "
                   "No inventes resultados de herramientas. No solicites ni reveles "
                   "secretos. Si una herramienta devuelve error, explícalo de forma "
                   "clara y ofrece el siguiente paso razonable."
               ),
           },
           {"role": "user", "content": user_message},
       ]

       final_answer = (
           "No fue posible completar la solicitud con la información disponible."
       )
       termination_reason = "unknown"

       for iteration in range(1, MAX_TOOL_ITERATIONS + 1):
           response = client.chat.completions.create(
               model=deployment,
               messages=messages,
               tools=TOOL_DEFINITIONS,
               tool_choice="auto",
               temperature=0,
           )

           assistant_message = response.choices[0].message
           assistant_dict = assistant_message.model_dump(exclude_none=True)
           messages.append(assistant_dict)

           if not assistant_message.tool_calls:
               final_answer = assistant_message.content or final_answer
               termination_reason = "final_model_response"
               break

           for tool_call in assistant_message.tool_calls:
               name = tool_call.function.name
               raw_arguments = tool_call.function.arguments
               result = execute_tool(name, raw_arguments)

               tool_calls_log.append(
                   {
                       "iteration": iteration,
                       "tool_call_id": tool_call.id,
                       "name": name,
                       "raw_arguments": raw_arguments,
                       "result": result,
                   }
               )

               messages.append(
                   {
                       "role": "tool",
                       "tool_call_id": tool_call.id,
                       "content": json.dumps(result, ensure_ascii=False),
                   }
               )
       else:
           final_answer = (
               "La solicitud excedió el límite de cinco iteraciones de herramientas. "
               "Intente formular una consulta más específica."
           )
           termination_reason = "max_tool_iterations"

       duration_ms = round((time.perf_counter() - started) * 1000, 2)
       record = {
           "run_id": run_id,
           "scenario_id": scenario_id,
           "timestamp_utc": datetime.now(timezone.utc).isoformat(),
           "model_deployment": deployment,
           "user_message": user_message,
           "tool_calls": tool_calls_log,
           "final_answer": final_answer,
           "termination_reason": termination_reason,
           "duration_ms": duration_ms,
       }
       append_run(record)
       return record


   def run_scenarios(scenarios_path: Path) -> list[dict[str, Any]]:
       scenarios = load_json(scenarios_path)
       if not isinstance(scenarios, list):
           raise RuntimeError("scenarios.json debe contener una lista JSON.")

       results = []
       for scenario in scenarios:
           if not isinstance(scenario, dict) or "user_message" not in scenario:
               raise RuntimeError("Cada escenario debe incluir user_message.")
           results.append(
               run_agent(
                   user_message=str(scenario["user_message"]),
                   scenario_id=str(scenario.get("id", "")) or None,
               )
           )
       return results


   if __name__ == "__main__":
       from dotenv import load_dotenv

       load_dotenv(PROJECT_ROOT / ".env")
       results = run_scenarios(PROJECT_ROOT / "data" / "evaluation" / "scenarios.json")
       print(f"Escenarios ejecutados: {len(results)}")
       print(f"Trazas guardadas en: {RUNS_PATH}")
   PY
   ```

2. Revise los controles implementados:

   ```bash
   grep -nE "additionalProperties|extra=\"forbid\"|ast.parse|MAX_TOOL_ITERATIONS|eval" \
     src/agent/function_calling_agent.py
   ```

3. Confirme específicamente que el código no usa `eval()` ni `exec()`:

   ```bash
   grep -nE '\beval\(|\bexec\(' src/agent/function_calling_agent.py || true
   ```

**Salida esperada**

El primer comando muestra referencias a:

- `additionalProperties: False`
- `extra="forbid"`
- `ast.parse(..., mode="eval")`
- `MAX_TOOL_ITERATIONS = 5`

El segundo comando no debe mostrar llamadas a `eval(` ni `exec(`.

**Verificación**

La función `execute_tool()` es el único punto de entrada para ejecutar herramientas. Debe comprobar:

1. Que el nombre de herramienta exista en `TOOL_REGISTRY`.
2. Que los argumentos sean JSON válido.
3. Que Pydantic acepte los argumentos.
4. Que la función autorizada se ejecute solo después de la validación.

---

### Paso 3. Crear pruebas unitarias para las herramientas y los controles de seguridad

**Objetivo:** validar las herramientas sin depender de una llamada real al modelo de Azure OpenAI.

**Instrucciones**

1. Cree el archivo de pruebas:

   ```bash
   cat > tests/test_function_calling_agent.py <<'PY'
   import json

   import pytest

   from src.agent.function_calling_agent import (
       calculate,
       execute_tool,
       get_ticket_status,
   )


   def test_calculate_accepts_safe_expression():
       result = calculate("(125 * 4) - 30")

       assert result["ok"] is True
       assert result["result"] == 470


   @pytest.mark.parametrize(
       "expression",
       [
           "__import__('os').system('id')",
           "open('/etc/passwd')",
           "1; 2",
           "[x for x in range(3)]",
       ],
   )
   def test_calculate_rejects_unsafe_expression(expression):
       result = calculate(expression)

       assert result["ok"] is False
       assert result["error"] == "invalid_expression"


   def test_calculate_rejects_division_by_zero():
       result = calculate("10 / 0")

       assert result["ok"] is False
       assert result["error"] == "division_by_zero"


   def test_ticket_status_returns_controlled_data():
       result = get_ticket_status("TKT-1002")

       assert result["ok"] is True
       assert result["ticket_id"] == "TKT-1002"
       assert result["status"] == "en_progreso"


   def test_tool_rejects_unknown_name():
       result = execute_tool("delete_ticket", "{}")

       assert result["ok"] is False
       assert result["error"] == "tool_not_authorized"


   def test_ticket_rejects_invalid_identifier():
       result = execute_tool(
           "get_ticket_status",
           json.dumps({"ticket_id": "../../etc/passwd"}),
       )

       assert result["ok"] is False
       assert result["error"] == "tool_argument_validation_failed"


   def test_tool_rejects_extra_properties():
       result = execute_tool(
           "calculate",
           json.dumps(
               {
                   "expression": "2 + 2",
                   "command": "rm -rf /",
               }
           ),
       )

       assert result["ok"] is False
       assert result["error"] == "tool_argument_validation_failed"
   PY
   ```

2. Ejecute las pruebas:

   ```bash
   pytest -q tests/test_function_calling_agent.py
   ```

3. Ejecute una comprobación manual de argumentos válidos e inválidos:

   ```bash
   python - <<'PY'
   import json
   from src.agent.function_calling_agent import execute_tool

   print(execute_tool("calculate", json.dumps({"expression": "20 * 3 + 5"})))
   print(execute_tool("get_ticket_status", json.dumps({"ticket_id": "TKT-1001"})))
   print(execute_tool("get_ticket_status", json.dumps({"ticket_id": "TKT-12"})))
   PY
   ```

**Salida esperada**

La ejecución de `pytest` termina con un resultado similar a:

```text
9 passed
```

La comprobación manual devuelve:

- Un resultado de cálculo con `result: 65`.
- Un resultado válido para `TKT-1001`.
- Un error de validación para `TKT-12`.

**Verificación**

Las pruebas deben demostrar que:

- No se puede invocar una herramienta no registrada.
- No se aceptan propiedades extras.
- No se aceptan identificadores de ticket que no cumplan `TKT-0000`.
- La calculadora rechaza expresiones que intenten invocar funciones, importar módulos o ejecutar código.

---

### Paso 4. Ejecutar el agente con Azure OpenAI y generar trazas JSONL

**Objetivo:** conectar el orquestador con el despliegue Azure OpenAI, procesar los escenarios y registrar la trazabilidad requerida.

**Instrucciones**

1. Confirme que las variables de entorno están disponibles sin imprimir secretos:

   ```bash
   python - <<'PY'
   from dotenv import load_dotenv
   import os

   load_dotenv(".env")

   required = [
       "AZURE_OPENAI_ENDPOINT",
       "AZURE_OPENAI_API_KEY",
       "AZURE_OPENAI_DEPLOYMENT",
   ]

   for name in required:
       print(f"{name}: {'configurada' if os.getenv(name) else 'FALTA'}")
   PY
   ```

2. Elimine trazas anteriores para que la ejecución sea reproducible:

   ```bash
   rm -f runs/function_calling_runs.jsonl
   ```

3. Ejecute los escenarios:

   ```bash
   PYTHONPATH=. python -m src.agent.function_calling_agent
   ```

4. Inspeccione el número de registros generados:

   ```bash
   wc -l runs/function_calling_runs.jsonl
   ```

5. Formatee el primer registro para revisar su estructura:

   ```bash
   head -n 1 runs/function_calling_runs.jsonl | python -m json.tool
   ```

6. Extraiga una vista resumida de los escenarios, herramientas y criterios de finalización:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("runs/function_calling_runs.jsonl")

   for line in path.read_text(encoding="utf-8").splitlines():
       run = json.loads(line)
       tools = [item["name"] for item in run["tool_calls"]]
       print(
           f"scenario={run['scenario_id']}, "
           f"tools={tools}, "
           f"termination={run['termination_reason']}, "
           f"duration_ms={run['duration_ms']}"
       )
   PY
   ```

**Salida esperada**

Debe visualizar un mensaje similar a:

```text
Escenarios ejecutados: 4
Trazas guardadas en: /home/<usuario>/genai-agent-labs/runs/function_calling_runs.jsonl
```

El archivo JSONL contiene un objeto JSON por escenario. Cada objeto incluye al menos:

```json
{
  "run_id": "identificador-uuid",
  "scenario_id": "ticket-001",
  "timestamp_utc": "2026-08-18T...",
  "model_deployment": "nombre-del-despliegue",
  "user_message": "¿Cuál es el estado del ticket TKT-1002?",
  "tool_calls": [
    {
      "iteration": 1,
      "tool_call_id": "call_...",
      "name": "get_ticket_status",
      "raw_arguments": "{\"ticket_id\":\"TKT-1002\"}",
      "result": {
        "ok": true,
        "status": "en_progreso"
      }
    }
  ],
  "final_answer": "El ticket TKT-1002 está en progreso...",
  "termination_reason": "final_model_response",
  "duration_ms": 0.0
}
```

**Verificación**

Confirme las siguientes condiciones:

```bash
python - <<'PY'
import json
from pathlib import Path

path = Path("runs/function_calling_runs.jsonl")
runs = [json.loads(line) for line in path.read_text(encoding="utf-8").splitlines()]

assert len(runs) == 4, "Se esperaban cuatro escenarios."
assert all("run_id" in run for run in runs)
assert all("tool_calls" in run for run in runs)
assert all(len(run["tool_calls"]) <= 5 for run in runs)
assert all(run["final_answer"] for run in runs)

print("Validación estructural de trazas correcta.")
PY
```

> El modelo puede responder directamente a algunas preguntas según su interpretación. Para los escenarios de cálculo y tickets, revise que la herramienta esperada se haya utilizado. Si no ocurrió, ajuste el mensaje de sistema o la descripción de la herramienta sin eliminar las restricciones de seguridad.

---

### Paso 5. Revisar el flujo de orquestación y preparar la entrega

**Objetivo:** comprobar que el agente aplica límites, conserva trazabilidad y no expone secretos.

**Instrucciones**

1. Revise las llamadas de herramientas registradas:

   ```bash
   python - <<'PY'
   import json

   with open("runs/function_calling_runs.jsonl", encoding="utf-8") as file:
       for line in file:
           run = json.loads(line)
           print(f"\nEscenario: {run['scenario_id']}")
           for call in run["tool_calls"]:
               print(f"  Iteración {call['iteration']}: {call['name']}")
               print(f"  Resultado: {call['result'].get('ok')}")
   PY
   ```

2. Compruebe que ningún secreto fue escrito en el código fuente o las trazas:

   ```bash
   grep -RniE "AZURE_OPENAI_API_KEY=|sk-[A-Za-z0-9]|api[_-]?key.{0,20}[A-Za-z0-9]{20,}" \
     src runs data --exclude="*.jsonl" || true
   ```

3. Revise el estado de Git:

   ```bash
   git status
   ```

4. Agregue los artefactos permitidos. No agregue `.env`, entornos virtuales ni archivos temporales no requeridos:

   ```bash
   git add \
     .gitignore \
     src/agent/function_calling_agent.py \
     data/mock/tickets.json \
     data/evaluation/scenarios.json \
     tests/test_function_calling_agent.py \
     runs/function_calling_runs.jsonl
   ```

5. Verifique el contenido del área de preparación:

   ```bash
   git diff --cached --stat
   git diff --cached -- . ':!runs/function_calling_runs.jsonl'
   ```

6. Realice el commit indicado para esta práctica:

   ```bash
   git commit -m "lab-02-00-03"
   ```

**Salida esperada**

Git confirma un nuevo commit que contiene el agente, los datos controlados, las pruebas y las trazas de ejecución.

**Verificación**

Ejecute:

```bash
git log -1 --oneline
git status
```

La primera salida debe mostrar el mensaje `lab-02-00-03`. La segunda debe indicar un árbol de trabajo limpio, salvo archivos locales intencionalmente no versionados.

## Validación y pruebas

Ejecute la validación completa desde la raíz del repositorio:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

pytest -q tests/test_function_calling_agent.py

python - <<'PY'
import json
from pathlib import Path

runs_path = Path("runs/function_calling_runs.jsonl")
assert runs_path.exists(), "No existe el archivo de trazas."

runs = [
    json.loads(line)
    for line in runs_path.read_text(encoding="utf-8").splitlines()
    if line.strip()
]

assert runs, "El archivo JSONL no contiene ejecuciones."

required_run_fields = {
    "run_id",
    "timestamp_utc",
    "user_message",
    "tool_calls",
    "final_answer",
    "termination_reason",
    "duration_ms",
}

for index, run in enumerate(runs, start=1):
    missing = required_run_fields - run.keys()
    assert not missing, f"Registro {index} sin campos: {missing}"
    assert len(run["tool_calls"]) <= 5, f"Registro {index} excede cinco iteraciones."

    for tool_call in run["tool_calls"]:
        assert tool_call["name"] in {
            "search_faq",
            "calculate",
            "get_ticket_status",
        }, f"Herramienta no autorizada: {tool_call['name']}"

print(f"Validación correcta: {len(runs)} trazas listas para evaluación.")
PY
```

Criterios de aceptación:

| Criterio | Resultado esperado |
|---|---|
| Contratos de herramientas | Las tres herramientas incluyen JSON Schema con `additionalProperties: false`. |
| Validación de argumentos | Pydantic rechaza campos adicionales y formatos de ticket inválidos. |
| Seguridad del cálculo | No se utiliza `eval`; el AST admite solo constantes y operadores aritméticos permitidos. |
| Mínimo privilegio | Solo existen herramientas de lectura y cálculo; no hay herramientas de escritura, shell o acceso de red. |
| Límite de ejecución | El ciclo tiene un máximo de cinco iteraciones. |
| Trazabilidad | Existe `runs/function_calling_runs.jsonl` con mensajes, herramientas, resultados, duración y respuesta final. |
| Pruebas | Todas las pruebas de `pytest` finalizan correctamente. |

## Solución de problemas

### Problema 1: Azure OpenAI devuelve un error de autenticación o configuración

**Síntomas**

Al ejecutar el agente aparece un error similar a:

```text
AuthenticationError
```

o:

```text
RuntimeError: Faltan AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_API_KEY o AZURE_OPENAI_DEPLOYMENT en el entorno.
```

**Causa**

Las variables no están definidas en `.env`, el endpoint tiene un formato incorrecto, la clave no pertenece al recurso indicado o el nombre del despliegue no coincide con el despliegue real de Azure OpenAI.

**Solución**

1. Confirme que el archivo se encuentra en la raíz del repositorio:

   ```bash
   ls -la ~/genai-agent-labs/.env
   ```

2. Revise los nombres de variables sin mostrar la clave:

   ```bash
   grep -E "^(AZURE_OPENAI_ENDPOINT|AZURE_OPENAI_DEPLOYMENT)=" .env
   ```

3. Verifique que el endpoint tenga formato similar a:

   ```text
   https://<nombre-recurso>.openai.azure.com/
   ```

4. Compruebe en Azure Portal que `AZURE_OPENAI_DEPLOYMENT` sea el nombre del despliegue, no necesariamente el nombre base del modelo.

5. Vuelva a ejecutar:

   ```bash
   PYTHONPATH=. python -m src.agent.function_calling_agent
   ```

### Problema 2: La calculadora rechaza una expresión aparentemente válida

**Síntomas**

La herramienta devuelve:

```json
{
  "ok": false,
  "error": "invalid_expression"
}
```

para una expresión enviada por el modelo.

**Causa**

La calculadora acepta únicamente números, espacios, paréntesis y los operadores `+`, `-`, `*`, `/`, `%` y `**`. No acepta nombres de variables, funciones como `round()`, comas, notación científica, corchetes ni texto como `resultado de 2 + 2`.

**Solución**

1. Use expresiones estrictamente aritméticas, por ejemplo:

   ```text
   (125 * 4) - 30
   ```

2. No agregue funciones ni variables:

   ```text
   Incorrecto: round(10 / 3, 2)
   Incorrecto: total + impuesto
   Correcto: 10 / 3
   ```

3. Mantenga el contrato estricto. No relaje el validador para aceptar código Python, ya que eso rompería el control de seguridad requerido por el laboratorio.

## Limpieza

1. No elimine los archivos requeridos para evaluación:

   - `src/agent/function_calling_agent.py`
   - `data/mock/tickets.json`
   - `data/evaluation/scenarios.json`
   - `tests/test_function_calling_agent.py`
   - `runs/function_calling_runs.jsonl`

2. Si creó archivos temporales, elimínelos:

   ```bash
   rm -f /tmp/function_calling_debug.json
   ```

3. Desactive el entorno virtual cuando termine:

   ```bash
   deactivate
   ```

4. Confirme el estado final del repositorio:

   ```bash
   cd ~/genai-agent-labs
   git status
   ```

## Resumen

En esta práctica implementó un agente de soporte con Azure OpenAI Function Calling. El modelo puede decidir entre responder o solicitar una herramienta, pero Python conserva el control efectivo mediante un registro explícito de herramientas, contratos JSON Schema, validación Pydantic, manejo de errores y un máximo de cinco iteraciones.

También construyó tres herramientas con privilegios mínimos: una búsqueda sobre un catálogo FAQ local, una calculadora con AST restringido y una consulta simulada de tickets validada por patrón. Finalmente, generó trazas JSONL estructuradas que servirán como entrada para la evaluación automática y para el laboratorio posterior de MCP.

### Recursos opcionales

- [Azure OpenAI chat completions y Function Calling](https://learn.microsoft.com/azure/ai-services/openai/how-to/function-calling)
- [OpenAI Python SDK](https://github.com/openai/openai-python)
- [Documentación de Pydantic](https://docs.pydantic.dev/latest/)
- [Módulo ast de Python](https://docs.python.org/3/library/ast.html)
- [OWASP Top 10 para aplicaciones de LLM](https://genai.owasp.org/llmrisk/)

---

# 2. Práctica 2. Desarrollar un servidor MCP que permita a un agente interactuar de forma segura con sistemas externos y administrar múltiples herramientas disponibles.

## Metadatos

| Atributo | Valor |
|---|---|
| Duración | 75 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica migrará las capacidades de consulta creadas en el laboratorio 05-00-01 a un servidor basado en Model Context Protocol (MCP). El servidor expondrá herramientas reutilizables mediante transporte `stdio`, validará todos los argumentos con Pydantic y aplicará el principio de mínimo privilegio al ofrecer únicamente operaciones de lectura y cálculo controlado.

También construirá un cliente MCP que iniciará el servidor como un subproceso, descubrirá sus herramientas, leerá un recurso de políticas y ejecutará escenarios equivalentes a los del agente de Function Calling anterior. Finalmente, incorporará pruebas automatizadas para validar descubrimiento, entradas inválidas, ausencia de secretos y rechazo de operaciones no permitidas.

## Objetivos de aprendizaje

- [ ] Implementar un servidor MCP con contratos explícitos para `search_faq`, `calculate` y `get_ticket_status`.
- [ ] Reutilizar capacidades locales del laboratorio 05-00-01 mediante un módulo de herramientas independiente del transporte MCP.
- [ ] Validar tipos, rangos, patrones, categorías permitidas y operaciones matemáticas antes de ejecutar una herramienta.
- [ ] Construir un cliente MCP con transporte `stdio` que descubra y consuma herramientas y recursos.
- [ ] Generar evidencia reproducible en `runs/mcp_runs.jsonl` y ejecutar pruebas de seguridad y comportamiento.

## Prerrequisitos

### Conocimientos requeridos

- Laboratorio **05-00-01** completado, con herramientas locales funcionales para FAQ, cálculo y consulta de tickets.
- Comprensión básica de Python, procesos hijo, JSON, JSON-RPC y Function Calling.
- Familiaridad con modelos Pydantic, pruebas con `pytest` y variables de entorno.
- Comprensión del principio de mínimo privilegio: una herramienta expone solo las capacidades estrictamente necesarias.

### Acceso y archivos requeridos

- Repositorio local `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Archivo `data/processed/faq_catalog.json` generado o validado en el laboratorio 05-00-01.
- Archivo `.env` local configurado si fue requerido por laboratorios anteriores. Esta práctica no debe leer ni mostrar secretos.
- Permisos de escritura sobre el repositorio local para crear código, pruebas y evidencias en `runs/`.

> **Importante:** MCP con transporte `stdio` usa la salida estándar para los mensajes del protocolo. El servidor no debe escribir mensajes de depuración, `print()` ni logs informativos en `stdout`; cualquier log técnico debe dirigirse a `stderr`.

## Entorno del laboratorio

### Software utilizado

| Componente | Versión objetivo |
|---|---:|
| Python | 3.12.1 |
| MCP Python SDK | 1.2.0 |
| Pydantic | 2.10.4 |
| pytest | 8.3.4 |
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS |
| Transporte MCP | `stdio` |
| Formato de mensajes | JSON-RPC |

### Estructura esperada

Al finalizar, la estructura relevante será similar a la siguiente:

```text
~/genai-agent-labs/
├── data/
│   └── processed/
│       └── faq_catalog.json
├── runs/
│   ├── function_calling_reference.jsonl
│   └── mcp_runs.jsonl
├── src/
│   ├── __init__.py
│   ├── tools/
│   │   ├── __init__.py
│   │   └── support_tools.py
│   ├── mcp_server/
│   │   ├── __init__.py
│   │   └── support_mcp_server.py
│   └── mcp_client/
│       ├── __init__.py
│       └── mcp_agent_client.py
└── tests/
    ├── __init__.py
    └── test_mcp_support_server.py
```

### Preparación inicial

1. Abra una terminal y active el entorno virtual compartido.

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate
   ```

2. Confirme la versión de Python y actualice las dependencias específicas del laboratorio.

   ```bash
   python --version
   python -m pip install --upgrade pip
   python -m pip install "mcp==1.2.0" "pydantic==2.10.4" "pytest==8.3.4"
   ```

3. Cree los directorios y archivos de paquete necesarios.

   ```bash
   mkdir -p src/tools src/mcp_server src/mcp_client tests runs
   touch src/__init__.py
   touch src/tools/__init__.py
   touch src/mcp_server/__init__.py
   touch src/mcp_client/__init__.py
   touch tests/__init__.py
   ```

4. Verifique que el catálogo FAQ existe y contiene JSON válido.

   ```bash
   test -f data/processed/faq_catalog.json && python -m json.tool data/processed/faq_catalog.json > /dev/null
   ```

**Salida esperada**

```text
Python 3.12.x
```

El último comando no debe producir salida ni errores.

**Verificación**

Ejecute:

```bash
python -c "import mcp, pydantic; print('MCP y Pydantic disponibles')"
```

Debe obtener:

```text
MCP y Pydantic disponibles
```

## Procedimiento paso a paso

### Paso 1. Revisar el catálogo de FAQ y definir el límite de seguridad

**Objetivo:** confirmar que el servidor usará una fuente de datos fija, de solo lectura y ubicada dentro del repositorio.

**Instrucciones**

1. Inspeccione la estructura del archivo FAQ sin revelar información ajena al laboratorio.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("data/processed/faq_catalog.json")
   records = json.loads(path.read_text(encoding="utf-8"))

   print(f"Tipo raíz: {type(records).__name__}")
   print(f"Total de registros: {len(records)}")
   if records:
       print("Campos del primer registro:", sorted(records[0].keys()))
       print("Categoría del primer registro:", records[0].get("category"))
   PY
   ```

2. El catálogo debe contener una lista de objetos con una estructura equivalente a la siguiente:

   ```json
   [
     {
       "id": "FAQ-001",
       "category": "technical",
       "question": "¿Cómo restablezco mi contraseña?",
       "answer": "Use el flujo de recuperación de contraseña.",
       "keywords": ["contraseña", "acceso", "recuperación"]
     }
   ]
   ```

3. Si los nombres de campos de su catálogo son distintos, adapte el archivo durante este paso o ajuste el mapeo de carga en el módulo de herramientas. Para este laboratorio se usarán las categorías permitidas:

   ```text
   technical
   account
   billing
   general
   ```

4. Cree o reemplace el archivo `.gitignore` con reglas mínimas para impedir el versionado de secretos, cachés y evidencia temporal local.

   ```bash
   cat >> .gitignore <<'EOF'

   # Secretos y configuración local
   .env
   .env.*
   !.env.example

   # Python
   __pycache__/
   .pytest_cache/
   *.py[cod]

   # Resultados temporales locales
   runs/*.tmp
   EOF
   ```

5. No copie valores de `.env` a código, comentarios, pruebas, respuestas de herramientas ni archivos JSONL.

**Salida esperada**

El catálogo se identifica como una lista y muestra al menos un registro. El directorio `data/processed/` será la única ubicación desde la que la herramienta FAQ leerá datos.

**Verificación**

Confirme que no hay secretos visibles en los archivos que se crearán posteriormente:

```bash
grep -RInE '(AZURE_OPENAI_API_KEY|OPENAI_API_KEY|ANTHROPIC_API_KEY|api[_-]?key\s*=)' \
  src tests runs 2>/dev/null || true
```

En este momento no debe aparecer ningún resultado.

---

### Paso 2. Implementar las herramientas locales con contratos Pydantic

**Objetivo:** separar la lógica de negocio del transporte MCP para poder reutilizar las mismas capacidades que usaba el agente de Function Calling en 05-00-01.

**Instrucciones**

1. Cree el archivo `src/tools/support_tools.py`.

2. Pegue el siguiente código. Si en 05-00-01 ya dispone de funciones equivalentes, conserve su lógica de dominio siempre que mantenga los contratos, validaciones y respuestas seguras de este laboratorio.

   ```python
   from __future__ import annotations

   import json
   import math
   import re
   from pathlib import Path
   from typing import Literal

   from pydantic import (
       BaseModel,
       ConfigDict,
       Field,
       ValidationError,
       field_validator,
   )

   PROJECT_ROOT = Path(__file__).resolve().parents[2]
   FAQ_CATALOG_PATH = PROJECT_ROOT / "data" / "processed" / "faq_catalog.json"

   ALLOWED_CATEGORIES = {"technical", "account", "billing", "general"}
   TICKET_PATTERN = re.compile(r"^TCK-\d{4,8}$")
   QUERY_PATTERN = re.compile(
       r"^[A-Za-zÁÉÍÓÚÜÑáéíóúüñ0-9\s.,!?¿¡()/_-]+$"
   )

   # Datos simulados de solo lectura. No existe ninguna función de actualización.
   TICKETS: dict[str, dict[str, str]] = {
       "TCK-1001": {
           "status": "open",
           "priority": "high",
           "summary": "No se puede iniciar sesión.",
       },
       "TCK-1002": {
           "status": "in_progress",
           "priority": "medium",
           "summary": "Error de facturación en revisión.",
       },
       "TCK-1003": {
           "status": "resolved",
           "priority": "low",
           "summary": "Solicitud de información resuelta.",
       },
   }


   class FAQRecord(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       id: str = Field(min_length=1, max_length=64)
       category: Literal["technical", "account", "billing", "general"]
       question: str = Field(min_length=1, max_length=500)
       answer: str = Field(min_length=1, max_length=2000)
       keywords: list[str] = Field(default_factory=list, max_length=20)


   class SearchFAQInput(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       query: str = Field(min_length=2, max_length=120)
       category: Literal["technical", "account", "billing", "general"] | None = None
       max_results: int = Field(default=3, ge=1, le=5)

       @field_validator("query")
       @classmethod
       def validate_query(cls, value: str) -> str:
           normalized = " ".join(value.strip().split())
           if not QUERY_PATTERN.fullmatch(normalized):
               raise ValueError("La consulta contiene caracteres no permitidos.")
           return normalized


   class CalculateInput(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       operation: Literal["add", "subtract", "multiply", "divide"]
       left: float
       right: float

       @field_validator("left", "right", mode="before")
       @classmethod
       def validate_number_type(cls, value: object) -> object:
           if isinstance(value, bool) or not isinstance(value, (int, float)):
               raise ValueError("Los operandos deben ser números JSON.")
           if not math.isfinite(float(value)):
               raise ValueError("Los operandos deben ser números finitos.")
           if abs(float(value)) > 1_000_000:
               raise ValueError("Cada operando debe estar entre -1000000 y 1000000.")
           return value


   class TicketStatusInput(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       ticket_id: str = Field(min_length=8, max_length=12)

       @field_validator("ticket_id")
       @classmethod
       def validate_ticket_id(cls, value: str) -> str:
           normalized = value.strip().upper()
           if not TICKET_PATTERN.fullmatch(normalized):
               raise ValueError(
                   "ticket_id debe tener el formato TCK- seguido de 4 a 8 dígitos."
               )
           return normalized


   def _safe_validation_error(error: ValidationError) -> dict[str, object]:
       """Devuelve un error estable sin reflejar datos sensibles de entrada."""
       return {
           "ok": False,
           "error": "invalid_arguments",
           "message": "Los argumentos no cumplen el contrato de la herramienta.",
           "validation_errors": [
               {
                   "field": ".".join(str(part) for part in item["loc"]),
                   "type": item["type"],
               }
               for item in error.errors()
           ],
       }


   def _load_faq_catalog() -> list[FAQRecord]:
       """Lee solamente el catálogo fijo permitido por el laboratorio."""
       if not FAQ_CATALOG_PATH.is_file():
           raise RuntimeError("El catálogo FAQ permitido no está disponible.")

       raw_data = json.loads(FAQ_CATALOG_PATH.read_text(encoding="utf-8"))
       if not isinstance(raw_data, list):
           raise RuntimeError("El catálogo FAQ debe ser una lista JSON.")

       return [FAQRecord.model_validate(item) for item in raw_data]


   def search_faq(arguments: dict[str, object]) -> dict[str, object]:
       """Busca preguntas frecuentes en el catálogo local de solo lectura."""
       try:
           request = SearchFAQInput.model_validate(arguments)
       except ValidationError as error:
           return _safe_validation_error(error)

       try:
           records = _load_faq_catalog()
       except (OSError, json.JSONDecodeError, ValidationError, RuntimeError):
           return {
               "ok": False,
               "error": "catalog_unavailable",
               "message": "El catálogo FAQ no está disponible.",
           }

       query_terms = set(request.query.casefold().split())
       candidates: list[tuple[int, FAQRecord]] = []

       for record in records:
           if request.category is not None and record.category != request.category:
               continue

           searchable_text = " ".join(
               [record.question, record.answer, *record.keywords]
           ).casefold()
           score = sum(term in searchable_text for term in query_terms)

           if score > 0:
               candidates.append((score, record))

       candidates.sort(key=lambda item: (-item[0], item[1].id))

       results = [
           {
               "id": record.id,
               "category": record.category,
               "question": record.question,
               "answer": record.answer,
           }
           for _, record in candidates[: request.max_results]
       ]

       return {
           "ok": True,
           "query": request.query,
           "category": request.category,
           "results": results,
       }


   def calculate(arguments: dict[str, object]) -> dict[str, object]:
       """Ejecuta exclusivamente operaciones aritméticas permitidas."""
       try:
           request = CalculateInput.model_validate(arguments)
       except ValidationError as error:
           return _safe_validation_error(error)

       if request.operation == "divide" and request.right == 0:
           return {
               "ok": False,
               "error": "invalid_arguments",
               "message": "No se permite la división por cero.",
           }

       operations = {
           "add": request.left + request.right,
           "subtract": request.left - request.right,
           "multiply": request.left * request.right,
           "divide": request.left / request.right,
       }

       return {
           "ok": True,
           "operation": request.operation,
           "left": request.left,
           "right": request.right,
           "result": operations[request.operation],
       }


   def get_ticket_status(arguments: dict[str, object]) -> dict[str, object]:
       """Consulta un ticket en una fuente local de solo lectura."""
       try:
           request = TicketStatusInput.model_validate(arguments)
       except ValidationError as error:
           return _safe_validation_error(error)

       ticket = TICKETS.get(request.ticket_id)
       if ticket is None:
           return {
               "ok": False,
               "error": "ticket_not_found",
               "message": "No existe un ticket con el identificador indicado.",
           }

       return {
           "ok": True,
           "ticket_id": request.ticket_id,
           "status": ticket["status"],
           "priority": ticket["priority"],
           "summary": ticket["summary"],
       }
   ```

3. Analice los controles implementados:

   - `extra="forbid"` rechaza campos inesperados.
   - `strict=True` evita conversiones implícitas de cadenas a tipos válidos.
   - `max_results` está limitado entre 1 y 5.
   - El identificador de ticket solo admite el patrón `TCK-` seguido de 4 a 8 dígitos.
   - El cálculo solo permite cuatro operaciones predefinidas.
   - No se recibe una ruta de archivo como argumento; el catálogo está fijado en `FAQ_CATALOG_PATH`.
   - No existe `create_ticket`, `update_ticket`, `delete_ticket`, ejecución de comandos ni escritura de archivos.

4. Ejecute pruebas manuales de las funciones locales.

   ```bash
   python - <<'PY'
   from src.tools.support_tools import calculate, get_ticket_status, search_faq

   print(calculate({"operation": "multiply", "left": 6, "right": 7}))
   print(get_ticket_status({"ticket_id": "TCK-1001"}))
   print(search_faq({"query": "contraseña", "category": "technical"}))
   print(calculate({"operation": "divide", "left": 10, "right": 0}))
   PY
   ```

**Salida esperada**

Debe observar:

- Un resultado correcto para `6 × 7`.
- La información de solo lectura de `TCK-1001`.
- Una lista de resultados FAQ, si el catálogo contiene contenido coincidente.
- Un error estructurado `invalid_arguments` para división por cero.

**Verificación**

Ejecute una entrada deliberadamente inválida:

```bash
python - <<'PY'
from src.tools.support_tools import get_ticket_status

print(get_ticket_status({
    "ticket_id": "../../etc/passwd",
    "unexpected_field": "not_allowed"
}))
PY
```

La respuesta debe contener:

```text
"ok": False
"error": "invalid_arguments"
```

No debe intentar abrir ningún archivo ni incluir una ruta real del sistema.

---

### Paso 3. Crear el servidor MCP seguro con transporte `stdio`

**Objetivo:** exponer las herramientas locales mediante MCP, con descripciones claras, contratos derivados de tipos y un recurso de políticas de solo lectura.

**Instrucciones**

1. Cree el archivo `src/mcp_server/support_mcp_server.py`.

2. Agregue el siguiente código:

   ```python
   from __future__ import annotations

   import logging
   import sys
   from typing import Literal

   from mcp.server.fastmcp import FastMCP
   from pydantic import BaseModel, ConfigDict, Field

   from src.tools.support_tools import (
       calculate as calculate_local,
       get_ticket_status as get_ticket_status_local,
       search_faq as search_faq_local,
   )

   # stdio está reservado para JSON-RPC; los logs técnicos van a stderr.
   logging.basicConfig(
       level=logging.WARNING,
       stream=sys.stderr,
       format="%(asctime)s %(levelname)s %(name)s %(message)s",
   )

   mcp = FastMCP(
       "support-mcp",
       instructions=(
           "Servidor de soporte de solo lectura. Use únicamente las herramientas "
           "publicadas y respete las políticas disponibles en support://policies."
       ),
   )


   class SearchFAQArguments(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       query: str = Field(
           min_length=2,
           max_length=120,
           description="Texto de búsqueda sin caracteres de control.",
       )
       category: Literal["technical", "account", "billing", "general"] | None = Field(
           default=None,
           description="Categoría opcional permitida.",
       )
       max_results: int = Field(
           default=3,
           ge=1,
           le=5,
           description="Cantidad máxima de resultados, entre 1 y 5.",
       )


   class CalculateArguments(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       operation: Literal["add", "subtract", "multiply", "divide"]
       left: float = Field(ge=-1_000_000, le=1_000_000)
       right: float = Field(ge=-1_000_000, le=1_000_000)


   class TicketStatusArguments(BaseModel):
       model_config = ConfigDict(extra="forbid", strict=True)

       ticket_id: str = Field(
           min_length=8,
           max_length=12,
           description="Identificador con formato TCK- seguido de 4 a 8 dígitos.",
       )


   @mcp.tool(
       name="search_faq",
       description=(
           "Busca respuestas en el catálogo FAQ local de solo lectura. "
           "No acepta rutas de archivos ni realiza búsquedas fuera del catálogo permitido."
       ),
   )
   def search_faq(
       query: str,
       category: Literal["technical", "account", "billing", "general"] | None = None,
       max_results: int = 3,
   ) -> dict[str, object]:
       """Recupera preguntas frecuentes relevantes por texto y categoría."""
       arguments = SearchFAQArguments(
           query=query,
           category=category,
           max_results=max_results,
       )
       return search_faq_local(arguments.model_dump())


   @mcp.tool(
       name="calculate",
       description=(
           "Calcula add, subtract, multiply o divide con dos números finitos. "
           "No evalúa expresiones, código ni comandos."
       ),
   )
   def calculate(
       operation: Literal["add", "subtract", "multiply", "divide"],
       left: float,
       right: float,
   ) -> dict[str, object]:
       """Ejecuta una operación aritmética incluida explícitamente en el contrato."""
       arguments = CalculateArguments(
           operation=operation,
           left=left,
           right=right,
       )
       return calculate_local(arguments.model_dump())


   @mcp.tool(
       name="get_ticket_status",
       description=(
           "Consulta el estado de un ticket de soporte existente. "
           "La operación es de solo lectura y no permite modificar tickets."
       ),
   )
   def get_ticket_status(ticket_id: str) -> dict[str, object]:
       """Devuelve estado, prioridad y resumen de un ticket permitido."""
       arguments = TicketStatusArguments(ticket_id=ticket_id)
       return get_ticket_status_local(arguments.model_dump())


   @mcp.resource(
       "support://policies",
       name="Políticas operativas del agente de soporte",
       description="Políticas de seguridad y operación que el cliente debe consultar.",
   )
   def support_policies() -> str:
       """Expone políticas estáticas de solo lectura para agentes MCP."""
       return """# Políticas operativas de soporte

   1. Solo están autorizadas las herramientas search_faq, calculate y get_ticket_status.
   2. El servidor no ofrece creación, actualización, eliminación ni cierre de tickets.
   3. search_faq solo lee data/processed/faq_catalog.json; no recibe rutas de archivo.
   4. calculate solo admite add, subtract, multiply y divide con números finitos.
   5. Los secretos, variables de entorno y configuraciones privadas no deben solicitarse,
      registrarse ni devolverse en respuestas.
   6. Ante argumentos inválidos o una operación no autorizada, el cliente debe informar
      el error sin intentar una alternativa con privilegios mayores.
   """


   if __name__ == "__main__":
       mcp.run(transport="stdio")
   ```

3. Observe el diseño de seguridad:

   - El proceso publica solamente tres herramientas.
   - El transporte es `stdio`; no se inicia un puerto HTTP ni se expone un servicio de red.
   - `support://policies` es un recurso estático de lectura.
   - Los argumentos de cada herramienta se reconstruyen mediante modelos Pydantic en el límite MCP.
   - El módulo del servidor no carga `.env`, no consulta proveedores de IA y no devuelve variables de entorno.
   - El servidor no acepta nombres de herramientas arbitrarios: MCP solo descubrirá las decoradas con `@mcp.tool`.

4. Valide que el módulo se puede importar sin iniciar el servidor.

   ```bash
   python -c "from src.mcp_server.support_mcp_server import mcp; print(mcp.name)"
   ```

**Salida esperada**

```text
support-mcp
```

**Verificación**

Compruebe que el archivo del servidor no contiene instrucciones inseguras obvias:

```bash
grep -nE 'os\.system|subprocess|eval\(|exec\(|open\(.+request|Path\(.+request|print\(' \
  src/mcp_server/support_mcp_server.py || true
```

No debe aparecer ninguna coincidencia.

> No ejecute manualmente `python -m src.mcp_server.support_mcp_server` en una terminal para inspeccionarlo: el proceso esperará mensajes JSON-RPC por `stdin`. El cliente del siguiente paso es quien debe iniciarlo y comunicarse con él.

---

### Paso 4. Construir el cliente MCP y registrar ejecuciones

**Objetivo:** iniciar el servidor como proceso hijo, descubrir herramientas y recursos, ejecutar escenarios controlados y persistir evidencia comparable con el agente anterior.

**Instrucciones**

1. Cree el archivo `src/mcp_client/mcp_agent_client.py`.

2. Agregue el siguiente código:

   ```python
   from __future__ import annotations

   import asyncio
   import json
   import os
   import sys
   from datetime import UTC, datetime
   from pathlib import Path
   from typing import Any

   from mcp import ClientSession, StdioServerParameters
   from mcp.client.stdio import stdio_client

   from src.tools.support_tools import calculate, get_ticket_status, search_faq

   PROJECT_ROOT = Path(__file__).resolve().parents[2]
   RUNS_PATH = PROJECT_ROOT / "runs" / "mcp_runs.jsonl"

   SCENARIOS: list[dict[str, Any]] = [
       {
           "scenario_id": "faq_technical",
           "tool_name": "search_faq",
           "arguments": {
               "query": "contraseña",
               "category": "technical",
               "max_results": 3,
           },
       },
       {
           "scenario_id": "calculate_multiply",
           "tool_name": "calculate",
           "arguments": {
               "operation": "multiply",
               "left": 6,
               "right": 7,
           },
       },
       {
           "scenario_id": "ticket_known",
           "tool_name": "get_ticket_status",
           "arguments": {
               "ticket_id": "TCK-1001",
           },
       },
       {
           "scenario_id": "ticket_unknown",
           "tool_name": "get_ticket_status",
           "arguments": {
               "ticket_id": "TCK-9999",
           },
       },
   ]


   def run_local_reference(tool_name: str, arguments: dict[str, Any]) -> dict[str, Any]:
       """Representa la salida de referencia de las herramientas de 05-00-01."""
       local_tools = {
           "search_faq": search_faq,
           "calculate": calculate,
           "get_ticket_status": get_ticket_status,
       }
       return local_tools[tool_name](arguments)


   def decode_tool_result(result: Any) -> dict[str, Any]:
       """Extrae el objeto JSON serializado por una respuesta MCP de herramienta."""
       if getattr(result, "isError", False):
           return {
               "ok": False,
               "error": "mcp_tool_error",
               "message": "El servidor informó un error de herramienta.",
           }

       for content in result.content:
           text = getattr(content, "text", None)
           if text is not None:
               decoded = json.loads(text)
               if isinstance(decoded, dict):
                   return decoded

       return {
           "ok": False,
           "error": "invalid_mcp_response",
           "message": "La respuesta MCP no contenía un objeto JSON.",
       }


   def append_jsonl(record: dict[str, Any]) -> None:
       RUNS_PATH.parent.mkdir(parents=True, exist_ok=True)
       with RUNS_PATH.open("a", encoding="utf-8") as file:
           file.write(json.dumps(record, ensure_ascii=False, sort_keys=True) + "\n")


   async def execute_scenarios() -> list[dict[str, Any]]:
       """Ejecuta escenarios mediante un subproceso MCP con stdio."""
       server_params = StdioServerParameters(
           command=sys.executable,
           args=["-m", "src.mcp_server.support_mcp_server"],
           env={
               "PYTHONPATH": str(PROJECT_ROOT),
               "PATH": os.environ.get("PATH", ""),
           },
       )

       records: list[dict[str, Any]] = []

       async with stdio_client(server_params) as (read_stream, write_stream):
           async with ClientSession(read_stream, write_stream) as session:
               await session.initialize()

               tools_response = await session.list_tools()
               discovered_tools = sorted(tool.name for tool in tools_response.tools)

               resources_response = await session.list_resources()
               discovered_resources = sorted(
                   str(resource.uri) for resource in resources_response.resources
               )

               policy_response = await session.read_resource("support://policies")
               policy_text = policy_response.contents[0].text

               for scenario in SCENARIOS:
                   started = datetime.now(UTC)
                   mcp_response = await session.call_tool(
                       scenario["tool_name"],
                       arguments=scenario["arguments"],
                   )
                   mcp_output = decode_tool_result(mcp_response)
                   reference_output = run_local_reference(
                       scenario["tool_name"],
                       scenario["arguments"],
                   )

                   record = {
                       "timestamp_utc": started.isoformat(),
                       "transport": "stdio",
                       "server_name": "support-mcp",
                       "scenario_id": scenario["scenario_id"],
                       "tool_name": scenario["tool_name"],
                       "arguments": scenario["arguments"],
                       "reference_source": "function_calling_05-00-01",
                       "reference_output": reference_output,
                       "mcp_output": mcp_output,
                       "matches_reference": mcp_output == reference_output,
                       "discovered_tools": discovered_tools,
                       "discovered_resources": discovered_resources,
                       "policy_resource_read": "Solo están autorizadas las herramientas"
                       in policy_text,
                   }
                   append_jsonl(record)
                   records.append(record)

       return records


   async def main() -> None:
       records = await execute_scenarios()
       matches = sum(record["matches_reference"] for record in records)

       print(f"Escenarios ejecutados: {len(records)}")
       print(f"Coincidencias con referencia: {matches}/{len(records)}")
       print(f"Evidencia generada: {RUNS_PATH}")


   if __name__ == "__main__":
       asyncio.run(main())
   ```

3. La comparación se realiza contra las mismas funciones locales que daban soporte al agente de Function Calling del laboratorio 05-00-01. Esto verifica que el cambio de interfaz, de Function Calling local a MCP, no alteró el resultado funcional esperado.

4. Ejecute el cliente desde la raíz del repositorio.

   ```bash
   cd ~/genai-agent-labs
   python -m src.mcp_client.mcp_agent_client
   ```

5. Inspeccione la evidencia generada.

   ```bash
   cat runs/mcp_runs.jsonl
   ```

**Salida esperada**

La salida debe ser equivalente a:

```text
Escenarios ejecutados: 4
Coincidencias con referencia: 4/4
Evidencia generada: /home/<usuario>/genai-agent-labs/runs/mcp_runs.jsonl
```

El archivo JSONL debe contener un objeto JSON por escenario, incluyendo:

- `scenario_id`
- `tool_name`
- `arguments`
- `reference_output`
- `mcp_output`
- `matches_reference`
- `discovered_tools`
- `discovered_resources`
- `policy_resource_read`

**Verificación**

Compruebe descubrimiento, coincidencias y recurso:

```bash
python - <<'PY'
import json
from pathlib import Path

records = [
    json.loads(line)
    for line in Path("runs/mcp_runs.jsonl").read_text(encoding="utf-8").splitlines()
]

assert records, "No se generaron registros MCP."
assert all(item["matches_reference"] for item in records)
assert all(
    item["discovered_tools"] == ["calculate", "get_ticket_status", "search_faq"]
    for item in records
)
assert all("support://policies" in item["discovered_resources"] for item in records)
assert all(item["policy_resource_read"] for item in records)

print("Evidencia MCP válida.")
PY
```

Debe obtener:

```text
Evidencia MCP válida.
```

---

### Paso 5. Crear pruebas de descubrimiento, validación y seguridad

**Objetivo:** automatizar la validación de los controles requeridos para que los resultados puedan incorporarse posteriormente al conjunto de evaluación del laboratorio 06-00-01.

**Instrucciones**

1. Cree el archivo `tests/test_mcp_support_server.py`.

2. Agregue el siguiente contenido:

   ```python
   from __future__ import annotations

   import json
   import os
   import sys
   from pathlib import Path

   import pytest
   from mcp import ClientSession, StdioServerParameters
   from mcp.client.stdio import stdio_client

   from src.tools.support_tools import calculate, get_ticket_status, search_faq

   PROJECT_ROOT = Path(__file__).resolve().parents[1]


   @pytest.fixture
   def anyio_backend():
       return "asyncio"


   def decode_tool_result(result):
       for content in result.content:
           text = getattr(content, "text", None)
           if text is not None:
               return json.loads(text)
       raise AssertionError("La respuesta MCP no incluyó contenido JSON.")


   @pytest.mark.anyio
   async def test_discovery_exposes_only_allowed_tools_and_policy_resource():
       params = StdioServerParameters(
           command=sys.executable,
           args=["-m", "src.mcp_server.support_mcp_server"],
           env={
               "PYTHONPATH": str(PROJECT_ROOT),
               "PATH": os.environ.get("PATH", ""),
           },
       )

       async with stdio_client(params) as (read_stream, write_stream):
           async with ClientSession(read_stream, write_stream) as session:
               await session.initialize()

               tools = await session.list_tools()
               tool_names = {tool.name for tool in tools.tools}

               resources = await session.list_resources()
               resource_uris = {str(resource.uri) for resource in resources.resources}

               assert tool_names == {
                   "search_faq",
                   "calculate",
                   "get_ticket_status",
               }
               assert resource_uris == {"support://policies"}


   def test_invalid_ticket_pattern_is_rejected():
       result = get_ticket_status({"ticket_id": "../../etc/passwd"})

       assert result["ok"] is False
       assert result["error"] == "invalid_arguments"


   def test_calculate_rejects_division_by_zero():
       result = calculate(
           {
               "operation": "divide",
               "left": 10,
               "right": 0,
           }
       )

       assert result["ok"] is False
       assert result["error"] == "invalid_arguments"


   def test_search_faq_rejects_unexpected_field_and_path_like_query():
       result = search_faq(
           {
               "query": "../secret.env",
               "max_results": 99,
               "path": "/etc/passwd",
           }
       )

       assert result["ok"] is False
       assert result["error"] == "invalid_arguments"


   def test_unknown_ticket_returns_controlled_tool_error():
       result = get_ticket_status({"ticket_id": "TCK-9999"})

       assert result["ok"] is False
       assert result["error"] == "ticket_not_found"
       assert "TCK-9999" not in result["message"]


   def test_responses_do_not_include_environment_secrets(monkeypatch):
       monkeypatch.setenv("AZURE_OPENAI_API_KEY", "lab-secret-value")
       monkeypatch.setenv("ANTHROPIC_API_KEY", "another-secret-value")

       responses = [
           calculate({"operation": "add", "left": 1, "right": 2}),
           get_ticket_status({"ticket_id": "TCK-1001"}),
           search_faq({"query": "contraseña", "max_results": 1}),
       ]

       serialized = json.dumps(responses, ensure_ascii=False)
       assert "lab-secret-value" not in serialized
       assert "another-secret-value" not in serialized
       assert "AZURE_OPENAI_API_KEY" not in serialized
       assert "ANTHROPIC_API_KEY" not in serialized


   def test_write_operations_are_not_exposed():
       allowed_tools = {
           "search_faq",
           "calculate",
           "get_ticket_status",
       }
       prohibited_names = {
           "create_ticket",
           "update_ticket",
           "delete_ticket",
           "close_ticket",
           "read_file",
           "write_file",
           "run_command",
       }

       assert allowed_tools.isdisjoint(prohibited_names)
       assert not (allowed_tools & prohibited_names)
   ```

3. Ejecute las pruebas.

   ```bash
   pytest -q tests/test_mcp_support_server.py
   ```

4. Revise que las pruebas cubren los requisitos de seguridad solicitados:

   | Requisito | Prueba |
   |---|---|
   | Descubrimiento de herramientas | `test_discovery_exposes_only_allowed_tools_and_policy_resource` |
   | Argumentos inválidos | `test_invalid_ticket_pattern_is_rejected` |
   | Error controlado de herramienta | `test_unknown_ticket_returns_controlled_tool_error` |
   | Ausencia de secretos | `test_responses_do_not_include_environment_secrets` |
   | Denegación de capacidades no permitidas | `test_write_operations_are_not_exposed` |

**Salida esperada**

```text
.......                                                                  [100%]
7 passed in <tiempo>s
```

**Verificación**

Ejecute además la validación completa del repositorio de pruebas:

```bash
pytest -q
```

No debe haber pruebas fallidas.

---

### Paso 6. Validar contratos MCP y evidencia para evaluación futura

**Objetivo:** verificar que el servidor, el cliente y las evidencias cumplen los contratos operativos antes de integrar resultados en el laboratorio 06-00-01.

**Instrucciones**

1. Vuelva a ejecutar el cliente para generar una ejecución reciente. El archivo JSONL conservará una línea por escenario y por ejecución.

   ```bash
   python -m src.mcp_client.mcp_agent_client
   ```

2. Compruebe que no existe ninguna herramienta de escritura publicada.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   last_record = json.loads(
       Path("runs/mcp_runs.jsonl").read_text(encoding="utf-8").splitlines()[-1]
   )

   forbidden = {
       "create_ticket",
       "update_ticket",
       "delete_ticket",
       "close_ticket",
       "read_file",
       "write_file",
       "run_command",
   }

   discovered = set(last_record["discovered_tools"])
   assert not discovered.intersection(forbidden)
   print("No se expusieron herramientas prohibidas.")
   PY
   ```

3. Compruebe que las evidencias no contienen valores típicos de secretos.

   ```bash
   grep -RInE \
     '(AZURE_OPENAI_API_KEY|OPENAI_API_KEY|ANTHROPIC_API_KEY|sk-[A-Za-z0-9_-]{10,})' \
     runs src tests 2>/dev/null && exit 1 || echo "No se detectaron secretos."
   ```

4. Formatee y valide el JSONL generado.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("runs/mcp_runs.jsonl")
   records = []

   for line_number, line in enumerate(
       path.read_text(encoding="utf-8").splitlines(),
       start=1,
   ):
       record = json.loads(line)
       assert record["transport"] == "stdio"
       assert record["server_name"] == "support-mcp"
       assert record["matches_reference"] is True
       records.append(record)

   print(f"Registros válidos: {len(records)}")
   PY
   ```

5. Revise el estado de Git y agregue los artefactos relevantes.

   ```bash
   git status --short
   git add .gitignore src/tools src/mcp_server src/mcp_client tests runs/mcp_runs.jsonl
   git commit -m "lab-05-00-02"
   ```

**Salida esperada**

Las comprobaciones deben producir mensajes equivalentes a:

```text
No se expusieron herramientas prohibidas.
No se detectaron secretos.
Registros válidos: 8
```

El número de registros puede variar porque cada ejecución del cliente agrega cuatro líneas al archivo JSONL.

**Verificación**

Compruebe el último commit:

```bash
git log -1 --oneline
```

Debe mostrar un commit con el mensaje:

```text
lab-05-00-02
```

## Validación y pruebas

Ejecute la siguiente secuencia final desde `~/genai-agent-labs`:

```bash
source .venv/bin/activate

pytest -q

python -m src.mcp_client.mcp_agent_client

python - <<'PY'
import json
from pathlib import Path

records = [
    json.loads(line)
    for line in Path("runs/mcp_runs.jsonl").read_text(encoding="utf-8").splitlines()
]

latest_by_scenario = {}
for record in records:
    latest_by_scenario[record["scenario_id"]] = record

assert len(latest_by_scenario) == 4
assert all(record["matches_reference"] for record in latest_by_scenario.values())
assert all(record["transport"] == "stdio" for record in latest_by_scenario.values())

print("Validación final completada correctamente.")
PY
```

### Criterios de aceptación

La práctica está completada cuando se cumplen todos los criterios:

- El servidor MCP se inicia como subproceso y usa exclusivamente transporte `stdio`.
- `list_tools()` descubre exactamente `search_faq`, `calculate` y `get_ticket_status`.
- `list_resources()` descubre `support://policies`.
- El recurso de políticas confirma que no existen operaciones de escritura.
- Las herramientas validan entradas mediante Pydantic y devuelven errores controlados.
- No se acepta una ruta de archivo proporcionada por el usuario.
- No se expone ejecución de comandos, escritura de archivos ni modificación de tickets.
- `runs/mcp_runs.jsonl` contiene resultados MCP comparados con la salida de referencia local.
- Las pruebas automatizadas finalizan correctamente.
- Ningún secreto aparece en código, logs de evidencia ni respuestas de herramientas.

## Resolución de problemas

### Problema 1: el cliente se bloquea, muestra `BrokenResourceError` o no descubre herramientas

**Síntoma:** al ejecutar `python -m src.mcp_client.mcp_agent_client`, el proceso termina con un error de conexión `stdio`, un flujo cerrado o el cliente queda esperando indefinidamente.

**Causa probable:** el servidor escribió texto no perteneciente a JSON-RPC en `stdout`, normalmente por un `print()`, un logger configurado para salida estándar o una traza generada antes de iniciar MCP.

**Solución:**

1. Compruebe que `support_mcp_server.py` no contiene `print()`.
2. Confirme que `logging.basicConfig(..., stream=sys.stderr)` está configurado.
3. No agregue mensajes de depuración a `stdout`.
4. Ejecute nuevamente:

   ```bash
   pytest -q tests/test_mcp_support_server.py
   python -m src.mcp_client.mcp_agent_client
   ```

### Problema 2: `catalog_unavailable` o fallos de validación al buscar FAQ

**Síntoma:** `search_faq` devuelve `"error": "catalog_unavailable"` o las pruebas fallan al cargar `faq_catalog.json`.

**Causa probable:** el archivo no está en `data/processed/faq_catalog.json`, contiene JSON inválido o sus registros no cumplen el contrato esperado (`id`, `category`, `question`, `answer`, `keywords`).

**Solución:**

1. Valide el JSON:

   ```bash
   python -m json.tool data/processed/faq_catalog.json > /dev/null
   ```

2. Inspeccione el primer registro:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   data = json.loads(Path("data/processed/faq_catalog.json").read_text())
   print(data[0])
   PY
   ```

3. Asegúrese de que `category` sea una de estas opciones: `technical`, `account`, `billing` o `general`.
4. Agregue `keywords` como lista, aunque esté vacía:

   ```json
   "keywords": []
   ```

5. Vuelva a ejecutar las pruebas y el cliente MCP.

## Limpieza

No se crean recursos de Azure, contenedores ni servicios de red en esta práctica. El servidor MCP termina automáticamente cuando finaliza el cliente o la prueba.

Para eliminar únicamente resultados temporales de ejecución y conservar el código, ejecute:

```bash
rm -f runs/mcp_runs.jsonl
```

Si debe reiniciar la evidencia desde cero, vuelva a ejecutar:

```bash
python -m src.mcp_client.mcp_agent_client
```

No elimine:

- `data/processed/faq_catalog.json`
- `src/tools/support_tools.py`
- `src/mcp_server/support_mcp_server.py`
- `src/mcp_client/mcp_agent_client.py`
- `tests/test_mcp_support_server.py`

Estos artefactos serán insumos para el conjunto de evaluación del laboratorio 06-00-01.

## Resumen

En esta práctica creó un servidor MCP de soporte con herramientas de lectura y cálculo controlado. La lógica de negocio se mantuvo separada del transporte MCP, permitiendo reutilizar las capacidades del agente de Function Calling creado anteriormente.

También aplicó controles de seguridad concretos: catálogo de archivos fijo, validación estricta de argumentos, categorías y operaciones permitidas, ausencia de capacidades de escritura, protección de secretos y registro estructurado de resultados. El cliente MCP verificó descubrimiento, lectura del recurso de políticas y equivalencia funcional entre las respuestas locales de referencia y las respuestas obtenidas a través de MCP.

### Recursos opcionales

- [Especificación de Model Context Protocol](https://modelcontextprotocol.io/)
- [SDK de MCP para Python](https://github.com/modelcontextprotocol/python-sdk)
- [Documentación de Pydantic](https://docs.pydantic.dev/)
- [OWASP Top 10 para aplicaciones LLM](https://genai.owasp.org/)
