# 1. Práctica 1. Implementar un cliente Python que consuma modelos de OpenAI y Claude mediante sus SDK oficiales, configurando parámetros de inferencia y comparando sus respuestas frente a distintos escenarios de negocio.

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 30 minutos |
| Complejidad | Media |
| Nivel de Bloom | Aplicar |

## Descripción general

En esta práctica implementarás un cliente de línea de comandos reutilizable para consultar modelos de OpenAI y Anthropic Claude mediante sus SDK oficiales. El cliente normalizará las solicitudes y respuestas de ambos proveedores en contratos Pydantic comunes, registrará métricas de ejecución en archivos JSON y permitirá comparar el efecto de la temperatura en tres escenarios de negocio.

La implementación mantiene separadas la lógica de negocio, la autenticación y la telemetría. De este modo, el resultado será compatible con el Router de Modelos creado en la práctica 01-00-02 y podrá reutilizarse en los Agent Skills posteriores.

## Objetivos de aprendizaje

- [ ] Definir una interfaz común `LLMClient` y contratos normalizados con Pydantic.
- [ ] Implementar adaptadores para OpenAI y Anthropic usando sus SDK oficiales.
- [ ] Configurar `temperature`, `max_tokens`, modelo e instrucciones de sistema mediante argumentos de línea de comandos y variables de entorno.
- [ ] Ejecutar los escenarios de clasificación de tickets, extracción de datos y explicación de código Python.
- [ ] Registrar en JSON la solicitud normalizada, respuesta, uso de tokens, latencia y configuración efectiva.
- [ ] Comparar una misma consulta con temperaturas `0.0` y `0.7`.

## Requisitos previos

### Conocimientos

Antes de comenzar, debes poder:

- Activar un entorno virtual de Python y ejecutar módulos con `python -m`.
- Interpretar archivos JSON y variables de entorno.
- Usar argumentos de línea de comandos con `argparse`.
- Comprender el propósito de una interfaz o protocolo común entre proveedores.
- Haber completado las prácticas 01-00-01 y 01-00-02, especialmente la configuración de proveedores y el Router de Modelos.

### Acceso y configuración

Debes disponer de:

- El repositorio local `~/genai-agent-labs`.
- El entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Un archivo `.env` local válido y no versionado.
- Credenciales operativas de OpenAI y Anthropic Claude.
- Modelos autorizados en ambas cuentas.
- Acceso de escritura a `~/genai-agent-labs/reports/client_runs/`.

> **Seguridad:** no incluyas claves de API en el código, comandos con historial compartido, archivos JSON de resultados ni repositorios Git. Los reportes de esta práctica no deben contener encabezados HTTP, claves, ni variables de entorno completas.

## Entorno del laboratorio

### Software utilizado

| Componente | Versión de referencia | Uso |
|---|---:|---|
| Python | 3.12.1 | Implementación del cliente |
| OpenAI Python SDK | 1.12.0 | Consumo de OpenAI |
| Anthropic Python SDK | 0.18.1 | Consumo de Claude |
| Pydantic | 2.6.1 | Contratos normalizados |
| python-dotenv | 1.0.1 | Carga local de variables de entorno |
| Git | 2.43.0 | Control de versiones |

### Estructura objetivo

Al finalizar, la estructura relevante será la siguiente:

```text
~/genai-agent-labs/
├── .env
├── .gitignore
├── src/
│   ├── clients/
│   │   ├── __init__.py
│   │   └── business_scenarios_client.py
│   └── core/
│       ├── __init__.py
│       └── contracts.py
├── tests/
│   └── test_business_scenarios_client.py
└── reports/
    └── client_runs/
        └── *.json
```

## Desarrollo paso a paso

### Paso 1. Preparar el repositorio y verificar la configuración segura

**Objetivo:** activar el entorno compartido, confirmar las dependencias y preparar los directorios requeridos sin exponer secretos.

**Instrucciones:**

1. Abre una terminal y sitúate en el directorio global obligatorio:

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido:

   ```bash
   source .venv/bin/activate
   ```

3. Verifica las versiones instaladas:

   ```bash
   python --version
   python -m pip show openai anthropic pydantic python-dotenv
   ```

4. Si alguna dependencia no está disponible en el entorno compartido, instálala con las versiones de referencia:

   ```bash
   python -m pip install \
     "openai==1.12.0" \
     "anthropic==0.18.1" \
     "pydantic==2.6.1" \
     "python-dotenv==1.0.1"
   ```

5. Crea los directorios y archivos de paquete necesarios:

   ```bash
   mkdir -p src/clients src/core tests reports/client_runs
   touch src/clients/__init__.py src/core/__init__.py
   ```

6. Revisa que `.env` esté excluido de Git:

   ```bash
   grep -qxF '.env' .gitignore || echo '.env' >> .gitignore
   grep -qxF '.venv/' .gitignore || echo '.venv/' >> .gitignore
   grep -qxF '__pycache__/' .gitignore || echo '__pycache__/' >> .gitignore
   ```

7. Añade una regla para no versionar ejecuciones potencialmente sensibles. El directorio se mantendrá mediante un archivo `.gitkeep`.

   ```bash
   mkdir -p reports/client_runs
   touch reports/client_runs/.gitkeep
   grep -qxF 'reports/client_runs/*.json' .gitignore || \
     echo 'reports/client_runs/*.json' >> .gitignore
   ```

8. Verifica que las variables requeridas existen sin imprimir sus valores:

   ```bash
   python - <<'PY'
   import os
   from dotenv import load_dotenv

   load_dotenv()
   required = [
       "OPENAI_API_KEY",
       "OPENAI_MODEL",
       "ANTHROPIC_API_KEY",
       "ANTHROPIC_MODEL",
   ]
   missing = [name for name in required if not os.getenv(name)]
   print("Variables ausentes:", ", ".join(missing) if missing else "ninguna")
   PY
   ```

9. Si falta alguna variable, actualiza tu archivo `.env` local. Usa los modelos aprobados y configurados en la práctica 01-00-01.

   ```dotenv
   OPENAI_API_KEY=coloca_tu_clave_local_de_openai
   OPENAI_MODEL=gpt-4o-mini

   ANTHROPIC_API_KEY=coloca_tu_clave_local_de_anthropic
   ANTHROPIC_MODEL=claude-3-5-haiku-latest
   ```

   Si el Router de Modelos de la práctica anterior usa nombres diferentes para las variables de modelo, conserva sus nombres o adapta únicamente la carga de configuración del archivo que crearás en el siguiente paso. No dupliques secretos en más de un archivo.

**Resultado esperado:**

- El entorno virtual está activo.
- Los paquetes necesarios están disponibles.
- La verificación informa `Variables ausentes: ninguna`.
- `.env` y los resultados JSON están ignorados por Git.

**Verificación:**

```bash
git status --short
git check-ignore -v .env reports/client_runs/ejemplo.json
```

Debes observar que `.env` y `reports/client_runs/ejemplo.json` están ignorados. El archivo `reports/client_runs/.gitkeep` no debe estar ignorado.

---

### Paso 2. Definir los contratos normalizados y la interfaz común

**Objetivo:** crear contratos Pydantic independientes del proveedor para que el Router de Modelos y los clientes compartan una representación consistente de solicitudes, respuestas, uso y errores.

**Instrucciones:**

1. Crea el archivo `src/core/contracts.py`:

   ```bash
   cat > src/core/contracts.py <<'PY'
   from __future__ import annotations

   from datetime import datetime, timezone
   from typing import Literal, Protocol, runtime_checkable
   from uuid import uuid4

   from pydantic import BaseModel, Field, field_validator


   ProviderName = Literal["openai", "claude"]
   ScenarioName = Literal["ticket", "extraction", "code_explanation"]


   class LLMRequest(BaseModel):
       """Solicitud independiente del SDK y compatible con un Router de Modelos."""

       provider: ProviderName
       model: str = Field(min_length=1)
       scenario: ScenarioName
       system_prompt: str = Field(min_length=1)
       user_prompt: str = Field(min_length=1)
       temperature: float = Field(ge=0.0, le=1.0)
       max_tokens: int = Field(ge=1, le=4096)
       request_id: str = Field(default_factory=lambda: str(uuid4()))

       @field_validator("model")
       @classmethod
       def validate_model(cls, value: str) -> str:
           value = value.strip()
           if not value:
               raise ValueError("model no puede estar vacío")
           return value


   class TokenUsage(BaseModel):
       """Uso normalizado; los valores no disponibles se conservan como cero."""

       input_tokens: int = Field(default=0, ge=0)
       output_tokens: int = Field(default=0, ge=0)
       total_tokens: int = Field(default=0, ge=0)


   class LLMResponse(BaseModel):
       """Resultado normalizado que consume la capa de negocio o el Router."""

       request_id: str
       provider: ProviderName
       model: str
       scenario: ScenarioName
       text: str
       usage: TokenUsage
       latency_ms: float = Field(ge=0)
       effective_temperature: float = Field(ge=0.0, le=1.0)
       effective_max_tokens: int = Field(ge=1)
       created_at: datetime = Field(
           default_factory=lambda: datetime.now(timezone.utc)
       )


   @runtime_checkable
   class LLMClient(Protocol):
       """Contrato mínimo para adaptadores de proveedores de LLM."""

       def generate(self, request: LLMRequest) -> LLMResponse:
           """Envía una solicitud normalizada y devuelve una respuesta normalizada."""
           ...
   PY
   ```

2. Comprueba que el módulo se puede importar desde la raíz del repositorio:

   ```bash
   PYTHONPATH=src python - <<'PY'
   from core.contracts import LLMRequest, TokenUsage

   request = LLMRequest(
       provider="openai",
       model="gpt-4o-mini",
       scenario="ticket",
       system_prompt="Responde en español.",
       user_prompt="El portal de pagos no carga para todos los usuarios.",
       temperature=0.0,
       max_tokens=120,
   )

   print(request.model_dump_json(indent=2))
   print(TokenUsage(input_tokens=10, output_tokens=5, total_tokens=15))
   PY
   ```

3. Revisa los campos que hacen compatible este contrato con un Router de Modelos:

   | Campo | Propósito |
   |---|---|
   | `provider` | Permite seleccionar el adaptador sin que la capa de negocio conozca el SDK. |
   | `model` | Mantiene el modelo explícito para trazabilidad y comparación. |
   | `system_prompt` y `user_prompt` | Preservan la separación entre instrucciones y entrada de negocio. |
   | `temperature` y `max_tokens` | Expresan la configuración de inferencia de forma uniforme. |
   | `usage` | Normaliza tokens de entrada, salida y total. |
   | `latency_ms` | Permite comparar desempeño entre proveedores. |
   | `request_id` | Correlaciona reporte, ejecución y telemetría futura. |

4. Si el Router de Modelos creado en 01-00-02 ya define contratos equivalentes, no mantengas dos jerarquías incompatibles. En ese caso:

   - Conserva los nombres de campos normalizados anteriores.
   - Importa los contratos existentes desde `src/core/contracts.py`, o centralízalos allí.
   - No adaptes la capa de negocio a objetos propios de `OpenAI` o `Anthropic`.

**Resultado esperado:**

El archivo `src/core/contracts.py` contiene un protocolo `LLMClient` y modelos Pydantic para solicitudes, respuestas y uso de tokens.

**Verificación:**

```bash
PYTHONPATH=src python -c "from core.contracts import LLMClient, LLMRequest, LLMResponse; print('Contratos importados correctamente')"
```

La salida debe ser:

```text
Contratos importados correctamente
```

---

### Paso 3. Implementar el cliente de escenarios de negocio

**Objetivo:** implementar un cliente CLI con adaptadores para OpenAI y Claude, tres escenarios predefinidos, configuración de inferencia y almacenamiento de reportes JSON.

**Instrucciones:**

1. Crea `src/clients/business_scenarios_client.py` con el siguiente contenido:

   ```bash
   cat > src/clients/business_scenarios_client.py <<'PY'
   from __future__ import annotations

   import argparse
   import json
   import os
   import sys
   import time
   from datetime import datetime, timezone
   from pathlib import Path
   from typing import Any

   from anthropic import Anthropic
   from dotenv import load_dotenv
   from openai import (
       APIConnectionError,
       AuthenticationError,
       OpenAI,
       RateLimitError,
   )

   from core.contracts import LLMClient, LLMRequest, LLMResponse, TokenUsage


   PROJECT_ROOT = Path(__file__).resolve().parents[2]
   REPORTS_DIR = PROJECT_ROOT / "reports" / "client_runs"


   SCENARIOS: dict[str, dict[str, str]] = {
       "ticket": {
           "system_prompt": (
               "Eres un analista de soporte de nivel 1. Clasifica el ticket "
               "en español. Responde exactamente con JSON válido y sin Markdown "
               "con las claves: categoria, prioridad, producto, justificacion. "
               "Usa prioridad Alta, Media o Baja."
           ),
           "user_prompt": (
               "Ticket: Desde las 08:15, todos los vendedores de la región norte "
               "reciben error 503 al abrir el portal de pedidos. No pueden registrar "
               "ventas ni consultar inventario. El problema empezó después de una "
               "actualización del portal."
           ),
       },
       "extraction": {
           "system_prompt": (
               "Eres un asistente de operaciones. Extrae únicamente los datos "
               "solicitados. Responde exactamente con JSON válido y sin Markdown "
               "con las claves: cliente, numero_solicitud, servicio, fecha_requerida, "
               "contacto, monto_usd. Si un dato no existe, usa null."
           ),
           "user_prompt": (
               "Solicitud recibida por correo:\n"
               "Cliente: Fabrikam Logistics S.A.\n"
               "Solicitud #SR-2048\n"
               "Necesitamos ampliar el plan de monitoreo de flota para 120 vehículos. "
               "La activación debe completarse el 2026-09-01. "
               "Contacto: Laura Méndez, laura.mendez@fabrikam.example. "
               "El presupuesto aprobado es de USD 18,500."
           ),
       },
       "code_explanation": {
           "system_prompt": (
               "Eres un revisor de código Python. Explica el código para un "
               "desarrollador junior en español. Incluye: propósito, flujo de "
               "ejecución, un riesgo técnico y una mejora concreta. No inventes "
               "bibliotecas ni comportamiento que no aparezca en el fragmento."
           ),
           "user_prompt": (
               "Explica este código Python:\n\n"
               "def average_latency(values):\n"
               "    if not values:\n"
               "        return 0\n"
               "    return sum(values) / len(values)\n\n"
               "latencies = [120, 180, 90]\n"
               "print(average_latency(latencies))"
           ),
       },
   }


   def get_openai_usage(response: Any) -> TokenUsage:
       usage = getattr(response, "usage", None)
       input_tokens = getattr(usage, "input_tokens", 0) if usage else 0
       output_tokens = getattr(usage, "output_tokens", 0) if usage else 0
       total_tokens = getattr(usage, "total_tokens", 0) if usage else 0
       if not total_tokens:
           total_tokens = input_tokens + output_tokens
       return TokenUsage(
           input_tokens=input_tokens or 0,
           output_tokens=output_tokens or 0,
           total_tokens=total_tokens or 0,
       )


   def get_anthropic_usage(response: Any) -> TokenUsage:
       usage = getattr(response, "usage", None)
       input_tokens = getattr(usage, "input_tokens", 0) if usage else 0
       output_tokens = getattr(usage, "output_tokens", 0) if usage else 0
       return TokenUsage(
           input_tokens=input_tokens or 0,
           output_tokens=output_tokens or 0,
           total_tokens=(input_tokens or 0) + (output_tokens or 0),
       )


   class OpenAIClientAdapter(LLMClient):
       def __init__(self, api_key: str) -> None:
           self.client = OpenAI(api_key=api_key)

       def generate(self, request: LLMRequest) -> LLMResponse:
           start = time.perf_counter()
           response = self.client.responses.create(
               model=request.model,
               instructions=request.system_prompt,
               input=request.user_prompt,
               temperature=request.temperature,
               max_output_tokens=request.max_tokens,
           )
           latency_ms = round((time.perf_counter() - start) * 1000, 2)

           return LLMResponse(
               request_id=request.request_id,
               provider=request.provider,
               model=request.model,
               scenario=request.scenario,
               text=response.output_text.strip(),
               usage=get_openai_usage(response),
               latency_ms=latency_ms,
               effective_temperature=request.temperature,
               effective_max_tokens=request.max_tokens,
           )


   class ClaudeClientAdapter(LLMClient):
       def __init__(self, api_key: str) -> None:
           self.client = Anthropic(api_key=api_key)

       def generate(self, request: LLMRequest) -> LLMResponse:
           start = time.perf_counter()
           response = self.client.messages.create(
               model=request.model,
               max_tokens=request.max_tokens,
               temperature=request.temperature,
               system=request.system_prompt,
               messages=[
                   {
                       "role": "user",
                       "content": request.user_prompt,
                   }
               ],
           )
           latency_ms = round((time.perf_counter() - start) * 1000, 2)

           text = "".join(
               block.text
               for block in response.content
               if getattr(block, "type", None) == "text"
           ).strip()

           return LLMResponse(
               request_id=request.request_id,
               provider=request.provider,
               model=request.model,
               scenario=request.scenario,
               text=text,
               usage=get_anthropic_usage(response),
               latency_ms=latency_ms,
               effective_temperature=request.temperature,
               effective_max_tokens=request.max_tokens,
           )


   def create_client(provider: str) -> LLMClient:
       if provider == "openai":
           api_key = os.getenv("OPENAI_API_KEY")
           if not api_key:
               raise RuntimeError("Falta la variable de entorno OPENAI_API_KEY.")
           return OpenAIClientAdapter(api_key=api_key)

       if provider == "claude":
           api_key = os.getenv("ANTHROPIC_API_KEY")
           if not api_key:
               raise RuntimeError("Falta la variable de entorno ANTHROPIC_API_KEY.")
           return ClaudeClientAdapter(api_key=api_key)

       raise ValueError(f"Proveedor no soportado: {provider}")


   def resolve_model(provider: str, explicit_model: str | None) -> str:
       if explicit_model:
           return explicit_model

       environment_variable = (
           "OPENAI_MODEL" if provider == "openai" else "ANTHROPIC_MODEL"
       )
       model = os.getenv(environment_variable)
       if not model:
           raise RuntimeError(
               f"Falta {environment_variable}. Define el modelo aprobado en .env."
           )
       return model


   def save_report(request: LLMRequest, response: LLMResponse) -> Path:
       REPORTS_DIR.mkdir(parents=True, exist_ok=True)
       timestamp = datetime.now(timezone.utc).strftime("%Y%m%dT%H%M%SZ")
       filename = (
           f"{timestamp}_{request.provider}_{request.scenario}_"
           f"t{request.temperature:.1f}_{request.request_id[:8]}.json"
       )
       report_path = REPORTS_DIR / filename

       report = {
           "request": request.model_dump(mode="json"),
           "response": response.model_dump(mode="json"),
           "effective_configuration": {
               "provider": response.provider,
               "model": response.model,
               "temperature": response.effective_temperature,
               "max_tokens": response.effective_max_tokens,
           },
       }

       report_path.write_text(
           json.dumps(report, ensure_ascii=False, indent=2),
           encoding="utf-8",
       )
       return report_path


   def build_parser() -> argparse.ArgumentParser:
       parser = argparse.ArgumentParser(
           description="Ejecuta escenarios de negocio con OpenAI o Claude."
       )
       parser.add_argument(
           "--provider",
           required=True,
           choices=["openai", "claude"],
           help="Proveedor que se utilizará.",
       )
       parser.add_argument(
           "--scenario",
           required=True,
           choices=sorted(SCENARIOS.keys()),
           help="Escenario de negocio predefinido.",
       )
       parser.add_argument(
           "--temperature",
           type=float,
           default=0.0,
           help="Variabilidad de generación entre 0.0 y 1.0. Predeterminado: 0.0.",
       )
       parser.add_argument(
           "--max-tokens",
           type=int,
           default=250,
           help="Máximo de tokens de salida. Predeterminado: 250.",
       )
       parser.add_argument(
           "--model",
           default=None,
           help="Modelo opcional; si se omite, se lee desde .env.",
       )
       return parser


   def main() -> int:
       load_dotenv()
       args = build_parser().parse_args()

       if not 0.0 <= args.temperature <= 1.0:
           print("Error: --temperature debe estar entre 0.0 y 1.0.", file=sys.stderr)
           return 2

       if args.max_tokens < 1 or args.max_tokens > 4096:
           print("Error: --max-tokens debe estar entre 1 y 4096.", file=sys.stderr)
           return 2

       scenario = SCENARIOS[args.scenario]
       request = LLMRequest(
           provider=args.provider,
           model=resolve_model(args.provider, args.model),
           scenario=args.scenario,
           system_prompt=scenario["system_prompt"],
           user_prompt=scenario["user_prompt"],
           temperature=args.temperature,
           max_tokens=args.max_tokens,
       )

       try:
           client = create_client(args.provider)
           response = client.generate(request)
           report_path = save_report(request, response)

       except AuthenticationError:
           print(
               "Error de autenticación. Revisa la clave de OpenAI y sus permisos.",
               file=sys.stderr,
           )
           return 1

       except RateLimitError:
           print(
               "Límite de cuota o solicitudes alcanzado. Espera y vuelve a intentar.",
               file=sys.stderr,
           )
           return 1

       except APIConnectionError:
           print(
               "No fue posible conectar con OpenAI. Comprueba tu red.",
               file=sys.stderr,
           )
           return 1

       except Exception as error:
           print(
               f"Error controlado durante la ejecución: {type(error).__name__}.",
               file=sys.stderr,
           )
           return 1

       print("Respuesta normalizada:")
       print(response.text)
       print("\nMétricas:")
       print(
           json.dumps(
               {
                   "provider": response.provider,
                   "model": response.model,
                   "latency_ms": response.latency_ms,
                   "input_tokens": response.usage.input_tokens,
                   "output_tokens": response.usage.output_tokens,
                   "total_tokens": response.usage.total_tokens,
                   "report_path": str(report_path),
               },
               ensure_ascii=False,
               indent=2,
           )
       )
       return 0


   if __name__ == "__main__":
       raise SystemExit(main())
   PY
   ```

