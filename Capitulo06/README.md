# 1. Práctica 1. Implementar un sistema de evaluación automática utilizando agentes especializados para medir precisión, consistencia y fidelidad de las respuestas generadas.

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 80 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica construirás un sistema de evaluación automática para comparar las respuestas producidas por las rutas de Function Calling directo y MCP implementadas en los laboratorios 05-00-01 y 05-00-02. El sistema combinará validaciones deterministas con tres evaluadores especializados: precisión, consistencia y fidelidad al catálogo FAQ.

El resultado será un conjunto de artefactos reutilizables: un *golden set*, resultados detallados en JSONL, un resumen segmentado por ruta y un análisis de fallos clasificado por severidad. Estos archivos serán utilizados posteriormente en el laboratorio 06-00-02 para observabilidad y evaluación con LangSmith.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Construir un conjunto de evaluación JSONL con expectativas verificables, evidencia permitida y umbrales de aceptación.
- [ ] Implementar validaciones deterministas para identificadores de ticket, valores numéricos y citas de FAQ.
- [ ] Implementar evaluadores LLM-as-a-Judge con Azure OpenAI, temperatura `0` y salidas JSON validadas mediante Pydantic.
- [ ] Medir estabilidad entre variantes semánticamente equivalentes de una misma pregunta.
- [ ] Generar reportes que comparen Function Calling directo y MCP e identifiquen alucinaciones, regresiones y defectos de severidad alta.

## Prerrequisitos

### Conocimientos requeridos

- Haber completado los laboratorios `05-00-01` y `05-00-02`.
- Comprender el formato JSONL y la diferencia entre JSON y JSONL.
- Conocer los conceptos de exactitud, consistencia, fidelidad al contexto y evaluación LLM-as-a-Judge.
- Saber ejecutar pruebas con `pytest`.
- Comprender que la fidelidad se evalúa contra el contexto permitido, no contra el conocimiento general del modelo.

### Acceso y artefactos requeridos

Debes disponer de los siguientes archivos:

```text
~/genai-agent-labs/runs/function_calling_runs.jsonl
~/genai-agent-labs/runs/mcp_runs.jsonl
~/genai-agent-labs/data/faq_catalog.json
```

También necesitas acceso a un recurso Azure OpenAI con un despliegue compatible con chat completions y las siguientes variables de entorno:

```bash
AZURE_OPENAI_ENDPOINT
AZURE_OPENAI_API_KEY
AZURE_OPENAI_DEPLOYMENT
```

> Si tu organización usa identidad administrada en lugar de API key, consulta con el instructor antes de modificar el cliente de este laboratorio. La implementación base usa API key para concentrar el ejercicio en evaluación y no en autenticación.

## Entorno del laboratorio

### Recursos de hardware

| Recurso | Mínimo recomendado |
|---|---:|
| Memoria RAM | 16 GB |
| Espacio libre en disco | 10 GB |
| Procesador | 4 núcleos |
| Conectividad | Acceso a Azure OpenAI |

### Software

| Componente | Versión objetivo |
|---|---:|
| Python | 3.12.x |
| OpenAI Python SDK | 1.58.1 |
| Pydantic | 2.10.4 |
| Pandas | 2.2.3 |
| pytest | 8.3.4 |
| Azure OpenAI REST API | 2024-10-21 |

### Preparación inicial

1. Abre una terminal y sitúate en el repositorio global obligatorio.

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate
   ```

2. Confirma que los artefactos de los laboratorios anteriores existen.

   ```bash
   ls -lh runs/function_calling_runs.jsonl runs/mcp_runs.jsonl data/faq_catalog.json
   ```

3. Instala o actualiza las dependencias específicas del laboratorio.

   ```bash
   pip install "openai==1.58.1" "pydantic==2.10.4" "pandas==2.2.3" "pytest==8.3.4"
   ```

4. Crea la estructura de directorios requerida.

   ```bash
   mkdir -p data/evaluation reports src/evaluation tests
   touch src/evaluation/__init__.py
   ```

5. Comprueba las variables de Azure OpenAI sin mostrar el secreto.

   ```bash
   test -n "$AZURE_OPENAI_ENDPOINT" && echo "AZURE_OPENAI_ENDPOINT configurada"
   test -n "$AZURE_OPENAI_API_KEY" && echo "AZURE_OPENAI_API_KEY configurada"
   test -n "$AZURE_OPENAI_DEPLOYMENT" && echo "AZURE_OPENAI_DEPLOYMENT configurada"
   ```

**Salida esperada**

```text
AZURE_OPENAI_ENDPOINT configurada
AZURE_OPENAI_API_KEY configurada
AZURE_OPENAI_DEPLOYMENT configurada
```

**Verificación**

No debes incluir claves API en archivos versionados, comandos con `echo`, reportes ni capturas de pantalla.

---

## Procedimiento paso a paso

### Paso 1. Inspeccionar y normalizar los artefactos de ejecución

**Objetivo:** identificar los campos disponibles en las ejecuciones Function Calling y MCP antes de crear el conjunto de evaluación.

**Instrucciones**

1. Muestra las primeras líneas de ambos archivos JSONL.

   ```bash
   head -n 2 runs/function_calling_runs.jsonl | python -m json.tool
   head -n 2 runs/mcp_runs.jsonl | python -m json.tool
   ```

2. Inspecciona los campos de los primeros registros mediante Python.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   for filename in [
       "runs/function_calling_runs.jsonl",
       "runs/mcp_runs.jsonl",
   ]:
       path = Path(filename)
       with path.open(encoding="utf-8") as file:
           record = json.loads(next(line for line in file if line.strip()))
       print(f"\nArchivo: {filename}")
       print("Campos:", sorted(record.keys()))
   PY
   ```

3. Identifica, para cada ejecución, los campos que equivalen a los siguientes conceptos:

   | Concepto | Nombres admitidos por el evaluador |
   |---|---|
   | Prompt del usuario | `prompt`, `question`, `input`, `user_prompt` |
   | Respuesta final | `response`, `answer`, `output`, `final_answer` |
   | Herramienta seleccionada | `tool_name`, `tool`, `selected_tool`, `function_name` |
   | Argumentos de herramienta | `tool_arguments`, `arguments`, `function_arguments` |
   | Latencia | `latency_ms`, `duration_ms`, `elapsed_ms` |
   | Identificador del caso | `case_id`, `evaluation_case_id`, `id` |

4. Si tus archivos usan nombres distintos, no los modifiques todavía. En el paso 3 configurarás alias adicionales en el adaptador `normalize_run`.

**Salida esperada**

Debes observar objetos JSON con información de prompt, respuesta y, según el laboratorio previo, trazas de herramientas o llamadas MCP.

Ejemplo ilustrativo:

```json
{
    "run_id": "fc-001",
    "prompt": "¿Cuál es el estado del ticket INC-1001?",
    "response": "El ticket INC-1001 está en estado abierto.",
    "tool_name": "get_ticket_status",
    "latency_ms": 842
}
```

**Verificación**

Confirma que ambos archivos contienen al menos un registro válido:

```bash
wc -l runs/function_calling_runs.jsonl runs/mcp_runs.jsonl
```

Si uno de los archivos no existe o está vacío, debes completar o repetir el laboratorio correspondiente antes de continuar.

---

### Paso 2. Construir el conjunto de evaluación dorado

**Objetivo:** crear un *golden set* trazable al catálogo FAQ, con criterios explícitos y sin depender de una única redacción de respuesta.

**Instrucciones**

1. Inspecciona la estructura del catálogo FAQ.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   catalog = json.loads(Path("data/faq_catalog.json").read_text(encoding="utf-8"))
   items = catalog if isinstance(catalog, list) else catalog.get("items", catalog.get("faqs", []))

   print(f"Entradas encontradas: {len(items)}")
   for item in items[:3]:
       print("\n--- FAQ ---")
       print(json.dumps(item, ensure_ascii=False, indent=2)[:1200])
   PY
   ```

2. Selecciona al menos cinco casos que ya estén presentes en los archivos de ejecución. Incluye categorías distintas:

   - consulta factual sobre ticket;
   - consulta numérica;
   - consulta que requiere una cita de FAQ;
   - consulta que exige seleccionar una herramienta;
   - consulta con una variante equivalente para medir consistencia.

3. Crea el archivo `data/evaluation/golden_set.jsonl`.

   > Sustituye los valores `FAQ-XXX`, `INC-XXXX`, textos y cifras por valores reales presentes en `faq_catalog.json` y en los archivos de ejecución. No inventes evidencia.

   ```bash
   cat > data/evaluation/golden_set.jsonl <<'EOF'
   {"id":"eval-001","prompt":"¿Cuál es el estado del ticket INC-XXXX?","category":"ticket_factual","risk":"high","expected_facts":["INC-XXXX","abierto"],"expected_tool":"get_ticket_status","expected_ticket_id":"INC-XXXX","expected_numbers":[],"required_faq_ids":[],"allowed_faq_ids":[],"acceptance":{"accuracy_min":3,"faithfulness_min":3,"consistency_min":3,"require_ticket_id":true,"require_faq_citation":false}}
   {"id":"eval-002","prompt":"¿Cuál es el límite de solicitudes permitido por la política descrita?","category":"numeric_policy","risk":"high","expected_facts":["límite","VALOR_REAL"],"expected_tool":null,"expected_ticket_id":null,"expected_numbers":["VALOR_REAL"],"required_faq_ids":["FAQ-XXX"],"allowed_faq_ids":["FAQ-XXX"],"acceptance":{"accuracy_min":3,"faithfulness_min":4,"consistency_min":3,"require_ticket_id":false,"require_faq_citation":true}}
   {"id":"eval-003","prompt":"¿Cómo debo proceder según la FAQ sobre acceso a la plataforma?","category":"faq_procedure","risk":"medium","expected_facts":["PASO_REAL_1","PASO_REAL_2"],"expected_tool":null,"expected_ticket_id":null,"expected_numbers":[],"required_faq_ids":["FAQ-XXX"],"allowed_faq_ids":["FAQ-XXX"],"acceptance":{"accuracy_min":3,"faithfulness_min":4,"consistency_min":3,"require_ticket_id":false,"require_faq_citation":true}}
   {"id":"eval-004","prompt":"Necesito consultar los detalles del ticket INC-YYYY.","category":"tool_selection","risk":"high","expected_facts":["INC-YYYY"],"expected_tool":"get_ticket_details","expected_ticket_id":"INC-YYYY","expected_numbers":[],"required_faq_ids":[],"allowed_faq_ids":[],"acceptance":{"accuracy_min":3,"faithfulness_min":3,"consistency_min":3,"require_ticket_id":true,"require_faq_citation":false}}
   {"id":"eval-005","prompt":"¿Qué indica la política para el caso REAL definido en la FAQ?","category":"faq_factual","risk":"medium","expected_facts":["HECHO_REAL_1","HECHO_REAL_2"],"expected_tool":null,"expected_ticket_id":null,"expected_numbers":[],"required_faq_ids":["FAQ-XXX"],"allowed_faq_ids":["FAQ-XXX"],"acceptance":{"accuracy_min":3,"faithfulness_min":4,"consistency_min":3,"require_ticket_id":false,"require_faq_citation":true}}
   EOF
   ```

4. Reemplaza todos los marcadores de posición.

   ```bash
   grep -nE 'FAQ-XXX|INC-XXXX|INC-YYYY|VALOR_REAL|HECHO_REAL|PASO_REAL' \
     data/evaluation/golden_set.jsonl
   ```

5. Cuando el comando anterior no produzca salida, valida el formato JSONL.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("data/evaluation/golden_set.jsonl")
   for number, line in enumerate(path.read_text(encoding="utf-8").splitlines(), start=1):
       if line.strip():
           json.loads(line)
           print(f"Línea {number}: JSON válido")
   PY
   ```

**Salida esperada**

Cada caso debe contener como mínimo:

```json
{
  "id": "eval-002",
  "category": "numeric_policy",
  "risk": "high",
  "expected_facts": ["límite", "5"],
  "expected_numbers": ["5"],
  "required_faq_ids": ["FAQ-012"],
  "acceptance": {
    "accuracy_min": 3,
    "faithfulness_min": 4,
    "consistency_min": 3,
    "require_faq_citation": true
  }
}
```

