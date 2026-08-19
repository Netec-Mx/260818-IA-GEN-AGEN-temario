# 1. Práctica 1. Desarrollar un script en Python que evalúe y compare automáticamente el costo, tiempo de respuesta y calidad de las respuestas obtenidas con modelos de OpenAI y Claude para un conjunto de solicitudes representativas del negocio.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 50 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica construirá un benchmark reproducible para comparar un modelo de OpenAI y un modelo Claude de Anthropic sobre solicitudes representativas de negocio. El script medirá latencia, tokens de entrada y salida, costo estimado, errores y una puntuación de calidad basada en reglas deterministas.

El resultado será un conjunto de reportes JSON y CSV que permitirá justificar una primera decisión de selección de modelo usando evidencia del dominio, en lugar de comparativas públicas o percepciones aisladas. La estructura, los adaptadores y el contrato `BenchmarkResult` se reutilizarán en laboratorios posteriores para implementar un Router de Modelos.

## Objetivos de aprendizaje

Al finalizar la práctica, podrá:

- [ ] Crear y configurar el repositorio compartido `~/genai-agent-labs` y su entorno virtual `.venv`.
- [ ] Consumir modelos de OpenAI y Anthropic mediante adaptadores Python con una interfaz normalizada.
- [ ] Medir latencia, tokens, costo estimado y errores para cada ejecución de benchmark.
- [ ] Evaluar respuestas mediante una rúbrica determinista por escenario de negocio.
- [ ] Generar reportes comparativos JSON y CSV para respaldar una decisión inicial de selección de modelo.

## Prerrequisitos

### Conocimientos requeridos

- Uso básico de terminal Linux, Git, Python, `venv` y `pip`.
- Comprensión de archivos JSON, CSV y variables de entorno.
- Conocimientos generales sobre tokens, prompts, latencia y costos de modelos de lenguaje.
- Conocimiento básico de llamadas a API y manejo de excepciones en Python.

### Accesos requeridos

Debe disponer de:

- Python 3.12.1 disponible en la terminal.
- Una API key válida de OpenAI.
- Una API key válida de Anthropic.
- Acceso de red a las APIs de OpenAI y Anthropic.
- Permisos para crear archivos en `~/genai-agent-labs`.

> **Importante:** No comparta ni versiona claves de API. El archivo `.env` se mantendrá exclusivamente en el equipo local y se agregará a `.gitignore`.

## Entorno del laboratorio

### Recursos de hardware orientativos

| Recurso | Mínimo | Recomendado |
|---|---:|---:|
| Procesador | 4 núcleos | 4 núcleos o superior |
| Memoria RAM | 16 GB | 32 GB |
| Espacio libre en disco | 10 GB | 20 GB SSD |
| Conectividad | Acceso a Internet | Conexión estable de banda ancha |

### Software utilizado

| Componente | Versión de referencia |
|---|---|
| Ubuntu | 22.04.4 LTS |
| Python | 3.12.1 |
| pip | 23.3.2 |
| Git | 2.43.0 |
| OpenAI Python SDK | 1.12.0 |
| Anthropic Python SDK | 0.18.1 |
| python-dotenv | 1.0.1 |

### Convenciones obligatorias

| Elemento | Valor |
|---|---|
| Directorio de trabajo global | `~/genai-agent-labs` |
| Entorno virtual compartido | `~/genai-agent-labs/.venv` |
| Código fuente | `~/genai-agent-labs/src` |
| Configuración | `~/genai-agent-labs/config` |
| Prompts | `~/genai-agent-labs/prompts` |
| Pruebas | `~/genai-agent-labs/tests` |
| Reportes | `~/genai-agent-labs/reports` |
| Datos | `~/genai-agent-labs/data` |
| Base de datos reservada | `genai_agents_db` |
| Nombre de contenedor reservado | `genai-agent-api` |

---

## Procedimiento paso a paso

### Paso 1. Crear el repositorio y el entorno virtual compartido

**Objetivo:** Crear la estructura base del repositorio local y preparar el entorno Python reutilizable para los siguientes laboratorios.

**Instrucciones:**

1. Abra una terminal y cree el directorio de trabajo obligatorio:

   ```bash
   mkdir -p ~/genai-agent-labs
   cd ~/genai-agent-labs
   ```

2. Inicialice el repositorio Git:

   ```bash
   git init
   ```

3. Cree la estructura inicial de directorios:

   ```bash
   mkdir -p src/benchmarks config prompts tests reports/benchmark data
   ```

4. Cree y active el entorno virtual compartido:

   ```bash
   python3.12 -m venv .venv
   source .venv/bin/activate
   ```

5. Actualice `pip` e instale las dependencias requeridas:

   ```bash
   python -m pip install --upgrade pip
   pip install openai==1.12.0 anthropic==0.18.1 python-dotenv==1.0.1
   ```

6. Registre las dependencias instaladas:

   ```bash
   pip freeze > requirements.txt
   ```

7. Cree el archivo `.gitignore`:

   ```bash
   cat > .gitignore <<'EOF'
   .venv/
   __pycache__/
   *.py[cod]
   .env
   reports/benchmark/*.json
   reports/benchmark/*.csv
   .pytest_cache/
   EOF
   ```

**Salida esperada:**

La estructura del proyecto debe contener al menos los directorios y archivos siguientes:

```text
~/genai-agent-labs/
├── .venv/
├── config/
├── data/
├── prompts/
├── reports/
│   └── benchmark/
├── src/
│   └── benchmarks/
├── tests/
├── .gitignore
└── requirements.txt
```

**Verificación:**

Ejecute:

```bash
python --version
pip show openai anthropic python-dotenv
git status
```

Debe observar Python 3.12.x, las tres dependencias instaladas y archivos sin confirmar en Git.

---

### Paso 2. Configurar variables de entorno de forma segura

**Objetivo:** Definir la configuración local de proveedores y modelos sin exponer claves en el repositorio.

**Instrucciones:**

1. Cree una plantilla de configuración compartible llamada `.env.example`:

   ```bash
   cat > .env.example <<'EOF'
   # Nunca agregue claves reales a este archivo.
   OPENAI_API_KEY=replace_with_openai_key
   ANTHROPIC_API_KEY=replace_with_anthropic_key

   OPENAI_MODEL=gpt-4o-mini-2024-07-18
   ANTHROPIC_MODEL=claude-3-haiku-20240307

   OPENAI_TEMPERATURE=0
   ANTHROPIC_TEMPERATURE=0
   MAX_TOKENS=400
   EOF
   ```

2. Cree el archivo local `.env` a partir de la plantilla:

   ```bash
   cp .env.example .env
   ```

3. Edite `.env` con un editor local y sustituya únicamente los valores de las claves:

   ```bash
   nano .env
   ```

   Ejemplo conceptual:

   ```dotenv
   OPENAI_API_KEY=sk-proj-...
   ANTHROPIC_API_KEY=sk-ant-...

   OPENAI_MODEL=gpt-4o-mini-2024-07-18
   ANTHROPIC_MODEL=claude-3-haiku-20240307

   OPENAI_TEMPERATURE=0
   ANTHROPIC_TEMPERATURE=0
   MAX_TOKENS=400
   ```

4. Compruebe que `.env` está ignorado por Git:

   ```bash
   git check-ignore .env
   ```

**Salida esperada:**

El comando debe mostrar:

```text
.env
```

**Verificación:**

Ejecute el siguiente comando sin imprimir las claves:

```bash
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print('OPENAI configurada:', bool(os.getenv('OPENAI_API_KEY'))); print('ANTHROPIC configurada:', bool(os.getenv('ANTHROPIC_API_KEY')))"
```

La salida esperada es:

```text
OPENAI configurada: True
ANTHROPIC configurada: True
```

---

### Paso 3. Definir la tabla de precios versionada

**Objetivo:** Crear una fuente explícita y revisable para calcular el costo estimado de cada invocación.

**Instrucciones:**

1. Cree el archivo `config/pricing.json`:

   ```bash
   cat > config/pricing.json <<'EOF'
   {
     "pricing_version": "2026-08-lab-reference",
     "currency": "USD",
     "unit": "cost_per_1m_tokens",
     "models": {
       "gpt-4o-mini-2024-07-18": {
         "input": 0.15,
         "output": 0.60
       },
       "claude-3-haiku-20240307": {
         "input": 0.25,
         "output": 1.25
       }
     }
   }
   EOF
   ```

2. Revise el contenido:

   ```bash
   cat config/pricing.json
   ```

3. Valide que el archivo es JSON correcto:

   ```bash
   python -m json.tool config/pricing.json > /dev/null
   ```

> **Nota técnica:** Los precios cambian con frecuencia y pueden variar por región, modalidad, caché, lote o condiciones comerciales. Los valores de este archivo se usan exclusivamente como referencia controlada del laboratorio. En un entorno real, actualice la tabla con precios oficiales vigentes y conserve su versión junto con los resultados.

**Salida esperada:**

El comando de validación no debe producir errores.

**Verificación:**

Ejecute:

```bash
python -c "import json; p=json.load(open('config/pricing.json')); print(p['pricing_version']); print(', '.join(p['models'].keys()))"
```

Debe listar la versión de precios y los dos modelos configurados.

---

### Paso 4. Definir solicitudes representativas y rúbricas deterministas

**Objetivo:** Crear un conjunto de al menos seis escenarios de negocio y criterios de calidad verificables sin evaluación subjetiva manual.

**Instrucciones:**

1. Cree el archivo `prompts/business_requests.json`:

   ```bash
   cat > prompts/business_requests.json <<'EOF'
   [
     {
       "id": "executive_summary",
       "name": "Resumen ejecutivo",
       "prompt": "Resume el siguiente informe en español en un máximo de 60 palabras. Debe incluir los términos ingresos, riesgo y recomendación. Informe: Los ingresos trimestrales crecieron 12 %. El principal riesgo es la dependencia de un único proveedor logístico. Se recomienda diversificar proveedores antes del próximo trimestre.",
       "quality_rules": {
         "max_words": 60,
         "required_terms": ["ingresos", "riesgo", "recomendación"]
       }
     },
     {
       "id": "incident_classification",
       "name": "Clasificación de incidencia",
       "prompt": "Clasifica la incidencia y responde exclusivamente con JSON válido con las claves categoria, prioridad y sla_horas. Incidencia: El portal de pagos no permite completar compras para todos los clientes desde hace 20 minutos.",
       "quality_rules": {
         "json_required": true,
         "required_fields": ["categoria", "prioridad", "sla_horas"],
         "expected_terms": ["alta"]
       }
     },
     {
       "id": "field_extraction",
       "name": "Extracción de campos",
       "prompt": "Extrae datos y responde exclusivamente con JSON válido con las claves cliente, factura, total y moneda. Texto: Factura F-2026-1048 emitida para Contoso S.A. por un total de 1250.50 EUR.",
       "quality_rules": {
         "json_required": true,
         "required_fields": ["cliente", "factura", "total", "moneda"],
         "expected_terms": ["contoso", "f-2026-1048", "eur"]
       }
     },
     {
       "id": "support_response",
       "name": "Respuesta de soporte",
       "prompt": "Redacta una respuesta de soporte en español para un cliente que no puede restablecer su contraseña. Incluye exactamente tres pasos numerados y menciona soporte si el problema continúa.",
       "quality_rules": {
         "required_terms": ["soporte"],
         "min_numbered_steps": 3,
         "max_words": 140
       }
     },
     {
       "id": "safe_sql",
       "name": "Generación de SQL segura",
       "prompt": "Genera una consulta SQL segura y parametrizada para obtener pedidos de un cliente por customer_id. Usa un marcador llamado :customer_id. No generes sentencias DELETE, UPDATE, INSERT, DROP ni datos ficticios.",
       "quality_rules": {
         "required_terms": ["select", ":customer_id"],
         "forbidden_terms": ["delete", "update", "insert", "drop"],
         "max_words": 100
       }
     },
     {
       "id": "technical_explanation",
       "name": "Explicación técnica",
       "prompt": "Explica en español, en un máximo de 100 palabras, la diferencia entre autenticación y autorización. Debes mencionar identidad, permisos y RBAC.",
       "quality_rules": {
         "max_words": 100,
         "required_terms": ["identidad", "permisos", "rbac"]
       }
     }
   ]
   EOF
   ```

2. Valide el JSON:

   ```bash
   python -m json.tool prompts/business_requests.json > /dev/null
   ```

3. Confirme que hay seis escenarios:

   ```bash
   python -c "import json; print(len(json.load(open('prompts/business_requests.json'))))"
   ```

**Salida esperada:**

La salida debe ser:

```text
6
```

**Verificación:**

Revise que los escenarios cubren tareas frecuentes de negocio:

- Resumen ejecutivo.
- Clasificación de incidencias.
- Extracción estructurada.
- Soporte al cliente.
- Generación SQL con restricciones de seguridad.
- Explicación técnica.

Estas tareas permiten comparar calidad, formato, seguridad básica, latencia y costo en una muestra heterogénea del dominio.

---

### Paso 5. Implementar adaptadores normalizados y el contrato de resultado

**Objetivo:** Implementar una interfaz uniforme para proveedores y definir el contrato `BenchmarkResult` reutilizable.

**Instrucciones:**

1. Cree los archivos de paquete Python:

   ```bash
   touch src/__init__.py
   touch src/benchmarks/__init__.py
   ```