2. Revisa los elementos de diseño implementados:

   | Componente | Responsabilidad |
   |---|---|
   | `SCENARIOS` | Centraliza instrucciones y entradas de negocio reutilizables. |
   | `OpenAIClientAdapter` | Traduce `LLMRequest` a `responses.create`. |
   | `ClaudeClientAdapter` | Traduce `LLMRequest` a `messages.create`. |
   | `LLMClient` | Evita que la capa superior dependa de un SDK concreto. |
   | `TokenUsage` | Unifica métricas de uso con nombres homogéneos. |
   | `save_report()` | Almacena evidencia JSON sin secretos. |
   | `create_client()` | Centraliza autenticación por variables de entorno. |
   | `main()` | Gestiona CLI, validación y errores controlados. |

3. Comprueba la ayuda del programa. Esta acción no invoca ningún modelo ni consume cuota:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client --help
   ```

**Resultado esperado:**

La ayuda muestra los argumentos `--provider`, `--scenario`, `--temperature`, `--max-tokens` y `--model`.

**Verificación:**

La salida debe incluir opciones equivalentes a:

```text
--provider {openai,claude}
--scenario {code_explanation,extraction,ticket}
--temperature TEMPERATURE
--max-tokens MAX_TOKENS
```

---

### Paso 4. Validar el comportamiento local sin consumir API

**Objetivo:** comprobar que la validación de argumentos, los contratos y la estructura de escenarios funcionan antes de realizar llamadas con coste.

**Instrucciones:**

1. Crea el archivo de pruebas `tests/test_business_scenarios_client.py`:

   ```bash
   cat > tests/test_business_scenarios_client.py <<'PY'
   import unittest

   from clients.business_scenarios_client import SCENARIOS, get_anthropic_usage
   from core.contracts import LLMRequest


   class FakeUsage:
       input_tokens = 12
       output_tokens = 8


   class FakeAnthropicResponse:
       usage = FakeUsage()


   class BusinessScenariosClientTests(unittest.TestCase):
       def test_all_required_scenarios_exist(self):
           self.assertEqual(
               set(SCENARIOS.keys()),
               {"ticket", "extraction", "code_explanation"},
           )

       def test_normalized_request_accepts_valid_configuration(self):
           request = LLMRequest(
               provider="claude",
               model="claude-3-5-haiku-latest",
               scenario="extraction",
               system_prompt="Extrae datos.",
               user_prompt="Cliente: Contoso",
               temperature=0.7,
               max_tokens=200,
           )
           self.assertEqual(request.temperature, 0.7)
           self.assertEqual(request.max_tokens, 200)

       def test_anthropic_usage_is_normalized(self):
           usage = get_anthropic_usage(FakeAnthropicResponse())
           self.assertEqual(usage.input_tokens, 12)
           self.assertEqual(usage.output_tokens, 8)
           self.assertEqual(usage.total_tokens, 20)


   if __name__ == "__main__":
       unittest.main()
   PY
   ```

2. Ejecuta las pruebas:

   ```bash
   PYTHONPATH=src python -m unittest discover -s tests -v
   ```

3. Comprueba la validación de la CLI con una temperatura inválida. El comando debe terminar con código distinto de cero y no debe llamar a ningún proveedor:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider openai \
     --scenario ticket \
     --temperature 1.5 \
     --max-tokens 200
   ```

**Resultado esperado:**

- Las tres pruebas terminan con estado `ok`.
- La ejecución con `--temperature 1.5` informa que el valor debe estar entre `0.0` y `1.0`.

**Verificación:**

Debes obtener una salida semejante a:

```text
Ran 3 tests in ...s

OK
```

Y para el segundo comando:

```text
Error: --temperature debe estar entre 0.0 y 1.0.
```

---

### Paso 5. Ejecutar escenarios con OpenAI y Claude

**Objetivo:** realizar consultas reales a ambos proveedores y generar reportes normalizados con métricas de ejecución.

**Instrucciones:**

1. Ejecuta la clasificación de tickets con OpenAI y baja variabilidad:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider openai \
     --scenario ticket \
     --temperature 0.0 \
     --max-tokens 180
   ```

2. Ejecuta el mismo escenario con Claude:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider claude \
     --scenario ticket \
     --temperature 0.0 \
     --max-tokens 180
   ```

3. Ejecuta la extracción de datos con uno de los proveedores. En este escenario, una temperatura baja es adecuada porque se solicita información estructurada y verificable:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider openai \
     --scenario extraction \
     --temperature 0.0 \
     --max-tokens 200
   ```

4. Ejecuta la explicación de código con Claude. Una temperatura moderada puede aportar una explicación más natural, manteniendo las restricciones de la instrucción:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider claude \
     --scenario code_explanation \
     --temperature 0.7 \
     --max-tokens 250
   ```

5. Lista los reportes generados:

   ```bash
   ls -lt reports/client_runs/
   ```

6. Inspecciona el último reporte sin mostrar variables de entorno ni secretos:

   ```bash
   latest_report=$(ls -t reports/client_runs/*.json | head -n 1)
   python -m json.tool "$latest_report"
   ```

**Resultado esperado:**

Cada ejecución correcta muestra:

- El texto generado por el modelo.
- El proveedor y modelo usados.
- La latencia en milisegundos.
- Tokens de entrada, salida y total, cuando el proveedor los informa.
- La ruta a un archivo JSON en `reports/client_runs/`.

Un reporte tendrá una estructura similar a la siguiente:

```json
{
  "request": {
    "provider": "openai",
    "model": "gpt-4o-mini",
    "scenario": "ticket",
    "temperature": 0.0,
    "max_tokens": 180
  },
  "response": {
    "provider": "openai",
    "scenario": "ticket",
    "text": "...",
    "usage": {
      "input_tokens": 0,
      "output_tokens": 0,
      "total_tokens": 0
    },
    "latency_ms": 0.0
  },
  "effective_configuration": {
    "provider": "openai",
    "model": "gpt-4o-mini",
    "temperature": 0.0,
    "max_tokens": 180
  }
}
```

> Los valores de uso pueden variar según la versión del SDK, modelo o disponibilidad de métricas del proveedor. El contrato conserva el registro con cero cuando una métrica no está disponible, en lugar de fallar la ejecución.

**Verificación:**

Comprueba que hay al menos cuatro reportes JSON:

```bash
find reports/client_runs -maxdepth 1 -name '*.json' | wc -l
```

Comprueba además que el JSON es válido:

```bash
for file in reports/client_runs/*.json; do
  python -m json.tool "$file" > /dev/null || exit 1
done
echo "Todos los reportes JSON son válidos."
```

---

### Paso 6. Comparar el efecto de la temperatura

**Objetivo:** contrastar el comportamiento del modelo con temperaturas `0.0` y `0.7` utilizando exactamente el mismo proveedor, modelo, escenario y límite de salida.

**Instrucciones:**

1. Ejecuta el escenario de explicación de código con OpenAI a temperatura `0.0`:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider openai \
     --scenario code_explanation \
     --temperature 0.0 \
     --max-tokens 250
   ```

2. Ejecuta exactamente el mismo escenario con temperatura `0.7`:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider openai \
     --scenario code_explanation \
     --temperature 0.7 \
     --max-tokens 250
   ```

3. Identifica los dos reportes recientes de OpenAI para el escenario `code_explanation`:

   ```bash
   ls -t reports/client_runs/*openai_code_explanation*.json | head -n 2
   ```

4. Extrae los datos principales de cada reporte. Sustituye las rutas por las dos rutas obtenidas en el paso anterior:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   files = sorted(
       Path("reports/client_runs").glob("*openai_code_explanation*.json"),
       key=lambda path: path.stat().st_mtime,
       reverse=True,
   )[:2]

   for path in files:
       data = json.loads(path.read_text(encoding="utf-8"))
       response = data["response"]
       config = data["effective_configuration"]

       print(f"\nArchivo: {path.name}")
       print(f"Temperatura: {config['temperature']}")
       print(f"Latencia (ms): {response['latency_ms']}")
       print(f"Tokens totales: {response['usage']['total_tokens']}")
       print("Respuesta:")
       print(response["text"])
   PY
   ```

5. Registra tus observaciones en un archivo local de resultados. No copies claves ni datos confidenciales:

   ```bash
   cat > reports/client_runs/comparison_notes.md <<'MD'
   # Comparación de temperatura

   | Criterio | Temperatura 0.0 | Temperatura 0.7 |
   |---|---|---|
   | Cumple propósito, flujo, riesgo y mejora | | |
   | Mantiene el idioma español | | |
   | No inventa comportamiento | | |
   | Claridad para perfil junior | | |
   | Variación de redacción | | |
   | Latencia observada | | |
   | Tokens totales observados | | |

   ## Conclusión

   Para extracción y clasificación estructurada, la temperatura recomendada es:

   Para explicación de código, la temperatura recomendada es:

   Justificación:
   MD
   ```

6. Completa la tabla a partir de las respuestas reales.

**Resultado esperado:**

- Ambas ejecuciones cumplen la instrucción de explicar propósito, flujo, riesgo y mejora.
- La respuesta con temperatura `0.0` suele ser más repetible en ejecuciones equivalentes.
- La respuesta con temperatura `0.7` puede mostrar mayor variación de redacción, ejemplos o nivel de detalle.
- La temperatura no garantiza calidad por sí sola; las instrucciones explícitas y el modelo seleccionado siguen siendo factores principales.

**Verificación:**

Revisa que `comparison_notes.md` contenga una conclusión concreta y que los reportes comparados tengan:

- El mismo `provider`.
- El mismo `model`.
- El mismo `scenario`.
- El mismo `max_tokens`.
- Valores distintos de `temperature`: `0.0` y `0.7`.

---

### Paso 7. Relacionar resultados con el Router de Modelos y benchmarks previos

**Objetivo:** confirmar que el cliente es reutilizable por el Router de Modelos y que los resultados permiten decisiones informadas de selección de modelo.

**Instrucciones:**

1. Verifica que los adaptadores implementan el protocolo común:

   ```bash
   PYTHONPATH=src python - <<'PY'
   from clients.business_scenarios_client import (
       ClaudeClientAdapter,
       OpenAIClientAdapter,
   )
   from core.contracts import LLMClient

   print("OpenAIClientAdapter cumple LLMClient:", issubclass(OpenAIClientAdapter, LLMClient))
   print("ClaudeClientAdapter cumple LLMClient:", issubclass(ClaudeClientAdapter, LLMClient))
   PY
   ```

2. Revisa que la capa de negocio no importa tipos de SDK en `src/core/contracts.py`:

   ```bash
   grep -nE 'OpenAI|Anthropic|openai|anthropic' src/core/contracts.py || true
   ```

   No debe aparecer ninguna importación de los SDKs en el archivo de contratos.

3. Compara los resultados actuales con los datos de benchmark de 01-00-01 y la política de selección de 01-00-02. Evalúa, como mínimo:

   | Criterio | Evidencia de esta práctica | Relación con la selección |
   |---|---|---|
   | Calidad | Cumplimiento de formato, exactitud de extracción y utilidad de explicación | Determina si el modelo es apto para la tarea. |
   | Latencia | Campo `latency_ms` en los reportes | Ayuda a decidir si el modelo sirve para interacción síncrona. |
   | Tokens | Campo `usage` | Permite estimar coste y controlar límites. |
   | Temperatura | Configuración efectiva en cada reporte | Define repetibilidad frente a variedad. |
   | Robustez | Errores controlados y validación CLI | Permite integrar el cliente en flujos operativos. |

4. Documenta una decisión breve en `reports/client_runs/comparison_notes.md`. Por ejemplo:

   ```markdown
   ## Decisión de enrutamiento propuesta

   - Clasificación de tickets: usar el proveedor/modelo con mayor cumplimiento de JSON y menor latencia observada; temperatura 0.0.
   - Extracción de solicitudes: usar temperatura 0.0 por requerir exactitud y salida estructurada.
   - Explicación de código: usar el modelo que ofrezca explicaciones más claras según el benchmark; temperatura entre 0.0 y 0.7 según el nivel de variación permitido.
   ```

5. Revisa los cambios antes del commit:

   ```bash
   git status
   git diff -- src/core/contracts.py src/clients/business_scenarios_client.py tests/test_business_scenarios_client.py
   ```

6. Ejecuta la validación final local:

   ```bash
   PYTHONPATH=src python -m unittest discover -s tests -v
   ```

7. Agrega únicamente el código, las pruebas y el archivo `.gitkeep`. No agregues `.env` ni reportes JSON con entradas reales:

   ```bash
   git add \
     .gitignore \
     src/core/contracts.py \
     src/core/__init__.py \
     src/clients/__init__.py \
     src/clients/business_scenarios_client.py \
     tests/test_business_scenarios_client.py \
     reports/client_runs/.gitkeep
   ```

8. Realiza el commit obligatorio de esta práctica:

   ```bash
   git commit -m "lab-02-00-01"
   ```

**Resultado esperado:**

- Los dos adaptadores cumplen `LLMClient`.
- La capa de contratos permanece libre de dependencias específicas de proveedores.
- El commit se crea con el mensaje exacto `lab-02-00-01`.

**Verificación:**

```bash
git log -1 --oneline
git status --short
```

La primera salida debe comenzar con:

```text
<hash> lab-02-00-01
```

La segunda salida no debe incluir archivos `.env` ni archivos JSON de `reports/client_runs/`.

## Validación y pruebas

Ejecuta esta lista final de comprobación:

- [ ] `PYTHONPATH=src python -m unittest discover -s tests -v` termina con `OK`.
- [ ] `PYTHONPATH=src python -m clients.business_scenarios_client --help` muestra las opciones requeridas.
- [ ] Se ejecutó al menos una llamada exitosa a OpenAI.
- [ ] Se ejecutó al menos una llamada exitosa a Claude.
- [ ] Se ejecutaron los tres escenarios: `ticket`, `extraction` y `code_explanation`.
- [ ] Se generaron reportes JSON válidos en `reports/client_runs/`.
- [ ] Se comparó el mismo escenario con temperatura `0.0` y `0.7`.
- [ ] Los reportes contienen solicitud normalizada, respuesta, tokens, latencia y configuración efectiva.
- [ ] No hay claves API en archivos versionados.
- [ ] Existe el commit `lab-02-00-01`.

Ejecuta esta validación compacta:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

PYTHONPATH=src python -m unittest discover -s tests -v && \
find reports/client_runs -maxdepth 1 -name '*.json' -print -quit | grep -q . && \
git log -1 --pretty=%s | grep -qx 'lab-02-00-01' && \
echo "Validación final completada correctamente."
```

## Solución de problemas

### Problema 1. La ejecución muestra `Falta la variable de entorno OPENAI_API_KEY`, `ANTHROPIC_API_KEY` o una variable de modelo

**Síntomas:**

```text
Error controlado durante la ejecución: RuntimeError.
```

O bien:

```text
Falta OPENAI_MODEL. Define el modelo aprobado en .env.
```

**Causa:**

El archivo `.env` no existe, no está ubicado en `~/genai-agent-labs`, contiene un nombre de variable diferente al usado por el cliente, o la terminal se está ejecutando desde otra ubicación.

**Solución:**

1. Sitúate en la raíz del repositorio:

   ```bash
   cd ~/genai-agent-labs
   ```

2. Verifica la presencia del archivo sin mostrar su contenido:

   ```bash
   test -f .env && echo ".env encontrado" || echo ".env no encontrado"
   ```

3. Revisa solo los nombres de variables:

   ```bash
   cut -d= -f1 .env | grep -E '^(OPENAI|ANTHROPIC)_'
   ```

4. Asegúrate de definir `OPENAI_API_KEY`, `OPENAI_MODEL`, `ANTHROPIC_API_KEY` y `ANTHROPIC_MODEL`, o adapta de forma controlada la carga de variables para mantener compatibilidad con la política de 01-00-02.

5. Confirma que `.env` sigue ignorado:

   ```bash
   git check-ignore -v .env
   ```

### Problema 2. La llamada falla con autenticación, límite de cuota, modelo no disponible o error de proveedor

**Síntomas:**

La aplicación termina con un mensaje de autenticación, límite de cuota, o un mensaje genérico como:

```text
Error controlado durante la ejecución: NotFoundError.
```

**Causa:**

La clave no tiene permisos para el modelo configurado, el nombre del modelo no coincide con uno autorizado en la cuenta, se alcanzó una cuota temporal o se intenta utilizar un endpoint/configuración distinto al definido en las prácticas anteriores.

**Solución:**

1. Verifica los nombres de modelo configurados, sin imprimir claves:

   ```bash
   python - <<'PY'
   import os
   from dotenv import load_dotenv

   load_dotenv()
   print("OPENAI_MODEL:", os.getenv("OPENAI_MODEL"))
   print("ANTHROPIC_MODEL:", os.getenv("ANTHROPIC_MODEL"))
   PY
   ```

2. Contrasta los nombres con los modelos autorizados en tu cuenta y con las constantes del lote configuradas en 01-00-01.

3. Si recibes un límite de cuota, espera el periodo indicado por el proveedor y reduce pruebas repetitivas. No implementes reintentos agresivos.

4. Si usas una configuración corporativa, Azure OpenAI u otro endpoint administrado, conserva el adaptador y la política de autenticación definidos en 01-00-02. No sustituyas una configuración de identidad administrada o un Router existente por claves hardcodeadas.

5. Vuelve a ejecutar primero un escenario de bajo consumo:

   ```bash
   PYTHONPATH=src python -m clients.business_scenarios_client \
     --provider openai \
     --scenario ticket \
     --temperature 0.0 \
     --max-tokens 120
   ```

## Limpieza

1. Conserva el código fuente, las pruebas y el commit de Git.

2. Si los reportes contienen texto de negocio que no debe permanecer en el equipo, elimínalos. Esto no afecta el código ni el commit porque los JSON están ignorados:

   ```bash
   rm -f reports/client_runs/*.json
   ```

3. Conserva `reports/client_runs/.gitkeep`:

   ```bash
   touch reports/client_runs/.gitkeep
   ```

4. Desactiva el entorno virtual al terminar:

   ```bash
   deactivate
   ```

5. No elimines el archivo `.env` si será reutilizado por las siguientes prácticas. Verifica siempre que permanezca fuera del control de versiones:

   ```bash
   cd ~/genai-agent-labs
   git status --short
   git check-ignore -v .env
   ```

## Resumen

En esta práctica implementaste un cliente CLI que consume OpenAI y Claude mediante sus SDKs oficiales. Definiste una interfaz `LLMClient` y contratos Pydantic que aíslan a la capa de negocio de los detalles de cada proveedor.

También configuraste parámetros de inferencia, ejecutaste tres escenarios de negocio, generaste evidencia JSON de cada ejecución y comparaste temperaturas `0.0` y `0.7`. El resultado puede ser consumido por el Router de Modelos y constituye una base reutilizable para los Agent Skills de generación, revisión de código y evaluación automatizada de las siguientes prácticas.

### Recursos