**Verificación**

Ejecuta:

```bash
python - <<'PY'
import json
from pathlib import Path

cases = [
    json.loads(line)
    for line in Path("data/evaluation/golden_set.jsonl").read_text(encoding="utf-8").splitlines()
    if line.strip()
]
assert len(cases) >= 5, "Se requieren al menos cinco casos."
assert len({case["id"] for case in cases}) == len(cases), "Los IDs deben ser únicos."
assert all(case["expected_facts"] for case in cases), "Cada caso requiere hechos esperados."
print(f"Golden set válido: {len(cases)} casos.")
PY
```

---

### Paso 3. Implementar los evaluadores especializados

**Objetivo:** crear un orquestador que aplique comprobaciones deterministas y evaluación semántica mediante Azure OpenAI.

**Instrucciones**

1. Crea el archivo `src/evaluation/evaluate_runs.py`.

   ```bash
   cat > src/evaluation/evaluate_runs.py <<'PY'
   import argparse
   import json
   import os
   import re
   from collections import defaultdict
   from pathlib import Path
   from statistics import mean
   from typing import Any, Literal

   from openai import AzureOpenAI
   from pydantic import BaseModel, Field


   ROOT = Path(__file__).resolve().parents[2]
   GOLDEN_PATH = ROOT / "data/evaluation/golden_set.jsonl"
   FAQ_PATH = ROOT / "data/faq_catalog.json"
   FC_PATH = ROOT / "runs/function_calling_runs.jsonl"
   MCP_PATH = ROOT / "runs/mcp_runs.jsonl"
   REPORTS_DIR = ROOT / "reports"


   class JudgeVerdict(BaseModel):
       score: int = Field(ge=1, le=4)
       passed: bool
       rationale: str = Field(min_length=1, max_length=800)
       unsupported_claims: list[str] = Field(default_factory=list)


   class EvaluationResult(BaseModel):
       case_id: str
       route: Literal["function_calling", "mcp"]
       category: str
       risk: str
       prompt: str
       response: str
       tool_name: str | None = None
       deterministic: dict[str, Any]
       accuracy: JudgeVerdict
       faithfulness: JudgeVerdict
       consistency_score: int
       passed: bool
       severity: str
       defects: list[str]
       latency_ms: float | None = None


   def read_jsonl(path: Path) -> list[dict[str, Any]]:
       rows = []
       with path.open(encoding="utf-8") as file:
           for line_number, line in enumerate(file, start=1):
               if not line.strip():
                   continue
               try:
                   rows.append(json.loads(line))
               except json.JSONDecodeError as error:
                   raise ValueError(f"{path}:{line_number} no contiene JSON válido") from error
       return rows


   def load_faqs() -> dict[str, str]:
       raw = json.loads(FAQ_PATH.read_text(encoding="utf-8"))
       items = raw if isinstance(raw, list) else raw.get("items", raw.get("faqs", []))
       faqs = {}
       for item in items:
           faq_id = str(item.get("id") or item.get("faq_id") or item.get("identifier") or "")
           text = " ".join(
               str(item.get(key, ""))
               for key in ("title", "question", "answer", "content", "text")
           ).strip()
           if faq_id and text:
               faqs[faq_id] = text
       return faqs


   def normalize_run(raw: dict[str, Any], route: str) -> dict[str, Any]:
       def first(*keys: str) -> Any:
           for key in keys:
               if raw.get(key) not in (None, ""):
                   return raw[key]
           return None

       response = first("response", "answer", "output", "final_answer", "content")
       prompt = first("prompt", "question", "input", "user_prompt")
       arguments = first("tool_arguments", "arguments", "function_arguments") or {}
       if isinstance(arguments, str):
           try:
               arguments = json.loads(arguments)
           except json.JSONDecodeError:
               pass

       return {
           "route": route,
           "case_id": first("case_id", "evaluation_case_id"),
           "prompt": str(prompt or ""),
           "response": str(response or ""),
           "tool_name": first("tool_name", "tool", "selected_tool", "function_name"),
           "arguments": arguments,
           "latency_ms": first("latency_ms", "duration_ms", "elapsed_ms"),
           "raw": raw,
       }


   def build_client() -> AzureOpenAI:
       required = ["AZURE_OPENAI_ENDPOINT", "AZURE_OPENAI_API_KEY", "AZURE_OPENAI_DEPLOYMENT"]
       missing = [name for name in required if not os.getenv(name)]
       if missing:
           raise RuntimeError(f"Faltan variables de entorno: {', '.join(missing)}")
       return AzureOpenAI(
           azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
           api_key=os.environ["AZURE_OPENAI_API_KEY"],
           api_version="2024-10-21",
       )


   def judge(client: AzureOpenAI, system: str, payload: dict[str, Any]) -> JudgeVerdict:
       completion = client.beta.chat.completions.parse(
           model=os.environ["AZURE_OPENAI_DEPLOYMENT"],
           temperature=0,
           messages=[
               {"role": "system", "content": system},
               {"role": "user", "content": json.dumps(payload, ensure_ascii=False)},
           ],
           response_format=JudgeVerdict,
       )
       parsed = completion.choices[0].message.parsed
       if parsed is None:
           raise RuntimeError("Azure OpenAI no devolvió una salida estructurada válida.")
       return parsed


   def deterministic_checks(case: dict[str, Any], run: dict[str, Any]) -> dict[str, Any]:
       response = run["response"].lower()
       expected_facts = [fact.lower() for fact in case.get("expected_facts", [])]
       facts_found = {fact: fact in response for fact in expected_facts}

       ticket = case.get("expected_ticket_id")
       ticket_found = (not ticket) or ticket.lower() in response

       numbers_found = {
           number: bool(re.search(rf"(?<!\d){re.escape(str(number))}(?!\d)", run["response"]))
           for number in case.get("expected_numbers", [])
       }

       expected_tool = case.get("expected_tool")
       tool_match = expected_tool is None or run["tool_name"] == expected_tool

       required_faq_ids = case.get("required_faq_ids", [])
       faq_citations_found = {
           faq_id: faq_id.lower() in response
           for faq_id in required_faq_ids
       }

       return {
           "facts_found": facts_found,
           "all_expected_facts_found": all(facts_found.values()),
           "ticket_id_found": ticket_found,
           "numbers_found": numbers_found,
           "all_numbers_found": all(numbers_found.values()),
           "tool_match": tool_match,
           "faq_citations_found": faq_citations_found,
           "all_required_faq_citations_found": all(faq_citations_found.values()),
       }


   def accuracy_evaluator(
       client: AzureOpenAI, case: dict[str, Any], run: dict[str, Any], checks: dict[str, Any]
   ) -> JudgeVerdict:
       return judge(
           client,
           """Eres AccuracyEvaluator. Evalúa exactitud y completitud funcional.
   Puntúa 4 si todos los hechos obligatorios, números, ticket y herramienta esperada son correctos.
   Puntúa 3 si hay una omisión menor que no altera la decisión.
   Puntúa 2 si falta o es incorrecto un hecho importante.
   Puntúa 1 si la respuesta es incorrecta, contradictoria o no responde.
   No otorgues puntuaciones altas solo por una redacción fluida. Devuelve exclusivamente el esquema solicitado.""",
           {
               "prompt": case["prompt"],
               "expected_facts": case.get("expected_facts", []),
               "expected_tool": case.get("expected_tool"),
               "response": run["response"],
               "tool_name": run["tool_name"],
               "deterministic_checks": checks,
           },
       )


   def faithfulness_evaluator(
       client: AzureOpenAI,
       case: dict[str, Any],
       run: dict[str, Any],
       allowed_context: str,
   ) -> JudgeVerdict:
       return judge(
           client,
           """Eres FaithfulnessEvaluator para un sistema RAG.
   Evalúa únicamente si las afirmaciones de la respuesta están respaldadas por el CONTEXTO_PERMITIDO.
   No uses conocimiento externo. Si el contexto está vacío, una respuesta factual no puede recibir más de 2.
   Escala: 4 totalmente respaldada; 3 imprecisión menor; 2 mezcla afirmaciones respaldadas y no respaldadas;
   1 contiene una afirmación importante no respaldada o contradicha.
   Incluye en unsupported_claims cada afirmación no sustentada. Devuelve exclusivamente el esquema solicitado.""",
           {
               "prompt": case["prompt"],
               "contexto_permitido": allowed_context,
               "response": run["response"],
               "faq_ids_permitidos": case.get("allowed_faq_ids", []),
           },
       )


   def consistency_score(case: dict[str, Any], route_runs: list[dict[str, Any]]) -> int:
       equivalents = [
           run for run in route_runs
           if run["case_id"] == case["id"] or run["prompt"].strip().lower() == case["prompt"].strip().lower()
       ]
       if len(equivalents) < 2:
           return 3

       tools = {str(run["tool_name"]) for run in equivalents}
       required_facts = [fact.lower() for fact in case.get("expected_facts", [])]
       fact_patterns = {
           tuple(fact in run["response"].lower() for fact in required_facts)
           for run in equivalents
       }

       if len(tools) == 1 and len(fact_patterns) == 1:
           return 4
       if len(tools) == 1:
           return 3
       if len(tools) == 2:
           return 2
       return 1


   def severity(case: dict[str, Any], result: EvaluationResult) -> tuple[str, list[str]]:
       defects = []
       checks = result.deterministic

       if not checks["all_expected_facts_found"]:
           defects.append("hechos_esperados_ausentes")
       if not checks["tool_match"]:
           defects.append("herramienta_incorrecta")
       if not checks["ticket_id_found"]:
           defects.append("ticket_id_ausente")
       if not checks["all_numbers_found"]:
           defects.append("resultado_numerico_incorrecto")
       if result.faithfulness.score <= 2 or result.faithfulness.unsupported_claims:
           defects.append("posible_alucinacion")
       if result.consistency_score <= 2:
           defects.append("inconsistencia")

       if not defects:
           return "none", defects
       if case["risk"] == "high" and (
           "posible_alucinacion" in defects
           or "herramienta_incorrecta" in defects
           or "resultado_numerico_incorrecto" in defects
       ):
           return "critical", defects
       if result.accuracy.score <= 2 or result.faithfulness.score <= 2:
           return "high", defects
       return "medium", defects


   def select_run(case: dict[str, Any], runs: list[dict[str, Any]]) -> dict[str, Any] | None:
       for run in reversed(runs):
           if run["case_id"] == case["id"]:
               return run
       for run in reversed(runs):
           if run["prompt"].strip().lower() == case["prompt"].strip().lower():
               return run
       return None


   def markdown_failure_analysis(results: list[EvaluationResult]) -> str:
       lines = [
           "# Análisis de fallos de evaluación",
           "",
           "## Criterio de severidad",
           "",
           "- **critical**: riesgo alto con alucinación, herramienta incorrecta o resultado numérico incorrecto.",
           "- **high**: exactitud o fidelidad menor o igual a 2.",
           "- **medium**: inconsistencia u omisiones no críticas.",
           "",
           "## Fallos detectados",
           "",
       ]
       failures = [result for result in results if not result.passed]
       if not failures:
           lines.append("No se detectaron fallos contra los umbrales definidos.")
       else:
           for result in failures:
               lines.extend([
                   f"### {result.case_id} — {result.route} — {result.severity}",
                   f"- Categoría: `{result.category}`; riesgo: `{result.risk}`.",
                   f"- Defectos: {', '.join(result.defects) or 'umbral LLM no alcanzado'}.",
                   f"- Exactitud: {result.accuracy.score}/4. Fidelidad: {result.faithfulness.score}/4. Consistencia: {result.consistency_score}/4.",
                   f"- Justificación de fidelidad: {result.faithfulness.rationale}",
               ])
               if result.faithfulness.unsupported_claims:
                   lines.append(f"- Afirmaciones no respaldadas: {result.faithfulness.unsupported_claims}")
               lines.append("")
       return "\n".join(lines)


   def main() -> None:
       parser = argparse.ArgumentParser()
       parser.add_argument("--skip-llm", action="store_true", help="Útil para validar estructura sin consumir Azure OpenAI.")
       args = parser.parse_args()

       cases = read_jsonl(GOLDEN_PATH)
       faqs = load_faqs()
       route_runs = {
           "function_calling": [normalize_run(row, "function_calling") for row in read_jsonl(FC_PATH)],
           "mcp": [normalize_run(row, "mcp") for row in read_jsonl(MCP_PATH)],
       }
       client = None if args.skip_llm else build_client()
       results: list[EvaluationResult] = []

       for case in cases:
           allowed_ids = case.get("allowed_faq_ids", [])
           missing_faqs = [faq_id for faq_id in allowed_ids if faq_id not in faqs]
           if missing_faqs:
               raise ValueError(f"{case['id']} referencia FAQ inexistente: {missing_faqs}")
           allowed_context = "\n\n".join(f"[{faq_id}] {faqs[faq_id]}" for faq_id in allowed_ids)

           for route, runs in route_runs.items():
               run = select_run(case, runs)
               if not run:
                   continue

               checks = deterministic_checks(case, run)
               if client:
                   accuracy = accuracy_evaluator(client, case, run, checks)
                   faithfulness = faithfulness_evaluator(client, case, run, allowed_context)
               else:
                   score = 4 if checks["all_expected_facts_found"] else 1
                   accuracy = JudgeVerdict(score=score, passed=score >= 3, rationale="Modo determinista.", unsupported_claims=[])
                   faithfulness = JudgeVerdict(score=score, passed=score >= 3, rationale="Modo determinista.", unsupported_claims=[])

               consistency = consistency_score(case, runs)
               acceptance = case["acceptance"]
               passed = (
                   accuracy.score >= acceptance["accuracy_min"]
                   and faithfulness.score >= acceptance["faithfulness_min"]
                   and consistency >= acceptance["consistency_min"]
                   and (not acceptance["require_ticket_id"] or checks["ticket_id_found"])
                   and (not acceptance["require_faq_citation"] or checks["all_required_faq_citations_found"])
                   and checks["all_numbers_found"]
                   and checks["tool_match"]
               )

               provisional = EvaluationResult(
                   case_id=case["id"], route=route, category=case["category"], risk=case["risk"],
                   prompt=case["prompt"], response=run["response"], tool_name=run["tool_name"],
                   deterministic=checks, accuracy=accuracy, faithfulness=faithfulness,
                   consistency_score=consistency, passed=passed, severity="none", defects=[],
                   latency_ms=run["latency_ms"],
               )
               provisional.severity, provisional.defects = severity(case, provisional)
               results.append(provisional)

       REPORTS_DIR.mkdir(exist_ok=True)
       with (REPORTS_DIR / "evaluation_results.jsonl").open("w", encoding="utf-8") as file:
           for result in results:
               file.write(result.model_dump_json() + "\n")

       grouped = defaultdict(list)
       for result in results:
           grouped[result.route].append(result)

       summary = {
           "total_results": len(results),
           "routes": {},
           "missing_cases": {},
       }
       for route, route_result in grouped.items():
           summary["routes"][route] = {
               "total": len(route_result),
               "passed": sum(result.passed for result in route_result),
               "success_rate": round(sum(result.passed for result in route_result) / len(route_result) * 100, 2),
               "accuracy_mean": round(mean(result.accuracy.score for result in route_result), 2),
               "faithfulness_mean": round(mean(result.faithfulness.score for result in route_result), 2),
               "consistency_mean": round(mean(result.consistency_score for result in route_result), 2),
               "p95_latency_ms": sorted(
                   [result.latency_ms for result in route_result if result.latency_ms is not None]
               )[max(0, int(len([r for r in route_result if r.latency_ms is not None]) * 0.95) - 1)]
               if any(result.latency_ms is not None for result in route_result) else None,
               "critical_failures": sum(result.severity == "critical" for result in route_result),
           }

       for route, runs in route_runs.items():
           summary["missing_cases"][route] = [
               case["id"] for case in cases if select_run(case, runs) is None
           ]

       (REPORTS_DIR / "evaluation_summary.json").write_text(
           json.dumps(summary, ensure_ascii=False, indent=2), encoding="utf-8"
       )
       (REPORTS_DIR / "failure_analysis.md").write_text(
           markdown_failure_analysis(results), encoding="utf-8"
       )
       print(json.dumps(summary, ensure_ascii=False, indent=2))


   if __name__ == "__main__":
       main()
   PY
   ```