2. Cree el archivo `src/benchmarks/compare_models.py`:

   ```bash
   cat > src/benchmarks/compare_models.py <<'EOF'
   import csv
   import json
   import os
   import re
   import time
   from dataclasses import asdict, dataclass
   from datetime import datetime, timezone
   from pathlib import Path
   from typing import Any

   from anthropic import Anthropic
   from dotenv import load_dotenv
   from openai import OpenAI


   PROJECT_ROOT = Path(__file__).resolve().parents[2]
   PROMPTS_FILE = PROJECT_ROOT / "prompts" / "business_requests.json"
   PRICING_FILE = PROJECT_ROOT / "config" / "pricing.json"
   REPORTS_DIR = PROJECT_ROOT / "reports" / "benchmark"


   @dataclass
   class BenchmarkResult:
       scenario_id: str
       scenario_name: str
       provider: str
       model: str
       latency_ms: float
       input_tokens: int
       output_tokens: int
       estimated_cost_usd: float
       quality_score: float
       quality_checks: dict[str, bool]
       response_text: str
       error: str | None
       executed_at_utc: str


   class OpenAIAdapter:
       def __init__(self, model: str, temperature: float, max_tokens: int):
           self.model = model
           self.temperature = temperature
           self.max_tokens = max_tokens
           self.client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

       def generate(self, prompt: str) -> tuple[str, int, int]:
           response = self.client.chat.completions.create(
               model=self.model,
               temperature=self.temperature,
               max_tokens=self.max_tokens,
               messages=[
                   {
                       "role": "system",
                       "content": "Responde con precisión, sigue estrictamente el formato solicitado y no agregues explicaciones innecesarias."
                   },
                   {"role": "user", "content": prompt}
               ]
           )
           text = response.choices[0].message.content or ""
           input_tokens = response.usage.prompt_tokens if response.usage else 0
           output_tokens = response.usage.completion_tokens if response.usage else 0
           return text, input_tokens, output_tokens


   class AnthropicAdapter:
       def __init__(self, model: str, temperature: float, max_tokens: int):
           self.model = model
           self.temperature = temperature
           self.max_tokens = max_tokens
           self.client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

       def generate(self, prompt: str) -> tuple[str, int, int]:
           response = self.client.messages.create(
               model=self.model,
               max_tokens=self.max_tokens,
               temperature=self.temperature,
               system="Responde con precisión, sigue estrictamente el formato solicitado y no agregues explicaciones innecesarias.",
               messages=[{"role": "user", "content": prompt}]
           )
           text = "".join(
               block.text for block in response.content
               if getattr(block, "type", "") == "text"
           )
           input_tokens = response.usage.input_tokens if response.usage else 0
           output_tokens = response.usage.output_tokens if response.usage else 0
           return text, input_tokens, output_tokens


   def load_json(path: Path) -> Any:
       with path.open(encoding="utf-8") as file:
           return json.load(file)


   def estimate_cost(model: str, input_tokens: int, output_tokens: int, pricing: dict[str, Any]) -> float:
       model_pricing = pricing["models"].get(model)
       if not model_pricing:
           return 0.0

       input_cost = input_tokens / 1_000_000 * model_pricing["input"]
       output_cost = output_tokens / 1_000_000 * model_pricing["output"]
       return round(input_cost + output_cost, 8)


   def count_words(text: str) -> int:
       return len(re.findall(r"\b[\wáéíóúüñÁÉÍÓÚÜÑ-]+\b", text))


   def evaluate_quality(response_text: str, rules: dict[str, Any]) -> tuple[float, dict[str, bool]]:
       text_lower = response_text.lower()
       checks: dict[str, bool] = {}

       if "max_words" in rules:
           checks["max_words"] = count_words(response_text) <= rules["max_words"]

       if "required_terms" in rules:
           checks["required_terms"] = all(
               term.lower() in text_lower for term in rules["required_terms"]
           )

       if "expected_terms" in rules:
           checks["expected_terms"] = all(
               term.lower() in text_lower for term in rules["expected_terms"]
           )

       if "forbidden_terms" in rules:
           checks["forbidden_terms"] = all(
               term.lower() not in text_lower for term in rules["forbidden_terms"]
           )

       if "min_numbered_steps" in rules:
           numbered_steps = re.findall(r"(?m)^\s*\d+[.)]", response_text)
           checks["min_numbered_steps"] = len(numbered_steps) >= rules["min_numbered_steps"]

       parsed_json: dict[str, Any] | None = None
       if rules.get("json_required"):
           try:
               parsed_json = json.loads(response_text.strip())
               checks["json_valid"] = isinstance(parsed_json, dict)
           except json.JSONDecodeError:
               checks["json_valid"] = False

       if "required_fields" in rules:
           checks["required_fields"] = bool(parsed_json) and all(
               field in parsed_json for field in rules["required_fields"]
           )

       if not checks:
           return 1.0, {}

       score = round(sum(checks.values()) / len(checks), 2)
       return score, checks


   def execute_benchmark(
       adapter: Any,
       provider: str,
       scenario: dict[str, Any],
       pricing: dict[str, Any]
   ) -> BenchmarkResult:
       started = time.perf_counter()
       error = None
       response_text = ""
       input_tokens = 0
       output_tokens = 0

       try:
           response_text, input_tokens, output_tokens = adapter.generate(scenario["prompt"])
       except Exception as exception:
           error = f"{type(exception).__name__}: {exception}"

       latency_ms = round((time.perf_counter() - started) * 1000, 2)

       if error:
           quality_score = 0.0
           quality_checks = {"request_success": False}
       else:
           quality_score, quality_checks = evaluate_quality(
               response_text,
               scenario["quality_rules"]
           )
           quality_checks["request_success"] = True

       return BenchmarkResult(
           scenario_id=scenario["id"],
           scenario_name=scenario["name"],
           provider=provider,
           model=adapter.model,
           latency_ms=latency_ms,
           input_tokens=input_tokens,
           output_tokens=output_tokens,
           estimated_cost_usd=estimate_cost(
               adapter.model,
               input_tokens,
               output_tokens,
               pricing
           ),
           quality_score=quality_score,
           quality_checks=quality_checks,
           response_text=response_text,
           error=error,
           executed_at_utc=datetime.now(timezone.utc).isoformat()
       )


   def write_reports(results: list[BenchmarkResult]) -> None:
       REPORTS_DIR.mkdir(parents=True, exist_ok=True)

       json_path = REPORTS_DIR / "benchmark_results.json"
       csv_path = REPORTS_DIR / "benchmark_summary.csv"

       serialized_results = [asdict(result) for result in results]
       with json_path.open("w", encoding="utf-8") as file:
           json.dump(serialized_results, file, ensure_ascii=False, indent=2)

       csv_fields = [
           "scenario_id", "scenario_name", "provider", "model",
           "latency_ms", "input_tokens", "output_tokens",
           "estimated_cost_usd", "quality_score", "error"
       ]

       with csv_path.open("w", newline="", encoding="utf-8") as file:
           writer = csv.DictWriter(file, fieldnames=csv_fields)
           writer.writeheader()
           for result in serialized_results:
               writer.writerow({field: result[field] for field in csv_fields})

       print(f"Reporte JSON generado: {json_path}")
       print(f"Reporte CSV generado:  {csv_path}")


   def print_summary(results: list[BenchmarkResult]) -> None:
       print("\nResumen por proveedor")
       print("-" * 72)

       providers = sorted({result.provider for result in results})
       for provider in providers:
           provider_results = [result for result in results if result.provider == provider]
           successful = [result for result in provider_results if result.error is None]

           average_latency = (
               sum(result.latency_ms for result in successful) / len(successful)
               if successful else 0
           )
           average_quality = (
               sum(result.quality_score for result in successful) / len(successful)
               if successful else 0
           )
           total_cost = sum(result.estimated_cost_usd for result in provider_results)
           errors = sum(result.error is not None for result in provider_results)

           print(
               f"{provider:<10} "
               f"calidad_promedio={average_quality:.2f} "
               f"latencia_promedio_ms={average_latency:.2f} "
               f"costo_total_usd={total_cost:.8f} "
               f"errores={errors}"
           )


   def main() -> None:
       load_dotenv()

       required_variables = ["OPENAI_API_KEY", "ANTHROPIC_API_KEY"]
       missing = [name for name in required_variables if not os.getenv(name)]
       if missing:
           raise RuntimeError(
               "Faltan variables de entorno requeridas: " + ", ".join(missing)
           )

       scenarios = load_json(PROMPTS_FILE)
       pricing = load_json(PRICING_FILE)

       max_tokens = int(os.getenv("MAX_TOKENS", "400"))
       openai_adapter = OpenAIAdapter(
           model=os.getenv("OPENAI_MODEL", "gpt-4o-mini-2024-07-18"),
           temperature=float(os.getenv("OPENAI_TEMPERATURE", "0")),
           max_tokens=max_tokens
       )
       anthropic_adapter = AnthropicAdapter(
           model=os.getenv("ANTHROPIC_MODEL", "claude-3-haiku-20240307"),
           temperature=float(os.getenv("ANTHROPIC_TEMPERATURE", "0")),
           max_tokens=max_tokens
       )

       adapters = [
           ("openai", openai_adapter),
           ("anthropic", anthropic_adapter)
       ]

       results: list[BenchmarkResult] = []
       for scenario in scenarios:
           for provider, adapter in adapters:
               print(f"Ejecutando {scenario['id']} con {provider}...")
               result = execute_benchmark(adapter, provider, scenario, pricing)
               results.append(result)

               state = "ERROR" if result.error else "OK"
               print(
                   f"  [{state}] latencia={result.latency_ms} ms "
                   f"calidad={result.quality_score:.2f} "
                   f"costo=${result.estimated_cost_usd:.8f}"
               )

       write_reports(results)
       print_summary(results)


   if __name__ == "__main__":
       main()
   EOF
   ```

3. Revise la sintaxis del script:

   ```bash
   python -m py_compile src/benchmarks/compare_models.py
   ```

**Salida esperada:**

No debe aparecer ningún error de sintaxis.

**Verificación:**

Confirme que el contrato de resultado contiene las métricas requeridas:

```bash
grep -E "latency_ms|input_tokens|output_tokens|estimated_cost_usd|quality_score|error" src/benchmarks/compare_models.py
```

Debe encontrar los campos de latencia, tokens, costo, calidad y error dentro de `BenchmarkResult`.

---

### Paso 6. Ejecutar el benchmark y generar reportes

**Objetivo:** Ejecutar los seis escenarios contra ambos proveedores y persistir los resultados normalizados.

**Instrucciones:**

1. Asegúrese de que el entorno virtual continúa activo:

   ```bash
   source ~/genai-agent-labs/.venv/bin/activate
   ```

2. Sitúese en la raíz del repositorio:

   ```bash
   cd ~/genai-agent-labs
   ```

3. Ejecute el benchmark:

   ```bash
   python -m src.benchmarks.compare_models
   ```

4. Espere a que finalicen las doce solicitudes: seis escenarios por dos proveedores.

5. Revise los reportes creados:

   ```bash
   ls -lh reports/benchmark/
   ```

6. Inspeccione el reporte CSV:

   ```bash
   column -s, -t < reports/benchmark/benchmark_summary.csv
   ```

7. Consulte un resultado JSON de ejemplo:

   ```bash
   python -c "import json; r=json.load(open('reports/benchmark/benchmark_results.json')); print(json.dumps(r[0], ensure_ascii=False, indent=2))"
   ```

**Salida esperada:**

Durante la ejecución aparecerán mensajes similares a los siguientes:

```text
Ejecutando executive_summary con openai...
  [OK] latencia=842.15 ms calidad=1.00 costo=$0.00003150
Ejecutando executive_summary con anthropic...
  [OK] latencia=765.20 ms calidad=1.00 costo=$0.00007250
...
Reporte JSON generado: /home/usuario/genai-agent-labs/reports/benchmark/benchmark_results.json
Reporte CSV generado:  /home/usuario/genai-agent-labs/reports/benchmark/benchmark_summary.csv

Resumen por proveedor
------------------------------------------------------------------------
anthropic  calidad_promedio=0.92 latencia_promedio_ms=...
openai     calidad_promedio=0.96 latencia_promedio_ms=...
```

Los valores concretos de tokens, latencia, costo y calidad variarán según la respuesta del modelo, la red, el estado del proveedor y la versión efectiva disponible.

**Verificación:**

Ejecute:

```bash
python -c "
import json
results = json.load(open('reports/benchmark/benchmark_results.json'))
print('Resultados:', len(results))
print('Proveedores:', sorted(set(r['provider'] for r in results)))
print('Escenarios:', sorted(set(r['scenario_id'] for r in results)))
print('Errores:', sum(r['error'] is not None for r in results))
"
```

El resultado esperado es:

```text
Resultados: 12
Proveedores: ['anthropic', 'openai']
Escenarios: ['executive_summary', 'field_extraction', 'incident_classification', 'safe_sql', 'support_response', 'technical_explanation']
Errores: 0
```

> Si existen errores, continúe con la sección de resolución de problemas antes de realizar el commit.

---

### Paso 7. Analizar resultados y justificar una selección inicial

**Objetivo:** Interpretar las métricas obtenidas usando los criterios de calidad, costo y rendimiento operacional tratados en la lección.

**Instrucciones:**