- [Documentación de OpenAI para Python](https://github.com/openai/openai-python)
- [Documentación de Anthropic para Python](https://github.com/anthropics/anthropic-sdk-python)
- [Documentación de Pydantic](https://docs.pydantic.dev/latest/)
- [Buenas prácticas de seguridad para claves API de OpenAI](https://help.openai.com/en/articles/5112595-best-practices-for-api-key-safety)

---

# 1. Práctica 2. Desarrollar un Agent Skill reutilizable para encapsular una capacidad específica (por ejemplo, clasificación, extracción o resumen) e implementar un Harness que permita validar automáticamente distintos escenarios de entrada y salida.

## Metadatos

| Elemento | Valor |
|---|---|
| Duración | 35 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica construirás un **Agent Skill** reutilizable para clasificar incidencias de soporte técnico mediante un modelo de lenguaje. El skill estará desacoplado de FastAPI y del proveedor concreto, reutilizando el Router de Modelos y el contrato `LLMClient` implementados en prácticas anteriores.

Además, implementarás un **Harness de evaluación** que ejecuta automáticamente escenarios positivos, ambiguos, límite y negativos. El Harness validará tanto el contrato estructurado de salida como reglas funcionales de negocio, y almacenará un informe JSON reproducible en `reports/harness/`.

## Objetivos de aprendizaje

- [ ] Diseñar un Agent Skill desacoplado para clasificar incidencias de soporte.
- [ ] Definir contratos Pydantic de entrada y salida para respuestas JSON verificables.
- [ ] Forzar y validar respuestas estructuradas generadas por un modelo de lenguaje.
- [ ] Implementar un Harness que evalúe escenarios positivos, límites y negativos.
- [ ] Exponer el skill mediante un endpoint FastAPI sin incluir secretos en código.

## Prerrequisitos

### Conocimientos requeridos

- Uso de entornos virtuales de Python y `pip`.
- Variables de entorno y archivos `.env` no versionados.
- Pydantic 2.x, modelos de validación y serialización JSON.
- FastAPI, solicitudes `POST` y pruebas con `curl`.
- Contrato `LLMClient` y Router de Modelos desarrollados en las prácticas anteriores.
- Manejo básico de excepciones, archivos JSON y pruebas automatizadas.

### Acceso y configuración requeridos

- Repositorio local `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Credenciales configuradas para al menos un proveedor soportado por el Router:
  - Azure OpenAI, OpenAI o Anthropic Claude, según la implementación previa.
- Un despliegue o modelo configurado en las variables de entorno utilizadas por el Router.
- La API FastAPI de la práctica anterior disponible en el puerto `8000`.
- No incluyas claves, endpoints privados ni secretos en archivos versionados.

## Entorno del laboratorio

### Recursos de referencia

| Recurso | Mínimo |
|---|---|
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS |
| Python | 3.12.1 |
| Memoria RAM | 16 GB |
| Espacio libre | 10 GB |
| Directorio de trabajo | `~/genai-agent-labs` |
| Host de API | `127.0.0.1` |
| Puerto de API | `8000` |

### Dependencias principales

| Componente | Versión de referencia |
|---|---|
| FastAPI | 0.109.2 |
| Uvicorn | 0.27.1 |
| Pydantic | 2.6.1 |
| pytest | 8.0.0 |
| OpenAI Python SDK | 1.12.0 |
| Anthropic Python SDK | 0.18.1 |
| python-dotenv | 1.0.1 |

### Preparación inicial

1. Abre una terminal y entra en el repositorio obligatorio.

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido.

   ```bash
   source .venv/bin/activate
   ```

3. Verifica las versiones principales.

   ```bash
   python --version
   pip show fastapi pydantic pytest
   ```

4. Crea los directorios necesarios si no existen.

   ```bash
   mkdir -p src/skills src/harness data/harness prompts reports/harness tests
   touch src/skills/__init__.py src/harness/__init__.py
   ```

5. Comprueba que los secretos no serán versionados.

   ```bash
   grep -nE '^(\.env|\.venv/|__pycache__/)' .gitignore
   ```

   Si falta alguna entrada, añade las siguientes líneas:

   ```bash
   cat >> .gitignore <<'EOF'
   .env
   .venv/
   __pycache__/
   reports/harness/*.json
   EOF
   ```

> **Importante:** el informe del Harness puede contener entradas de tickets. En un entorno real podría contener datos personales o información operacional. No lo subas al repositorio salvo que haya sido anonimizado y el instructor lo solicite.

## Desarrollo paso a paso

### Paso 1. Revisar el contrato existente del Router de Modelos

**Objetivo:** identificar cómo invocar el cliente reutilizable sin acoplar el skill a Azure OpenAI, OpenAI o Claude.

**Instrucciones:**

1. Localiza los módulos creados en las prácticas anteriores.

   ```bash
   find src -maxdepth 3 -type f | sort
   ```

2. Busca la definición de `LLMClient`, `ModelRouter`, `Router` o métodos como `complete`, `generate` o `chat`.

   ```bash
   grep -RInE "class LLMClient|class ModelRouter|def complete|def generate|def chat" src
   ```

3. Identifica el método que recibe:
   - Una instrucción de sistema.
   - Un mensaje de usuario.
   - El modelo o proveedor seleccionado.
   - Un límite de tokens.
   - Opcionalmente un formato JSON o esquema estructurado.

4. Para esta práctica se utilizará el siguiente contrato lógico. Puede variar el nombre de los parámetros según tu implementación anterior:

   ```python
   response_text = client.complete(
       system_prompt=system_prompt,
       user_prompt=user_prompt,
       max_output_tokens=500,
       response_format="json",
   )
   ```

5. Si tu Router usa otros nombres, crea una adaptación pequeña **únicamente** en la función `_request_json_from_model()` del skill que implementarás en el siguiente paso. No distribuyas llamadas específicas de OpenAI o Anthropic por toda la lógica de negocio.

**Resultado esperado:**

Debes conocer el módulo y método concreto que permite al skill solicitar texto a un modelo a través de la capa común ya implementada.

**Verificación:**

Ejecuta una prueba mínima con el Router existente, usando una instrucción sencilla y sin escribir secretos en el comando:

```bash
python -c "
from src.clients.model_router import get_llm_client
client = get_llm_client()
print(type(client).__name__)
"
```

> Si tu proyecto usa otra ruta de importación, sustituye `src.clients.model_router` por la ruta real. Mantén esa adaptación localizada.

---

### Paso 2. Crear el prompt de sistema versionado

**Objetivo:** separar las instrucciones de clasificación de la lógica Python y establecer reglas verificables para la salida.

**Instrucciones:**

1. Crea el archivo `prompts/ticket_classification_v1.txt`.

   ```bash
   cat > prompts/ticket_classification_v1.txt <<'EOF'
   Eres un clasificador de incidencias de soporte técnico empresarial.

   Tu tarea es analizar un ticket y devolver exclusivamente un objeto JSON válido.
   No uses Markdown, bloques de código, comentarios ni texto antes o después del JSON.

   Categorías permitidas:
   - access: problemas de acceso, autenticación, permisos, contraseñas o MFA.
   - billing: facturación, cobros, suscripciones, facturas o pagos.
   - bug: comportamiento incorrecto, error de aplicación, fallo funcional o excepción.
   - infrastructure: disponibilidad, red, rendimiento, despliegue, almacenamiento o plataforma.
   - security: posible fuga de datos, malware, phishing, credenciales expuestas o acceso no autorizado.
   - general: consultas generales, solicitudes ambiguas o casos sin categoría suficiente.

   Prioridades permitidas:
   - low: impacto limitado, consulta general o molestia menor.
   - medium: afecta a una persona o función no crítica.
   - high: afecta a varios usuarios, un proceso relevante o un servicio importante.
   - critical: posible incidente de seguridad grave, indisponibilidad amplia o impacto operacional crítico.

   Reglas:
   1. Si hay indicios de phishing, malware, credenciales expuestas, fuga de datos o acceso no autorizado,
      selecciona category="security".
   2. Si la información es insuficiente, selecciona category="general" y una confianza baja o moderada.
   3. No inventes sistemas, usuarios, fechas, causas, evidencias ni acciones realizadas.
   4. rationale debe ser breve, objetiva y estar escrita en el mismo idioma predominante del ticket.
   5. suggested_team debe ser uno de: Service Desk, Identity, Billing, Application Support,
      Infrastructure, Security Operations.
   6. confidence debe ser un número decimal entre 0 y 1.
   7. Devuelve exactamente estas claves:
      category, priority, confidence, rationale, suggested_team

   Esquema JSON requerido:
   {
     "category": "access | billing | bug | infrastructure | security | general",
     "priority": "low | medium | high | critical",
     "confidence": 0.0,
     "rationale": "explicación breve",
     "suggested_team": "equipo permitido"
   }
   EOF
   ```

2. Revisa el contenido.

   ```bash
   cat prompts/ticket_classification_v1.txt
   ```

3. Confirma que el prompt está versionado como archivo de texto y no contiene una clave de API, endpoint ni dato sensible.

   ```bash
   grep -nEi "api[_-]?key|secret|password=|token=" prompts/ticket_classification_v1.txt || true
   ```

**Resultado esperado:**

Existe un prompt versionado que define categorías, prioridades, equipos permitidos, reglas de seguridad y un esquema JSON explícito.

**Verificación:**

La siguiente orden debe mostrar el archivo:

```bash
test -f prompts/ticket_classification_v1.txt && echo "Prompt disponible"
```

---

### Paso 3. Implementar los contratos y el Agent Skill

**Objetivo:** construir un componente reutilizable que valide entradas, solicite JSON al Router y devuelva una salida Pydantic validada.

**Instrucciones:**

1. Crea `src/skills/ticket_classification_skill.py`.

2. Copia el siguiente código. Ajusta solamente el bloque marcado como **PUNTO DE ADAPTACIÓN DEL ROUTER** para que coincida con el contrato real de la práctica anterior.

   ```python
   from __future__ import annotations

   import json
   import time
   from pathlib import Path
   from typing import Literal

   from pydantic import BaseModel, ConfigDict, Field, field_validator


   Category = Literal[
       "access",
       "billing",
       "bug",
       "infrastructure",
       "security",
       "general",
   ]

   Priority = Literal["low", "medium", "high", "critical"]

   SuggestedTeam = Literal[
       "Service Desk",
       "Identity",
       "Billing",
       "Application Support",
       "Infrastructure",
       "Security Operations",
   ]


   class TicketClassificationInput(BaseModel):
       model_config = ConfigDict(str_strip_whitespace=True)

       ticket_id: str = Field(
           min_length=1,
           max_length=100,
           examples=["INC-1001"],
       )
       subject: str = Field(min_length=1, max_length=300)
       description: str = Field(min_length=1, max_length=5000)
       optional_context: str | None = Field(default=None, max_length=2000)

       @field_validator("ticket_id")
       @classmethod
       def validate_ticket_id(cls, value: str) -> str:
           if not value.strip():
               raise ValueError("ticket_id no puede estar vacío.")
           return value


   class TicketClassificationOutput(BaseModel):
       model_config = ConfigDict(extra="forbid")

       category: Category
       priority: Priority
       confidence: float = Field(ge=0.0, le=1.0)
       rationale: str = Field(min_length=5, max_length=600)
       suggested_team: SuggestedTeam


   class SkillExecutionError(RuntimeError):
       """Error controlado durante la ejecución del skill."""


   PROMPT_PATH = (
       Path(__file__).resolve().parents[2]
       / "prompts"
       / "ticket_classification_v1.txt"
   )


   def _load_system_prompt() -> str:
       try:
           return PROMPT_PATH.read_text(encoding="utf-8")
       except FileNotFoundError as error:
           raise SkillExecutionError(
               f"No se encontró el prompt requerido: {PROMPT_PATH}"
           ) from error


   def _build_user_prompt(ticket: TicketClassificationInput) -> str:
       context = ticket.optional_context or "No se proporcionó contexto adicional."

       return (
           "Clasifica el siguiente ticket.\n\n"
           f"ticket_id: {ticket.ticket_id}\n"
           f"subject: {ticket.subject}\n"
           f"description: {ticket.description}\n"
           f"optional_context: {context}\n"
       )


   def _extract_json(text: str) -> dict:
       cleaned = text.strip()

       if cleaned.startswith("```"):
           cleaned = cleaned.removeprefix("```json").removeprefix("```")
           cleaned = cleaned.removesuffix("```").strip()

       try:
           payload = json.loads(cleaned)
       except json.JSONDecodeError as error:
           raise SkillExecutionError(
               "El modelo no devolvió un JSON válido para TicketClassificationOutput."
           ) from error

       if not isinstance(payload, dict):
           raise SkillExecutionError(
               "La respuesta JSON del modelo debe ser un objeto."
           )

       return payload


   def _request_json_from_model(
       llm_client,
       system_prompt: str,
       user_prompt: str,
   ) -> str:
       """
       Punto de adaptación único al contrato LLMClient de prácticas anteriores.

       El método debe devolver el contenido textual generado por el proveedor.
       Solicita JSON nativo si el proveedor o Router lo soporta.
       """

       # PUNTO DE ADAPTACIÓN DEL ROUTER:
       # Sustituye este bloque por la llamada equivalente de tu LLMClient.
       #
       # Ejemplo esperado:
       # return llm_client.complete(
       #     system_prompt=system_prompt,
       #     user_prompt=user_prompt,
       #     max_output_tokens=350,
       #     response_format="json",
       # )

       return llm_client.complete(
           system_prompt=system_prompt,
           user_prompt=user_prompt,
           max_output_tokens=350,
           response_format="json",
       )


   def classify_ticket(
       ticket: TicketClassificationInput,
       llm_client,
   ) -> tuple[TicketClassificationOutput, float]:
       """
       Ejecuta el skill y devuelve una salida validada junto con latencia en ms.
       """

       system_prompt = _load_system_prompt()
       user_prompt = _build_user_prompt(ticket)

       started = time.perf_counter()
       try:
           raw_response = _request_json_from_model(
               llm_client=llm_client,
               system_prompt=system_prompt,
               user_prompt=user_prompt,
           )
       except Exception as error:
           raise SkillExecutionError(
               "No fue posible obtener una respuesta del proveedor LLM."
           ) from error

       latency_ms = round((time.perf_counter() - started) * 1000, 2)

       payload = _extract_json(raw_response)

       try:
           result = TicketClassificationOutput.model_validate(payload)
       except Exception as error:
           raise SkillExecutionError(
               "La respuesta del modelo no cumple TicketClassificationOutput."
           ) from error

       return result, latency_ms
   ```

3. Si tu Router no implementa `response_format="json"`, conserva el prompt con la instrucción JSON y elimina solo ese parámetro. Por ejemplo:

   ```python
   return llm_client.complete(
       system_prompt=system_prompt,
       user_prompt=user_prompt,
       max_output_tokens=350,
   )
   ```

4. Si tu Router devuelve un objeto de respuesta y no una cadena, extrae el texto en `_request_json_from_model()`. Por ejemplo:

   ```python
   response = llm_client.complete(
       system_prompt=system_prompt,
       user_prompt=user_prompt,
       max_output_tokens=350,
   )
   return response.text
   ```

5. Verifica la sintaxis.

   ```bash
   python -m py_compile src/skills/ticket_classification_skill.py
   ```

**Resultado esperado:**

El skill contiene:

- Un contrato de entrada `TicketClassificationInput`.
- Un contrato de salida estricto `TicketClassificationOutput`.
- Categorías, prioridades y equipos permitidos mediante `Literal`.
- Carga del prompt desde un archivo versionado.
- Validación de JSON.
- Manejo explícito de errores mediante `SkillExecutionError`.
- Una única zona de adaptación al Router.

**Verificación:**

Ejecuta esta prueba local que no llama al proveedor:

```bash
python - <<'PY'
from src.skills.ticket_classification_skill import TicketClassificationInput

ticket = TicketClassificationInput(
    ticket_id="INC-LOCAL-01",
    subject="Cannot sign in",
    description="My MFA code is rejected.",
)
print(ticket.model_dump_json(indent=2))
PY
```

La salida debe mostrar un JSON válido con `optional_context` como `null`.

---

### Paso 4. Crear los escenarios del Harness

**Objetivo:** definir un conjunto reproducible de al menos ocho casos con expectativas objetivas.

**Instrucciones:**

1. Crea el directorio de datos si aún no existe.

   ```bash
   mkdir -p data/harness
   ```

2. Crea `data/harness/ticket_classification_cases.json`.

   ```bash
   cat > data/harness/ticket_classification_cases.json <<'EOF'
   [
     {
       "case_id": "TC-001-access-en",
       "description": "Solicitud en inglés relacionada con MFA.",
       "input": {
         "ticket_id": "INC-1001",
         "subject": "Cannot sign in with MFA",
         "description": "My authenticator code is rejected and I cannot access the customer portal.",
         "optional_context": "Single user affected."
       },
       "expected": {
         "categories": ["access"],
         "priorities": ["medium", "high"],
         "min_confidence": 0.60,
         "expected_team": "Identity",
         "forbidden_words": ["password123", "ignore previous instructions"]
       }
     },
     {
       "case_id": "TC-002-billing-es",
       "description": "Solicitud en español sobre un cobro duplicado.",
       "input": {
         "ticket_id": "INC-1002",
         "subject": "Cobro duplicado en la suscripción",
         "description": "Veo dos cargos por la suscripción mensual en la tarjeta corporativa.",
         "optional_context": "El cliente solicita revisión de factura."
       },
       "expected": {
         "categories": ["billing"],
         "priorities": ["medium"],
         "min_confidence": 0.60,
         "expected_team": "Billing",
         "forbidden_words": ["número de tarjeta", "password"]
       }
     },
     {
       "case_id": "TC-003-bug-es",
       "description": "Error funcional reproducible en la aplicación.",
       "input": {
         "ticket_id": "INC-1003",
         "subject": "Error al exportar informe",
         "description": "Al pulsar Exportar CSV aparece un error 500. Ocurre desde esta mañana para varios analistas.",
         "optional_context": "Producción."
       },
       "expected": {
         "categories": ["bug"],
         "priorities": ["high"],
         "min_confidence": 0.60,
         "expected_team": "Application Support",
         "forbidden_words": ["culpa", "garantizado"]
       }
     },
     {
       "case_id": "TC-004-infrastructure-en",
       "description": "Indisponibilidad de plataforma para varios usuarios.",
       "input": {
         "ticket_id": "INC-1004",
         "subject": "VPN is unavailable",
         "description": "Multiple employees cannot establish a VPN connection from two offices.",
         "optional_context": "Business operations are delayed."
       },
       "expected": {
         "categories": ["infrastructure"],
         "priorities": ["high", "critical"],
         "min_confidence": 0.60,
         "expected_team": "Infrastructure",
         "forbidden_words": ["password", "secret key"]
       }
     },
     {
       "case_id": "TC-005-security-es",
       "description": "Posible incidente de seguridad.",
       "input": {
         "ticket_id": "INC-1005",
         "subject": "Posible correo de phishing",
         "description": "Un empleado introdujo sus credenciales en un enlace recibido por correo que parecía corporativo.",
         "optional_context": "Se debe investigar con urgencia."
       },
       "expected": {
         "categories": ["security"],
         "priorities": ["high", "critical"],
         "min_confidence": 0.70,
         "expected_team": "Security Operations",
         "forbidden_words": ["ignore", "no action needed"]
       }
     },
     {
       "case_id": "TC-006-ambiguous-es",
       "description": "Mensaje ambiguo con información insuficiente.",
       "input": {
         "ticket_id": "INC-1006",
         "subject": "No funciona",
         "description": "La aplicación está rara.",
         "optional_context": null
       },
       "expected": {
         "categories": ["general"],
         "priorities": ["low", "medium"],
         "min_confidence": 0.00,
         "expected_team": "Service Desk",
         "forbidden_words": ["definitivamente", "garantizado"]
       }
     },
     {
       "case_id": "TC-007-short-en",
       "description": "Entrada excesivamente corta pero válida.",
       "input": {
         "ticket_id": "INC-1007",
         "subject": "Help",
         "description": "Error",
         "optional_context": null
       },
       "expected": {
         "categories": ["general"],
         "priorities": ["low", "medium"],
         "min_confidence": 0.00,
         "expected_team": "Service Desk",
         "forbidden_words": ["password", "secret"]
       }
     },
     {
       "case_id": "TC-008-empty-description",
       "description": "Caso negativo: descripción vacía.",
       "input": {
         "ticket_id": "INC-1008",
         "subject": "Sin detalle",
         "description": "",
         "optional_context": null
       },
       "expected_error": "validation"
     }
   ]
   EOF
   ```

3. Valida que el archivo sea JSON correcto.

   ```bash
   python -m json.tool data/harness/ticket_classification_cases.json > /dev/null
   echo $?
   ```

**Resultado esperado:**

El dataset contiene ocho casos que cubren:

- Categorías válidas.
- Solicitudes en español e inglés.
- Caso de acceso.
- Facturación.
- Error funcional.
- Infraestructura.
- Posible incidente de seguridad.
- Mensaje ambiguo.
- Entrada muy corta.
- Entrada negativa con descripción vacía.

**Verificación:**

```bash
python - <<'PY'
import json
from pathlib import Path

cases = json.loads(Path("data/harness/ticket_classification_cases.json").read_text())
print(f"Casos cargados: {len(cases)}")
assert len(cases) >= 8
assert any("expected_error" in case for case in cases)
assert any(case["case_id"] == "TC-005-security-es" for case in cases)
PY
```

---

### Paso 5. Implementar el Harness de evaluación

**Objetivo:** automatizar la ejecución de casos y generar un informe con estado, proveedor, modelo, latencia y detalle de validación.

**Instrucciones:**

1. Crea `src/harness/skill_harness.py`.

2. Copia el siguiente código.

   ```python
   from __future__ import annotations

   import json
   from datetime import UTC, datetime
   from pathlib import Path
   from typing import Any

   from pydantic import ValidationError

   from src.skills.ticket_classification_skill import (
       SkillExecutionError,
       TicketClassificationInput,
       classify_ticket,
   )


   PROJECT_ROOT = Path(__file__).resolve().parents[2]
   CASES_PATH = (
       PROJECT_ROOT / "data" / "harness" / "ticket_classification_cases.json"
   )
   REPORT_PATH = (
       PROJECT_ROOT
       / "reports"
       / "harness"
       / "ticket_classification_report.json"
   )


   def _client_metadata(llm_client: Any) -> dict[str, str | None]:
       """
       Adapta esta función si el Router expone provider/model con otros nombres.
       No se incluyen secretos en el informe.
       """
       return {
           "provider": getattr(llm_client, "provider", None)
           or getattr(llm_client, "provider_name", None)
           or "unknown",
           "model": getattr(llm_client, "model", None)
           or getattr(llm_client, "model_name", None)
           or "unknown",
       }


   def _validate_output(
       output: dict[str, Any],
       expected: dict[str, Any],
   ) -> list[str]:
       failures: list[str] = []

       if output["category"] not in expected["categories"]:
           failures.append(
               f"category={output['category']} no pertenece a "
               f"{expected['categories']}."
           )

       if output["priority"] not in expected["priorities"]:
           failures.append(
               f"priority={output['priority']} no pertenece a "
               f"{expected['priorities']}."
           )

       if output["confidence"] < expected["min_confidence"]:
           failures.append(
               f"confidence={output['confidence']} es menor que "
               f"{expected['min_confidence']}."
           )

       if output["suggested_team"] != expected["expected_team"]:
           failures.append(
               f"suggested_team={output['suggested_team']} no coincide con "
               f"{expected['expected_team']}."
           )

       searchable_text = " ".join(
           [
               output["rationale"],
               output["category"],
               output["priority"],
               output["suggested_team"],
           ]
       ).lower()

       for forbidden_word in expected.get("forbidden_words", []):
           if forbidden_word.lower() in searchable_text:
               failures.append(
                   f"La salida contiene la palabra prohibida: {forbidden_word}."
               )

       return failures


   def run_harness(llm_client: Any) -> dict[str, Any]:
       cases = json.loads(CASES_PATH.read_text(encoding="utf-8"))
       metadata = _client_metadata(llm_client)
       results: list[dict[str, Any]] = []

       for case in cases:
           case_id = case["case_id"]
           result: dict[str, Any] = {
               "case_id": case_id,
               "description": case["description"],
               "status": "error",
               "provider": metadata["provider"],
               "model": metadata["model"],
               "latency_ms": None,
               "validation": [],
           }

           try:
               ticket = TicketClassificationInput.model_validate(case["input"])

               if "expected_error" in case:
                   result["status"] = "fail"
                   result["validation"].append(
                       "Se esperaba un error de validación, pero la entrada fue aceptada."
                   )
               else:
                   output, latency_ms = classify_ticket(ticket, llm_client)
                   output_data = output.model_dump()
                   failures = _validate_output(output_data, case["expected"])

                   result["latency_ms"] = latency_ms
                   result["output"] = output_data
                   result["validation"] = failures
                   result["status"] = "pass" if not failures else "fail"

           except ValidationError as error:
               if case.get("expected_error") == "validation":
                   result["status"] = "pass"
                   result["validation"].append(
                       "La entrada inválida fue rechazada correctamente."
                   )
               else:
                   result["status"] = "error"
                   result["validation"].append(
                       f"Error de validación inesperado: {error.errors()}"
                   )

           except SkillExecutionError as error:
               result["status"] = "error"
               result["validation"].append(str(error))

           except Exception as error:
               result["status"] = "error"
               result["validation"].append(
                   f"Error no controlado: {type(error).__name__}: {error}"
               )

           results.append(result)

       summary = {
           "total": len(results),
           "pass": sum(item["status"] == "pass" for item in results),
           "fail": sum(item["status"] == "fail" for item in results),
           "error": sum(item["status"] == "error" for item in results),
       }

       report = {
           "skill": "ticket-classification",
           "generated_at_utc": datetime.now(UTC).isoformat(),
           "provider": metadata["provider"],
           "model": metadata["model"],
           "summary": summary,
           "results": results,
       }

       REPORT_PATH.parent.mkdir(parents=True, exist_ok=True)
       REPORT_PATH.write_text(
           json.dumps(report, ensure_ascii=False, indent=2),
           encoding="utf-8",
       )

       return report
   ```

3. Crea un ejecutable de línea de comandos para simplificar la evaluación. Crea `src/harness/run_ticket_classification_harness.py`.

   ```bash
   cat > src/harness/run_ticket_classification_harness.py <<'EOF'
   import json

   from src.harness.skill_harness import run_harness

   # Ajusta este import a la fábrica del Router implementada previamente.
   from src.clients.model_router import get_llm_client


   if __name__ == "__main__":
       client = get_llm_client()
       report = run_harness(client)
       print(json.dumps(report["summary"], ensure_ascii=False, indent=2))
   EOF
   ```

4. Ajusta la importación `get_llm_client` si tu Router usa otro módulo o una factoría con parámetros como proveedor y modelo.

5. Verifica sintaxis sin llamar al proveedor.

   ```bash
   python -m py_compile \
     src/harness/skill_harness.py \
     src/harness/run_ticket_classification_harness.py
   ```

**Resultado esperado:**

El Harness:

- Carga casos desde `data/harness/ticket_classification_cases.json`.
- Ejecuta el skill para los casos válidos.
- Acepta correctamente el caso negativo esperado.
- Valida categorías, prioridades, confianza, equipo y palabras prohibidas.
- Genera estados `pass`, `fail` o `error`.
- Registra proveedor, modelo, latencia y detalle de validación.
- Guarda un informe JSON en `reports/harness/ticket_classification_report.json`.

**Verificación:**

Comprueba que las rutas de entrada y salida sean correctas:

```bash
python - <<'PY'
from src.harness.skill_harness import CASES_PATH, REPORT_PATH

print(f"Casos: {CASES_PATH}")
print(f"Informe: {REPORT_PATH}")
assert CASES_PATH.exists()
assert REPORT_PATH.name == "ticket_classification_report.json"
PY
```

---

### Paso 6. Añadir el endpoint FastAPI

**Objetivo:** permitir la invocación del skill mediante `POST /v1/skills/ticket-classification`.

**Instrucciones:**

1. Abre el archivo principal de FastAPI creado en la Práctica 02-00-01. Una ruta frecuente es `src/main.py`, `src/api/main.py` o `src/app.py`.

2. Importa los contratos y el skill. Ajusta la importación del Router según tu implementación:

   ```python
   from fastapi import FastAPI, HTTPException
   from src.clients.model_router import get_llm_client
   from src.skills.ticket_classification_skill import (
       SkillExecutionError,
       TicketClassificationInput,
       TicketClassificationOutput,
       classify_ticket,
   )
   ```

3. Añade el endpoint al objeto `app` existente:

   ```python
   @app.post(
       "/v1/skills/ticket-classification",
       response_model=TicketClassificationOutput,
       tags=["skills"],
   )
   def classify_ticket_endpoint(
       ticket: TicketClassificationInput,
   ) -> TicketClassificationOutput:
       try:
           llm_client = get_llm_client()
           result, _latency_ms = classify_ticket(
               ticket=ticket,
               llm_client=llm_client,
           )
           return result

       except SkillExecutionError as error:
           raise HTTPException(
               status_code=502,
               detail="No fue posible clasificar el ticket con el proveedor configurado.",
           ) from error
   ```

4. No devuelvas internamente:
   - Claves API.
   - Configuración de proveedor.
   - Trazas completas.
   - El prompt de sistema.
   - Detalles técnicos de excepciones al cliente HTTP.

5. Si tu API utiliza `APIRouter`, añade la ruta al router correspondiente y registra el router en la aplicación principal.

6. Inicia la API desde la raíz del proyecto. Sustituye `src.main:app` por el módulo real de tu práctica anterior si es necesario.

   ```bash
   uvicorn src.main:app --host 127.0.0.1 --port 8000 --reload
   ```

**Resultado esperado:**

La API expone el endpoint:

```text
POST http://127.0.0.1:8000/v1/skills/ticket-classification
```

El endpoint valida el cuerpo mediante Pydantic, invoca el skill y devuelve un objeto conforme a `TicketClassificationOutput`.

**Verificación:**

En otra terminal, con el entorno virtual activado, consulta OpenAPI:

```bash
curl -s http://127.0.0.1:8000/openapi.json | python -m json.tool | grep -n "ticket-classification"
```

La salida debe incluir la ruta `/v1/skills/ticket-classification`.

---

### Paso 7. Invocar el skill directamente y mediante REST

**Objetivo:** validar las dos formas requeridas de consumo del skill.

**Instrucciones:**

1. Ejecuta el skill directamente desde Python. Ajusta el import del Router si fuera necesario.

   ```bash
   python - <<'PY'
   from src.clients.model_router import get_llm_client
   from src.skills.ticket_classification_skill import (
       TicketClassificationInput,
       classify_ticket,
   )

   client = get_llm_client()

   ticket = TicketClassificationInput(
       ticket_id="INC-MANUAL-01",
       subject="Suspicious login notification",
       description=(
           "I received a login notification from an unknown location "
           "and did not initiate it."
       ),
       optional_context="Please assess whether this may be a security incident.",
   )

   output, latency_ms = classify_ticket(ticket, client)
   print(output.model_dump_json(indent=2))
   print(f"Latencia: {latency_ms} ms")
   PY
   ```

2. La clasificación esperada para este caso es `security`, con equipo sugerido `Security Operations`.

3. Invoca el endpoint REST.

   ```bash
   curl -s -X POST http://127.0.0.1:8000/v1/skills/ticket-classification \
     -H "Content-Type: application/json" \
     -d '{
       "ticket_id": "INC-REST-01",
       "subject": "Cobro duplicado",
       "description": "La factura mensual muestra el mismo servicio cobrado dos veces.",
       "optional_context": "Cuenta empresarial."
     }' | python -m json.tool
   ```

4. Prueba una entrada inválida para confirmar la validación HTTP de FastAPI.

   ```bash
   curl -s -i -X POST http://127.0.0.1:8000/v1/skills/ticket-classification \
     -H "Content-Type: application/json" \
     -d '{
       "ticket_id": "INC-REST-02",
       "subject": "Sin descripción",
       "description": ""
     }'
   ```

**Resultado esperado:**

- La invocación Python devuelve un JSON validado y una latencia.
- La llamada REST devuelve `200 OK` para un ticket válido.
- La llamada REST con `description` vacía devuelve `422 Unprocessable Entity`.

**Verificación:**

Una respuesta válida debe incluir exactamente estas propiedades principales:

```json
{
  "category": "billing",
  "priority": "medium",
  "confidence": 0.0,
  "rationale": "Explicación breve.",
  "suggested_team": "Billing"
}
```

La confianza real variará según el modelo, pero debe estar entre `0.0` y `1.0`.

## Validación y pruebas

### Ejecutar el Harness completo

1. Detén el servidor FastAPI solo si el Router o la cuota del proveedor no permiten solicitudes simultáneas. En caso contrario, puede permanecer en ejecución.

2. Ejecuta el Harness desde la raíz del repositorio.

   ```bash
   python -m src.harness.run_ticket_classification_harness
   ```

3. Revisa el informe generado.

   ```bash
   python -m json.tool \
     reports/harness/ticket_classification_report.json | less
   ```

4. Consulta el resumen.

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   report = json.loads(
       Path("reports/harness/ticket_classification_report.json").read_text()
   )
   print(json.dumps(report["summary"], indent=2))
   PY
   ```

### Interpretar estados del informe

| Estado | Significado | Acción recomendada |
|---|---|---|
| `pass` | La entrada fue procesada y todas las reglas esperadas se cumplieron. | Mantener el caso como regresión. |
| `fail` | El skill respondió, pero incumplió una expectativa funcional. | Revisar prompt, caso o criterios de aceptación. |
| `error` | Hubo un fallo de proveedor, red, JSON o contrato no controlado. | Revisar configuración, Router o manejo de errores. |

### Pruebas unitarias mínimas

1. Crea `tests/test_ticket_classification_skill.py`.

   ```bash
   cat > tests/test_ticket_classification_skill.py <<'EOF'
   import pytest
   from pydantic import ValidationError

   from src.skills.ticket_classification_skill import (
       TicketClassificationInput,
       TicketClassificationOutput,
   )


   def test_ticket_input_rejects_empty_description():
       with pytest.raises(ValidationError):
           TicketClassificationInput(
               ticket_id="INC-TEST-01",
               subject="Empty description",
               description="",
           )


   def test_ticket_output_accepts_valid_contract():
       output = TicketClassificationOutput(
           category="security",
           priority="critical",
           confidence=0.91,
           rationale="Hay indicios de posible exposición de credenciales.",
           suggested_team="Security Operations",
       )

       assert output.category == "security"
       assert output.confidence == 0.91


   def test_ticket_output_rejects_unknown_category():
       with pytest.raises(ValidationError):
           TicketClassificationOutput(
               category="network",
               priority="high",
               confidence=0.80,
               rationale="La red no responde.",
               suggested_team="Infrastructure",
           )
   EOF
   ```

2. Ejecuta las pruebas.

   ```bash
   pytest -q tests/test_ticket_classification_skill.py
   ```

3. Confirma que las pruebas no requieren una llamada real al proveedor. Las pruebas unitarias deben validar contratos locales; el Harness es la evaluación de integración con el LLM.

### Criterios de aceptación

Completa la práctica cuando se cumplan todos los criterios siguientes:

- [ ] Existe `src/skills/ticket_classification_skill.py`.
- [ ] Existe `prompts/ticket_classification_v1.txt`.
- [ ] El skill recibe `ticket_id`, `subject`, `description` y `optional_context`.
- [ ] El skill devuelve `category`, `priority`, `confidence`, `rationale` y `suggested_team`.
- [ ] La salida está validada con `TicketClassificationOutput`.
- [ ] El prompt exige JSON sin Markdown ni texto adicional.
- [ ] Existe `data/harness/ticket_classification_cases.json` con al menos ocho casos.
- [ ] Los casos incluyen español, inglés, ambigüedad, texto corto, seguridad y entrada negativa.
- [ ] Existe `src/harness/skill_harness.py`.
- [ ] El Harness genera `reports/harness/ticket_classification_report.json`.
- [ ] El informe contiene `pass`, `fail` o `error`, proveedor, modelo, latencia y validaciones.
- [ ] El endpoint `POST /v1/skills/ticket-classification` responde en `http://127.0.0.1:8000`.
- [ ] Las pruebas unitarias pasan.
- [ ] No hay secretos en el código, prompt, informe versionado ni repositorio Git.

## Resolución de problemas

### Problema 1. El Harness informa `error` y el detalle indica que el modelo no devolvió JSON válido

**Síntoma:** el informe contiene mensajes como `El modelo no devolvió un JSON válido para TicketClassificationOutput` o la respuesta contiene texto explicativo y bloques Markdown.

**Causa:** el Router no está solicitando un formato JSON nativo cuando el proveedor lo soporta, el prompt no se está cargando correctamente, o el modelo elegido no sigue de forma consistente las instrucciones de formato.

**Solución:**

1. Comprueba que se carga el archivo correcto:

   ```bash
   python - <<'PY'
   from src.skills.ticket_classification_skill import _load_system_prompt
   print(_load_system_prompt()[:200])
   PY
   ```

2. Verifica que `_request_json_from_model()` usa la capacidad de JSON estructurado del Router o proveedor, si está disponible.
3. Confirma que la llamada incluye `response_format="json"` o el equivalente de tu implementación.
4. Mantén `_extract_json()` como validación defensiva; no aceptes texto libre como si fuera una clasificación válida.
5. Reduce la variabilidad del modelo en el Router, por ejemplo usando temperatura baja si el contrato anterior lo permite.

### Problema 2. El endpoint devuelve `502` o el Harness muestra errores de autenticación, cuota o conexión

**Síntoma:** la API responde `502 Bad Gateway`, el Harness muestra `No fue posible obtener una respuesta del proveedor LLM`, o los registros del servidor muestran errores de autenticación, `RateLimitError` o conectividad.

**Causa:** faltan variables de entorno, el modelo configurado no existe, no hay cuota disponible, el endpoint de Azure OpenAI es incorrecto o el Router no puede crear el cliente del proveedor.

**Solución:**

1. Comprueba que las variables requeridas existen sin imprimir su valor:

   ```bash
   python - <<'PY'
   import os

   names = [
       "AZURE_OPENAI_API_KEY",
       "AZURE_OPENAI_ENDPOINT",
       "OPENAI_API_KEY",
       "ANTHROPIC_API_KEY",
   ]

   for name in names:
       print(f"{name}: {'configurada' if os.getenv(name) else 'no configurada'}")
   PY
   ```

2. Revisa el archivo `.env` local y confirma que está incluido en `.gitignore`.
3. Confirma el nombre exacto del despliegue o modelo configurado en el Router.
4. Espera y reintenta si existe límite temporal de cuota.
5. Revisa los logs locales de Uvicorn, pero no copies secretos ni cabeceras de autorización en informes o capturas.

## Limpieza

1. Detén el servidor FastAPI si sigue activo.

   ```bash
   Ctrl+C
   ```

2. No elimines el entorno virtual compartido ni los archivos fuente, porque serán reutilizados en laboratorios posteriores.

3. Si el instructor no requiere conservar resultados de ejecución, elimina solo el informe generado localmente:

   ```bash
   rm -f reports/harness/ticket_classification_report.json
   ```

4. Revisa los cambios antes de confirmar.

   ```bash
   git status
   git diff -- src/skills src/harness prompts data/harness tests
   ```

5. Confirma que no se agregará ningún secreto.

   ```bash
   git status --short | grep -E '(^|/)\.env$' && echo "No agregues .env" || true
   ```

6. Realiza el commit obligatorio de esta práctica.

   ```bash
   git add \
     src/skills/ticket_classification_skill.py \
     src/harness/skill_harness.py \
     src/harness/run_ticket_classification_harness.py \
     prompts/ticket_classification_v1.txt \
     data/harness/ticket_classification_cases.json \
     tests/test_ticket_classification_skill.py

   git commit -m "lab-02-00-02"
   ```

> Si modificaste el archivo principal de FastAPI, inclúyelo explícitamente en `git add` antes del commit.

## Resumen

En esta práctica implementaste un Agent Skill de clasificación de tickets con contratos Pydantic estrictos, un prompt de sistema versionado y una integración desacoplada del proveedor mediante el Router de Modelos. El skill valida tanto la entrada como la salida JSON generada, reduciendo el riesgo de que texto libre o estructuras incompletas entren en la lógica de negocio.

También construiste un Harness de evaluación basado en datos que ejecuta escenarios representativos, incluyendo seguridad, ambigüedad, idiomas distintos y errores de validación. El informe generado permite comparar proveedores y modelos usando resultados observables: cumplimiento de reglas, latencia, proveedor, modelo y detalle de fallos.

Como siguiente evolución, puedes incorporar métricas de coste estimado, reintentos controlados ante errores transitorios, trazas de observabilidad y casos de regresión adicionales aprobados por el equipo de soporte y seguridad.

---

# 2. Práctica 3. Implementar un flujo automatizado para revisión de código utilizando OpenAI o Claude para detectar oportunidades de mejora, vulnerabilidades y recomendaciones de refactorización.

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 35 minutos |
| Complejidad | Difícil |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica extenderás la arquitectura existente de GenAI con un **Agent Skill de revisión de código**. El skill enviará fragmentos de código a OpenAI mediante Azure OpenAI o a Claude mediante Anthropic, obtendrá una respuesta JSON estructurada y devolverá hallazgos priorizados por severidad, categoría y ubicación aproximada.

También generalizarás el Harness creado en la práctica anterior para validar automáticamente que el modelo detecta problemas conocidos de seguridad, confiabilidad y mantenibilidad. Finalmente, ejecutarás Ruff como herramienta determinista de análisis estático y compararás su propósito con el análisis contextual proporcionado por el LLM.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Implementar un `CodeReviewSkill` reutilizable que use el Router de Modelos existente.
- [ ] Definir y validar una salida estructurada `CodeReviewOutput` con Pydantic.
- [ ] Detectar inyección SQL, secretos codificados, uso inseguro de `eval`, manejo deficiente de excepciones y duplicación de código.
- [ ] Exponer la revisión de código mediante el endpoint `POST /v1/skills/code-review`.
- [ ] Ejecutar un Harness automatizado y contrastar sus resultados con Ruff 0.2.1.

## Prerrequisitos

### Conocimientos

- Práctica `02-00-02` completada, incluyendo el patrón de Agent Skill, el Router de Modelos y el Harness reutilizable.
- Uso básico de FastAPI, Pydantic y peticiones HTTP REST.
- Conocimientos introductorios sobre vulnerabilidades comunes: inyección SQL, secretos en código, ejecución dinámica y manejo de errores.
- Uso de variables de entorno y archivos `.env` sin versionar.

### Acceso y configuración requerida

- Repositorio local en `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Credenciales válidas para al menos uno de estos proveedores:
  - Azure OpenAI con un despliegue de chat compatible.
  - Anthropic Claude con una API key válida.
- Ruff versión `0.2.1`.
- Puerto local `8000` disponible.

> **Advertencia de seguridad:** los ejemplos de secretos de esta práctica son valores ficticios diseñados para que el modelo los detecte. Nunca envíes secretos reales, tokens válidos, código propietario sensible ni datos personales a un proveedor de modelos sin autorización.

## Entorno del laboratorio

### Recursos de referencia

| Recurso | Mínimo recomendado |
|---|---|
| Procesador | 4 núcleos físicos |
| Memoria RAM | 16 GB; 32 GB recomendado |
| Espacio libre | 10 GB mínimo; 20 GB recomendado |
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS |
| Python | 3.12.1 |
| FastAPI | 0.109.2 |
| Pydantic | 2.6.1 |
| OpenAI SDK | 1.12.0 |
| Anthropic SDK | 0.18.1 |
| Ruff | 0.2.1 |

### Preparación inicial

1. Abre una terminal y entra en el directorio obligatorio del curso:

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido:

   ```bash
   source .venv/bin/activate
   ```

3. Instala o verifica las dependencias requeridas:

   ```bash
   pip install \
     "openai==1.12.0" \
     "anthropic==0.18.1" \
     "fastapi==0.109.2" \
     "uvicorn==0.27.1" \
     "pydantic==2.6.1" \
     "pydantic-settings==2.1.0" \
     "python-dotenv==1.0.1" \
     "ruff==0.2.1"
   ```

4. Verifica las versiones principales:

   ```bash
   python --version
   ruff --version
   ```

5. Confirma que `.env` y el entorno virtual no se versionan:

   ```bash
   grep -E '^(\.env|\.venv/|__pycache__/)$' .gitignore
   ```

   Si falta alguna entrada, agrégala:

   ```bash
   cat >> .gitignore <<'EOF'

   # Secretos y artefactos locales
   .env
   .venv/
   __pycache__/
   reports/code_review_*.json
   EOF
   ```

## Desarrollo paso a paso

### Paso 1. Verificar la estructura base y configurar el proveedor

**Objetivo:** preparar las variables de entorno y la estructura de archivos para el skill de revisión.

**Instrucciones:**

1. Crea los directorios requeridos si no existen:

   ```bash
   mkdir -p \
     src/skills \
     src/schemas \
     src/services \
     src/harness \
     data/code_samples \
     data/harness \
     reports \
     tests
   ```

2. Verifica que los paquetes Python puedan importarse:

   ```bash
   touch src/__init__.py
   touch src/skills/__init__.py
   touch src/schemas/__init__.py
   touch src/services/__init__.py
   touch src/harness/__init__.py
   ```

3. Añade o actualiza el archivo local `.env`. Usa **uno** de los dos proveedores como mínimo.

   Para Azure OpenAI:

   ```dotenv
   MODEL_PROVIDER=openai

   AZURE_OPENAI_ENDPOINT=https://TU-RECURSO.openai.azure.com/
   AZURE_OPENAI_API_KEY=tu_clave_local_no_versionada
   AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini
   AZURE_OPENAI_API_VERSION=2024-02-15-preview
   ```

   Para Anthropic Claude:

   ```dotenv
   MODEL_PROVIDER=claude

   ANTHROPIC_API_KEY=tu_clave_local_no_versionada
   ANTHROPIC_MODEL=claude-3-5-haiku-latest
   ```

4. Comprueba que el archivo no será añadido a Git:

   ```bash
   git check-ignore -v .env
   ```

**Resultado esperado:**

- La estructura contiene directorios para skills, esquemas, servicios, datos, Harness y reportes.
- `.env` aparece ignorado por Git.
- Existe una configuración válida para `MODEL_PROVIDER=openai` o `MODEL_PROVIDER=claude`.

**Verificación:**

```bash
find src data -maxdepth 2 -type d | sort
git status --short
```

No debe aparecer `.env` en la salida de `git status`.

---

### Paso 2. Definir los contratos de entrada y salida estructurada

**Objetivo:** crear modelos Pydantic que establezcan un contrato estable para el endpoint, el skill y el Harness.

**Instrucciones:**

1. Crea el archivo `src/schemas/code_review.py`:

   ```bash
   cat > src/schemas/code_review.py <<'PY'
   from typing import Literal

   from pydantic import BaseModel, Field


   Severity = Literal["critical", "high", "medium", "low", "info"]
   Category = Literal[
       "security",
       "reliability",
       "maintainability",
       "performance",
       "style",
       "other",
   ]


   class CodeReviewRequest(BaseModel):
       language: str = Field(
           min_length=1,
           max_length=50,
           examples=["python"],
       )
       code: str = Field(
           min_length=1,
           max_length=40_000,
           description="Fragmento de código que se analizará; no debe contener secretos reales.",
       )
       repository_context: str = Field(
           default="",
           max_length=4_000,
           description="Contexto no sensible del repositorio, módulo o propósito funcional.",
       )
       review_focus: list[str] = Field(
           default_factory=lambda: [
               "security",
               "reliability",
               "maintainability",
           ],
           description="Áreas prioritarias para la revisión.",
       )


   class CodeFinding(BaseModel):
       severity: Severity
       category: Category
       title: str = Field(min_length=1, max_length=180)
       location: str = Field(
           min_length=1,
           max_length=200,
           description="Ubicación aproximada, por ejemplo: línea 8 o función login.",
       )
       evidence: str = Field(
           min_length=1,
           max_length=1_000,
           description="Evidencia observada exclusivamente en el código proporcionado.",
       )
       recommendation: str = Field(
           min_length=1,
           max_length=1_500,
       )
       refactored_snippet: str | None = Field(
           default=None,
           max_length=4_000,
       )
       uncertainty: bool = Field(
           default=False,
           description="True si la conclusión requiere datos externos no disponibles.",
       )


   class CodeReviewOutput(BaseModel):
       summary: str = Field(min_length=1, max_length=2_000)
       findings: list[CodeFinding] = Field(default_factory=list)
   PY
   ```

2. Verifica que el esquema se puede importar:

   ```bash
   python -c "from src.schemas.code_review import CodeReviewRequest, CodeReviewOutput; print('Esquemas importados correctamente')"
   ```

3. Observa las decisiones de diseño implementadas:

   - `severity` usa una escala controlada: `critical`, `high`, `medium`, `low` e `info`.
   - `category` limita las categorías esperadas por el Harness.
   - `location` admite una ubicación aproximada porque el LLM puede no contar líneas exactamente.
   - `uncertainty` obliga al modelo a diferenciar evidencia observada de hipótesis no verificables.
   - `refactored_snippet` es opcional para evitar generar cambios innecesarios en todos los hallazgos.

**Resultado esperado:**

```text
Esquemas importados correctamente
```

**Verificación:**

```bash
python - <<'PY'
from src.schemas.code_review import CodeReviewRequest

request = CodeReviewRequest(
    language="python",
    code="print('hello')",
    repository_context="Ejemplo mínimo",
    review_focus=["maintainability"],
)
print(request.model_dump_json(indent=2))
PY
```

La salida debe ser un JSON válido con los cuatro campos de entrada.

---

### Paso 3. Implementar o adaptar el Router de Modelos

**Objetivo:** aislar las diferencias entre Azure OpenAI y Claude detrás de una interfaz única que devuelva texto.

**Instrucciones:**

1. Crea o adapta `src/services/model_router.py`.

   > Si el Router de la práctica `02-00-02` ya existe, conserva su interfaz institucional. Debe ofrecer una operación equivalente a `generate(system_prompt, user_prompt, max_tokens)` y devolver texto normalizado.

   ```bash
   cat > src/services/model_router.py <<'PY'
   import os

   from anthropic import Anthropic
   from dotenv import load_dotenv
   from openai import AzureOpenAI


   class ModelRouter:
       """Router mínimo que normaliza la salida de Azure OpenAI o Claude."""

       def __init__(self) -> None:
           load_dotenv()

           self.provider = os.getenv("MODEL_PROVIDER", "openai").lower()

           if self.provider == "openai":
               endpoint = os.environ["AZURE_OPENAI_ENDPOINT"]
               api_key = os.environ["AZURE_OPENAI_API_KEY"]
               self.deployment = os.environ["AZURE_OPENAI_DEPLOYMENT"]
               api_version = os.getenv(
                   "AZURE_OPENAI_API_VERSION",
                   "2024-02-15-preview",
               )
               self.client = AzureOpenAI(
                   azure_endpoint=endpoint,
                   api_key=api_key,
                   api_version=api_version,
               )

           elif self.provider == "claude":
               self.model = os.getenv(
                   "ANTHROPIC_MODEL",
                   "claude-3-5-haiku-latest",
               )
               self.client = Anthropic(
                   api_key=os.environ["ANTHROPIC_API_KEY"],
               )

           else:
               raise ValueError(
                   "MODEL_PROVIDER debe ser 'openai' o 'claude'."
               )

       def generate(
           self,
           system_prompt: str,
           user_prompt: str,
           max_tokens: int = 1800,
       ) -> str:
           if self.provider == "openai":
               response = self.client.chat.completions.create(
                   model=self.deployment,
                   temperature=0,
                   max_tokens=max_tokens,
                   response_format={"type": "json_object"},
                   messages=[
                       {"role": "system", "content": system_prompt},
                       {"role": "user", "content": user_prompt},
                   ],
               )
               content = response.choices[0].message.content
               if not content:
                   raise RuntimeError(
                       "Azure OpenAI devolvió una respuesta vacía."
                   )
               return content

           response = self.client.messages.create(
               model=self.model,
               max_tokens=max_tokens,
               temperature=0,
               system=system_prompt,
               messages=[
                   {
                       "role": "user",
                       "content": user_prompt,
                   }
               ],
           )

           text_blocks = [
               block.text
               for block in response.content
               if block.type == "text"
           ]
           content = "".join(text_blocks)

           if not content:
               raise RuntimeError("Claude devolvió una respuesta vacía.")

           return content
   PY
   ```

2. Revisa que no existan claves codificadas en el Router:

   ```bash
   grep -nE '(api_key=sk-|ANTHROPIC_API_KEY=|AZURE_OPENAI_API_KEY=)' \
     src/services/model_router.py || true
   ```

3. Verifica la sintaxis:

   ```bash
   python -m py_compile src/services/model_router.py
   ```

**Resultado esperado:**

- El Router selecciona el proveedor usando `MODEL_PROVIDER`.
- Las credenciales se leen exclusivamente desde variables de entorno.
- Azure OpenAI recibe una solicitud con `response_format={"type": "json_object"}`.
- Claude recibe el requisito de JSON mediante el prompt del skill.

**Verificación:**

```bash
python - <<'PY'
from src.services.model_router import ModelRouter

router = ModelRouter()
print(f"Proveedor activo: {router.provider}")
PY
```

Debe mostrarse `Proveedor activo: openai` o `Proveedor activo: claude`.

---

### Paso 4. Crear muestras de código inseguro y casos del Harness

**Objetivo:** preparar un conjunto reproducible de entradas con vulnerabilidades y problemas de calidad conocidos.

**Instrucciones:**

1. Crea una muestra con inyección SQL por concatenación:

   ```bash
   cat > data/code_samples/sql_injection.py <<'PY'
   import sqlite3


   def find_user(username: str):
       connection = sqlite3.connect("users.db")
       cursor = connection.cursor()
       query = "SELECT id, username FROM users WHERE username = '" + username + "'"
       cursor.execute(query)
       return cursor.fetchone()
   PY
   ```

2. Crea una muestra con secretos codificados:

   ```bash
   cat > data/code_samples/hardcoded_secret.py <<'PY'
   API_KEY = "sk_example_training_secret_do_not_use"
   DATABASE_PASSWORD = "training-password-123"


   def get_headers():
       return {"Authorization": f"Bearer {API_KEY}"}
   PY
   ```

3. Crea una muestra con uso inseguro de `eval`:

   ```bash
   cat > data/code_samples/unsafe_eval.py <<'PY'
   def calculate(expression: str):
       return eval(expression)
   PY
   ```

4. Crea una muestra con manejo deficiente de excepciones:

   ```bash
   cat > data/code_samples/weak_exceptions.py <<'PY'
   import json


   def load_configuration(raw_value: str) -> dict:
       try:
           return json.loads(raw_value)
       except Exception:
           return {}
   PY
   ```

5. Crea una muestra con código duplicado:

   ```bash
   cat > data/code_samples/duplicated_code.py <<'PY'
   def format_customer(name: str, email: str) -> dict:
       clean_name = name.strip().title()
       clean_email = email.strip().lower()
       return {"name": clean_name, "email": clean_email}


   def format_supplier(name: str, email: str) -> dict:
       clean_name = name.strip().title()
       clean_email = email.strip().lower()
       return {"name": clean_name, "email": clean_email}
   PY
   ```

6. Crea el archivo `data/harness/code_review_cases.json`:

   ```bash
   cat > data/harness/code_review_cases.json <<'JSON'
   [
     {
       "id": "sql-injection",
       "file": "data/code_samples/sql_injection.py",
       "language": "python",
       "repository_context": "Módulo de acceso a usuarios con SQLite.",
       "review_focus": ["security", "reliability"],
       "expected_categories": ["security"],
       "minimum_severity": "high"
     },
     {
       "id": "hardcoded-secret",
       "file": "data/code_samples/hardcoded_secret.py",
       "language": "python",
       "repository_context": "Módulo de integración que construye cabeceras HTTP.",
       "review_focus": ["security"],
       "expected_categories": ["security"],
       "minimum_severity": "high"
     },
     {
       "id": "unsafe-eval",
       "file": "data/code_samples/unsafe_eval.py",
       "language": "python",
       "repository_context": "Función que calcula expresiones proporcionadas por un usuario.",
       "review_focus": ["security", "reliability"],
       "expected_categories": ["security"],
       "minimum_severity": "high"
     },
     {
       "id": "weak-exceptions",
       "file": "data/code_samples/weak_exceptions.py",
       "language": "python",
       "repository_context": "Carga de configuración JSON.",
       "review_focus": ["reliability"],
       "expected_categories": ["reliability"],
       "minimum_severity": "medium"
     },
     {
       "id": "duplicated-code",
       "file": "data/code_samples/duplicated_code.py",
       "language": "python",
       "repository_context": "Funciones de normalización de entidades de negocio.",
       "review_focus": ["maintainability"],
       "expected_categories": ["maintainability"],
       "minimum_severity": "low"
     }
   ]
   JSON
   ```

**Resultado esperado:**

```text
data/code_samples/
├── duplicated_code.py
├── hardcoded_secret.py
├── sql_injection.py
├── unsafe_eval.py
└── weak_exceptions.py

data/harness/
└── code_review_cases.json
```

**Verificación:**

```bash
python -m json.tool data/harness/code_review_cases.json > /dev/null
find data/code_samples -type f -name '*.py' -print | sort
```

El primer comando debe finalizar sin errores y el segundo debe listar las cinco muestras.

---

### Paso 5. Implementar el Agent Skill de revisión de código

**Objetivo:** construir el skill que genera una solicitud segura, analiza la respuesta JSON del modelo y valida su estructura.

**Instrucciones:**

1. Crea `src/skills/code_review_skill.py`:

   ```bash
   cat > src/skills/code_review_skill.py <<'PY'
   import json

   from pydantic import ValidationError

   from src.schemas.code_review import CodeReviewOutput, CodeReviewRequest
   from src.services.model_router import ModelRouter


   SYSTEM_PROMPT = """
   Eres un revisor senior de código y seguridad de aplicaciones.

   Analiza exclusivamente el código y el contexto proporcionados. No ejecutes
   código, no simules ejecuciones, no llames herramientas externas y no inventes
   dependencias, configuraciones, rutas, permisos ni vulnerabilidades que no
   estén respaldadas por evidencia visible.

   Identifica oportunidades reales de mejora en seguridad, confiabilidad,
   mantenibilidad, rendimiento o estilo. Para toda afirmación que dependa de
   información no disponible, marca uncertainty como true y explica el límite
   en evidence.

   Devuelve exclusivamente un objeto JSON válido sin Markdown, sin bloques de
   código y sin texto antes o después del JSON.

   El JSON debe cumplir exactamente esta estructura:
   {
     "summary": "resumen breve en español",
     "findings": [
       {
         "severity": "critical|high|medium|low|info",
         "category": "security|reliability|maintainability|performance|style|other",
         "title": "título breve",
         "location": "línea aproximada, función o bloque",
         "evidence": "evidencia presente en el código recibido",
         "recommendation": "recomendación accionable",
         "refactored_snippet": "código opcional o null",
         "uncertainty": false
       }
     ]
   }

   Usa severidad high o critical para vulnerabilidades conocidas que permitan
   inyección, ejecución arbitraria o exposición de secretos. No declares una
   vulnerabilidad como confirmada si solo es una posibilidad no verificable.
   """.strip()


   class CodeReviewSkill:
       def __init__(self, router: ModelRouter | None = None) -> None:
           self.router = router or ModelRouter()

       def review(self, request: CodeReviewRequest) -> CodeReviewOutput:
           user_prompt = f"""
   Lenguaje: {request.language}

   Contexto del repositorio:
   {request.repository_context or "No proporcionado."}

   Focos de revisión:
   {", ".join(request.review_focus)}

   Código a revisar:
   ```{request.language}
   {request.code}
   ```
   """.strip()

           raw_response = self.router.generate(
               system_prompt=SYSTEM_PROMPT,
               user_prompt=user_prompt,
               max_tokens=1800,
           )

           parsed_response = self._parse_json(raw_response)

           try:
               return CodeReviewOutput.model_validate(parsed_response)
           except ValidationError as error:
               raise ValueError(
                   "La respuesta del modelo no cumple el contrato "
                   "CodeReviewOutput."
               ) from error

       @staticmethod
       def _parse_json(raw_response: str) -> dict:
           cleaned = raw_response.strip()

           if cleaned.startswith("```"):
               cleaned = cleaned.split("\n", maxsplit=1)[1]
               cleaned = cleaned.rsplit("```", maxsplit=1)[0].strip()

           start = cleaned.find("{")
           end = cleaned.rfind("}")

           if start == -1 or end == -1 or end < start:
               raise ValueError(
                   "El modelo no devolvió un objeto JSON reconocible."
               )

           try:
               return json.loads(cleaned[start : end + 1])
           except json.JSONDecodeError as error:
               raise ValueError(
                   "El modelo devolvió JSON inválido para la revisión."
               ) from error
   PY
   ```

2. Compila el archivo:

   ```bash
   python -m py_compile src/skills/code_review_skill.py
   ```

3. Realiza una prueba directa del skill con la muestra de `eval`:

   ```bash
   python - <<'PY'
   from pathlib import Path

   from src.schemas.code_review import CodeReviewRequest
   from src.skills.code_review_skill import CodeReviewSkill

   request = CodeReviewRequest(
       language="python",
       code=Path("data/code_samples/unsafe_eval.py").read_text(),
       repository_context="Función que procesa expresiones controladas por usuarios.",
       review_focus=["security", "reliability"],
   )

   result = CodeReviewSkill().review(request)
   print(result.model_dump_json(indent=2))
   PY
   ```

**Resultado esperado:**

La respuesta debe ser JSON válido y debe contener, como mínimo, un hallazgo de categoría `security`. Normalmente incluirá evidencia relacionada con `eval(expression)` y una recomendación de sustituirlo por una alternativa restringida, como un parser controlado o una lista permitida de operaciones.

**Verificación:**

Comprueba que la salida incluya valores similares a los siguientes:

```json
{
  "severity": "high",
  "category": "security",
  "location": "función calculate",
  "uncertainty": false
}
```

La redacción exacta puede variar entre modelos, pero la categoría y severidad deben cumplir el contrato.

---

### Paso 6. Exponer el endpoint REST del skill

**Objetivo:** integrar el skill en FastAPI mediante `POST /v1/skills/code-review`.

**Instrucciones:**

1. Crea o actualiza `src/main.py` con el endpoint. Si tu proyecto ya contiene otros endpoints, agrega únicamente las partes necesarias sin eliminarlos.

   ```bash
   cat > src/main.py <<'PY'
   import logging

   from fastapi import FastAPI, HTTPException, status
   from openai import APIConnectionError, AuthenticationError, RateLimitError

   from src.schemas.code_review import CodeReviewOutput, CodeReviewRequest
   from src.skills.code_review_skill import CodeReviewSkill


   logging.basicConfig(level=logging.INFO)
   logger = logging.getLogger(__name__)

   app = FastAPI(
       title="GenAI Agent Labs",
       version="1.0.0",
   )

   code_review_skill = CodeReviewSkill()


   @app.get("/health")
   def health() -> dict[str, str]:
       return {"status": "ok"}


   @app.post(
       "/v1/skills/code-review",
       response_model=CodeReviewOutput,
       status_code=status.HTTP_200_OK,
   )
   def review_code(request: CodeReviewRequest) -> CodeReviewOutput:
       try:
           return code_review_skill.review(request)

       except AuthenticationError as error:
           logger.error("Error de autenticación con el proveedor: %s", type(error).__name__)
           raise HTTPException(
               status_code=status.HTTP_502_BAD_GATEWAY,
               detail="No fue posible autenticar la solicitud con el proveedor.",
           ) from error

       except RateLimitError as error:
           logger.warning("Límite de cuota del proveedor: %s", type(error).__name__)
           raise HTTPException(
               status_code=status.HTTP_429_TOO_MANY_REQUESTS,
               detail="El proveedor alcanzó un límite temporal de cuota.",
           ) from error

       except APIConnectionError as error:
           logger.error("Error de conexión con el proveedor: %s", type(error).__name__)
           raise HTTPException(
               status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
               detail="No fue posible conectar con el proveedor de modelos.",
           ) from error

       except ValueError as error:
           logger.warning("Respuesta no válida del modelo: %s", str(error))
           raise HTTPException(
               status_code=status.HTTP_502_BAD_GATEWAY,
               detail="El proveedor devolvió una respuesta no estructurada.",
           ) from error

       except Exception as error:
           logger.exception("Error no controlado durante la revisión")
           raise HTTPException(
               status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
               detail="Error interno durante la revisión de código.",
           ) from error
   PY
   ```

2. Inicia FastAPI en el host y puerto obligatorios:

   ```bash
   uvicorn src.main:app --host 127.0.0.1 --port 8000
   ```

3. En una segunda terminal, activa el entorno virtual y valida el estado:

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate

   curl -s http://127.0.0.1:8000/health | python -m json.tool
   ```

4. Envía una solicitud REST de revisión usando la muestra de SQL:

   ```bash
   python - <<'PY' > /tmp/code_review_request.json
   import json
   from pathlib import Path

   payload = {
       "language": "python",
       "code": Path("data/code_samples/sql_injection.py").read_text(),
       "repository_context": "Módulo de consulta de usuarios sobre SQLite.",
       "review_focus": ["security", "reliability"],
   }

   print(json.dumps(payload))
   PY

   curl -sS \
     -X POST http://127.0.0.1:8000/v1/skills/code-review \
     -H "Content-Type: application/json" \
     --data @/tmp/code_review_request.json \
     | python -m json.tool
   ```

**Resultado esperado:**

La solicitud debe devolver `HTTP 200` y una estructura similar a:

```json
{
  "summary": "Se identificó una consulta SQL construida por concatenación...",
  "findings": [
    {
      "severity": "high",
      "category": "security",
      "title": "Posible inyección SQL",
      "location": "función find_user, construcción de query",
      "evidence": "La variable username se concatena directamente en la consulta SQL.",
      "recommendation": "Usar consultas parametrizadas...",
      "refactored_snippet": "...",
      "uncertainty": false
    }
  ]
}
```

**Verificación:**

```bash
curl -s http://127.0.0.1:8000/openapi.json \
  | python -c "import json,sys; d=json.load(sys.stdin); print('/v1/skills/code-review' in d['paths'])"
```

La salida debe ser:

```text
True
```

---

### Paso 7. Implementar y ejecutar el Harness automatizado

**Objetivo:** validar que el skill identifica categorías esperadas y una severidad mínima en casos de prueba conocidos.

**Instrucciones:**

1. Crea `src/harness/code_review_harness.py`:

   ```bash
   cat > src/harness/code_review_harness.py <<'PY'
   import json
   from datetime import UTC, datetime
   from pathlib import Path

   from src.schemas.code_review import CodeReviewRequest
   from src.skills.code_review_skill import CodeReviewSkill


   SEVERITY_ORDER = {
       "info": 0,
       "low": 1,
       "medium": 2,
       "high": 3,
       "critical": 4,
   }


   def has_minimum_severity(
       findings: list[dict],
       expected_categories: list[str],
       minimum_severity: str,
   ) -> bool:
       expected_level = SEVERITY_ORDER[minimum_severity]

       return any(
           finding["category"] in expected_categories
           and SEVERITY_ORDER[finding["severity"]] >= expected_level
           for finding in findings
       )


   def run_harness() -> dict:
       cases_path = Path("data/harness/code_review_cases.json")
       cases = json.loads(cases_path.read_text(encoding="utf-8"))
       skill = CodeReviewSkill()
       results = []

       for case in cases:
           code = Path(case["file"]).read_text(encoding="utf-8")

           request = CodeReviewRequest(
               language=case["language"],
               code=code,
               repository_context=case["repository_context"],
               review_focus=case["review_focus"],
           )

           output = skill.review(request)
           output_data = output.model_dump()

           actual_categories = sorted(
               {finding["category"] for finding in output_data["findings"]}
           )

           missing_categories = sorted(
               set(case["expected_categories"]) - set(actual_categories)
           )

           severity_ok = has_minimum_severity(
               findings=output_data["findings"],
               expected_categories=case["expected_categories"],
               minimum_severity=case["minimum_severity"],
           )

           passed = not missing_categories and severity_ok

           results.append(
               {
                   "id": case["id"],
                   "passed": passed,
                   "expected_categories": case["expected_categories"],
                   "actual_categories": actual_categories,
                   "minimum_severity": case["minimum_severity"],
                   "missing_categories": missing_categories,
                   "severity_ok": severity_ok,
                   "review": output_data,
               }
           )

       total = len(results)
       passed_count = sum(item["passed"] for item in results)

       report = {
           "generated_at": datetime.now(UTC).isoformat(),
           "total_cases": total,
           "passed_cases": passed_count,
           "failed_cases": total - passed_count,
           "results": results,
       }

       report_path = Path("reports/code_review_harness_report.json")
       report_path.write_text(
           json.dumps(report, ensure_ascii=False, indent=2),
           encoding="utf-8",
       )

       return report


   if __name__ == "__main__":
       report = run_harness()
       print(
           f"Harness completado: {report['passed_cases']}/"
           f"{report['total_cases']} casos aprobados."
       )

       if report["failed_cases"]:
           raise SystemExit(1)
   PY
   ```

2. Ejecuta el Harness:

   ```bash
   python -m src.harness.code_review_harness
   ```

3. Consulta el resumen del reporte:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   report = json.loads(
       Path("reports/code_review_harness_report.json").read_text()
   )

   print(f"Casos aprobados: {report['passed_cases']}/{report['total_cases']}")

   for result in report["results"]:
       print(
           f"- {result['id']}: "
           f"{'APROBADO' if result['passed'] else 'FALLIDO'}; "
           f"categorías={result['actual_categories']}"
       )
   PY
   ```

**Resultado esperado:**

El Harness debe aprobar los cinco casos o, como mínimo, identificar claramente los casos no aprobados. Una salida satisfactoria tiene esta forma:

```text
Harness completado: 5/5 casos aprobados.
Casos aprobados: 5/5
- sql-injection: APROBADO; categorías=['security']
- hardcoded-secret: APROBADO; categorías=['security']
- unsafe-eval: APROBADO; categorías=['security']
- weak-exceptions: APROBADO; categorías=['reliability']
- duplicated-code: APROBADO; categorías=['maintainability']
```

**Verificación:**

```bash
python -m json.tool reports/code_review_harness_report.json > /dev/null
```

Además, inspecciona un resultado individual:

```bash
python - <<'PY'
import json
from pathlib import Path

report = json.loads(
    Path("reports/code_review_harness_report.json").read_text()
)

case = next(item for item in report["results"] if item["id"] == "sql-injection")
print(json.dumps(case, ensure_ascii=False, indent=2))
PY
```

El caso `sql-injection` debe incluir `security` y una severidad `high` o `critical`.

> **Nota sobre variabilidad:** un LLM no es una regla determinista. Si un caso falla por una clasificación demasiado conservadora, revisa el prompt y la evidencia producida antes de cambiar artificialmente las expectativas del Harness. El objetivo es mejorar el contrato y la instrucción, no ocultar resultados inconsistentes.

## Validación y pruebas

### Ejecutar Ruff 0.2.1

Ejecuta Ruff sobre el código fuente del proyecto:

```bash
ruff check src tests
```

Si deseas revisar también las muestras intencionalmente inseguras:

```bash
ruff check src tests data/code_samples
```

Ruff puede informar problemas como imports no utilizados, formato, variables no usadas o construcciones detectables por reglas estáticas. Sin embargo, no debe esperarse que encuentre todos los problemas semánticos del conjunto de muestras.

Para almacenar el resultado sin interrumpir el laboratorio por código de salida distinto de cero:

```bash
ruff check src tests data/code_samples \
  > reports/ruff_code_review.txt 2>&1 || true

cat reports/ruff_code_review.txt
```

### Comparación entre Ruff y el LLM

| Aspecto | Ruff | Skill con LLM |
|---|---|---|
| Naturaleza | Análisis estático determinista basado en reglas | Análisis probabilístico y contextual |
| Repetibilidad | Alta: misma entrada y versión producen el mismo resultado | Puede variar entre ejecuciones o modelos |
| Secretos codificados | Depende de reglas y configuración instaladas | Puede detectarlos por contexto y patrones visibles |
| Inyección SQL | Puede requerir reglas especializadas | Puede inferir el riesgo al observar concatenación de entradas |
| Código duplicado | Cobertura limitada en Ruff básico | Puede proponer abstracciones y refactorizaciones |
| Dependencias inexistentes | No debe inventarlas | El prompt obliga al LLM a no inventarlas |
| Sustitución de herramientas SAST | No aplica | No: complementa, no reemplaza herramientas deterministas |

### Criterios de aceptación

Considera la práctica completada cuando se cumplan todos estos criterios:

- [ ] `POST /v1/skills/code-review` responde en `http://127.0.0.1:8000`.
- [ ] La respuesta cumple el esquema `CodeReviewOutput`.
- [ ] El skill usa el Router de Modelos y no contiene claves en el código.
- [ ] El Harness procesa `data/harness/code_review_cases.json`.
- [ ] Los casos de SQL, secretos y `eval` reciben categoría `security` y severidad mínima `high`.
- [ ] El caso de excepciones recibe categoría `reliability` y severidad mínima `medium`.
- [ ] El caso de duplicación recibe categoría `maintainability` y severidad mínima `low`.
- [ ] Ruff fue ejecutado y su resultado fue comparado conceptualmente con el análisis del LLM.
- [ ] No se versionó `.env`, claves, archivos temporales ni secretos reales.

### Prueba mínima automatizada del contrato

Crea `tests/test_code_review_schema.py` para validar el contrato sin consumir ninguna API:

```bash
cat > tests/test_code_review_schema.py <<'PY'
from src.schemas.code_review import CodeReviewOutput


def test_code_review_output_accepts_valid_finding():
    output = CodeReviewOutput.model_validate(
        {
            "summary": "Se encontró una vulnerabilidad de seguridad.",
            "findings": [
                {
                    "severity": "high",
                    "category": "security",
                    "title": "Inyección SQL",
                    "location": "función find_user",
                    "evidence": "La entrada se concatena en una consulta.",
                    "recommendation": "Usar parámetros SQL.",
                    "refactored_snippet": None,
                    "uncertainty": False,
                }
            ],
        }
    )

    assert output.findings[0].category == "security"
    assert output.findings[0].severity == "high"
PY
```

Ejecuta la prueba:

```bash
python -m pytest -q tests/test_code_review_schema.py
```

Si `pytest` no está instalado:

```bash
pip install pytest
python -m pytest -q tests/test_code_review_schema.py
```

## Solución de problemas

### Problema 1: el endpoint devuelve `502` con el mensaje “El proveedor devolvió una respuesta no estructurada”

**Síntoma:**

```json
{
  "detail": "El proveedor devolvió una respuesta no estructurada."
}
```

o el log indica que el JSON no cumple `CodeReviewOutput`.

**Causa probable:** el modelo devolvió texto adicional, Markdown, una categoría no permitida o un JSON incompleto. Esto puede ocurrir especialmente si se cambia el prompt, se usa un modelo distinto o se incrementa demasiado la temperatura.

**Solución:**

1. Verifica que `temperature=0` se mantenga en el Router.
2. Confirma que `SYSTEM_PROMPT` exige “exclusivamente un objeto JSON válido”.
3. No elimines la validación `CodeReviewOutput.model_validate(...)`.
4. Inspecciona la respuesta bruta temporalmente en un entorno de desarrollo, sin registrar código sensible.
5. Si usas Azure OpenAI, confirma que permanece:

   ```python
   response_format={"type": "json_object"}
   ```

6. Reintenta la solicitud después de corregir el prompt o la configuración.

### Problema 2: el Harness falla en uno o más casos aunque el código contiene una vulnerabilidad evidente

**Síntoma:**

```text
Harness completado: 3/5 casos aprobados.
```

o un caso muestra `missing_categories: ["security"]`.

**Causa probable:** el modelo clasificó el hallazgo con una categoría distinta, asignó una severidad inferior a la requerida o interpretó insuficiente el contexto entregado.

**Solución:**

1. Abre el reporte para revisar la evidencia real devuelta:

   ```bash
   python -m json.tool reports/code_review_harness_report.json | less
   ```

2. Verifica que `review_focus` incluya la categoría esperada.
3. Mejora el `repository_context` del caso para aclarar si la entrada puede ser controlada por usuarios.
4. Revisa que el prompt mantenga la regla de usar severidad `high` o `critical` para inyección, ejecución arbitraria y exposición de secretos.
5. No reduzcas la severidad mínima sin justificarlo técnicamente; primero corrige el contrato, el contexto o la instrucción.
6. Ejecuta de nuevo el Harness y compara el nuevo reporte.

## Limpieza

1. Detén FastAPI en la terminal donde ejecutaste Uvicorn:

   ```text
   Ctrl+C
   ```

2. Elimina únicamente archivos temporales creados fuera del repositorio:

   ```bash
   rm -f /tmp/code_review_request.json
   ```

3. Conserva el reporte local si lo necesitas como evidencia, pero no lo versiones si contiene análisis de código sensible:

   ```bash
   git status --short
   ```

4. Comprueba nuevamente que no hay secretos ni `.env` en el área de preparación:

   ```bash
   git status --short
   git check-ignore -v .env
   ```

5. Añade los archivos de código, muestras, casos y pruebas requeridos:

   ```bash
   git add \
     .gitignore \
     src \
     data/code_samples \
     data/harness \
     tests
   ```

6. Revisa el contenido preparado antes de confirmar:

   ```bash
   git diff --cached --check
   git diff --cached
   ```

7. Realiza el commit obligatorio de la práctica:

   ```bash
   git commit -m "lab-02-00-03"
   ```

## Resumen

En esta práctica implementaste una capacidad de revisión de código asistida por LLM como un Agent Skill reutilizable. El flujo recibe código y contexto, selecciona Azure OpenAI o Claude mediante un Router, exige una respuesta JSON validada por Pydantic y expone el resultado mediante un endpoint REST.

También creaste un Harness que comprueba automáticamente categorías y severidades mínimas para casos conocidos de inyección SQL, secretos codificados, `eval` inseguro, excepciones amplias y duplicación. Ruff se utilizó como contraste determinista: sus reglas estáticas aportan consistencia y repetibilidad, mientras que el LLM añade razonamiento contextual, explicación, priorización y sugerencias de refactorización. Ambos mecanismos deben utilizarse de forma complementaria dentro de un proceso de revisión supervisado.

### Recursos opcionales

- [Documentación de Azure OpenAI](https://learn.microsoft.com/azure/ai-services/openai/)
- [Documentación de OpenAI Python SDK](https://github.com/openai/openai-python)
- [Documentación de Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Documentación de Ruff](https://docs.astral.sh/ruff/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

# 2. Práctica 4. Integrar los Agent Skills desarrollados en el módulo anterior dentro del flujo de revisión e implementar un Harness de evaluación para validar automáticamente la calidad, seguridad y consistencia de las respuestas antes de incorporarlas al proceso de desarrollo.

## Metadatos

| Campo | Valor |
|---|---|
| Duración | 40 minutos |
| Complejidad | Alta |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica integrarás los Agent Skills de análisis de requisitos, generación de código y revisión de código en un flujo único y controlado. Implementarás un *harness* de evaluación que combina validaciones deterministas y evaluación opcional mediante LLM-as-a-Judge para decidir si una propuesta generada puede aprobarse, debe rechazarse o requiere revisión humana.

El flujo persistirá sus resultados en formato JSON Lines (`JSONL`) dentro de `data/evaluations/evaluation_results.jsonl`. La implementación evitará exponer claves de API en el código, en archivos versionados o en las respuestas generadas.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Integrar `RequirementAnalysisSkill`, `CodeGenerationSkill` y `CodeReviewSkill` mediante contratos compartidos de Pydantic.
- [ ] Construir un orquestador que procese una tarea de desarrollo desde el análisis hasta la evaluación final.
- [ ] Implementar validaciones deterministas para esquema, secretos, patrones inseguros, cobertura y consistencia.
- [ ] Incorporar un evaluador LLM-as-a-Judge desacoplado del proveedor mediante variables de entorno.
- [ ] Clasificar resultados con las decisiones `PASS`, `FAIL` o `HUMAN_REVIEW` y persistir evidencia de evaluación.

## Requisitos previos

### Conocimientos

- Python intermedio: módulos, clases, excepciones, pruebas con `pytest` y tipado básico.
- Uso de `venv`, `pip`, Git y archivos `.env`.
- Conocimientos básicos de FastAPI y consumo de APIs REST.
- Agent Skills de análisis de requisitos, generación de código y revisión implementados en el módulo anterior.
- Conceptos de seguridad básica: secretos, validación de entradas, patrones peligrosos y revisión humana.

### Acceso y configuración

- Repositorio local `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Python 3.12.3 o compatible.
- Una clave de OpenAI configurada opcionalmente para ejecutar el juez LLM.
- Despliegue o modelo compatible configurado en `OPENAI_MODEL`. Para API pública puede utilizarse un modelo autorizado por la cuenta, por ejemplo `gpt-4.1-mini`.
- Puerto local `8000` disponible para FastAPI.
- Permisos de escritura en `~/genai-agent-labs/data/evaluations`.

> **Importante:** esta práctica no requiere Docker ni una base de datos. La persistencia se realiza exclusivamente en archivos JSONL. No crees ni uses una base de datos llamada `genai_agents_db`.

## Entorno del laboratorio

### Recursos recomendados

| Recurso | Mínimo | Recomendado |
|---|---:|---:|
| CPU | 4 núcleos | 4 núcleos o superior |
| Memoria RAM | 16 GB | 32 GB |
| Espacio libre SSD | 10 GB | 20 GB |
| Sistema operativo de referencia | Ubuntu 22.04.4 LTS | Ubuntu 22.04.4 LTS |

### Dependencias de Python

| Paquete | Versión objetivo | Uso |
|---|---:|---|
| `openai` | 1.35.13 | Evaluador LLM opcional |
| `anthropic` | 0.28.0 | Extensión opcional para otro proveedor |
| `pydantic` | 2.7.4 | Contratos, esquemas y validaciones |
| `fastapi` | 0.109.2 o compatible | API REST local |
| `uvicorn` | 0.27.1 o compatible | Servidor ASGI |
| `python-dotenv` | 1.0.1 | Carga local de variables |
| `pytest` | 8.2.2 | Pruebas automatizadas |

### Preparación inicial

Abre una terminal y ejecuta los siguientes comandos:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

python --version
git status
```

Instala o actualiza las dependencias requeridas:

```bash
pip install \
  "openai==1.35.13" \
  "anthropic==0.28.0" \
  "pydantic==2.7.4" \
  "fastapi==0.109.2" \
  "uvicorn==0.27.1" \
  "python-dotenv==1.0.1" \
  "pytest==8.2.2"
```

Crea la estructura de directorios del laboratorio:

```bash
mkdir -p src/tech_support_agent
mkdir -p src/tech_support_agent/skills
mkdir -p src/tech_support_agent/evaluation
mkdir -p tests
mkdir -p data/evaluations
mkdir -p config prompts reports
touch src/tech_support_agent/__init__.py
touch src/tech_support_agent/skills/__init__.py
touch src/tech_support_agent/evaluation/__init__.py
```

## Procedimiento paso a paso

### Paso 1. Preparar la configuración segura del proyecto

**Objetivo:** definir variables de entorno para proveedores y comportamiento del harness sin versionar secretos.

**Instrucciones:**

1. Verifica que `.env` esté excluido de Git:

   ```bash
   cd ~/genai-agent-labs

   grep -qxF ".env" .gitignore || echo ".env" >> .gitignore
   grep -qxF ".venv/" .gitignore || echo ".venv/" >> .gitignore
   grep -qxF "__pycache__/" .gitignore || echo "__pycache__/" >> .gitignore
   grep -qxF "data/evaluations/evaluation_results.jsonl" .gitignore || \
     echo "data/evaluations/evaluation_results.jsonl" >> .gitignore
   ```

2. Crea el archivo de ejemplo versionable:

   ```bash
   cat > .env.example <<'EOF'
   # API de OpenAI: use una clave real solo en el archivo .env local.
   OPENAI_API_KEY=
   OPENAI_MODEL=gpt-4.1-mini

   # Activa el juez LLM solo si dispone de credenciales y cuota.
   ENABLE_LLM_JUDGE=false

   # Umbrales del harness.
   MIN_QUALITY_SCORE=0.70
   MIN_REQUIREMENT_COVERAGE=0.80
   MIN_CONSISTENCY_SCORE=0.80
   EOF
   ```

3. Crea tu configuración local a partir del ejemplo:

   ```bash
   cp -n .env.example .env
   ```

4. Si dispones de una clave de OpenAI, edita exclusivamente `.env`:

   ```bash
   nano .env
   ```

   Establece los valores necesarios:

   ```text
   OPENAI_API_KEY=tu_clave_local_no_versionada
   OPENAI_MODEL=gpt-4.1-mini
   ENABLE_LLM_JUDGE=true
   ```

5. Si no dispones de clave o no deseas generar consumo durante las pruebas, conserva:

   ```text
   ENABLE_LLM_JUDGE=false
   ```

   En ese caso, el sistema realizará todas las evaluaciones deterministas y clasificará la falta del juicio LLM como `HUMAN_REVIEW` cuando el flujo real lo requiera. Las pruebas automatizadas usarán un juez controlado que no llama a servicios externos.

**Resultado esperado:**

- Existe `.env.example` sin claves reales.
- Existe opcionalmente `.env` local.
- `.env` no aparece como archivo preparado para commit.

**Verificación:**

```bash
git check-ignore -v .env
git status --short
```

Debes observar una regla de `.gitignore` para `.env`. El archivo `.env.example` sí debe poder versionarse.

---

### Paso 2. Definir contratos compartidos con Pydantic

**Objetivo:** crear esquemas explícitos para que los Skills y el harness intercambien datos verificables, serializables y consistentes.

**Instrucciones:**

1. Crea `src/tech_support_agent/contracts.py`:

   ```bash
   cat > src/tech_support_agent/contracts.py <<'PY'
   from __future__ import annotations

   from enum import StrEnum
   from typing import Literal, Protocol

   from pydantic import BaseModel, Field


   class Decision(StrEnum):
       PASS = "PASS"
       FAIL = "FAIL"
       HUMAN_REVIEW = "HUMAN_REVIEW"


   class Severity(StrEnum):
       INFO = "INFO"
       WARNING = "WARNING"
       CRITICAL = "CRITICAL"


   class DevelopmentTask(BaseModel):
       task_id: str = Field(min_length=3, max_length=80)
       title: str = Field(min_length=5, max_length=200)
       description: str = Field(min_length=10)
       requirements: list[str] = Field(min_length=1)
       language: Literal["python"] = "python"


   class RequirementAnalysis(BaseModel):
       task_id: str
       normalized_requirements: list[str] = Field(min_length=1)
       acceptance_criteria: list[str] = Field(min_length=1)
       assumptions: list[str] = Field(default_factory=list)
       risks: list[str] = Field(default_factory=list)


   class GeneratedFile(BaseModel):
       path: str = Field(
           pattern=r"^[a-zA-Z0-9_./-]+\.py$",
           description="Ruta relativa de un archivo Python."
       )
       content: str = Field(min_length=1)


   class CodeProposal(BaseModel):
       task_id: str
       summary: str = Field(min_length=10)
       files: list[GeneratedFile] = Field(min_length=1)
       tests: list[GeneratedFile] = Field(default_factory=list)
       requirement_mapping: dict[str, str] = Field(default_factory=dict)


   class ReviewFinding(BaseModel):
       severity: Severity
       category: str
       message: str
       file_path: str | None = None


   class CodeReview(BaseModel):
       task_id: str
       approved: bool
       findings: list[ReviewFinding] = Field(default_factory=list)
       summary: str = Field(min_length=5)


   class JudgeVerdict(BaseModel):
       quality_score: float = Field(ge=0.0, le=1.0)
       requirement_coverage: float = Field(ge=0.0, le=1.0)
       consistency_score: float = Field(ge=0.0, le=1.0)
       rationale: str = Field(min_length=5)
       requires_human_review: bool = False


   class EvaluationFinding(BaseModel):
       evaluator: str
       severity: Severity
       code: str
       message: str


   class EvaluationResult(BaseModel):
       task_id: str
       decision: Decision
       findings: list[EvaluationFinding]
       deterministic_passed: bool
       llm_judge_available: bool
       judge_verdict: JudgeVerdict | None = None


   class RequirementAnalysisSkill(Protocol):
       def analyze(self, task: DevelopmentTask) -> RequirementAnalysis:
           """Analiza y normaliza requisitos de una tarea."""


   class CodeGenerationSkill(Protocol):
       def generate(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
       ) -> CodeProposal:
           """Genera una propuesta de código trazable a requisitos."""


   class CodeReviewSkill(Protocol):
       def review(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
           proposal: CodeProposal,
       ) -> CodeReview:
           """Revisa seguridad, calidad y coherencia de la propuesta."""
   PY
   ```

2. Observa los contratos principales:

   - `DevelopmentTask` representa la solicitud de desarrollo.
   - `RequirementAnalysis` normaliza requisitos y criterios de aceptación.
   - `CodeProposal` contiene archivos generados y trazabilidad.
   - `CodeReview` representa la revisión producida por el Skill existente.
   - `EvaluationResult` representa la decisión del harness.
   - Los `Protocol` permiten integrar implementaciones existentes sin obligarlas a heredar de una clase concreta.

3. Verifica que el módulo compile:

   ```bash
   PYTHONPATH=src python -c "from tech_support_agent.contracts import DevelopmentTask; print('Contratos cargados correctamente')"
   ```

**Resultado esperado:**

```text
Contratos cargados correctamente
```

**Verificación:**

Ejecuta la siguiente validación de esquema:

```bash
PYTHONPATH=src python - <<'PY'
from tech_support_agent.contracts import DevelopmentTask

task = DevelopmentTask(
    task_id="TS-201",
    title="Crear normalizador de prioridades",
    description="Normalizar prioridades recibidas desde tickets de soporte.",
    requirements=["Aceptar valores low, medium y high."],
)
print(task.model_dump_json(indent=2))
PY
```

Debes obtener un objeto JSON válido con el campo `language` igual a `python`.

---

### Paso 3. Adaptar los Agent Skills al contrato común

**Objetivo:** integrar los Skills del módulo anterior mediante adaptadores que respeten los contratos compartidos.

**Instrucciones:**

1. Crea implementaciones de demostración en `src/tech_support_agent/skills/demo_skills.py`.

   > En un repositorio que ya contenga los Skills del módulo anterior, sustituye estas implementaciones de demostración por adaptadores a tus clases existentes. Mantén las firmas `analyze`, `generate` y `review`.

   ```bash
   cat > src/tech_support_agent/skills/demo_skills.py <<'PY'
   from tech_support_agent.contracts import (
       CodeProposal,
       CodeReview,
       DevelopmentTask,
       GeneratedFile,
       RequirementAnalysis,
       ReviewFinding,
       Severity,
   )


   class DemoRequirementAnalysisSkill:
       def analyze(self, task: DevelopmentTask) -> RequirementAnalysis:
           normalized = [
               requirement.strip().lower()
               for requirement in task.requirements
               if requirement.strip()
           ]

           return RequirementAnalysis(
               task_id=task.task_id,
               normalized_requirements=normalized,
               acceptance_criteria=[
                   f"Se implementa: {requirement}"
                   for requirement in normalized
               ],
               assumptions=["La entrada se procesa localmente."],
               risks=["Los valores no reconocidos deben generar un error controlado."],
           )


   class DemoCodeGenerationSkill:
       def generate(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
       ) -> CodeProposal:
           source = '''"""Utilidades de prioridad para tickets de soporte."""

   PRIORITIES = {"low", "medium", "high"}


   def normalize_priority(value: str) -> str:
       """Normaliza y valida una prioridad de ticket."""
       normalized = value.strip().lower()
       if normalized not in PRIORITIES:
           raise ValueError("Unsupported priority")
       return normalized
   '''

           test = '''import pytest

   from tech_support_agent.priority import normalize_priority


   def test_normalize_priority_accepts_uppercase() -> None:
       assert normalize_priority("HIGH") == "high"


   def test_normalize_priority_rejects_unknown_value() -> None:
       with pytest.raises(ValueError):
           normalize_priority("urgent")
   '''

           mapping = {
               requirement: "src/tech_support_agent/priority.py"
               for requirement in analysis.normalized_requirements
           }

           return CodeProposal(
               task_id=task.task_id,
               summary="Implementa normalización y validación de prioridades permitidas.",
               files=[
                   GeneratedFile(
                       path="src/tech_support_agent/priority.py",
                       content=source,
                   )
               ],
               tests=[
                   GeneratedFile(
                       path="tests/test_priority.py",
                       content=test,
                   )
               ],
               requirement_mapping=mapping,
           )


   class DemoCodeReviewSkill:
       def review(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
           proposal: CodeProposal,
       ) -> CodeReview:
           findings: list[ReviewFinding] = []

           if not proposal.tests:
               findings.append(
                   ReviewFinding(
                       severity=Severity.WARNING,
                       category="testing",
                       message="La propuesta no incluye pruebas automatizadas.",
                   )
               )

           return CodeReview(
               task_id=task.task_id,
               approved=not any(
                   finding.severity == Severity.CRITICAL
                   for finding in findings
               ),
               findings=findings,
               summary="Revisión estática inicial completada.",
           )
   PY
   ```

2. El generador de demostración produce una propuesta pequeña y segura. El propósito no es reemplazar un generador real, sino demostrar el contrato que debe respetar cualquier Skill.

3. El resultado del generador incluye `requirement_mapping`. Este campo permite al harness verificar que cada requisito normalizado está vinculado a uno o más archivos.

4. Verifica la integración mínima:

   ```bash
   PYTHONPATH=src python - <<'PY'
   from tech_support_agent.contracts import DevelopmentTask
   from tech_support_agent.skills.demo_skills import (
       DemoCodeGenerationSkill,
       DemoRequirementAnalysisSkill,
       DemoCodeReviewSkill,
   )

   task = DevelopmentTask(
       task_id="TS-201",
       title="Crear normalizador de prioridades",
       description="Normalizar prioridades recibidas desde tickets de soporte.",
       requirements=["Aceptar valores low, medium y high."],
   )

   analysis = DemoRequirementAnalysisSkill().analyze(task)
   proposal = DemoCodeGenerationSkill().generate(task, analysis)
   review = DemoCodeReviewSkill().review(task, analysis, proposal)

   print(analysis.normalized_requirements)
   print(proposal.files[0].path)
   print(review.summary)
   PY
   ```

**Resultado esperado:**

```text
['aceptar valores low, medium y high.']
src/tech_support_agent/priority.py
Revisión estática inicial completada.
```

**Verificación:**

Confirma que no se ha creado ningún archivo de código generado automáticamente en el repositorio. En esta práctica el harness evalúa una propuesta en memoria; su incorporación al árbol de trabajo sigue requiriendo una decisión explícita del equipo de desarrollo.

---

### Paso 4. Implementar evaluadores deterministas de seguridad y consistencia

**Objetivo:** detectar problemas objetivos antes de depender de una evaluación probabilística de un modelo de lenguaje.

**Instrucciones:**

1. Crea `src/tech_support_agent/evaluation/deterministic.py`:

   ```bash
   cat > src/tech_support_agent/evaluation/deterministic.py <<'PY'
   from __future__ import annotations

   import re

   from tech_support_agent.contracts import (
       CodeProposal,
       CodeReview,
       EvaluationFinding,
       RequirementAnalysis,
       Severity,
   )

   SECRET_PATTERNS = {
       "OPENAI_API_KEY": r"sk-[A-Za-z0-9_-]{12,}",
       "ANTHROPIC_API_KEY": r"sk-ant-[A-Za-z0-9_-]{12,}",
       "AZURE_OPENAI_KEY": r"(?i)(api[_-]?key\s*=\s*[\"'][^\"']{12,}[\"'])",
       "GENERIC_TOKEN": r"(?i)(token|secret|password)\s*=\s*[\"'][^\"']{8,}[\"']",
   }

   INSECURE_PATTERNS = {
       "EVAL_USAGE": r"\beval\s*\(",
       "EXEC_USAGE": r"\bexec\s*\(",
       "SHELL_TRUE": r"subprocess\.(run|call|Popen)\(.*shell\s*=\s*True",
       "PICKLE_LOAD": r"\bpickle\.loads?\(",
       "DISABLED_TLS_VERIFY": r"verify\s*=\s*False",
   }


   def all_contents(proposal: CodeProposal) -> str:
       files = [*proposal.files, *proposal.tests]
       return "\n".join(item.content for item in files)


   def evaluate_security(proposal: CodeProposal) -> list[EvaluationFinding]:
       findings: list[EvaluationFinding] = []
       content = all_contents(proposal)

       for code, pattern in SECRET_PATTERNS.items():
           if re.search(pattern, content):
               findings.append(
                   EvaluationFinding(
                       evaluator="deterministic_security",
                       severity=Severity.CRITICAL,
                       code=f"SECRET_{code}",
                       message="La propuesta contiene un posible secreto o credencial.",
                   )
               )

       for code, pattern in INSECURE_PATTERNS.items():
           if re.search(pattern, content, flags=re.DOTALL):
               findings.append(
                   EvaluationFinding(
                       evaluator="deterministic_security",
                       severity=Severity.CRITICAL,
                       code=f"INSECURE_{code}",
                       message=f"Se detectó el patrón inseguro: {code}.",
                   )
               )

       return findings


   def evaluate_coverage(
       analysis: RequirementAnalysis,
       proposal: CodeProposal,
   ) -> list[EvaluationFinding]:
       findings: list[EvaluationFinding] = []

       for requirement in analysis.normalized_requirements:
           mapped_file = proposal.requirement_mapping.get(requirement)
           if not mapped_file:
               findings.append(
                   EvaluationFinding(
                       evaluator="deterministic_coverage",
                       severity=Severity.WARNING,
                       code="MISSING_REQUIREMENT_MAPPING",
                       message=f"El requisito no tiene trazabilidad: {requirement}",
                   )
               )

       return findings


   def evaluate_consistency(
       analysis: RequirementAnalysis,
       proposal: CodeProposal,
       review: CodeReview,
   ) -> list[EvaluationFinding]:
       findings: list[EvaluationFinding] = []

       if analysis.task_id != proposal.task_id or proposal.task_id != review.task_id:
           findings.append(
               EvaluationFinding(
                   evaluator="deterministic_consistency",
                   severity=Severity.CRITICAL,
                   code="TASK_ID_MISMATCH",
                   message="Los artefactos pertenecen a tareas diferentes.",
               )
           )

       if not proposal.tests:
           findings.append(
               EvaluationFinding(
                   evaluator="deterministic_consistency",
                   severity=Severity.WARNING,
                   code="MISSING_TESTS",
                   message="La propuesta no contiene pruebas automatizadas.",
               )
           )

       if not review.approved:
           findings.append(
               EvaluationFinding(
                   evaluator="deterministic_consistency",
                   severity=Severity.WARNING,
                   code="SKILL_REVIEW_NOT_APPROVED",
                   message="El CodeReviewSkill no aprobó la propuesta.",
               )
           )

       for finding in review.findings:
           if finding.severity == Severity.CRITICAL:
               findings.append(
                   EvaluationFinding(
                       evaluator="code_review_skill",
                       severity=Severity.CRITICAL,
                       code="CRITICAL_REVIEW_FINDING",
                       message=finding.message,
                   )
               )

       return findings
   PY
   ```

2. Revisa las reglas implementadas:

   | Evaluación | Ejemplos de detección | Consecuencia |
   |---|---|---|
   | Secretos | `sk-...`, `sk-ant-...`, `password="..."` | `FAIL` |
   | Ejecución insegura | `eval()`, `exec()`, `shell=True` | `FAIL` |
   | Deserialización insegura | `pickle.load()` | `FAIL` |
   | Trazabilidad | Requisito sin archivo asociado | Advertencia |
   | Consistencia | `task_id` distinto | `FAIL` |
   | Calidad básica | Sin pruebas o revisión no aprobada | Revisión humana |

3. Ejecuta una comprobación manual de detección de secretos:

   ```bash
   PYTHONPATH=src python - <<'PY'
   from tech_support_agent.contracts import CodeProposal, GeneratedFile
   from tech_support_agent.evaluation.deterministic import evaluate_security

   proposal = CodeProposal(
       task_id="TS-SEC-01",
       summary="Ejemplo inseguro para validar el detector.",
       files=[
           GeneratedFile(
               path="src/example.py",
               content='OPENAI_API_KEY = "sk-abcdefghijklmnopqrstuv"',
           )
       ],
   )

   for finding in evaluate_security(proposal):
       print(f"{finding.severity}: {finding.code}")
   PY
   ```

**Resultado esperado:**

```text
CRITICAL: SECRET_OPENAI_API_KEY
```

**Verificación:**

La regla detecta el secreto en memoria. No añadas claves reales al archivo fuente ni a las pruebas.

---

### Paso 5. Implementar el evaluador LLM-as-a-Judge desacoplado

**Objetivo:** usar un LLM como evaluador complementario de calidad, cobertura semántica y consistencia, sin sustituir controles deterministas.

**Instrucciones:**

1. Crea `src/tech_support_agent/evaluation/llm_judge.py`:

   ```bash
   cat > src/tech_support_agent/evaluation/llm_judge.py <<'PY'
   from __future__ import annotations

   import json
   import os
   from typing import Protocol

   from dotenv import load_dotenv
   from openai import OpenAI

   from tech_support_agent.contracts import (
       CodeProposal,
       DevelopmentTask,
       JudgeVerdict,
       RequirementAnalysis,
   )


   class Judge(Protocol):
       def evaluate(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
           proposal: CodeProposal,
       ) -> JudgeVerdict:
           """Evalúa semánticamente la propuesta."""


   class OpenAIJudge:
       def __init__(self) -> None:
           load_dotenv()
           api_key = os.environ["OPENAI_API_KEY"]
           self.model = os.environ.get("OPENAI_MODEL", "gpt-4.1-mini")
           self.client = OpenAI(api_key=api_key)

       def evaluate(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
           proposal: CodeProposal,
       ) -> JudgeVerdict:
           payload = {
               "task": task.model_dump(),
               "analysis": analysis.model_dump(),
               "proposal": proposal.model_dump(),
           }

           instructions = """
   Eres un evaluador técnico estricto de propuestas de código Python.
   Evalúa calidad, cobertura de requisitos y consistencia con la solicitud.
   No apruebes secretos, ejecuciones de shell inseguras, eval, exec ni afirmaciones
   no sustentadas por el código. Devuelve exclusivamente JSON válido con:
   quality_score, requirement_coverage, consistency_score, rationale,
   requires_human_review.
   Los tres puntajes deben estar entre 0.0 y 1.0.
   """

           response = self.client.responses.create(
               model=self.model,
               instructions=instructions,
               input=json.dumps(payload, ensure_ascii=False),
               max_output_tokens=350,
           )

           return JudgeVerdict.model_validate_json(response.output_text)


   class StaticJudge:
       """Juez controlado para pruebas locales sin consumo de API."""

       def evaluate(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
           proposal: CodeProposal,
       ) -> JudgeVerdict:
           return JudgeVerdict(
               quality_score=0.90,
               requirement_coverage=0.90,
               consistency_score=0.90,
               rationale="Evaluación controlada para pruebas automatizadas.",
               requires_human_review=False,
           )
   PY
   ```

2. El juez usa `OpenAI()` y toma la clave exclusivamente desde `OPENAI_API_KEY`. La lógica de negocio no conoce los detalles de la API de OpenAI.

3. El resultado del modelo se valida otra vez con `JudgeVerdict.model_validate_json()`. Si el modelo devuelve texto no válido, el resultado no debe considerarse automáticamente aprobado.

4. Este diseño permite crear posteriormente un `AnthropicJudge` que implemente el mismo `Protocol`, utilizando `Anthropic()` y el formato de mensajes de Claude.

**Resultado esperado:**

- El proyecto contiene una interfaz `Judge`.
- `OpenAIJudge` usa variables de entorno.
- `StaticJudge` permite pruebas repetibles sin red ni coste.

**Verificación:**

```bash
PYTHONPATH=src python - <<'PY'
from tech_support_agent.evaluation.llm_judge import StaticJudge
from tech_support_agent.contracts import DevelopmentTask, RequirementAnalysis, CodeProposal, GeneratedFile

task = DevelopmentTask(
    task_id="TS-202",
    title="Validar propuesta con juez controlado",
    description="Comprobar la interfaz de evaluación LLM.",
    requirements=["Generar una función Python documentada."],
)
analysis = RequirementAnalysis(
    task_id="TS-202",
    normalized_requirements=["generar una función python documentada."],
    acceptance_criteria=["Existe una función documentada."],
)
proposal = CodeProposal(
    task_id="TS-202",
    summary="Incluye una función Python documentada.",
    files=[GeneratedFile(path="src/example.py", content='def hello():\n    """Saluda."""\n    return "hola"\n')],
)

print(StaticJudge().evaluate(task, analysis, proposal).model_dump_json(indent=2))
PY
```

---

### Paso 6. Construir el harness, las reglas de decisión y la persistencia JSONL

**Objetivo:** unificar resultados de los evaluadores, tomar una decisión y registrar evidencia auditable.

**Instrucciones:**

1. Crea `src/tech_support_agent/evaluation/harness.py`:

   ```bash
   cat > src/tech_support_agent/evaluation/harness.py <<'PY'
   from __future__ import annotations

   import json
   import os
   from datetime import datetime, timezone
   from pathlib import Path

   from tech_support_agent.contracts import (
       CodeProposal,
       CodeReview,
       Decision,
       DevelopmentTask,
       EvaluationFinding,
       EvaluationResult,
       JudgeVerdict,
       RequirementAnalysis,
       Severity,
   )
   from tech_support_agent.evaluation.deterministic import (
       evaluate_consistency,
       evaluate_coverage,
       evaluate_security,
   )
   from tech_support_agent.evaluation.llm_judge import Judge


   class EvaluationHarness:
       def __init__(
           self,
           judge: Judge | None = None,
           output_path: Path | None = None,
       ) -> None:
           self.judge = judge
           self.output_path = output_path or Path(
               "data/evaluations/evaluation_results.jsonl"
           )
           self.min_quality = float(os.getenv("MIN_QUALITY_SCORE", "0.70"))
           self.min_coverage = float(
               os.getenv("MIN_REQUIREMENT_COVERAGE", "0.80")
           )
           self.min_consistency = float(
               os.getenv("MIN_CONSISTENCY_SCORE", "0.80")
           )

       def evaluate(
           self,
           task: DevelopmentTask,
           analysis: RequirementAnalysis,
           proposal: CodeProposal,
           review: CodeReview,
       ) -> EvaluationResult:
           findings: list[EvaluationFinding] = []
           findings.extend(evaluate_security(proposal))
           findings.extend(evaluate_coverage(analysis, proposal))
           findings.extend(evaluate_consistency(analysis, proposal, review))

           critical = any(
               finding.severity == Severity.CRITICAL
               for finding in findings
           )

           judge_verdict: JudgeVerdict | None = None
           llm_available = self.judge is not None

           if self.judge is not None and not critical:
               try:
                   judge_verdict = self.judge.evaluate(task, analysis, proposal)
               except Exception:
                   llm_available = False
                   findings.append(
                       EvaluationFinding(
                           evaluator="llm_judge",
                           severity=Severity.WARNING,
                           code="LLM_JUDGE_UNAVAILABLE",
                           message=(
                               "No fue posible obtener una evaluación LLM. "
                               "La propuesta requiere revisión humana."
                           ),
                       )
                   )

           deterministic_passed = not critical
           decision = self._decide(
               findings=findings,
               deterministic_passed=deterministic_passed,
               llm_available=llm_available,
               verdict=judge_verdict,
           )

           result = EvaluationResult(
               task_id=task.task_id,
               decision=decision,
               findings=findings,
               deterministic_passed=deterministic_passed,
               llm_judge_available=llm_available,
               judge_verdict=judge_verdict,
           )
           self._persist(result)
           return result

       def _decide(
           self,
           findings: list[EvaluationFinding],
           deterministic_passed: bool,
           llm_available: bool,
           verdict: JudgeVerdict | None,
       ) -> Decision:
           if not deterministic_passed:
               return Decision.FAIL

           if any(
               finding.code == "SKILL_REVIEW_NOT_APPROVED"
               for finding in findings
           ):
               return Decision.HUMAN_REVIEW

           if verdict is None or not llm_available:
               return Decision.HUMAN_REVIEW

           if verdict.requires_human_review:
               return Decision.HUMAN_REVIEW

           if (
               verdict.quality_score < self.min_quality
               or verdict.requirement_coverage < self.min_coverage
               or verdict.consistency_score < self.min_consistency
           ):
               return Decision.HUMAN_REVIEW

           return Decision.PASS

       def _persist(self, result: EvaluationResult) -> None:
           self.output_path.parent.mkdir(parents=True, exist_ok=True)

           record = {
               "timestamp_utc": datetime.now(timezone.utc).isoformat(),
               **result.model_dump(mode="json"),
           }

           with self.output_path.open("a", encoding="utf-8") as file:
               file.write(json.dumps(record, ensure_ascii=False) + "\n")
   PY
   ```

2. Aplica las siguientes reglas de decisión:

   | Condición | Decisión |
   |---|---|
   | Secreto, patrón inseguro o inconsistencia crítica | `FAIL` |
   | El `CodeReviewSkill` no aprueba la propuesta | `HUMAN_REVIEW` |
   | El juez LLM no está disponible | `HUMAN_REVIEW` |
   | El juez pide revisión humana | `HUMAN_REVIEW` |
   | Puntajes por debajo de los umbrales | `HUMAN_REVIEW` |
   | Todo el conjunto de reglas se satisface | `PASS` |

3. La ausencia del juez LLM no debe convertirse en un `PASS`. Esta política evita que un fallo de red, cuota o autenticación reduzca los controles del proceso.

4. La persistencia usa JSONL: cada línea es un objeto JSON completo, lo que facilita auditoría, procesamiento posterior y exportación a herramientas de observabilidad.

**Resultado esperado:**

El harness contiene reglas explícitas y reproducibles. Una propuesta con un secreto siempre se rechaza, incluso si un evaluador LLM la considerara correcta.

**Verificación:**

```bash
PYTHONPATH=src python - <<'PY'
from pathlib import Path

from tech_support_agent.contracts import DevelopmentTask
from tech_support_agent.evaluation.harness import EvaluationHarness
from tech_support_agent.evaluation.llm_judge import StaticJudge
from tech_support_agent.skills.demo_skills import (
    DemoCodeGenerationSkill,
    DemoCodeReviewSkill,
    DemoRequirementAnalysisSkill,
)

task = DevelopmentTask(
    task_id="TS-203",
    title="Crear normalizador de prioridades",
    description="Normalizar prioridades de tickets de soporte técnico.",
    requirements=["Aceptar valores low, medium y high."],
)

analysis = DemoRequirementAnalysisSkill().analyze(task)
proposal = DemoCodeGenerationSkill().generate(task, analysis)
review = DemoCodeReviewSkill().review(task, analysis, proposal)

harness = EvaluationHarness(
    judge=StaticJudge(),
    output_path=Path("data/evaluations/evaluation_results.jsonl"),
)
result = harness.evaluate(task, analysis, proposal, review)

print(result.decision)
PY
```

La salida esperada es:

```text
PASS
```

Consulta la última evidencia persistida:

```bash
tail -n 1 data/evaluations/evaluation_results.jsonl | python -m json.tool
```

---

### Paso 7. Implementar el orquestador y una API REST local

**Objetivo:** exponer el flujo integrado mediante una API local de FastAPI y conservar la separación entre Skills, harness y capa de transporte.

**Instrucciones:**

1. Crea `src/tech_support_agent/orchestrator.py`:

   ```bash
   cat > src/tech_support_agent/orchestrator.py <<'PY'
   from __future__ import annotations

   from tech_support_agent.contracts import (
       CodeGenerationSkill,
       CodeReviewSkill,
       DevelopmentTask,
       EvaluationResult,
       RequirementAnalysisSkill,
   )
   from tech_support_agent.evaluation.harness import EvaluationHarness


   class DevelopmentOrchestrator:
       def __init__(
           self,
           requirement_skill: RequirementAnalysisSkill,
           generation_skill: CodeGenerationSkill,
           review_skill: CodeReviewSkill,
           harness: EvaluationHarness,
       ) -> None:
           self.requirement_skill = requirement_skill
           self.generation_skill = generation_skill
           self.review_skill = review_skill
           self.harness = harness

       def run(self, task: DevelopmentTask) -> EvaluationResult:
           analysis = self.requirement_skill.analyze(task)
           proposal = self.generation_skill.generate(task, analysis)
           review = self.review_skill.review(task, analysis, proposal)

           return self.harness.evaluate(
               task=task,
               analysis=analysis,
               proposal=proposal,
               review=review,
           )
   PY
   ```

2. Crea `src/tech_support_agent/api.py`:

   ```bash
   cat > src/tech_support_agent/api.py <<'PY'
   from __future__ import annotations

   import os

   from dotenv import load_dotenv
   from fastapi import FastAPI, HTTPException

   from tech_support_agent.contracts import DevelopmentTask, EvaluationResult
   from tech_support_agent.evaluation.harness import EvaluationHarness
   from tech_support_agent.evaluation.llm_judge import OpenAIJudge
   from tech_support_agent.orchestrator import DevelopmentOrchestrator
   from tech_support_agent.skills.demo_skills import (
       DemoCodeGenerationSkill,
       DemoCodeReviewSkill,
       DemoRequirementAnalysisSkill,
   )

   load_dotenv()

   app = FastAPI(
       title="tech-support-agent",
       version="0.1.0",
       description="Flujo controlado de generación, revisión y evaluación.",
   )


   def build_orchestrator() -> DevelopmentOrchestrator:
       enable_llm_judge = (
           os.getenv("ENABLE_LLM_JUDGE", "false").lower() == "true"
       )

       judge = None
       if enable_llm_judge:
           try:
               judge = OpenAIJudge()
           except KeyError as error:
               raise HTTPException(
                   status_code=500,
                   detail=(
                       "ENABLE_LLM_JUDGE=true requiere OPENAI_API_KEY "
                       "configurada en el entorno."
                   ),
               ) from error

       return DevelopmentOrchestrator(
           requirement_skill=DemoRequirementAnalysisSkill(),
           generation_skill=DemoCodeGenerationSkill(),
           review_skill=DemoCodeReviewSkill(),
           harness=EvaluationHarness(judge=judge),
       )


   @app.get("/health")
   def health() -> dict[str, str]:
       return {"status": "ok"}


   @app.post("/evaluate", response_model=EvaluationResult)
   def evaluate(task: DevelopmentTask) -> EvaluationResult:
       return build_orchestrator().run(task)
   PY
   ```

3. Inicia el servidor local en el host y puerto obligatorios:

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate
   uvicorn tech_support_agent.api:app --app-dir src --host 127.0.0.1 --port 8000
   ```

4. En una segunda terminal, activa el mismo entorno virtual y consulta el estado:

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate

   curl http://127.0.0.1:8000/health
   ```

5. Ejecuta una evaluación REST. Con `ENABLE_LLM_JUDGE=false`, el resultado esperado será `HUMAN_REVIEW`, porque no se utilizó un juez LLM real:

   ```bash
   curl -X POST http://127.0.0.1:8000/evaluate \
     -H "Content-Type: application/json" \
     -d '{
       "task_id": "TS-204",
       "title": "Crear normalizador de prioridades",
       "description": "Normalizar prioridades de tickets de soporte técnico.",
       "requirements": [
         "Aceptar valores low, medium y high."
       ],
       "language": "python"
     }'
   ```

**Resultado esperado:**

La consulta de estado debe devolver:

```json
{"status":"ok"}
```

La evaluación debe incluir una estructura equivalente a:

```json
{
  "task_id": "TS-204",
  "decision": "HUMAN_REVIEW",
  "deterministic_passed": true,
  "llm_judge_available": false
}
```

**Verificación:**

Abre la documentación interactiva local:

```text
http://127.0.0.1:8000/docs
```

Confirma que existen los endpoints `GET /health` y `POST /evaluate`.

---

### Paso 8. Crear pruebas automatizadas del harness

**Objetivo:** validar automáticamente escenarios de aprobación, rechazo por secretos y escalamiento a revisión humana.

**Instrucciones:**

1. Detén el servidor FastAPI con `Ctrl+C` si continúa ejecutándose.

2. Crea `tests/test_harness.py`:

   ```bash
   cat > tests/test_harness.py <<'PY'
   from pathlib import Path

   from tech_support_agent.contracts import (
       CodeProposal,
       CodeReview,
       DevelopmentTask,
       GeneratedFile,
       RequirementAnalysis,
   )
   from tech_support_agent.evaluation.harness import EvaluationHarness
   from tech_support_agent.evaluation.llm_judge import StaticJudge


   def build_artifacts() -> tuple[
       DevelopmentTask,
       RequirementAnalysis,
       CodeProposal,
       CodeReview,
   ]:
       task = DevelopmentTask(
           task_id="TS-TEST-01",
           title="Crear normalizador de prioridades",
           description="Normalizar prioridades de tickets técnicos.",
           requirements=["Aceptar valores low, medium y high."],
       )
       analysis = RequirementAnalysis(
           task_id=task.task_id,
           normalized_requirements=[
               "aceptar valores low, medium y high."
           ],
           acceptance_criteria=["La función acepta prioridades válidas."],
       )
       proposal = CodeProposal(
           task_id=task.task_id,
           summary="Incluye una función que normaliza prioridades.",
           files=[
               GeneratedFile(
                   path="src/tech_support_agent/priority.py",
                   content=(
                       'def normalize_priority(value: str) -> str:\n'
                       '    """Normaliza una prioridad."""\n'
                       '    return value.strip().lower()\n'
                   ),
               )
           ],
           tests=[
               GeneratedFile(
                   path="tests/test_priority.py",
                   content="def test_placeholder() -> None:\n    assert True\n",
               )
           ],
           requirement_mapping={
               "aceptar valores low, medium y high.": (
                   "src/tech_support_agent/priority.py"
               )
           },
       )
       review = CodeReview(
           task_id=task.task_id,
           approved=True,
           summary="Sin hallazgos críticos.",
       )
       return task, analysis, proposal, review


   def test_pass_with_static_judge(tmp_path: Path) -> None:
       task, analysis, proposal, review = build_artifacts()
       harness = EvaluationHarness(
           judge=StaticJudge(),
           output_path=tmp_path / "results.jsonl",
       )

       result = harness.evaluate(task, analysis, proposal, review)

       assert result.decision == "PASS"
       assert result.deterministic_passed is True


   def test_fail_when_secret_is_detected(tmp_path: Path) -> None:
       task, analysis, proposal, review = build_artifacts()
       proposal.files[0].content += (
           '\nOPENAI_API_KEY = "sk-abcdefghijklmnopqrstuv"\n'
       )

       harness = EvaluationHarness(
           judge=StaticJudge(),
           output_path=tmp_path / "results.jsonl",
       )
       result = harness.evaluate(task, analysis, proposal, review)

       assert result.decision == "FAIL"
       assert any(
           finding.code == "SECRET_OPENAI_API_KEY"
           for finding in result.findings
       )


   def test_human_review_without_llm_judge(tmp_path: Path) -> None:
       task, analysis, proposal, review = build_artifacts()
       harness = EvaluationHarness(
           judge=None,
           output_path=tmp_path / "results.jsonl",
       )

       result = harness.evaluate(task, analysis, proposal, review)

       assert result.decision == "HUMAN_REVIEW"
       assert result.llm_judge_available is False
   PY
   ```

3. Ejecuta las pruebas:

   ```bash
   cd ~/genai-agent-labs
   source .venv/bin/activate
   PYTHONPATH=src pytest -q
   ```

4. Guarda una evidencia de la ejecución:

   ```bash
   PYTHONPATH=src pytest -q | tee reports/lab-02-00-04-pytest.txt
   ```

**Resultado esperado:**

```text
3 passed
```

**Verificación:**

Comprueba el archivo de reporte:

```bash
cat reports/lab-02-00-04-pytest.txt
```

La prueba de secreto debe demostrar que la decisión es `FAIL` aunque se use `StaticJudge`. Esto confirma que los controles deterministas tienen prioridad sobre el juicio del LLM.

## Validación y pruebas

Ejecuta la siguiente lista de validación final desde `~/genai-agent-labs`:

```bash
source .venv/bin/activate

PYTHONPATH=src pytest -q

PYTHONPATH=src python -m compileall -q src

test -f .env.example && echo ".env.example presente"
test -f src/tech_support_agent/contracts.py && echo "Contratos presentes"
test -f src/tech_support_agent/evaluation/harness.py && echo "Harness presente"
test -f tests/test_harness.py && echo "Pruebas presentes"

tail -n 1 data/evaluations/evaluation_results.jsonl | python -m json.tool
```

Verifica específicamente estos criterios:

| Criterio | Evidencia esperada |
|---|---|
| Contratos compartidos | `contracts.py` usa modelos Pydantic y `Protocol` |
| Integración de Skills | El orquestador llama análisis, generación y revisión en orden |
| Seguridad | Se detectan secretos, `eval`, `exec`, `shell=True`, `pickle.load` y TLS deshabilitado |
| Formato | Los contratos Pydantic validan la estructura de entrada y salida |
| Cobertura | Cada requisito normalizado tiene una entrada en `requirement_mapping` |
| Consistencia | Los artefactos comparten el mismo `task_id` |
| Decisiones | Se generan `PASS`, `FAIL` y `HUMAN_REVIEW` en escenarios distintos |
| Persistencia | Se crean líneas JSON en `data/evaluations/evaluation_results.jsonl` |
| Pruebas sin red | `pytest` usa `StaticJudge` y no requiere claves reales |
| API REST | `GET /health` y `POST /evaluate` responden en `http://127.0.0.1:8000` |

Antes de finalizar, revisa que no haya secretos en los archivos versionables:

```bash
git grep -nE 'sk-[A-Za-z0-9_-]{12,}|sk-ant-[A-Za-z0-9_-]{12,}' || true
git status --short
```

El primer comando no debe mostrar claves reales. Es aceptable que las expresiones regulares de detección aparezcan en el código del harness; no deben aparecer credenciales funcionales.

Realiza el commit correspondiente a esta práctica:

```bash
git add \
  .gitignore \
  .env.example \
  src/tech_support_agent \
  tests/test_harness.py \
  reports/lab-02-00-04-pytest.txt

git commit -m "lab-02-00-04"
```

> Si el historial de la tanda exige los mensajes de commit previos definidos por el curso, no los reescribas. Para esta práctica utiliza el identificador del laboratorio como mensaje de commit, salvo que el instructor indique una convención distinta.

## Solución de problemas

### Problema 1: la API devuelve `HUMAN_REVIEW` aunque el código generado parece correcto

**Síntoma:** `POST /evaluate` devuelve `"decision": "HUMAN_REVIEW"` y `"llm_judge_available": false`.

**Causa:** `ENABLE_LLM_JUDGE=false`, falta `OPENAI_API_KEY`, el modelo configurado no está disponible, o el juez LLM produjo una excepción. El harness está diseñado para no aprobar automáticamente cuando el control semántico LLM no está disponible.

**Solución:** para pruebas locales, este resultado es correcto y debe escalarse a una revisión humana. Si deseas usar el juez LLM, configura en `.env` una clave válida y un modelo autorizado:

```text
OPENAI_API_KEY=tu_clave_local
OPENAI_MODEL=gpt-4.1-mini
ENABLE_LLM_JUDGE=true
```

Reinicia Uvicorn después de modificar `.env`. Si el problema persiste, verifica cuota, permisos, conectividad y el nombre de modelo habilitado en tu cuenta.

### Problema 2: `pytest` falla con `ModuleNotFoundError: No module named 'tech_support_agent'`

**Síntoma:** al ejecutar `pytest`, Python no encuentra el paquete ubicado en `src/`.

**Causa:** el directorio `src` no está incluido en la ruta de importación de Python, o el comando se ejecutó fuera de `~/genai-agent-labs`.

**Solución:** ejecuta las pruebas desde la raíz del repositorio e incluye `src` en `PYTHONPATH`:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate
PYTHONPATH=src pytest -q
```

Como alternativa permanente para un proyecto posterior, puede incorporarse un archivo `pyproject.toml` con configuración de paquete editable, pero no es necesario para esta práctica.

## Limpieza

1. Detén el servidor FastAPI si está en ejecución:

   ```bash
   Ctrl+C
   ```

2. Conserva el archivo `.env` únicamente en tu equipo local. No lo agregues al commit:

   ```bash
   git status --short
   ```

3. Si generaste resultados temporales que no deseas conservar localmente, limpia solo el archivo de evaluación:

   ```bash
   rm -f data/evaluations/evaluation_results.jsonl
   ```

4. No elimines:

   - `.env.example`
   - Los contratos Pydantic.
   - El harness y las pruebas.
   - El reporte de pruebas que será evidencia del laboratorio.
   - El entorno compartido `.venv`, salvo que el instructor indique lo contrario.

## Resumen

En esta práctica construiste una base controlada para integrar agentes de desarrollo asistidos por LLM. Los Skills de requisitos, generación y revisión se conectan mediante contratos Pydantic, lo que reduce ambigüedad y facilita la validación estructural.

El harness aplica primero reglas deterministas de seguridad y consistencia, y después usa un juez LLM opcional para evaluar aspectos semánticos menos mecánicos. La decisión final sigue una política conservadora:

- `PASS`: evidencia suficiente, sin problemas críticos y con puntajes aceptables.
- `FAIL`: secreto, patrón inseguro o inconsistencia crítica.
- `HUMAN_REVIEW`: evidencia incompleta, juez no disponible, calidad insuficiente o necesidad explícita de supervisión.

La persistencia JSONL y las pruebas automatizadas proporcionan trazabilidad para el siguiente módulo, donde el agente incorporará recuperación documental como contexto verificable antes de generar o revisar código.

### Recursos opcionales

- [Documentación de OpenAI Python SDK](https://github.com/openai/openai-python)
- [Documentación de Pydantic](https://docs.pydantic.dev/)
- [Documentación de FastAPI](https://fastapi.tiangolo.com/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Guía de seguridad para claves de API de OpenAI](https://help.openai.com/en/articles/5112595-best-practices-for-api-key-safety)