2. Comprueba la sintaxis del archivo.

   ```bash
   python -m py_compile src/evaluation/evaluate_runs.py
   ```

3. Ejecuta una validación inicial sin consumir llamadas al modelo.

   ```bash
   python src/evaluation/evaluate_runs.py --skip-llm
   ```

**Salida esperada**

Se crearán los siguientes archivos:

```text
reports/evaluation_results.jsonl
reports/evaluation_summary.json
reports/failure_analysis.md
```

El resumen debe contener métricas por ruta:

```json
{
  "total_results": 10,
  "routes": {
    "function_calling": {
      "total": 5,
      "passed": 4,
      "success_rate": 80.0,
      "accuracy_mean": 3.4,
      "faithfulness_mean": 3.4,
      "consistency_mean": 3.0
    }
  }
}
```

**Verificación**

Comprueba que los reportes no están vacíos:

```bash
wc -l reports/evaluation_results.jsonl
cat reports/evaluation_summary.json
```

---

### Paso 4. Ejecutar la evaluación LLM-as-a-Judge

**Objetivo:** evaluar exactitud y fidelidad con salida estructurada de Azure OpenAI.

**Instrucciones**

1. Revisa que no existan marcadores de posición en el *golden set*.

   ```bash
   grep -nE 'FAQ-XXX|INC-XXXX|INC-YYYY|VALOR_REAL|HECHO_REAL|PASO_REAL' \
     data/evaluation/golden_set.jsonl || true
   ```

2. Ejecuta el orquestador con los evaluadores LLM.

   ```bash
   python src/evaluation/evaluate_runs.py
   ```

3. Revisa un resultado estructurado.

   ```bash
   head -n 1 reports/evaluation_results.jsonl | python -m json.tool
   ```

4. Revisa el análisis de fallos.

   ```bash
   sed -n '1,220p' reports/failure_analysis.md
   ```

**Salida esperada**

Cada resultado contiene la puntuación de los tres evaluadores y las comprobaciones deterministas.

Ejemplo:

```json
{
  "case_id": "eval-002",
  "route": "mcp",
  "accuracy": {
    "score": 4,
    "passed": true,
    "rationale": "Incluye el límite requerido y responde a la consulta.",
    "unsupported_claims": []
  },
  "faithfulness": {
    "score": 4,
    "passed": true,
    "rationale": "La afirmación está respaldada por la FAQ permitida.",
    "unsupported_claims": []
  },
  "consistency_score": 3,
  "passed": true,
  "severity": "none"
}
```

**Verificación**

Confirma que todas las puntuaciones están entre 1 y 4:

```bash
python - <<'PY'
import json
from pathlib import Path

for line in Path("reports/evaluation_results.jsonl").read_text(encoding="utf-8").splitlines():
    result = json.loads(line)
    assert 1 <= result["accuracy"]["score"] <= 4
    assert 1 <= result["faithfulness"]["score"] <= 4
    assert 1 <= result["consistency_score"] <= 4

print("Puntuaciones válidas.")
PY
```

---

### Paso 5. Añadir pruebas automatizadas del evaluador

**Objetivo:** proteger las validaciones deterministas contra regresiones de implementación.

**Instrucciones**

1. Crea el archivo `tests/test_evaluation.py`.

   ```bash
   cat > tests/test_evaluation.py <<'PY'
   from src.evaluation.evaluate_runs import deterministic_checks


   def test_detecta_ticket_y_numero_correctos():
       case = {
           "expected_facts": ["INC-1001", "abierto"],
           "expected_ticket_id": "INC-1001",
           "expected_numbers": ["5"],
           "expected_tool": "get_ticket_status",
           "required_faq_ids": [],
       }
       run = {
           "response": "El ticket INC-1001 está abierto y tiene prioridad 5.",
           "tool_name": "get_ticket_status",
       }

       checks = deterministic_checks(case, run)

       assert checks["all_expected_facts_found"] is True
       assert checks["ticket_id_found"] is True
       assert checks["all_numbers_found"] is True
       assert checks["tool_match"] is True


   def test_detecta_herramienta_y_cita_faltantes():
       case = {
           "expected_facts": ["límite"],
           "expected_ticket_id": None,
           "expected_numbers": [],
           "expected_tool": "get_policy",
           "required_faq_ids": ["FAQ-012"],
       }
       run = {
           "response": "El límite se aplica según la política.",
           "tool_name": "get_ticket_status",
       }

       checks = deterministic_checks(case, run)

       assert checks["tool_match"] is False
       assert checks["all_required_faq_citations_found"] is False
   PY
   ```

2. Ejecuta las pruebas.

   ```bash
   pytest -q tests/test_evaluation.py
   ```

3. Ejecuta las pruebas de todo el repositorio si no interfiere con prácticas anteriores.

   ```bash
   pytest -q
   ```

**Salida esperada**

```text
2 passed
```

**Verificación**

Las pruebas deben finalizar sin errores. Si falla una prueba, revisa los nombres de los campos y no cambies el comportamiento esperado sin justificarlo.

---

### Paso 6. Comparar rutas y analizar regresiones

**Objetivo:** interpretar resultados agregados sin ocultar fallos críticos mediante un promedio global.

**Instrucciones**

1. Consulta el resumen consolidado.

   ```bash
   cat reports/evaluation_summary.json | python -m json.tool
   ```

2. Muestra únicamente fallos de severidad `critical` o `high`.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   results = [
       json.loads(line)
       for line in Path("reports/evaluation_results.jsonl").read_text(encoding="utf-8").splitlines()
       if line.strip()
   ]

   for result in results:
       if result["severity"] in {"critical", "high"}:
           print(
               f"{result['route']:18} {result['case_id']:10} "
               f"{result['severity']:8} {result['defects']}"
           )
   PY
   ```

3. Identifica una posible regresión entre rutas. Considera regresión cuando una ruta:

   - aprueba un caso que la otra ruta falla;
   - usa una herramienta distinta a la esperada;
   - obtiene fidelidad menor o igual a `2`;
   - introduce un ticket, cifra o política no respaldada;
   - tiene peor tasa de éxito en una categoría de alto riesgo.

4. Añade una conclusión breve al final de `reports/failure_analysis.md`.

   ```bash
   cat >> reports/failure_analysis.md <<'EOF'

   ## Conclusión de comparación entre rutas

   - Ruta recomendada para continuar: **REEMPLAZAR_CON_FUNCTION_CALLING_O_MCP**.
   - Evidencia: indicar tasa de éxito, casos críticos y diferencias de fidelidad observadas.
   - Acción correctiva prioritaria: describir una corrección concreta, por ejemplo, reforzar la instrucción de citar FAQ o corregir el mapeo de herramientas.
   EOF
   ```

5. Sustituye el texto de marcador por una conclusión basada en tus resultados reales.

**Salida esperada**

El reporte debe permitir responder preguntas operativas como:

- ¿Cuál ruta produce menos alucinaciones?
- ¿Qué categoría concentra los defectos críticos?
- ¿Las respuestas citan la FAQ requerida?
- ¿Existen respuestas correctas pero no fieles al contexto?
- ¿Qué caso debe convertirse en prueba de regresión prioritaria?

**Verificación**

Comprueba que no quedan marcadores de conclusión:

```bash
grep -n 'REEMPLAZAR_CON' reports/failure_analysis.md && \
  echo "Debes completar la conclusión." || \
  echo "Conclusión completada."
```

---

## Validación y pruebas

Ejecuta la siguiente secuencia final desde la raíz del repositorio:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

python -m py_compile src/evaluation/evaluate_runs.py
pytest -q tests/test_evaluation.py
python src/evaluation/evaluate_runs.py

test -s data/evaluation/golden_set.jsonl
test -s reports/evaluation_results.jsonl
test -s reports/evaluation_summary.json
test -s reports/failure_analysis.md

echo "Validación final completada."
```

### Criterios de aceptación

La práctica se considera completada cuando se cumplen todos los criterios:

| Criterio | Validación |
|---|---|
| Golden set | Al menos 5 casos JSONL válidos, con IDs únicos y evidencia FAQ real. |
| Trazabilidad | Los IDs de FAQ del golden set existen en `data/faq_catalog.json`. |
| Evaluador de precisión | Comprueba hechos, números, ticket y herramienta esperada. |
| Evaluador de fidelidad | Recibe exclusivamente las FAQ permitidas por cada caso. |
| Consistencia | Produce una puntuación de estabilidad por ruta y caso. |
| Salida estructurada | Las respuestas del juez se validan con Pydantic. |
| Reportes | Existen los tres archivos requeridos en `reports/`. |
| Segmentación | El resumen compara Function Calling directo y MCP. |
| Pruebas | `pytest -q tests/test_evaluation.py` finaliza correctamente. |
| Seguridad | No hay secretos de Azure OpenAI en código ni archivos versionados. |

Finalmente, revisa los cambios y realiza el commit requerido para esta práctica:

```bash
git status
git add data/evaluation/golden_set.jsonl \
        src/evaluation/evaluate_runs.py \
        tests/test_evaluation.py \
        reports/evaluation_results.jsonl \
        reports/evaluation_summary.json \
        reports/failure_analysis.md
git commit -m "lab-02-00-03"
```

> No agregues archivos `.env`, claves API, volcados completos de trazas con información sensible ni datos no anonimizados.

## Solución de problemas

### Problema 1. Azure OpenAI devuelve error de autenticación, despliegue inexistente o salida estructurada vacía

**Síntomas**

```text
AuthenticationError
ResourceNotFoundError
RuntimeError: Azure OpenAI no devolvió una salida estructurada válida.
```

**Causa probable**

Las variables `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY` o `AZURE_OPENAI_DEPLOYMENT` no están configuradas correctamente; el nombre del despliegue no coincide con el configurado en Azure OpenAI; o el despliegue no admite la operación solicitada.

**Solución**

1. Confirma que las variables existen sin imprimir secretos:

   ```bash
   env | grep '^AZURE_OPENAI_' | sed 's/=.*/=CONFIGURADA/'
   ```

2. Verifica en Azure Portal o Azure AI Foundry el nombre exacto del **deployment**, no solo el nombre del modelo.
3. Exporta el nombre correcto y vuelve a ejecutar:

   ```bash
   export AZURE_OPENAI_DEPLOYMENT="nombre-real-del-despliegue"
   python src/evaluation/evaluate_runs.py
   ```

4. Si el problema persiste, ejecuta temporalmente el modo determinista para validar datos y estructura:

   ```bash
   python src/evaluation/evaluate_runs.py --skip-llm
   ```

### Problema 2. El resumen indica casos faltantes o todos los resultados tienen puntuación baja

**Síntomas**

```json
"missing_cases": {
  "function_calling": ["eval-001", "eval-002"]
}
```

o bien:

```text
hechos_esperados_ausentes
posible_alucinacion
```

**Causa probable**

El prompt del golden set no coincide exactamente con los prompts registrados, no se añadió `case_id` a los archivos de ejecución, o los hechos/IDs de FAQ fueron inventados o transcritos de forma distinta al catálogo y a las respuestas reales.

**Solución**

1. Busca el prompt real en los archivos de ejecución:

   ```bash
   grep -in "fragmento distintivo del prompt" runs/function_calling_runs.jsonl
   grep -in "fragmento distintivo del prompt" runs/mcp_runs.jsonl
   ```

2. Copia el prompt real al campo `prompt` del golden set, o agrega el campo `case_id` a las ejecuciones futuras.
3. Verifica que los valores de `required_faq_ids` existan realmente:

   ```bash
   grep -o '"id"[[:space:]]*:[[:space:]]*"[^"]*"' data/faq_catalog.json | head -n 30
   ```

4. Ajusta los hechos esperados para que expresen requisitos verificables y presentes en la fuente, no una redacción completa ni conocimiento externo.

## Limpieza

Esta práctica genera artefactos que serán reutilizados en el laboratorio 06-00-02; por tanto, no elimines el *golden set* ni los reportes.

Para eliminar únicamente archivos temporales de Python:

```bash
cd ~/genai-agent-labs
find src tests -type d -name "__pycache__" -prune -exec rm -rf {} +
find . -type f -name "*.pyc" -delete
```

Si exportaste variables de entorno exclusivamente para esta sesión y deseas quitarlas:

```bash
unset AZURE_OPENAI_ENDPOINT
unset AZURE_OPENAI_API_KEY
unset AZURE_OPENAI_DEPLOYMENT
```

No elimines el entorno virtual compartido `~/genai-agent-labs/.venv`.

## Resumen

En esta práctica implementaste una estrategia de evaluación multidimensional para respuestas generadas por agentes. Construiste un *golden set* con evidencia permitida, aplicaste validaciones deterministas y usaste Azure OpenAI como juez con respuestas estructuradas y temperatura `0`.

Los reportes resultantes separan exactitud de fidelidad: una respuesta puede ser aparentemente correcta, pero fallar si no está respaldada por la FAQ recuperada. También comparaste las rutas Function Calling directo y MCP, evitando que una métrica promedio oculte fallos críticos en consultas de mayor riesgo.

Los siguientes archivos deben conservarse para el laboratorio `06-00-02`:

```text
data/evaluation/golden_set.jsonl
reports/evaluation_results.jsonl
reports/evaluation_summary.json
reports/failure_analysis.md
```

### Recursos opcionales