1. Calcule métricas agregadas por proveedor:

   ```bash
   python -c "
   import json
   from collections import defaultdict

   results = json.load(open('reports/benchmark/benchmark_results.json'))
   groups = defaultdict(list)

   for result in results:
       if result['error'] is None:
           groups[result['provider']].append(result)

   for provider, rows in groups.items():
       avg_quality = sum(r['quality_score'] for r in rows) / len(rows)
       avg_latency = sum(r['latency_ms'] for r in rows) / len(rows)
       total_cost = sum(r['estimated_cost_usd'] for r in rows)
       print(f'{provider}: calidad={avg_quality:.2f}, latencia_ms={avg_latency:.2f}, costo_usd={total_cost:.8f}')
   "
   ```

2. Identifique los escenarios cuya puntuación sea inferior a `1.0`:

   ```bash
   python -c "
   import json
   results = json.load(open('reports/benchmark/benchmark_results.json'))
   for r in results:
       if r['quality_score'] < 1.0:
           print(f\"{r['provider']} | {r['scenario_id']} | calidad={r['quality_score']} | controles={r['quality_checks']}\")
   "
   ```

3. Revise específicamente el escenario `safe_sql` para verificar que no se hayan generado operaciones destructivas:

   ```bash
   python -c "
   import json
   results = json.load(open('reports/benchmark/benchmark_results.json'))
   for r in results:
       if r['scenario_id'] == 'safe_sql':
           print('\\nProveedor:', r['provider'])
           print('Calidad:', r['quality_score'])
           print('Respuesta:', r['response_text'])
   "
   ```

4. Documente una conclusión breve en `reports/benchmark/decision.md`:

   ```bash
   cat > reports/benchmark/decision.md <<'EOF'
   # Decisión inicial de selección de modelo

   ## Evidencia evaluada

   - Se ejecutaron seis escenarios representativos de negocio.
   - Cada escenario se probó con OpenAI y Anthropic.
   - Se midieron calidad determinista, latencia, tokens, costo estimado y errores.
   - La temperatura se configuró en cero para reducir variabilidad.

   ## Decisión inicial

   Seleccione el proveedor que presente la mejor combinación de:
   1. Calidad promedio y cumplimiento de formatos estructurados.
   2. Ausencia de errores en los escenarios críticos.
   3. Latencia compatible con el caso de uso.
   4. Costo sostenible para el volumen esperado.

   ## Limitaciones

   Esta evaluación es inicial y no reemplaza pruebas de seguridad, cumplimiento,
   carga, evaluación humana, recuperación de información ni análisis de costos
   operativos completos.
   EOF
   ```

5. Complete el archivo con su decisión usando los resultados reales obtenidos.

**Salida esperada:**

Debe poder identificar un proveedor que sea más adecuado para una estrategia inicial, o concluir que ambos requieren más evidencia en escenarios críticos.

**Verificación:**

Abra el archivo de decisión:

```bash
cat reports/benchmark/decision.md
```

Su decisión debe hacer referencia explícita a:

- Calidad.
- Latencia.
- Costo.
- Errores o estabilidad.
- Limitaciones de la evaluación.

> **Interpretación recomendada:** Un modelo no debe seleccionarse solamente por ser más económico o más rápido. Si falla en JSON estructurado, genera SQL no seguro o incumple restricciones relevantes, ese riesgo puede superar el ahorro directo de tokens. Para producción, la decisión debe incluir seguridad, cumplimiento, disponibilidad regional, privacidad, límites de tasa y revisión humana cuando corresponda.

---

### Paso 8. Registrar el trabajo en Git

**Objetivo:** Versionar los artefactos reutilizables del laboratorio sin incluir secretos ni resultados locales potencialmente sensibles.

**Instrucciones:**

1. Compruebe el estado del repositorio:

   ```bash
   git status
   ```

2. Confirme que `.env` y los reportes de ejecución no aparecen como archivos para agregar. Deben estar ignorados por `.gitignore`.

3. Agregue los archivos de implementación y configuración:

   ```bash
   git add .gitignore requirements.txt .env.example config prompts src tests
   ```

4. Cree el commit obligatorio:

   ```bash
   git commit -m "lab-01-00-01"
   ```

5. Verifique el historial:

   ```bash
   git log --oneline -1
   ```

**Salida esperada:**

La salida debe incluir un commit similar a:

```text
<hash> lab-01-00-01
```

**Verificación:**

Ejecute:

```bash
git status
```

El repositorio debe quedar sin cambios pendientes, excepto si decide conservar localmente el archivo no versionado `reports/benchmark/decision.md`. Si desea versionar la decisión sin incluir los resultados JSON y CSV, agregue el archivo explícitamente:

```bash
git add -f reports/benchmark/decision.md
git commit --amend --no-edit
```

---

## Validación y pruebas

Ejecute las siguientes validaciones desde `~/genai-agent-labs` con el entorno virtual activo.

### Validación de estructura

```bash
test -d src/benchmarks && \
test -d config && \
test -d prompts && \
test -d reports/benchmark && \
test -f config/pricing.json && \
test -f prompts/business_requests.json && \
test -f src/benchmarks/compare_models.py && \
echo "Estructura válida"
```

Salida esperada:

```text
Estructura válida
```

### Validación de archivos JSON

```bash
python -m json.tool config/pricing.json > /dev/null && \
python -m json.tool prompts/business_requests.json > /dev/null && \
python -m json.tool reports/benchmark/benchmark_results.json > /dev/null && \
echo "JSON válido"
```

Salida esperada:

```text
JSON válido
```

### Validación de cantidad de ejecuciones

```bash
python -c "
import json
results = json.load(open('reports/benchmark/benchmark_results.json'))
assert len(results) == 12, f'Se esperaban 12 resultados y se obtuvieron {len(results)}'
assert all('latency_ms' in r for r in results)
assert all('input_tokens' in r for r in results)
assert all('output_tokens' in r for r in results)
assert all('estimated_cost_usd' in r for r in results)
assert all('quality_score' in r for r in results)
print('Contrato BenchmarkResult validado para 12 ejecuciones')
"
```

Salida esperada:

```text
Contrato BenchmarkResult validado para 12 ejecuciones
```

### Validación de resultados sin errores

```bash
python -c "
import json
results = json.load(open('reports/benchmark/benchmark_results.json'))
errors = [r for r in results if r['error']]
assert not errors, f'Se detectaron {len(errors)} errores: {errors}'
print('Todas las solicitudes finalizaron correctamente')
"
```

Salida esperada:

```text
Todas las solicitudes finalizaron correctamente
```

### Criterios de aceptación

Se considera completada la práctica si se cumplen todos los criterios siguientes:

- Existe el repositorio `~/genai-agent-labs` con el entorno `.venv`.
- El archivo `.env` no está versionado.
- Existen seis escenarios de negocio en `prompts/business_requests.json`.
- Existe una tabla de precios explícita en `config/pricing.json`.
- El script usa adaptadores para OpenAI y Anthropic.
- El contrato `BenchmarkResult` contiene métricas de latencia, tokens, costo, calidad y error.
- Se generan `reports/benchmark/benchmark_results.json` y `reports/benchmark/benchmark_summary.csv`.
- El reporte contiene doce ejecuciones: seis por cada proveedor.
- Se creó el commit `lab-01-00-01`.

## Resolución de problemas

### Problema 1. Error de autenticación o variable de entorno ausente

**Síntoma:**

Durante la ejecución aparece un error similar a:

```text
RuntimeError: Faltan variables de entorno requeridas: OPENAI_API_KEY
```

o bien:

```text
AuthenticationError
```

**Causa:**

El archivo `.env` no existe, contiene una clave incorrecta, no se ejecuta desde la raíz del proyecto o la clave fue revocada por el proveedor.

**Solución:**

1. Confirme que está en la raíz del repositorio:

   ```bash
   cd ~/genai-agent-labs
   ```

2. Confirme que existe el archivo `.env`:

   ```bash
   ls -la .env
   ```

3. Compruebe que las variables se cargan sin mostrar secretos:

   ```bash
   python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(bool(os.getenv('OPENAI_API_KEY')), bool(os.getenv('ANTHROPIC_API_KEY')))"
   ```

4. Edite `.env`, corrija la clave correspondiente y vuelva a ejecutar:

   ```bash
   python -m src.benchmarks.compare_models
   ```

---

### Problema 2. El modelo no está disponible o se produce un error de límite de solicitudes

**Síntoma:**

Aparece un error similar a:

```text
NotFoundError
```

```text
BadRequestError: The model does not exist
```

o:

```text
RateLimitError
```

**Causa:**

El identificador del modelo no está habilitado para la cuenta, el proveedor retiró o modificó la versión, o se alcanzó un límite temporal de solicitudes o cuota.

**Solución:**

1. Verifique los nombres configurados:

   ```bash
   grep -E "OPENAI_MODEL|ANTHROPIC_MODEL" .env
   ```

2. Confirme en la documentación y consola del proveedor que su cuenta puede usar esos modelos.

3. Si el instructor autoriza una versión alternativa, actualice `.env` y agregue el mismo identificador en `config/pricing.json`.

4. Si el error es de límite de solicitudes, espere unos minutos antes de reintentar. No ejecute múltiples instancias del benchmark simultáneamente.

5. Ejecute de nuevo el script y revise que los errores anteriores no aparezcan en el JSON final:

   ```bash
   python -m src.benchmarks.compare_models
   ```

## Limpieza

Esta práctica no utiliza contenedores, bases de datos ni recursos de Azure que requieran eliminación. Para preservar los artefactos reutilizables, no elimine el repositorio ni el entorno virtual.

Si necesita eliminar únicamente resultados locales antes de una nueva ejecución, use:

```bash
rm -f reports/benchmark/benchmark_results.json
rm -f reports/benchmark/benchmark_summary.csv
```

Para salir del entorno virtual:

```bash
deactivate
```

> No elimine `.env` si desea reutilizar sus credenciales en los siguientes laboratorios. No comparta ni copie este archivo a repositorios remotos.

## Resumen

En esta práctica creó una base reproducible para comparar modelos de lenguaje con evidencia propia del dominio. Implementó adaptadores para OpenAI y Anthropic, un contrato normalizado `BenchmarkResult`, una tabla de precios versionada, seis escenarios de negocio y reglas deterministas de calidad.

También generó reportes JSON y CSV que permiten analizar costo, latencia, tokens, errores y cumplimiento de requisitos básicos. Esta información será la entrada para una estrategia posterior de enrutamiento de modelos, donde solicitudes simples pueden dirigirse a modelos más económicos y solicitudes complejas o críticas pueden escalarse a modelos con mayor capacidad o a revisión humana.

### Recursos opcionales