- [OpenAI: guía de evaluaciones](https://platform.openai.com/docs/guides/evals)
- [Azure OpenAI: documentación de chat completions](https://learn.microsoft.com/azure/ai-services/openai/)
- [Pydantic: modelos y validación](https://docs.pydantic.dev/)
- [RAGAS: evaluación de sistemas RAG](https://docs.ragas.io/)

---

# 2. Práctica 2. Instrumentar una solución basada en LangChain utilizando LangSmith para identificar cuellos de botella, errores y regresiones durante la ejecución de agentes.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 80 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica instrumentarás un agente de soporte técnico construido con LangChain y Azure OpenAI para registrar trazas detalladas en LangSmith. Publicarás el *golden set* como un dataset reutilizable, ejecutarás una línea base evaluable y analizarás latencia, errores de herramientas, consumo de tokens y calidad de respuestas.

Finalmente, introducirás una regresión controlada para comprobar que los evaluadores y la comparación de experimentos permiten detectar degradaciones antes de que lleguen a producción.

## Objetivos de aprendizaje

Al completar esta práctica podrás:

- [ ] Instrumentar etapas de clasificación, selección de herramienta, invocación, recuperación de evidencia y respuesta final con LangChain y LangSmith.
- [ ] Configurar trazabilidad distribuida mediante `LANGCHAIN_TRACING_V2`, metadatos y etiquetas operativas.
- [ ] Crear o actualizar el dataset `support-golden-set-v1` desde un archivo JSONL local.
- [ ] Ejecutar y comparar experimentos de LangSmith utilizando métricas de precisión y fidelidad.
- [ ] Definir una línea base operacional y detectar una regresión intencional de calidad o recuperación.

## Prerrequisitos

### Conocimientos requeridos

Antes de comenzar, debes haber completado o comprender los resultados de:

- Laboratorio `05-00-01`: agente de soporte con las herramientas `search_faq`, `calculate` y `get_ticket_status`.
- Laboratorio `06-00-01`: creación de un *golden set* y evaluadores de precisión y fidelidad.
- Conceptos de evaluación de respuestas: exactitud, relevancia, fidelidad al contexto, tasa de éxito, percentiles p50/p95 y regresión.
- Uso básico de Azure OpenAI, variables de entorno, Python, Git y entornos virtuales.

### Accesos requeridos

Debes contar con:

- Una cuenta de LangSmith con una API key válida y permisos para crear proyectos, datasets y experimentos.
- Una suscripción de Azure con acceso a un recurso Azure OpenAI.
- Un despliegue compatible con chat y Function Calling, preferiblemente `gpt-4o-mini` versión `2024-07-18`.
- Un archivo `.env` local con credenciales de Azure OpenAI y LangSmith.
- Acceso de escritura al repositorio local `~/genai-agent-labs`.

> **Importante:** no publiques claves de Azure OpenAI ni de LangSmith en Git, código fuente, capturas de pantalla o archivos versionados.

## Entorno del laboratorio

### Recursos de referencia

| Recurso | Requisito |
|---|---|
| Sistema operativo | Ubuntu 22.04.4 LTS o compatible |
| Python | 3.12.1 |
| Memoria RAM | 16 GB mínimo; 32 GB recomendados |
| Espacio libre | 10 GB mínimo |
| Repositorio | `~/genai-agent-labs` |
| Entorno virtual compartido | `~/genai-agent-labs/.venv` |
| Proyecto LangSmith | `genai-agents-batch3` |
| Dataset LangSmith | `support-golden-set-v1` |

### Versiones de paquetes

Esta práctica utiliza las siguientes versiones:

| Paquete | Versión |
|---|---|
| `langchain` | `0.3.14` |
| `langchain-openai` | `0.2.14` |
| `langsmith` | `0.2.11` |
| `openai` | `1.12.0` |
| `pydantic` | `2.6.1` |
| `python-dotenv` | `1.0.1` |

### Paso 1. Preparar el repositorio, entorno y dependencias

**Objetivo:** verificar que el repositorio compartido, el entorno virtual y las dependencias de LangChain/LangSmith están listos.

**Instrucciones:**

1. Abre una terminal y accede al directorio obligatorio del curso.

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido.

   ```bash
   source .venv/bin/activate
   ```

3. Verifica la versión de Python.

   ```bash
   python --version
   ```

4. Instala las dependencias requeridas para esta práctica.

   ```bash
   pip install \
     "langchain==0.3.14" \
     "langchain-openai==0.2.14" \
     "langsmith==0.2.11" \
     "python-dotenv==1.0.1" \
     "pydantic==2.6.1"
   ```

5. Crea las rutas necesarias si aún no existen.

   ```bash
   mkdir -p \
     src/observability \
     src/support \
     data/evaluation \
     reports \
     tests
   ```

6. Verifica las versiones instaladas.

   ```bash
   python -c "import langchain, langchain_openai, langsmith; print('langchain=', langchain.__version__); print('langchain-openai=', langchain_openai.__version__); print('langsmith=', langsmith.__version__)"
   ```

**Salida esperada:**

Debes observar versiones equivalentes a las siguientes:

```text
langchain= 0.3.14
langchain-openai= 0.2.14
langsmith= 0.2.11
```

**Verificación:**

Ejecuta:

```bash
git status --short
```

El comando no debe mostrar secretos ni archivos `.env` preparados para ser confirmados.

---

### Paso 2. Configurar variables de entorno para Azure OpenAI y LangSmith

**Objetivo:** habilitar la trazabilidad de LangSmith sin exponer secretos en código fuente.

**Instrucciones:**

1. Comprueba que `.env` esté ignorado por Git.

   ```bash
   grep -n "^\.env$" .gitignore || echo ".env" >> .gitignore
   ```

2. Crea o actualiza el archivo `.env`.

   ```bash
   nano .env
   ```

3. Agrega los siguientes valores. Sustituye los marcadores por valores reales de tu entorno.

   ```dotenv
   AZURE_OPENAI_API_KEY=<tu-clave-de-azure-openai>
   AZURE_OPENAI_ENDPOINT=https://<tu-recurso>.openai.azure.com/
   AZURE_OPENAI_API_VERSION=2024-06-01
   AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini

   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_API_KEY=<tu-clave-de-langsmith>
   LANGCHAIN_PROJECT=genai-agents-batch3
   LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
   ```

4. Establece permisos restrictivos para el archivo.

   ```bash
   chmod 600 .env
   ```

5. Crea un archivo de ejemplo seguro para documentar las variables sin incluir valores reales.

   ```bash
   cat > .env.example <<'EOF'
   AZURE_OPENAI_API_KEY=
   AZURE_OPENAI_ENDPOINT=
   AZURE_OPENAI_API_VERSION=2024-06-01
   AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini

   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_API_KEY=
   LANGCHAIN_PROJECT=genai-agents-batch3
   LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
   EOF
   ```

6. Comprueba que `.env` no será incluido en Git.

   ```bash
   git check-ignore -v .env
   ```

7. Carga temporalmente las variables y valida que LangSmith puede autenticarse.

   ```bash
   set -a
   source .env
   set +a

   python -c "from langsmith import Client; client = Client(); print('Conexión LangSmith inicializada para el proyecto:', __import__('os').environ['LANGCHAIN_PROJECT'])"
   ```

**Salida esperada:**

Debes observar un mensaje similar a:

```text
Conexión LangSmith inicializada para el proyecto: genai-agents-batch3
```

**Verificación:**

Confirma los valores no secretos:

```bash
grep -E "^(LANGCHAIN_TRACING_V2|LANGCHAIN_PROJECT|LANGCHAIN_ENDPOINT)=" .env
```

La salida debe incluir:

```text
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=genai-agents-batch3
LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
```

> **Nota:** `LANGCHAIN_TRACING_V2=true` activa el envío de trazas. `LANGCHAIN_PROJECT` separa este laboratorio de otros proyectos y permite comparar ejecuciones homogéneas.

---

### Paso 3. Crear el golden set local de soporte técnico

**Objetivo:** disponer de un conjunto de evaluación reproducible que cubra recuperación de FAQ, cálculo y consulta de tickets.

**Instrucciones:**

1. Crea el archivo `data/evaluation/golden_set.jsonl`.

   ```bash
   cat > data/evaluation/golden_set.jsonl <<'EOF'
   {"id":"support-001","input":"¿Cómo restablezco mi contraseña?","expected_route":"faq","expected_tool":"search_faq","expected_terms":["restablecer","contraseña","correo"],"expected_evidence_id":"FAQ-001","risk":"medium","category":"factual"}
   {"id":"support-002","input":"¿Qué debo hacer si falla la autenticación multifactor?","expected_route":"faq","expected_tool":"search_faq","expected_terms":["mfa","soporte","dispositivo"],"expected_evidence_id":"FAQ-002","risk":"high","category":"factual"}
   {"id":"support-003","input":"Calcula 18 * 7 + 4","expected_route":"calculation","expected_tool":"calculate","expected_terms":["130"],"expected_evidence_id":"CALCULATION","risk":"low","category":"structured"}
   {"id":"support-004","input":"¿Cuál es el estado del ticket INC-1042?","expected_route":"ticket","expected_tool":"get_ticket_status","expected_terms":["inc-1042","en progreso","infraestructura"],"expected_evidence_id":"TICKET-INC-1042","risk":"high","category":"factual"}
   {"id":"support-005","input":"Necesito saber el estado de INC-9999","expected_route":"ticket","expected_tool":"get_ticket_status","expected_terms":["inc-9999","no encontrado"],"expected_evidence_id":"TICKET-INC-9999","risk":"medium","category":"negative"}
   EOF
   ```

2. Revisa que el archivo tenga cinco registros JSON válidos.

   ```bash
   wc -l data/evaluation/golden_set.jsonl
   python -c "import json; [json.loads(line) for line in open('data/evaluation/golden_set.jsonl', encoding='utf-8')]; print('JSONL válido')"
   ```

3. Muestra el contenido de forma legible.

   ```bash
   python -m json.tool < <(head -n 1 data/evaluation/golden_set.jsonl)
   ```

**Salida esperada:**

```text
5 data/evaluation/golden_set.jsonl
JSONL válido
```

**Verificación:**

Cada caso debe incluir, como mínimo:

- `id`
- `input`
- `expected_route`
- `expected_tool`
- `expected_terms`
- `expected_evidence_id`
- `risk`
- `category`

> **Criterio de calidad:** los casos de riesgo alto, como el estado de un incidente o el fallo de MFA, no deben quedar ocultos por una métrica promedio. Posteriormente se segmentarán al analizar resultados.

---

### Paso 4. Implementar herramientas reutilizables y recuperador de evidencia

**Objetivo:** conservar los contratos funcionales de las herramientas del agente de `05-00-01` y registrar evidencia verificable.

**Instrucciones:**

1. Crea el archivo `src/support/tools.py`.

   ```bash
   cat > src/support/tools.py <<'EOF'
   from __future__ import annotations

   import ast
   import operator
   import re
   from typing import Any


   FAQS = [
       {
           "id": "FAQ-000",
           "title": "Autenticación en aplicaciones internas",
           "content": (
               "Si no puede iniciar sesión en una aplicación interna, confirme "
               "su usuario corporativo y revise la conectividad antes de abrir un ticket."
           ),
       },
       {
           "id": "FAQ-001",
           "title": "Restablecimiento de contraseña",
           "content": (
               "Para restablecer la contraseña, use el portal de autoservicio. "
               "Recibirá un correo de confirmación y deberá definir una nueva contraseña."
           ),
       },
       {
           "id": "FAQ-002",
           "title": "Fallo de autenticación multifactor",
           "content": (
               "Si MFA falla, valide la hora del dispositivo y vuelva a registrar "
               "el método de autenticación. Si el problema continúa, contacte a soporte."
           ),
       },
   ]

   TICKETS = {
       "INC-1042": {
           "status": "En progreso",
           "team": "Infraestructura",
           "summary": "Investigación de pérdida intermitente de conectividad VPN.",
       }
   }

   ALLOWED_OPERATORS: dict[type, Any] = {
       ast.Add: operator.add,
       ast.Sub: operator.sub,
       ast.Mult: operator.mul,
       ast.Div: operator.truediv,
       ast.USub: operator.neg,
       ast.Pow: operator.pow,
   }


   def _score(query: str, document: str) -> int:
       terms = set(re.findall(r"\w+", query.lower()))
       content_terms = set(re.findall(r"\w+", document.lower()))
       return len(terms.intersection(content_terms))


   def search_faq(query: str, max_results: int = 3) -> list[dict[str, str]]:
       """Busca preguntas frecuentes y retorna evidencia ordenada por relevancia."""
       ranked = sorted(
           FAQS,
           key=lambda item: _score(query, f"{item['title']} {item['content']}"),
           reverse=True,
       )
       return ranked[:max_results]


   def _evaluate_expression(node: ast.AST) -> float | int:
       if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
           return node.value

       if isinstance(node, ast.BinOp) and type(node.op) in ALLOWED_OPERATORS:
           return ALLOWED_OPERATORS[type(node.op)](
               _evaluate_expression(node.left),
               _evaluate_expression(node.right),
           )

       if isinstance(node, ast.UnaryOp) and type(node.op) in ALLOWED_OPERATORS:
           return ALLOWED_OPERATORS[type(node.op)](_evaluate_expression(node.operand))

       raise ValueError("Expresión no permitida")


   def calculate(expression: str) -> dict[str, str]:
       """Evalúa una expresión aritmética limitada sin usar eval()."""
       try:
           tree = ast.parse(expression, mode="eval")
           result = _evaluate_expression(tree.body)
           return {
               "id": "CALCULATION",
               "expression": expression,
               "result": str(result),
           }
       except (SyntaxError, ValueError, TypeError, ZeroDivisionError) as exc:
           return {
               "id": "CALCULATION-ERROR",
               "expression": expression,
               "error": f"No se pudo calcular la expresión: {exc}",
           }


   def get_ticket_status(ticket_id: str) -> dict[str, str]:
       """Obtiene el estado de un ticket de soporte por identificador."""
       normalized_id = ticket_id.upper().strip()
       ticket = TICKETS.get(normalized_id)

       if ticket is None:
           return {
               "id": f"TICKET-{normalized_id}",
               "ticket_id": normalized_id,
               "status": "No encontrado",
               "team": "N/A",
               "summary": "No existe un ticket con ese identificador.",
           }

       return {
           "id": f"TICKET-{normalized_id}",
           "ticket_id": normalized_id,
           **ticket,
       }
   EOF
   ```

2. Ejecuta una prueba local de las tres herramientas.

   ```bash
   PYTHONPATH=src python - <<'PY'
   from support.tools import calculate, get_ticket_status, search_faq

   print(search_faq("restablecer contraseña"))
   print(calculate("18 * 7 + 4"))
   print(get_ticket_status("INC-1042"))
   PY
   ```

3. Confirma que no se usa `eval()`.

   ```bash
   grep -n "eval(" src/support/tools.py || true
   ```

**Salida esperada:**

La prueba debe mostrar:

- Una FAQ con `id` igual a `FAQ-001`.
- Un cálculo con resultado `130`.
- Un ticket `INC-1042` con estado `En progreso`.

**Verificación:**

Ejecuta:

```bash
PYTHONPATH=src python - <<'PY'
from support.tools import calculate, get_ticket_status, search_faq

assert search_faq("restablecer contraseña")[0]["id"] == "FAQ-001"
assert calculate("18 * 7 + 4")["result"] == "130"
assert get_ticket_status("INC-1042")["status"] == "En progreso"
print("Contratos de herramientas verificados.")
PY
```

> Si ya dispones de herramientas equivalentes creadas en `05-00-01`, conserva sus contratos y adapta solamente las importaciones del archivo de instrumentación. No cambies nombres, tipos de argumentos ni comportamiento esperado de `search_faq`, `calculate` y `get_ticket_status`.

---

### Paso 5. Implementar el agente instrumentado con LangChain y LangSmith

**Objetivo:** instrumentar cada etapa importante del flujo del agente y asociar las trazas a un laboratorio, escenario, versión de dataset y commit Git.

**Instrucciones:**

1. Crea el archivo `src/observability/langchain_support_agent.py`.

   ```bash
   cat > src/observability/langchain_support_agent.py <<'EOF'
   from __future__ import annotations

   import json
   import os
   import re
   import subprocess
   from typing import Any, Literal

   from dotenv import load_dotenv
   from langchain_core.messages import HumanMessage, SystemMessage
   from langchain_core.tools import tool
   from langchain_openai import AzureChatOpenAI
   from langsmith import traceable

   from support.tools import calculate, get_ticket_status, search_faq


   load_dotenv()

   LAB_ID = "06-00-02"
   DATASET_VERSION = "support-golden-set-v1"


   def get_git_commit() -> str:
       try:
           return subprocess.check_output(
               ["git", "rev-parse", "--short", "HEAD"],
               text=True,
               stderr=subprocess.DEVNULL,
           ).strip()
       except Exception:
           return "uncommitted"


   def build_llm() -> AzureChatOpenAI:
       return AzureChatOpenAI(
           azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
           api_key=os.environ["AZURE_OPENAI_API_KEY"],
           api_version=os.getenv("AZURE_OPENAI_API_VERSION", "2024-06-01"),
           azure_deployment=os.environ["AZURE_OPENAI_CHAT_DEPLOYMENT"],
           temperature=0,
           max_retries=2,
       )


   @tool
   def search_faq_tool(query: str) -> str:
       """Busca evidencia en preguntas frecuentes de soporte técnico."""
       max_results = int(os.getenv("FAQ_RESULT_LIMIT", "3"))
       results = search_faq(query, max_results=max_results)
       return json.dumps(results, ensure_ascii=False)


   @tool
   def calculate_tool(expression: str) -> str:
       """Calcula una expresión aritmética como 18 * 7 + 4."""
       return json.dumps(calculate(expression), ensure_ascii=False)


   @tool
   def get_ticket_status_tool(ticket_id: str) -> str:
       """Consulta el estado de un ticket con formato INC-1234."""
       return json.dumps(get_ticket_status(ticket_id), ensure_ascii=False)


   TOOLS = [search_faq_tool, calculate_tool, get_ticket_status_tool]
   TOOL_BY_NAME = {item.name: item for item in TOOLS}


   @traceable(name="clasificacion", run_type="chain")
   def classify_request(user_input: str, config: dict[str, Any]) -> str:
       llm = build_llm()
       prompt = [
           SystemMessage(
               content=(
                   "Clasifica la solicitud en exactamente una categoría: "
                   "faq, calculation o ticket. Responde solo con una categoría."
               )
           ),
           HumanMessage(content=user_input),
       ]
       result = llm.invoke(prompt, config=config)
       category = result.content.strip().lower()

       if category not in {"faq", "calculation", "ticket"}:
           if re.search(r"\binc-\d+\b", user_input, re.IGNORECASE):
               return "ticket"
           if re.search(r"[\d][\d\s+*/().-]*[\d]", user_input):
               return "calculation"
           return "faq"

       return category


   @traceable(name="seleccion_de_herramienta", run_type="chain")
   def select_tool(
       user_input: str,
       route: str,
       regression_mode: bool,
       config: dict[str, Any],
   ) -> dict[str, Any]:
       llm = build_llm().bind_tools(TOOLS, tool_choice="any")

       tool_description = (
           "Selecciona una herramienta. Para FAQ usa search_faq_tool; "
           "para cálculos usa calculate_tool; para tickets usa get_ticket_status_tool."
       )

       if regression_mode:
           tool_description = (
               "Selecciona la herramienta que parezca más útil. "
               "Las herramientas de FAQ y tickets pueden contener información similar."
           )

       messages = [
           SystemMessage(content=tool_description),
           HumanMessage(
               content=(
                   f"Ruta clasificada: {route}\n"
                   f"Solicitud del usuario: {user_input}\n"
                   "Invoca exactamente una herramienta."
               )
           ),
       ]
       response = llm.invoke(messages, config=config)

       if response.tool_calls:
           return response.tool_calls[0]

       fallback = {
           "faq": {"name": "search_faq_tool", "args": {"query": user_input}},
           "calculation": {"name": "calculate_tool", "args": {"expression": user_input}},
           "ticket": {
               "name": "get_ticket_status_tool",
               "args": {
                   "ticket_id": (
                       re.search(r"\bINC-\d+\b", user_input, re.IGNORECASE).group(0)
                       if re.search(r"\bINC-\d+\b", user_input, re.IGNORECASE)
                       else user_input
                   )
               },
           },
       }
       return fallback[route]


   @traceable(name="invocacion_de_herramienta", run_type="tool")
   def invoke_tool(tool_call: dict[str, Any]) -> dict[str, Any]:
       tool_name = tool_call["name"]
       selected_tool = TOOL_BY_NAME.get(tool_name)

       if selected_tool is None:
           return {
               "tool_name": tool_name,
               "error": "Herramienta no registrada",
               "raw_result": "",
           }

       try:
           result = selected_tool.invoke(tool_call["args"])
           return {
               "tool_name": tool_name,
               "raw_result": result,
               "error": None,
           }
       except Exception as exc:
           return {
               "tool_name": tool_name,
               "raw_result": "",
               "error": f"{type(exc).__name__}: {exc}",
           }


   @traceable(name="recuperacion_de_evidencia", run_type="retriever")
   def normalize_evidence(tool_result: dict[str, Any]) -> list[dict[str, Any]]:
       if tool_result["error"]:
           return [{"id": "TOOL-ERROR", "content": tool_result["error"]}]

       try:
           parsed = json.loads(tool_result["raw_result"])
       except json.JSONDecodeError:
           return [{"id": "PARSE-ERROR", "content": tool_result["raw_result"]}]

       if isinstance(parsed, list):
           return [
               {
                   "id": item.get("id", "UNKNOWN"),
                   "content": item.get("content", item.get("summary", str(item))),
               }
               for item in parsed
           ]

       return [
           {
               "id": parsed.get("id", "UNKNOWN"),
               "content": json.dumps(parsed, ensure_ascii=False),
           }
       ]


   @traceable(name="respuesta_final", run_type="chain")
   def generate_final_answer(
       user_input: str,
       route: str,
       evidence: list[dict[str, Any]],
       config: dict[str, Any],
   ) -> str:
       llm = build_llm()
       evidence_text = json.dumps(evidence, ensure_ascii=False)

       messages = [
           SystemMessage(
               content=(
                   "Eres un asistente de soporte técnico. Responde en español. "
                   "Usa únicamente la evidencia entregada. Si falta información, indícalo. "
                   "Incluye los identificadores de evidencia relevantes entre corchetes."
               )
           ),
           HumanMessage(
               content=(
                   f"Solicitud: {user_input}\n"
                   f"Ruta: {route}\n"
                   f"Evidencia: {evidence_text}"
               )
           ),
       ]
       return llm.invoke(messages, config=config).content.strip()


   @traceable(name="support_agent_langchain", run_type="chain")
   def run_agent(
       user_input: str,
       scenario_id: str,
       regression_mode: bool = False,
   ) -> dict[str, Any]:
       metadata = {
           "lab_id": LAB_ID,
           "route": "pending",
           "dataset_version": DATASET_VERSION,
           "git_commit": get_git_commit(),
           "scenario_id": scenario_id,
           "regression_mode": regression_mode,
       }
       config = {
           "metadata": metadata,
           "tags": [LAB_ID, DATASET_VERSION, "function-calling"],
       }

       route = classify_request(user_input, config)
       metadata["route"] = route

       tool_call = select_tool(user_input, route, regression_mode, config)
       tool_result = invoke_tool(tool_call)
       evidence = normalize_evidence(tool_result)
       answer = generate_final_answer(user_input, route, evidence, config)

       return {
           "answer": answer,
           "route": route,
           "tool_name": tool_result["tool_name"],
           "tool_error": tool_result["error"],
           "evidence": evidence,
       }


   if __name__ == "__main__":
       import argparse

       parser = argparse.ArgumentParser()
       parser.add_argument("input", help="Solicitud de soporte")
       parser.add_argument("--scenario-id", default="manual-001")
       parser.add_argument("--regression-mode", action="store_true")
       args = parser.parse_args()

       output = run_agent(
           user_input=args.input,
           scenario_id=args.scenario_id,
           regression_mode=args.regression_mode,
       )
       print(json.dumps(output, ensure_ascii=False, indent=2))
   EOF
   ```

2. Ejecuta una consulta manual instrumentada.

   ```bash
   PYTHONPATH=src python -m observability.langchain_support_agent \
     "¿Cuál es el estado del ticket INC-1042?" \
     --scenario-id manual-ticket-001
   ```

3. Conserva la salida JSON para inspeccionarla.

   ```bash
   PYTHONPATH=src python -m observability.langchain_support_agent \
     "Calcula 18 * 7 + 4" \
     --scenario-id manual-calc-001 \
     | tee reports/manual_agent_run.json
   ```

**Salida esperada:**

La respuesta debe incluir campos similares a:

```json
{
  "answer": "El resultado de 18 * 7 + 4 es 130. [CALCULATION]",
  "route": "calculation",
  "tool_name": "calculate_tool",
  "tool_error": null,
  "evidence": [
    {
      "id": "CALCULATION",
      "content": "{\"id\": \"CALCULATION\", \"expression\": \"18 * 7 + 4\", \"result\": \"130\"}"
    }
  ]
}
```

**Verificación:**

1. Abre LangSmith en el navegador.
2. Selecciona el proyecto `genai-agents-batch3`.
3. Busca una traza llamada `support_agent_langchain`.
4. Verifica que incluya subejecuciones con estos nombres:

   - `clasificacion`
   - `seleccion_de_herramienta`
   - `invocacion_de_herramienta`
   - `recuperacion_de_evidencia`
   - `respuesta_final`

5. Revisa los metadatos de la ejecución. Deben incluir:

   - `lab_id: 06-00-02`
   - `dataset_version: support-golden-set-v1`
   - `git_commit`
   - `scenario_id`
   - `route`
   - `regression_mode`

> **Interpretación:** una traza útil no solo registra el resultado final. Debe permitir responder qué ruta tomó el agente, qué herramienta eligió, qué evidencia recuperó, cuánto tardó cada etapa y dónde ocurrió un error.

---

### Paso 6. Implementar evaluadores de precisión y fidelidad

**Objetivo:** reutilizar la lógica de evaluación del laboratorio anterior para calificar resultados de LangSmith de forma automática.

**Instrucciones:**

1. Crea el archivo `src/observability/evaluation_metrics.py`.

   ```bash
   cat > src/observability/evaluation_metrics.py <<'EOF'
   from __future__ import annotations

   from typing import Any


   def _normalized(text: str) -> str:
       return text.lower().strip()


   def precision_evaluator(
       *,
       outputs: dict[str, Any],
       reference_outputs: dict[str, Any],
       **_: Any,
   ) -> dict[str, Any]:
       answer = _normalized(outputs.get("answer", ""))
       expected_terms = [
           _normalized(term)
           for term in reference_outputs.get("expected_terms", [])
       ]

       if not expected_terms:
           return {
               "key": "precision",
               "score": 0,
               "comment": "El caso no contiene expected_terms.",
           }

       matched_terms = [term for term in expected_terms if term in answer]
       score = len(matched_terms) / len(expected_terms)

       return {
           "key": "precision",
           "score": score,
           "comment": (
               f"Términos esperados encontrados: {matched_terms}; "
               f"total esperado: {expected_terms}"
           ),
       }


   def faithfulness_evaluator(
       *,
       outputs: dict[str, Any],
       reference_outputs: dict[str, Any],
       **_: Any,
   ) -> dict[str, Any]:
       expected_evidence_id = reference_outputs.get("expected_evidence_id", "")
       evidence_ids = {
           item.get("id", "")
           for item in outputs.get("evidence", [])
           if isinstance(item, dict)
       }

       tool_name = outputs.get("tool_name", "")
       expected_tool = reference_outputs.get("expected_tool", "")

       evidence_ok = expected_evidence_id in evidence_ids
       tool_ok = tool_name == expected_tool

       score = 1.0 if evidence_ok and tool_ok else 0.0

       return {
           "key": "faithfulness",
           "score": score,
           "comment": (
               f"Evidencia esperada={expected_evidence_id}; "
               f"evidencia observada={sorted(evidence_ids)}; "
               f"herramienta esperada={expected_tool}; "
               f"herramienta observada={tool_name}"
           ),
       }


   def route_evaluator(
       *,
       outputs: dict[str, Any],
       reference_outputs: dict[str, Any],
       **_: Any,
   ) -> dict[str, Any]:
       expected_route = reference_outputs.get("expected_route")
       observed_route = outputs.get("route")
       score = 1.0 if expected_route == observed_route else 0.0

       return {
           "key": "route_accuracy",
           "score": score,
           "comment": f"Ruta esperada={expected_route}; ruta observada={observed_route}",
       }
   EOF
   ```

2. Prueba los evaluadores de manera aislada.

   ```bash
   PYTHONPATH=src python - <<'PY'
   from observability.evaluation_metrics import (
       faithfulness_evaluator,
       precision_evaluator,
       route_evaluator,
   )

   outputs = {
       "answer": "El ticket INC-1042 está En progreso y lo atiende Infraestructura. [TICKET-INC-1042]",
       "route": "ticket",
       "tool_name": "get_ticket_status_tool",
       "evidence": [{"id": "TICKET-INC-1042"}],
   }

   reference = {
       "expected_route": "ticket",
       "expected_tool": "get_ticket_status_tool",
       "expected_terms": ["inc-1042", "en progreso", "infraestructura"],
       "expected_evidence_id": "TICKET-INC-1042",
   }

   print(precision_evaluator(outputs=outputs, reference_outputs=reference))
   print(faithfulness_evaluator(outputs=outputs, reference_outputs=reference))
   print(route_evaluator(outputs=outputs, reference_outputs=reference))
   PY
   ```

**Salida esperada:**

Los tres evaluadores deben devolver una puntuación de `1.0`.

**Verificación:**

Comprueba que las métricas representan dimensiones diferentes:

- `precision`: verifica términos esperados en la respuesta.
- `faithfulness`: verifica herramienta y evidencia recuperada.
- `route_accuracy`: verifica la clasificación de la solicitud.

> **Nota metodológica:** la precisión léxica no sustituye una revisión humana ni una rúbrica semántica. En este laboratorio se utiliza como señal automatizada y repetible. La fidelidad evita aprobar respuestas aparentemente correctas que hayan usado una herramienta o evidencia incorrecta.

---

### Paso 7. Publicar o actualizar el dataset en LangSmith

**Objetivo:** convertir el golden set local en un dataset centralizado y reutilizable para experimentos comparables.

**Instrucciones:**

1. Crea el script `src/observability/publish_dataset.py`.

   ```bash
   cat > src/observability/publish_dataset.py <<'EOF'
   from __future__ import annotations

   import json
   from pathlib import Path

   from dotenv import load_dotenv
   from langsmith import Client


   DATASET_NAME = "support-golden-set-v1"
   DATASET_DESCRIPTION = (
       "Golden set de soporte técnico para evaluación de rutas, herramientas, "
       "precisión y fidelidad del laboratorio 06-00-02."
   )


   def load_cases(path: Path) -> list[dict]:
       with path.open(encoding="utf-8") as file:
           return [json.loads(line) for line in file if line.strip()]


   def main() -> None:
       load_dotenv()
       client = Client()
       cases = load_cases(Path("data/evaluation/golden_set.jsonl"))

       try:
           dataset = client.read_dataset(dataset_name=DATASET_NAME)
           print(f"Dataset existente: {dataset.name} ({dataset.id})")
       except Exception:
           dataset = client.create_dataset(
               dataset_name=DATASET_NAME,
               description=DATASET_DESCRIPTION,
           )
           print(f"Dataset creado: {dataset.name} ({dataset.id})")

       existing_examples = list(client.list_examples(dataset_id=dataset.id))
       existing_ids = {
           example.inputs.get("scenario_id")
           for example in existing_examples
           if example.inputs
       }

       for case in cases:
           if case["id"] in existing_ids:
               print(f"Omitido, ya existe: {case['id']}")
               continue

           client.create_example(
               dataset_id=dataset.id,
               inputs={
                   "input": case["input"],
                   "scenario_id": case["id"],
               },
               outputs={
                   "expected_route": case["expected_route"],
                   "expected_tool": case["expected_tool"],
                   "expected_terms": case["expected_terms"],
                   "expected_evidence_id": case["expected_evidence_id"],
                   "risk": case["risk"],
                   "category": case["category"],
               },
           )
           print(f"Ejemplo creado: {case['id']}")

       total = len(list(client.list_examples(dataset_id=dataset.id)))
       print(f"Dataset listo: {DATASET_NAME}; ejemplos totales: {total}")


   if __name__ == "__main__":
       main()
   EOF
   ```

2. Ejecuta el publicador.

   ```bash
   PYTHONPATH=src python -m observability.publish_dataset
   ```

3. Ejecuta el script una segunda vez para comprobar que es idempotente.

   ```bash
   PYTHONPATH=src python -m observability.publish_dataset
   ```

**Salida esperada:**

En la primera ejecución se crearán el dataset y cinco ejemplos. En la segunda, los cinco ejemplos deberán aparecer como omitidos.

Ejemplo:

```text
Dataset creado: support-golden-set-v1 (...)
Ejemplo creado: support-001
...
Dataset listo: support-golden-set-v1; ejemplos totales: 5
```

**Verificación:**

En LangSmith:

1. Abre la sección **Datasets**.
2. Selecciona `support-golden-set-v1`.
3. Confirma que existen cinco ejemplos.
4. Revisa que cada ejemplo incluya:
   - Entrada `input`.
   - Entrada `scenario_id`.
   - Salidas de referencia, incluyendo `expected_terms`, `expected_tool` y `expected_evidence_id`.

---

### Paso 8. Ejecutar el experimento de línea base

**Objetivo:** establecer el experimento `baseline-function-calling-v1` como referencia de calidad y comportamiento operacional.

**Instrucciones:**

1. Crea el script `src/observability/run_experiment.py`.

   ```bash
   cat > src/observability/run_experiment.py <<'EOF'
   from __future__ import annotations

   import argparse
   import json
   from pathlib import Path
   from typing import Any

   from dotenv import load_dotenv
   from langsmith import Client, evaluate

   from observability.evaluation_metrics import (
       faithfulness_evaluator,
       precision_evaluator,
       route_evaluator,
   )
   from observability.langchain_support_agent import run_agent


   DATASET_NAME = "support-golden-set-v1"


   def target(inputs: dict[str, Any]) -> dict[str, Any]:
       return run_agent(
           user_input=inputs["input"],
           scenario_id=inputs["scenario_id"],
           regression_mode=False,
       )


   def regression_target(inputs: dict[str, Any]) -> dict[str, Any]:
       return run_agent(
           user_input=inputs["input"],
           scenario_id=inputs["scenario_id"],
           regression_mode=True,
       )


   def main() -> None:
       parser = argparse.ArgumentParser()
       parser.add_argument(
           "--experiment-prefix",
           required=True,
           help="Prefijo visible del experimento en LangSmith.",
       )
       parser.add_argument(
           "--regression-mode",
           action="store_true",
           help="Activa la regresión controlada de selección de herramienta.",
       )
       args = parser.parse_args()

       load_dotenv()
       client = Client()
       selected_target = regression_target if args.regression_mode else target

       results = evaluate(
           selected_target,
           data=DATASET_NAME,
           evaluators=[
               precision_evaluator,
               faithfulness_evaluator,
               route_evaluator,
           ],
           experiment_prefix=args.experiment_prefix,
           client=client,
           max_concurrency=1,
           metadata={
               "lab_id": "06-00-02",
               "dataset_version": DATASET_NAME,
               "execution_type": (
                   "regression" if args.regression_mode else "baseline"
               ),
           },
       )

       summary = {
           "experiment_prefix": args.experiment_prefix,
           "regression_mode": args.regression_mode,
           "results": str(results),
       }

       Path("reports").mkdir(exist_ok=True)
       Path(
           f"reports/{args.experiment_prefix}_launch.json"
       ).write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding="utf-8")

       print(f"Experimento enviado a LangSmith: {args.experiment_prefix}")


   if __name__ == "__main__":
       main()
   EOF
   ```

2. Ejecuta el experimento base.

   ```bash
   PYTHONPATH=src python -m observability.run_experiment \
     --experiment-prefix baseline-function-calling-v1
   ```

3. Espera a que finalice la ejecución y abre LangSmith.

4. En el proyecto `genai-agents-batch3`, abre la sección **Experiments**.

5. Localiza el experimento cuyo nombre comienza con:

   ```text
   baseline-function-calling-v1
   ```

6. Registra manualmente los valores observados en `reports/baseline_observations.md`.

   ```bash
   cat > reports/baseline_observations.md <<'EOF'
   # Línea base operacional: baseline-function-calling-v1

   | Métrica | Valor observado |
   |---|---:|
   | Número de casos | |
   | Precisión media | |
   | Fidelidad media | |
   | Exactitud de ruta media | |
   | Tasa de éxito con umbral >= 0.80 | |
   | Latencia p50 total | |
   | Latencia p95 total | |
   | Tokens promedio de entrada | |
   | Tokens promedio de salida | |
   | Errores de herramienta | |

   ## Casos de riesgo alto
   - support-002:
   - support-004:

   ## Observaciones de trazas
   - Llamadas redundantes:
   - Etapa más lenta:
   - Errores o reintentos:
   - Hipótesis de mejora:
   EOF
   ```

**Salida esperada:**

El script debe confirmar el envío del experimento:

```text
Experimento enviado a LangSmith: baseline-function-calling-v1
```

**Verificación:**

En el experimento de LangSmith, confirma:

- Existen cinco ejecuciones.
- Cada ejecución tiene resultados de:
  - `precision`
  - `faithfulness`
  - `route_accuracy`
- Cada ejecución enlaza con su traza completa.
- Las respuestas tienen una herramienta y evidencia asociadas.
- Los casos `support-002` y `support-004` pueden identificarse por `scenario_id`.

> **Criterio de línea base recomendado:** para aprobar esta práctica, la línea base debe alcanzar una puntuación media de fidelidad de al menos `0.80`, precisión media de al menos `0.80` y no debe presentar errores de herramienta en casos de riesgo alto.

---

### Paso 9. Analizar trazas, latencia, tokens y errores operacionales

**Objetivo:** identificar cuellos de botella y fallos que no son visibles al observar solamente una respuesta final.

**Instrucciones:**

1. En LangSmith, abre el experimento `baseline-function-calling-v1`.

2. Ordena las ejecuciones por duración total descendente.

3. Abre la ejecución con mayor duración y revisa el árbol de trazas.

4. Identifica la duración de las siguientes etapas:

   - `clasificacion`
   - `seleccion_de_herramienta`
   - `invocacion_de_herramienta`
   - `recuperacion_de_evidencia`
   - `respuesta_final`

5. En cada llamada al modelo, revisa los metadatos de uso disponibles:
   - Tokens de entrada.
   - Tokens de salida.
   - Tokens totales.
   - Duración.
   - Reintentos.
   - Errores de proveedor.

6. Filtra o busca ejecuciones que contengan:
   - `tool_error` distinto de `null`.
   - Puntuación de `faithfulness` inferior a `0.80`.
   - Puntuación de `precision` inferior a `0.80`.
   - Casos con `risk=high`.

7. Completa `reports/baseline_observations.md` con los valores observados.

8. Calcula los percentiles si exportaste las duraciones a CSV. Sustituye el contenido de `reports/latency_analysis.py` por el siguiente script y adapta el archivo CSV de entrada si es necesario.

   ```bash
   cat > reports/latency_analysis.py <<'EOF'
   import csv
   import statistics
   from pathlib import Path


   def percentile(values: list[float], percentage: float) -> float:
       ordered = sorted(values)
       index = (len(ordered) - 1) * percentage
       lower = int(index)
       upper = min(lower + 1, len(ordered) - 1)
       fraction = index - lower
       return ordered[lower] + (ordered[upper] - ordered[lower]) * fraction


   csv_path = Path("reports/experiment_latencies.csv")

   if not csv_path.exists():
       print(
           "Exporte las duraciones desde LangSmith a "
           "reports/experiment_latencies.csv con una columna duration_ms."
       )
       raise SystemExit(0)

   with csv_path.open(encoding="utf-8") as file:
       values = [
           float(row["duration_ms"])
           for row in csv.DictReader(file)
           if row.get("duration_ms")
       ]

   print(f"Casos: {len(values)}")
   print(f"Promedio: {statistics.mean(values):.2f} ms")
   print(f"p50: {percentile(values, 0.50):.2f} ms")
   print(f"p95: {percentile(values, 0.95):.2f} ms")
   EOF
   ```

9. Ejecuta el análisis cuando tengas el CSV.

   ```bash
   python reports/latency_analysis.py
   ```

**Salida esperada:**

Debes poder responder, con evidencia de trazas, preguntas como:

- ¿Qué etapa contribuye más a la latencia total?
- ¿Se invoca una herramienta más de una vez para el mismo caso?
- ¿Existen reintentos del modelo o errores de herramientas?
- ¿Qué casos tienen calidad inferior al umbral?
- ¿Los casos de riesgo alto aprueban las métricas?
- ¿Qué llamadas consumen más tokens?

**Verificación:**

Completa esta tabla en `reports/baseline_observations.md` con datos reales de tu experimento:

| Pregunta operacional | Evidencia esperada |
|---|---|
| ¿Cuál es la etapa más lenta? | Nombre de etapa y duración aproximada |
| ¿Hay llamadas redundantes? | Número de invocaciones por ejecución |
| ¿Hay errores de herramienta? | Caso, herramienta y detalle |
| ¿Cuál es la latencia p50/p95? | Valores en milisegundos o segundos |
| ¿Qué caso tiene menor puntuación? | `scenario_id` y métrica |
| ¿Hay degradación en alto riesgo? | Resultado de `support-002` y `support-004` |

> **Análisis esperado:** en un agente de este tipo, las llamadas al modelo suelen dominar la latencia. Si una herramienta local presenta una duración mayor que las llamadas al modelo, investiga serialización, dependencias externas, reintentos o procesamiento innecesario.

---

### Paso 10. Ejecutar una regresión controlada y demostrar su detección

**Objetivo:** demostrar que la línea base y los evaluadores detectan una degradación en la selección de herramienta o recuperación de evidencia.

**Instrucciones:**

1. Activa una condición de recuperación reducida para esta ejecución.

   ```bash
   export FAQ_RESULT_LIMIT=1
   ```

2. Ejecuta el experimento de regresión.

   ```bash
   PYTHONPATH=src python -m observability.run_experiment \
     --experiment-prefix regression-demo-v1 \
     --regression-mode
   ```

3. Abre el experimento `regression-demo-v1` en LangSmith.

4. Compáralo con `baseline-function-calling-v1` usando la opción de comparación de experimentos de LangSmith.

5. Revisa especialmente los casos:

   - `support-001`
   - `support-002`
   - `support-004`

6. Documenta el resultado en `reports/regression_analysis.md`.

   ```bash
   cat > reports/regression_analysis.md <<'EOF'
   # Análisis de regresión: regression-demo-v1

   ## Cambio controlado
   - Se activó `regression_mode=True`.
   - Se redujo `FAQ_RESULT_LIMIT` a `1`.
   - La descripción de selección de herramientas se volvió ambigua.

   ## Comparación con la línea base

   | Métrica | Línea base | Regresión | Diferencia |
   |---|---:|---:|---:|
   | Precisión media | | | |
   | Fidelidad media | | | |
   | Exactitud de ruta | | | |
   | Latencia p50 | | | |
   | Latencia p95 | | | |
   | Errores de herramienta | | | |

   ## Casos afectados
   - Caso:
     - Resultado base:
     - Resultado regresión:
     - Evidencia:
     - Causa probable:

   ## Decisión operacional
   - Umbral de regresión:
   - ¿Se bloquea el despliegue?:
   - Acción correctiva:
   EOF
   ```

7. Define y aplica el siguiente criterio operativo de regresión:

   ```text
   Bloquear una modificación si ocurre cualquiera de estas condiciones:
   - La fidelidad media disminuye más de 0.10 respecto de la línea base.
   - La precisión media disminuye más de 0.10 respecto de la línea base.
   - Un caso de riesgo alto obtiene fidelidad menor que 0.80.
   - Aparece un error de herramienta que no existía en la línea base.
   - La latencia p95 aumenta más de 25 % sin justificación aprobada.
   ```

8. Desactiva la variable temporal después de la demostración.

   ```bash
   unset FAQ_RESULT_LIMIT
   ```

**Salida esperada:**

La ejecución de regresión debe producir una o más señales observables:

- Selección incorrecta de herramienta.
- Evidencia diferente de la esperada.
- Reducción de fidelidad.
- Reducción de precisión.
- Mayor variabilidad de resultados.
- Casos de soporte FAQ con menor cobertura por limitar resultados.

**Verificación:**

La demostración se considera correcta si puedes presentar:

1. El enlace o evidencia del experimento base.
2. El enlace o evidencia del experimento de regresión.
3. Al menos una métrica empeorada o un caso individual degradado.
4. Una traza que explique la causa técnica.
5. Una decisión de bloqueo o corrección basada en el criterio definido.

> **Importante:** una regresión no debe evaluarse solo por un promedio. Si `support-002`, relacionado con MFA y clasificado como riesgo alto, falla la fidelidad, la modificación debe bloquearse aunque el promedio global parezca aceptable.

---

### Paso 11. Validar localmente y confirmar los artefactos del laboratorio

**Objetivo:** ejecutar validaciones reproducibles antes de registrar el trabajo en Git.

**Instrucciones:**

1. Crea pruebas básicas para los evaluadores y contratos de herramientas.

   ```bash
   cat > tests/test_observability.py <<'EOF'
   from observability.evaluation_metrics import (
       faithfulness_evaluator,
       precision_evaluator,
       route_evaluator,
   )
   from support.tools import calculate, get_ticket_status, search_faq


   def test_tool_contracts():
       assert search_faq("restablecer contraseña")[0]["id"] == "FAQ-001"
       assert calculate("18 * 7 + 4")["result"] == "130"
       assert get_ticket_status("INC-1042")["status"] == "En progreso"


   def test_evaluators_accept_expected_response():
       outputs = {
           "answer": (
               "El ticket INC-1042 está En progreso y es atendido por "
               "Infraestructura. [TICKET-INC-1042]"
           ),
           "route": "ticket",
           "tool_name": "get_ticket_status_tool",
           "evidence": [{"id": "TICKET-INC-1042"}],
       }
       reference = {
           "expected_route": "ticket",
           "expected_tool": "get_ticket_status_tool",
           "expected_terms": ["inc-1042", "en progreso", "infraestructura"],
           "expected_evidence_id": "TICKET-INC-1042",
       }

       assert precision_evaluator(
           outputs=outputs,
           reference_outputs=reference,
       )["score"] == 1.0

       assert faithfulness_evaluator(
           outputs=outputs,
           reference_outputs=reference,
       )["score"] == 1.0

       assert route_evaluator(
           outputs=outputs,
           reference_outputs=reference,
       )["score"] == 1.0
   EOF
   ```

2. Instala `pytest` si no está disponible.

   ```bash
   pip install pytest
   ```

3. Ejecuta las pruebas.

   ```bash
   PYTHONPATH=src pytest -q tests/test_observability.py
   ```

4. Verifica sintaxis de los módulos creados.

   ```bash
   PYTHONPATH=src python -m py_compile \
     src/support/tools.py \
     src/observability/langchain_support_agent.py \
     src/observability/evaluation_metrics.py \
     src/observability/publish_dataset.py \
     src/observability/run_experiment.py
   ```

5. Revisa el estado del repositorio.

   ```bash
   git status --short
   ```

**Salida esperada:**

```text
2 passed
```

Además, la compilación debe finalizar sin mensajes de error.

**Verificación:**

Comprueba que existen los siguientes artefactos:

```bash
find src/observability data/evaluation reports tests -maxdepth 2 -type f | sort
```

Debes identificar, como mínimo:

```text
data/evaluation/golden_set.jsonl
reports/baseline_observations.md
reports/regression_analysis.md
src/observability/evaluation_metrics.py
src/observability/langchain_support_agent.py
src/observability/publish_dataset.py
src/observability/run_experiment.py
src/support/tools.py
tests/test_observability.py
```

## Validación y pruebas

Antes de finalizar, completa esta lista de validación:

- [ ] El archivo `.env` existe localmente, tiene permisos restrictivos y está ignorado por Git.
- [ ] `LANGCHAIN_TRACING_V2=true` está configurado.
- [ ] El proyecto de LangSmith es `genai-agents-batch3`.
- [ ] El agente registra las cinco etapas: clasificación, selección, invocación, evidencia y respuesta final.
- [ ] Las trazas incluyen `lab_id`, `route`, `dataset_version`, `git_commit` y `scenario_id`.
- [ ] El dataset `support-golden-set-v1` contiene cinco casos.
- [ ] Existe un experimento base que comienza con `baseline-function-calling-v1`.
- [ ] Existe un experimento de regresión que comienza con `regression-demo-v1`.
- [ ] Los evaluadores producen resultados para precisión, fidelidad y exactitud de ruta.
- [ ] Se analizaron p50, p95, tokens, errores y casos inferiores al umbral.
- [ ] Se documentó una decisión operacional de bloqueo o aceptación.
- [ ] Las pruebas locales finalizan correctamente.

Finalmente, confirma que no vas a confirmar secretos:

```bash
git status --short
git check-ignore -v .env
```

Realiza el commit obligatorio correspondiente a esta práctica:

```bash
git add \
  .env.example \
  .gitignore \
  data/evaluation/golden_set.jsonl \
  src/support/tools.py \
  src/observability \
  tests/test_observability.py \
  reports/baseline_observations.md \
  reports/regression_analysis.md

git commit -m "lab-02-00-03"
```

> Si el instructor asignó otro mensaje de commit para esta práctica dentro de una secuencia diferente, utiliza el mensaje indicado por el instructor. Nunca agregues `.env` al área de preparación de Git.

## Solución de problemas

### Problema 1: LangSmith no muestra trazas o muestra errores de autenticación

**Síntomas:**

- No aparecen ejecuciones en el proyecto `genai-agents-batch3`.
- Aparece un error `401 Unauthorized`, `403 Forbidden` o relacionado con `LANGCHAIN_API_KEY`.
- La aplicación funciona localmente, pero LangSmith no recibe trazas.

**Causa probable:**

La API key de LangSmith es inválida, no se cargó el archivo `.env`, `LANGCHAIN_TRACING_V2` no está configurado como `true`, o `LANGCHAIN_PROJECT` apunta a otro proyecto.

**Corrección:**

1. Verifica que el entorno activo tenga las variables requeridas.

   ```bash
   set -a
   source .env
   set +a

   env | grep "^LANGCHAIN_"
   ```

2. Confirma que se muestran, como mínimo:

   ```text
   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_PROJECT=genai-agents-batch3
   LANGCHAIN_ENDPOINT=https://api.smith.langchain.com
   ```

3. Genera una nueva API key en LangSmith si la actual fue revocada.
4. Actualiza solamente el archivo `.env`.
5. Ejecuta de nuevo una consulta manual:

   ```bash
   PYTHONPATH=src python -m observability.langchain_support_agent \
     "¿Cómo restablezco mi contraseña?" \
     --scenario-id troubleshooting-langsmith-001
   ```

### Problema 2: El experimento falla por errores de Azure OpenAI o la selección de herramienta no devuelve `tool_calls`

**Síntomas:**

- Error `404`, `401`, `DeploymentNotFound` o `Resource not found`.
- El agente no invoca herramientas y usa la ruta de respaldo.
- Las métricas de fidelidad bajan porque se selecciona una herramienta inesperada.

**Causa probable:**

El nombre de despliegue no coincide con `AZURE_OPENAI_CHAT_DEPLOYMENT`, el endpoint o versión de API son incorrectos, el modelo no admite Function Calling, o la configuración de herramientas es demasiado ambigua.

**Corrección:**

1. Revisa las variables de Azure OpenAI sin imprimir secretos.

   ```bash
   grep -E "^(AZURE_OPENAI_ENDPOINT|AZURE_OPENAI_API_VERSION|AZURE_OPENAI_CHAT_DEPLOYMENT)=" .env
   ```

2. Confirma en Azure Portal que el nombre de despliegue coincide exactamente con `AZURE_OPENAI_CHAT_DEPLOYMENT`.
3. Verifica que el despliegue sea un modelo de chat compatible con Function Calling.
4. Ejecuta una prueba directa del agente con un ticket.

   ```bash
   PYTHONPATH=src python -m observability.langchain_support_agent \
     "¿Cuál es el estado del ticket INC-1042?" \
     --scenario-id troubleshooting-azure-001
   ```

5. Si la selección sigue siendo inestable, restaura la descripción explícita de herramientas y verifica que `FAQ_RESULT_LIMIT` no permanezca configurada:

   ```bash
   unset FAQ_RESULT_LIMIT
   ```

## Limpieza

1. Desactiva cualquier variable temporal utilizada para la regresión.

   ```bash
   unset FAQ_RESULT_LIMIT
   ```

2. Desactiva el entorno virtual al terminar la sesión.

   ```bash
   deactivate
   ```

3. Conserva los siguientes recursos para prácticas futuras:

   - El dataset `support-golden-set-v1`.
   - Los experimentos base y de regresión en LangSmith.
   - Los informes en `reports/`.
   - El agente instrumentado en `src/observability/`.

4. No elimines el proyecto `genai-agents-batch3` si será utilizado por el resto de la tanda.

5. Si debes revocar acceso por finalización del curso, elimina o rota las claves en Azure y LangSmith, y borra el archivo local `.env` únicamente después de confirmar que ya no lo necesitas.

## Resumen

En esta práctica instrumentaste un agente de soporte con LangChain y LangSmith, manteniendo los contratos de herramientas para búsqueda de FAQ, cálculos y consulta de tickets. Configuraste trazabilidad con metadatos operativos, publicaste un golden set como dataset reutilizable y ejecutaste un experimento base con evaluadores automáticos de precisión, fidelidad y exactitud de ruta.

También analizaste trazas para investigar latencia, tokens, errores, herramientas redundantes y fallos segmentados por riesgo. Finalmente, introdujiste una regresión controlada y definiste un criterio operacional para bloquear cambios que degraden la calidad, la fidelidad, la estabilidad o la latencia del agente.

### Recursos opcionales

- [Documentación de LangSmith](https://docs.smith.langchain.com/)
- [Evaluación con LangSmith](https://docs.smith.langchain.com/evaluation)
- [Trazabilidad de LangChain](https://python.langchain.com/docs/how_to/debugging/)
- [Azure OpenAI Service](https://learn.microsoft.com/azure/ai-services/openai/)
- [Guía de evaluación de modelos de OpenAI](https://platform.openai.com/docs/guides/evals)