- [Documentación de modelos de OpenAI](https://platform.openai.com/docs/models)
- [Documentación de Anthropic Claude](https://docs.anthropic.com/en/docs/about-claude/models)
- [OpenAI Python SDK](https://github.com/openai/openai-python)
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Guía de selección de modelos en Azure AI Foundry](https://learn.microsoft.com/es-es/azure/ai-foundry/how-to/model-catalog-overview)

---

# 2. Práctica 2. Construir la arquitectura base de una solución GenAI utilizando FastAPI, implementando un Router de Modelos y preparando su integración con Azure AI.

## Metadatos

| Atributo | Valor |
|---|---|
| Duración | 60 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica construirás una API reutilizable con FastAPI para generar respuestas mediante distintos proveedores de modelos. Implementarás un Router de Modelos que selecciona OpenAI, Claude o Azure AI según una preferencia explícita o una política automática basada en métricas de benchmark.

La API expondrá contratos JSON validados, métricas operativas básicas y errores controlados. El adaptador de Azure AI quedará preparado mediante variables de entorno, sin incluir secretos en el código ni requerir infraestructura Azure disponible durante la práctica.

## Objetivos de aprendizaje

Al finalizar la práctica podrás:

- [ ] Implementar una API FastAPI con los endpoints `GET /health` y `POST /v1/generate`.
- [ ] Definir contratos de entrada y salida validados mediante Pydantic.
- [ ] Implementar adaptadores para OpenAI, Anthropic y Azure AI Inference.
- [ ] Construir un Router de Modelos con selección explícita y política automática basada en costo y calidad.
- [ ] Devolver trazabilidad de proveedor, modelo, latencia, uso y `request_id` sin exponer secretos.

## Prerrequisitos

### Conocimientos requeridos

- Práctica `01-00-01` finalizada.
- Fundamentos de Python, entornos virtuales y módulos.
- Conceptos básicos de HTTP, JSON, FastAPI y códigos de estado HTTP.
- Comprensión de las métricas de benchmark: calidad, costo, latencia y proveedor.
- Conocimiento de que una política de enrutamiento es una decisión arquitectónica basada en evidencia y no una regla estática universal.

### Acceso y archivos requeridos

Debes disponer de:

- El repositorio local `~/genai-agent-labs`.
- El entorno virtual compartido en `~/genai-agent-labs/.venv`.
- El archivo `reports/benchmark/benchmark_results.json` producido en la práctica anterior.
- API keys válidas de OpenAI y Anthropic en un archivo `.env`.
- Opcionalmente, un endpoint y una credencial de Azure AI Inference si deseas probar ese proveedor.
- Permisos de lectura y escritura sobre el repositorio local.

> **Importante:** no se desplegará ningún recurso Azure ni se ejecutará Docker en esta práctica. El adaptador Azure AI se prepara para integración futura y devuelve un error controlado cuando su configuración no existe.

## Entorno del laboratorio

### Recursos de hardware recomendados

| Recurso | Mínimo | Recomendado |
|---|---:|---:|
| Procesador | 4 núcleos | 4 núcleos o superior |
| Memoria RAM | 16 GB | 32 GB |
| Espacio libre en SSD | 10 GB | 20 GB |

### Software utilizado

| Componente | Versión de referencia |
|---|---|
| Ubuntu | 22.04.4 LTS |
| Python | 3.12.1 |
| FastAPI | 0.109.2 |
| Uvicorn | 0.27.1 |
| Pydantic | 2.6.1 |
| OpenAI Python SDK | 1.12.0 |
| Anthropic Python SDK | 0.18.1 |
| Azure AI Inference SDK | 1.0.0b5 |

### Preparación inicial

Abre una terminal y ejecuta los siguientes comandos:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

python --version
git status
```

Instala las dependencias necesarias para esta práctica:

```bash
pip install \
  "fastapi==0.109.2" \
  "uvicorn==0.27.1" \
  "pydantic==2.6.1" \
  "pydantic-settings==2.1.0" \
  "python-dotenv==1.0.1" \
  "openai==1.12.0" \
  "anthropic==0.18.1" \
  "azure-ai-inference==1.0.0b5" \
  "pytest==8.0.0" \
  "httpx==0.26.0"
```

Verifica que el reporte de benchmark de la práctica anterior existe:

```bash
ls -lh reports/benchmark/benchmark_results.json
```

**Salida esperada**

Se debe mostrar el archivo `benchmark_results.json` con tamaño mayor que cero.

**Verificación**

Ejecuta:

```bash
python -c "from pathlib import Path; assert Path('reports/benchmark/benchmark_results.json').exists(); print('Reporte encontrado')"
```

La salida debe ser:

```text
Reporte encontrado
```

## Desarrollo paso a paso

### Paso 1. Crear la estructura de la aplicación

**Objetivo:** reorganizar el código bajo `src/app/` para separar rutas HTTP, servicios y proveedores.

**Instrucciones**

1. Desde la raíz del repositorio, crea los directorios y archivos de paquete:

   ```bash
   mkdir -p src/app/api/routes
   mkdir -p src/app/providers
   mkdir -p src/app/services
   mkdir -p tests

   touch src/app/__init__.py
   touch src/app/api/__init__.py
   touch src/app/api/routes/__init__.py
   touch src/app/providers/__init__.py
   touch src/app/services/__init__.py
   ```

2. Crea o actualiza `.gitignore` para evitar versionar secretos, cachés y artefactos locales:

   ```bash
   cat >> .gitignore <<'EOF'

   # Secretos y configuraciones locales
   .env
   .env.*
   !.env.example

   # Python
   __pycache__/
   *.py[cod]
   .pytest_cache/
   EOF
   ```

3. Crea un archivo de ejemplo de variables de entorno sin valores sensibles:

   ```bash
   cat > .env.example <<'EOF'
   OPENAI_API_KEY=
   OPENAI_MODEL=gpt-4o-mini

   ANTHROPIC_API_KEY=
   ANTHROPIC_MODEL=claude-3-haiku-20240307

   AZURE_AI_INFERENCE_ENDPOINT=
   AZURE_AI_INFERENCE_CREDENTIAL=
   AZURE_AI_MODEL=

   BENCHMARK_QUALITY_THRESHOLD=0.70
   EOF
   ```

4. Si no existe, crea tu archivo local `.env` a partir del ejemplo:

   ```bash
   cp -n .env.example .env
   ```

5. Edita `.env` y agrega tus claves reales de OpenAI y Anthropic. No subas este archivo al repositorio.

   Ejemplo de formato:

   ```dotenv
   OPENAI_API_KEY=sk-proveedor-openai
   OPENAI_MODEL=gpt-4o-mini

   ANTHROPIC_API_KEY=sk-ant-proveedor-anthropic
   ANTHROPIC_MODEL=claude-3-haiku-20240307

   BENCHMARK_QUALITY_THRESHOLD=0.70
   ```

**Salida esperada**

La estructura mínima debe ser:

```text
src/
└── app/
    ├── api/
    │   └── routes/
    ├── providers/
    └── services/
```

**Verificación**

```bash
find src/app -maxdepth 3 -type f | sort
```

Confirma que aparecen los archivos `__init__.py` de cada paquete.

---

### Paso 2. Definir configuración, contratos y excepciones de dominio

**Objetivo:** centralizar las variables de entorno y crear contratos validados para el endpoint de generación.

**Instrucciones**

1. Crea el archivo `src/app/config.py`:

   ```bash
   cat > src/app/config.py <<'PY'
   from functools import lru_cache

   from pydantic_settings import BaseSettings, SettingsConfigDict


   class Settings(BaseSettings):
       model_config = SettingsConfigDict(
           env_file=".env",
           env_file_encoding="utf-8",
           extra="ignore",
       )

       openai_api_key: str | None = None
       openai_model: str = "gpt-4o-mini"

       anthropic_api_key: str | None = None
       anthropic_model: str = "claude-3-haiku-20240307"

       azure_ai_inference_endpoint: str | None = None
       azure_ai_inference_credential: str | None = None
       azure_ai_model: str | None = None

       benchmark_quality_threshold: float = 0.70


   @lru_cache
   def get_settings() -> Settings:
       return Settings()
   PY
   ```

2. Crea el archivo `src/app/models.py` con los contratos de solicitud y respuesta:

   ```bash
   cat > src/app/models.py <<'PY'
   from typing import Literal

   from pydantic import BaseModel, Field


   ProviderPreference = Literal["auto", "openai", "anthropic", "azure_ai"]
   TaskType = Literal[
       "classification",
       "extraction",
       "technical_explanation",
       "general",
   ]


   class GenerateRequest(BaseModel):
       prompt: str = Field(
           min_length=1,
           max_length=20_000,
           description="Instrucción enviada al modelo.",
       )
       task_type: TaskType = "general"
       provider_preference: ProviderPreference = "auto"
       temperature: float = Field(default=0.2, ge=0.0, le=2.0)
       max_tokens: int = Field(default=500, ge=1, le=4_000)


   class Usage(BaseModel):
       input_tokens: int | None = None
       output_tokens: int | None = None
       total_tokens: int | None = None


   class GenerateResponse(BaseModel):
       content: str
       provider: str
       model: str
       latency_ms: float
       usage: Usage
       request_id: str


   class HealthResponse(BaseModel):
       status: str
       service: str
   PY
   ```

3. Crea excepciones que permitan separar errores de configuración de errores inesperados del proveedor:

   ```bash
   cat > src/app/exceptions.py <<'PY'
   class ProviderConfigurationError(Exception):
       """El proveedor solicitado no cuenta con configuración válida."""


   class ProviderUnavailableError(Exception):
       """El proveedor no puede atender la solicitud en este momento."""
   PY
   ```

**Salida esperada**

Los modelos Pydantic deben restringir:

- `provider_preference` a `auto`, `openai`, `anthropic` o `azure_ai`.
- `task_type` a los cuatro tipos definidos.
- `temperature` al intervalo de `0.0` a `2.0`.
- `max_tokens` al intervalo de `1` a `4000`.

**Verificación**

```bash
PYTHONPATH=src python - <<'PY'
from app.models import GenerateRequest

request = GenerateRequest(
    prompt="Clasifica este ticket como incidente o solicitud.",
    task_type="classification",
)
print(request.model_dump())
PY
```

La salida debe incluir `provider_preference: 'auto'`, `temperature: 0.2` y `max_tokens: 500`.

---

### Paso 3. Implementar los adaptadores de OpenAI y Anthropic

**Objetivo:** encapsular las diferencias de cada SDK detrás de una interfaz homogénea de resultados.

**Instrucciones**

1. Crea el adaptador de OpenAI en `src/app/providers/openai_provider.py`:

   ```bash
   cat > src/app/providers/openai_provider.py <<'PY'
   import time

   from openai import OpenAI

   from app.config import Settings
   from app.exceptions import ProviderConfigurationError, ProviderUnavailableError
   from app.models import Usage


   class OpenAIProvider:
       provider_name = "openai"

       def __init__(self, settings: Settings) -> None:
           self.settings = settings

       def generate(
           self,
           prompt: str,
           temperature: float,
           max_tokens: int,
       ) -> dict:
           if not self.settings.openai_api_key:
               raise ProviderConfigurationError(
                   "OpenAI no está habilitado: falta OPENAI_API_KEY."
               )

           started = time.perf_counter()

           try:
               client = OpenAI(api_key=self.settings.openai_api_key)
               response = client.chat.completions.create(
                   model=self.settings.openai_model,
                   messages=[{"role": "user", "content": prompt}],
                   temperature=temperature,
                   max_tokens=max_tokens,
               )
           except Exception as error:
               raise ProviderUnavailableError(
                   f"No fue posible completar la solicitud con OpenAI: {error}"
               ) from error

           latency_ms = round((time.perf_counter() - started) * 1000, 2)
           usage_data = response.usage

           return {
               "content": response.choices[0].message.content or "",
               "provider": self.provider_name,
               "model": response.model,
               "latency_ms": latency_ms,
               "usage": Usage(
                   input_tokens=usage_data.prompt_tokens if usage_data else None,
                   output_tokens=usage_data.completion_tokens if usage_data else None,
                   total_tokens=usage_data.total_tokens if usage_data else None,
               ),
           }
   PY
   ```

2. Crea el adaptador de Anthropic en `src/app/providers/anthropic_provider.py`:

   ```bash
   cat > src/app/providers/anthropic_provider.py <<'PY'
   import time

   from anthropic import Anthropic

   from app.config import Settings
   from app.exceptions import ProviderConfigurationError, ProviderUnavailableError
   from app.models import Usage


   class AnthropicProvider:
       provider_name = "anthropic"

       def __init__(self, settings: Settings) -> None:
           self.settings = settings

       def generate(
           self,
           prompt: str,
           temperature: float,
           max_tokens: int,
       ) -> dict:
           if not self.settings.anthropic_api_key:
               raise ProviderConfigurationError(
                   "Anthropic no está habilitado: falta ANTHROPIC_API_KEY."
               )

           started = time.perf_counter()

           try:
               client = Anthropic(api_key=self.settings.anthropic_api_key)
               response = client.messages.create(
                   model=self.settings.anthropic_model,
                   max_tokens=max_tokens,
                   temperature=temperature,
                   messages=[{"role": "user", "content": prompt}],
               )
           except Exception as error:
               raise ProviderUnavailableError(
                   f"No fue posible completar la solicitud con Anthropic: {error}"
               ) from error

           latency_ms = round((time.perf_counter() - started) * 1000, 2)
           content = "".join(
               block.text
               for block in response.content
               if getattr(block, "type", None) == "text"
           )

           return {
               "content": content,
               "provider": self.provider_name,
               "model": response.model,
               "latency_ms": latency_ms,
               "usage": Usage(
                   input_tokens=response.usage.input_tokens,
                   output_tokens=response.usage.output_tokens,
                   total_tokens=(
                       response.usage.input_tokens + response.usage.output_tokens
                   ),
               ),
           }
   PY
   ```

3. Compila los módulos para detectar errores de sintaxis:

   ```bash
   PYTHONPATH=src python -m compileall -q src/app
   ```

**Salida esperada**

El comando no debe mostrar errores. Los proveedores no se invocan todavía; solo se valida que el código sea sintácticamente correcto.

**Verificación**

```bash
PYTHONPATH=src python - <<'PY'
from app.config import get_settings
from app.providers.openai_provider import OpenAIProvider
from app.providers.anthropic_provider import AnthropicProvider

settings = get_settings()
print(OpenAIProvider(settings).provider_name)
print(AnthropicProvider(settings).provider_name)
PY
```

La salida debe ser:

```text
openai
anthropic
```

---

### Paso 4. Preparar el adaptador de Azure AI Inference

**Objetivo:** crear un adaptador configurable para Azure AI sin codificar secretos ni depender de un endpoint activo.

**Instrucciones**

1. Crea `src/app/providers/azure_ai_provider.py`:

   ```bash
   cat > src/app/providers/azure_ai_provider.py <<'PY'
   import time

   from azure.ai.inference import ChatCompletionsClient
   from azure.ai.inference.models import UserMessage
   from azure.core.credentials import AzureKeyCredential

   from app.config import Settings
   from app.exceptions import ProviderConfigurationError, ProviderUnavailableError
   from app.models import Usage


   class AzureAIProvider:
       provider_name = "azure_ai"

       def __init__(self, settings: Settings) -> None:
           self.settings = settings

       def generate(
           self,
           prompt: str,
           temperature: float,
           max_tokens: int,
       ) -> dict:
           endpoint = self.settings.azure_ai_inference_endpoint
           credential = self.settings.azure_ai_inference_credential
           model = self.settings.azure_ai_model

           if not endpoint or not credential or not model:
               raise ProviderConfigurationError(
                   "Azure AI no está habilitado. Configure "
                   "AZURE_AI_INFERENCE_ENDPOINT, "
                   "AZURE_AI_INFERENCE_CREDENTIAL y AZURE_AI_MODEL."
               )

           started = time.perf_counter()

           try:
               client = ChatCompletionsClient(
                   endpoint=endpoint,
                   credential=AzureKeyCredential(credential),
               )
               response = client.complete(
                   model=model,
                   messages=[UserMessage(content=prompt)],
                   temperature=temperature,
                   max_tokens=max_tokens,
               )
           except Exception as error:
               raise ProviderUnavailableError(
                   f"No fue posible completar la solicitud con Azure AI: {error}"
               ) from error

           latency_ms = round((time.perf_counter() - started) * 1000, 2)
           usage_data = getattr(response, "usage", None)

           return {
               "content": response.choices[0].message.content or "",
               "provider": self.provider_name,
               "model": model,
               "latency_ms": latency_ms,
               "usage": Usage(
                   input_tokens=getattr(usage_data, "prompt_tokens", None),
                   output_tokens=getattr(usage_data, "completion_tokens", None),
                   total_tokens=getattr(usage_data, "total_tokens", None),
               ),
           }
   PY
   ```

2. Verifica que el paquete Azure AI Inference está disponible:

   ```bash
   PYTHONPATH=src python - <<'PY'
   from app.config import get_settings
   from app.providers.azure_ai_provider import AzureAIProvider

   provider = AzureAIProvider(get_settings())
   print(f"Proveedor preparado: {provider.provider_name}")
   PY
   ```

3. No agregues valores ficticios de Azure AI al archivo `.env`. La ausencia de configuración es válida en esta práctica y será tratada como un error HTTP controlado.

**Salida esperada**

```text
Proveedor preparado: azure_ai
```

**Verificación**

El adaptador valida explícitamente estas tres variables antes de crear una llamada de red:

- `AZURE_AI_INFERENCE_ENDPOINT`
- `AZURE_AI_INFERENCE_CREDENTIAL`
- `AZURE_AI_MODEL`

Esto evita que una configuración incompleta provoque errores ambiguos o secretos incrustados en el código.

---

### Paso 5. Implementar la política automática y el Router de Modelos

**Objetivo:** seleccionar proveedor y modelo según la preferencia del cliente o el reporte de benchmark.

**Instrucciones**

1. Crea `src/app/services/model_router.py`:

   ```bash
   cat > src/app/services/model_router.py <<'PY'
   import json
   from pathlib import Path
   from typing import Any

   from app.config import Settings
   from app.models import GenerateRequest
   from app.providers.anthropic_provider import AnthropicProvider
   from app.providers.azure_ai_provider import AzureAIProvider
   from app.providers.openai_provider import OpenAIProvider


   class ModelRouter:
       def __init__(self, settings: Settings) -> None:
           self.settings = settings
           self.providers = {
               "openai": OpenAIProvider(settings),
               "anthropic": AnthropicProvider(settings),
               "azure_ai": AzureAIProvider(settings),
           }
           self.benchmark_path = Path("reports/benchmark/benchmark_results.json")

       def generate(self, request: GenerateRequest) -> dict:
           provider_name = self.select_provider(
               task_type=request.task_type,
               preference=request.provider_preference,
           )
           provider = self.providers[provider_name]

           return provider.generate(
               prompt=request.prompt,
               temperature=request.temperature,
               max_tokens=request.max_tokens,
           )

       def select_provider(self, task_type: str, preference: str) -> str:
           if preference != "auto":
               return preference

           records = self._load_benchmark_records()
           candidates = [
               record for record in records
               if record["provider"] in self.providers
           ]

           task_candidates = [
               record for record in candidates
               if record.get("task_type") in (None, task_type)
           ]
           if task_candidates:
               candidates = task_candidates

           if not candidates:
               return "openai"

           threshold = self.settings.benchmark_quality_threshold
           qualified = [
               record for record in candidates
               if record["quality_score"] >= threshold
           ]
           if not qualified:
               qualified = candidates

           if task_type in {"classification", "extraction"}:
               selected = min(
                   qualified,
                   key=lambda record: (
                       record["cost_usd"],
                       -record["quality_score"],
                       record["latency_ms"],
                   ),
               )
           elif task_type == "technical_explanation":
               selected = max(
                   qualified,
                   key=lambda record: (
                       record["quality_score"],
                       -record["latency_ms"],
                       -record["cost_usd"],
                   ),
               )
           else:
               selected = max(
                   qualified,
                   key=lambda record: (
                       record["quality_score"],
                       -record["latency_ms"],
                   ),
               )

           return selected["provider"]

       def _load_benchmark_records(self) -> list[dict[str, Any]]:
           if not self.benchmark_path.exists():
               return []

           try:
               raw_data = json.loads(self.benchmark_path.read_text(encoding="utf-8"))
           except (json.JSONDecodeError, OSError):
               return []

           if isinstance(raw_data, list):
               source_records = raw_data
           elif isinstance(raw_data, dict):
               source_records = raw_data.get("results", raw_data.get("models", []))
           else:
               source_records = []

           normalized = []
           for item in source_records:
               if not isinstance(item, dict) or "provider" not in item:
                   continue

               normalized.append(
                   {
                       "provider": item["provider"],
                       "task_type": item.get("task_type"),
                       "quality_score": float(
                           item.get("quality_score", item.get("quality", 0))
                       ),
                       "cost_usd": float(
                           item.get("cost_usd", item.get("estimated_cost_usd", 999999))
                       ),
                       "latency_ms": float(item.get("latency_ms", 999999)),
                   }
               )

           return normalized
   PY
   ```

2. Revisa la política implementada:

   - Si `provider_preference` es explícito, el Router respeta la elección.
   - Para `classification` y `extraction`, selecciona el menor costo que alcance `BENCHMARK_QUALITY_THRESHOLD`.
   - Para `technical_explanation`, selecciona la mayor puntuación de calidad.
   - Si no existe el benchmark o no hay registros válidos, usa OpenAI como ruta de respaldo.
   - La política considera los campos `provider`, `task_type`, `quality_score`, `cost_usd` y `latency_ms`.

3. Si el JSON generado en la práctica anterior usa nombres diferentes, adapta solamente la normalización dentro de `_load_benchmark_records`. Conserva la interfaz de salida normalizada.

4. Ejecuta una comprobación local de selección sin llamar a proveedores:

   ```bash
   PYTHONPATH=src python - <<'PY'
   from app.config import get_settings
   from app.services.model_router import ModelRouter

   router = ModelRouter(get_settings())

   for task in ["classification", "extraction", "technical_explanation", "general"]:
       print(f"{task}: {router.select_provider(task, 'auto')}")
   PY
   ```

**Salida esperada**

Se debe imprimir un proveedor válido para cada tipo de tarea:

```text
classification: openai
extraction: anthropic
technical_explanation: openai
general: openai
```

Los proveedores concretos pueden variar según las métricas reales de tu benchmark.

**Verificación**

Comprueba que el Router nunca devuelve un nombre fuera de esta lista:

```text
openai
anthropic
azure_ai
```

> **Decisión arquitectónica:** la política automática no afirma que un modelo sea universalmente mejor. Usa evidencia del benchmark para reservar modelos de mayor calidad para tareas más complejas y emplear modelos más económicos cuando cumplen el umbral de calidad.

---

### Paso 6. Crear las rutas HTTP y la aplicación FastAPI

**Objetivo:** exponer el Router mediante una API REST con contratos validados, trazabilidad y manejo controlado de errores.

**Instrucciones**

1. Crea `src/app/api/routes/generate.py`:

   ```bash
   cat > src/app/api/routes/generate.py <<'PY'
   from uuid import uuid4

   from fastapi import APIRouter, HTTPException, status

   from app.config import get_settings
   from app.exceptions import ProviderConfigurationError, ProviderUnavailableError
   from app.models import GenerateRequest, GenerateResponse
   from app.services.model_router import ModelRouter


   router = APIRouter(prefix="/v1", tags=["generation"])


   @router.post(
       "/generate",
       response_model=GenerateResponse,
       status_code=status.HTTP_200_OK,
   )
   def generate(request: GenerateRequest) -> GenerateResponse:
       request_id = str(uuid4())
       model_router = ModelRouter(get_settings())

       try:
           result = model_router.generate(request)
       except ProviderConfigurationError as error:
           raise HTTPException(
               status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
               detail={
                   "message": str(error),
                   "request_id": request_id,
               },
           ) from error
       except ProviderUnavailableError as error:
           raise HTTPException(
               status_code=status.HTTP_502_BAD_GATEWAY,
               detail={
                   "message": str(error),
                   "request_id": request_id,
               },
           ) from error

       return GenerateResponse(
           content=result["content"],
           provider=result["provider"],
           model=result["model"],
           latency_ms=result["latency_ms"],
           usage=result["usage"],
           request_id=request_id,
       )
   PY
   ```

2. Crea `src/app/main.py`:

   ```bash
   cat > src/app/main.py <<'PY'
   from fastapi import FastAPI

   from app.api.routes.generate import router as generate_router
   from app.models import HealthResponse


   app = FastAPI(
       title="GenAI Model Router API",
       version="1.0.0",
       description=(
           "API base para seleccionar proveedores de modelos GenAI "
           "mediante políticas explícitas o automáticas."
       ),
   )


   @app.get("/health", response_model=HealthResponse, tags=["health"])
   def health() -> HealthResponse:
       return HealthResponse(status="ok", service="genai-model-router")


   app.include_router(generate_router)
   PY
   ```

3. Inicia la aplicación. Mantén esta terminal abierta:

   ```bash
   PYTHONPATH=src uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
   ```

**Salida esperada**

Uvicorn debe indicar que está escuchando en el host y puerto obligatorios:

```text
Uvicorn running on http://127.0.0.1:8000
```

**Verificación**

En una segunda terminal, activa el entorno y consulta el estado:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

curl -i http://127.0.0.1:8000/health
```

Debes recibir una respuesta HTTP `200 OK` similar a:

```json
{
  "status": "ok",
  "service": "genai-model-router"
}
```

También puedes abrir la documentación interactiva en:

```text
http://127.0.0.1:8000/docs
```

---

### Paso 7. Realizar llamadas locales de generación

**Objetivo:** validar el contrato de generación, la selección explícita y el comportamiento de Azure AI no configurado.

**Instrucciones**

1. Realiza una llamada explícita a OpenAI:

   ```bash
   curl -sS -X POST http://127.0.0.1:8000/v1/generate \
     -H "Content-Type: application/json" \
     -d '{
       "prompt": "Explica en dos frases qué es un Router de Modelos.",
       "task_type": "technical_explanation",
       "provider_preference": "openai",
       "temperature": 0.2,
       "max_tokens": 120
     }' | python -m json.tool
   ```

2. Realiza una llamada con selección automática. La elección dependerá de tu archivo de benchmark:

   ```bash
   curl -sS -X POST http://127.0.0.1:8000/v1/generate \
     -H "Content-Type: application/json" \
     -d '{
       "prompt": "Clasifica el ticket: El usuario no puede restablecer su contraseña.",
       "task_type": "classification",
       "provider_preference": "auto",
       "temperature": 0.0,
       "max_tokens": 80
     }' | python -m json.tool
   ```

3. Comprueba el error controlado de Azure AI cuando no hay configuración:

   ```bash
   curl -sS -i -X POST http://127.0.0.1:8000/v1/generate \
     -H "Content-Type: application/json" \
     -d '{
       "prompt": "Resume esta solicitud.",
       "task_type": "general",
       "provider_preference": "azure_ai",
       "temperature": 0.2,
       "max_tokens": 50
     }'
   ```

4. Comprueba la validación automática de Pydantic enviando una temperatura fuera del rango permitido:

   ```bash
   curl -sS -i -X POST http://127.0.0.1:8000/v1/generate \
     -H "Content-Type: application/json" \
     -d '{
       "prompt": "Prueba de validación.",
       "task_type": "general",
       "provider_preference": "openai",
       "temperature": 3.5,
       "max_tokens": 50
     }'
   ```

**Salida esperada**

Una respuesta exitosa debe respetar este contrato:

```json
{
  "content": "Texto generado por el modelo.",
  "provider": "openai",
  "model": "gpt-4o-mini",
  "latency_ms": 421.57,
  "usage": {
    "input_tokens": 23,
    "output_tokens": 35,
    "total_tokens": 58
  },
  "request_id": "c3d7d5f9-1e7e-4d3a-9f4d-9e2b34c61ed6"
}
```

La llamada explícita a Azure AI sin configuración debe devolver `503 Service Unavailable` y un mensaje que indique las variables faltantes, sin revelar claves ni detalles sensibles.

La temperatura inválida debe devolver `422 Unprocessable Entity`.

**Verificación**

Confirma los siguientes criterios:

- Existe un `request_id` distinto para cada solicitud.
- `latency_ms` es numérico y no negativo.
- `provider` coincide con el proveedor seleccionado.
- `usage` contiene campos de tokens, aunque algún proveedor puede devolver valores `null`.
- La respuesta no contiene valores de `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` ni credenciales Azure.

## Validación y pruebas

Crea una prueba mínima para garantizar que la API puede inicializarse y que el endpoint de salud no depende de proveedores externos.

```bash
cat > tests/test_main.py <<'PY'
from fastapi.testclient import TestClient

from app.main import app


client = TestClient(app)


def test_health_returns_ok() -> None:
    response = client.get("/health")

    assert response.status_code == 200
    assert response.json() == {
        "status": "ok",
        "service": "genai-model-router",
    }
PY
```

Ejecuta las pruebas:

```bash
PYTHONPATH=src pytest -q
```

**Resultado esperado**

```text
1 passed
```

Realiza además la validación final del contrato HTTP:

```bash
curl -sS http://127.0.0.1:8000/openapi.json | python - <<'PY'
import json
import sys

schema = json.load(sys.stdin)
paths = schema["paths"]

assert "/health" in paths
assert "/v1/generate" in paths
assert "post" in paths["/v1/generate"]

print("Contrato OpenAPI validado")
PY
```

La salida esperada es:

```text
Contrato OpenAPI validado
```

Finalmente, revisa los archivos que se incluirán en el commit:

```bash
git status
git diff -- src/app tests .env.example .gitignore
```

No debe aparecer `.env` como archivo preparado para versionar.

## Resolución de problemas

### Problema 1: la llamada a `/v1/generate` devuelve HTTP 502 para OpenAI o Anthropic

**Síntomas**

La respuesta contiene `502 Bad Gateway` y un mensaje similar a:

```text
No fue posible completar la solicitud con OpenAI
```

o:

```text
No fue posible completar la solicitud con Anthropic
```

**Causa**

La clave API es inválida, está ausente, no tiene cuota disponible, el nombre de modelo no está disponible para la cuenta o existe un problema temporal de conectividad con el proveedor.

**Solución**

1. Confirma que `.env` existe y contiene la variable correcta:

   ```bash
   grep -E '^(OPENAI_API_KEY|OPENAI_MODEL|ANTHROPIC_API_KEY|ANTHROPIC_MODEL)=' .env
   ```

2. Verifica que no haya espacios accidentales ni comillas no requeridas alrededor de la clave.
3. Confirma que el modelo configurado está habilitado para tu cuenta.
4. Reinicia Uvicorn después de modificar `.env`:

   ```bash
   Ctrl+C
   PYTHONPATH=src uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
   ```

5. Prueba primero una llamada explícita a un solo proveedor usando `provider_preference: "openai"` o `provider_preference: "anthropic"`.

### Problema 2: el Router automático siempre selecciona OpenAI

**Síntomas**

Las solicitudes con `"provider_preference": "auto"` devuelven siempre `"provider": "openai"`, incluso cuando el benchmark debería favorecer otro proveedor.

**Causa**

El archivo `reports/benchmark/benchmark_results.json` no existe, no contiene JSON válido, usa campos distintos a los esperados o los registros no incluyen proveedores compatibles.

**Solución**

1. Confirma que el archivo existe y es JSON válido:

   ```bash
   python -m json.tool reports/benchmark/benchmark_results.json > /dev/null
   echo $?
   ```

   El resultado debe ser `0`.

2. Inspecciona su estructura:

   ```bash
   cat reports/benchmark/benchmark_results.json | python -m json.tool | head -n 80
   ```

3. Asegura que cada registro contenga, como mínimo, `provider`, `quality_score`, `cost_usd` y `latency_ms`.
4. Si la práctica anterior utilizó nombres como `quality` o `estimated_cost_usd`, conserva esos datos y ajusta la normalización en `_load_benchmark_records`.
5. Reinicia la API y vuelve a ejecutar una solicitud automática.

## Limpieza

1. Detén el servidor FastAPI con `Ctrl+C` en la terminal donde se ejecuta Uvicorn.
2. No elimines el entorno virtual, el reporte de benchmark ni los archivos creados; serán reutilizados en prácticas posteriores.
3. Confirma que `.env` no será incluido en Git:

   ```bash
   git check-ignore -v .env
   ```

4. Agrega los archivos de la práctica y realiza el commit obligatorio:

   ```bash
   git add src/app tests .env.example .gitignore
   git commit -m "lab-01-00-02"
   ```

5. Verifica el commit:

   ```bash
   git log -1 --oneline
   ```

**Salida esperada**

```text
<hash> lab-01-00-02
```

## Resumen

Has construido una API FastAPI reutilizable para acceso a modelos generativos mediante un patrón de adaptadores y un Router de Modelos. La solución valida contratos de entrada y salida, registra trazabilidad operativa mediante proveedor, modelo, latencia, uso y `request_id`, y evita exponer secretos en el código.

La política automática aplica los principios de selección de modelos estudiados: las tareas de clasificación y extracción priorizan el menor costo que mantiene un umbral de calidad, mientras que las explicaciones técnicas priorizan la calidad. El adaptador Azure AI queda preparado con validación explícita de configuración y errores controlados, listo para una integración posterior con infraestructura Azure.

### Recursos opcionales

- [Documentación de FastAPI](https://fastapi.tiangolo.com/)
- [Documentación de OpenAI Python SDK](https://github.com/openai/openai-python)
- [Documentación de Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Azure AI Inference SDK para Python](https://learn.microsoft.com/python/api/overview/azure/ai-inference-readme)
- [Documentación de Pydantic Settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/)
