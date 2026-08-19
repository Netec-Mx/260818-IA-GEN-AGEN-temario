# 1. Práctica 1. Preparar un dataset validado para Fine-Tuning a partir de una base de preguntas y respuestas utilizando Python.

## Metadatos

| Propiedad | Valor |
|---|---|
| Duración | 75 minutos |
| Complejidad | Media |
| Nivel de Bloom | Crear |

## Descripción general

En esta práctica construirás un pipeline reproducible para transformar preguntas y respuestas de soporte técnico en registros conversacionales JSONL compatibles con procesos posteriores de *Fine-Tuning*. El pipeline normalizará datos, detectará duplicados, rechazará registros de baja calidad y validará los ejemplos mediante Pydantic y JSON Schema.

No se ejecutará ningún trabajo de Fine-Tuning en Azure. El resultado será un conjunto de datos validado, un catálogo FAQ seguro para consultas futuras y un informe de calidad que permita decidir si el dataset está listo para una revisión humana o una carga posterior.

## Objetivos de aprendizaje

Al finalizar la práctica, podrás:

- [ ] Convertir una base CSV de preguntas y respuestas en ejemplos conversacionales con roles `system`, `user` y `assistant`.
- [ ] Aplicar reglas de validación sintáctica, semántica y de calidad usando Python, Pydantic y JSON Schema.
- [ ] Separar registros válidos, rechazados y duplicados en artefactos JSONL reproducibles.
- [ ] Generar métricas de calidad del dataset en formato JSON.
- [ ] Preparar un catálogo `faq_catalog.json` reutilizable por la herramienta `search_faq` de laboratorios posteriores.

## Prerrequisitos

### Conocimientos

- Python intermedio: funciones, archivos, excepciones y pruebas automatizadas.
- Lectura y escritura de archivos CSV, JSON y JSONL.
- Conocimiento básico del formato conversacional de modelos generativos.
- Comprensión de la diferencia entre Fine-Tuning, RAG e ingeniería de *prompts*.

### Acceso y entorno

- Repositorio local `~/genai-agent-labs`.
- Entorno virtual compartido en `~/genai-agent-labs/.venv`.
- Python 3.12.1.
- Acceso de escritura al directorio de trabajo.
- No se requieren recursos de Azure, credenciales ni Docker para esta práctica.

## Entorno del laboratorio

### Recursos de hardware recomendados

| Recurso | Mínimo |
|---|---:|
| Procesador | 4 núcleos |
| Memoria RAM | 16 GB |
| Espacio libre en disco | 10 GB |

### Dependencias de software

| Herramienta o biblioteca | Versión |
|---|---|
| Python | 3.12.1 |
| Pandas | 2.2.3 |
| Pydantic | 2.10.4 |
| jsonschema | 4.23.0 |
| pytest | 8.3.4 |

> **Importante:** usa el directorio global obligatorio `~/genai-agent-labs` y el entorno virtual compartido `~/genai-agent-labs/.venv`.

---

## Paso a paso

### Paso 1. Preparar la estructura base del repositorio

**Objetivo:** crear la estructura de datos, código, pruebas y reportes requerida por el pipeline.

**Instrucciones:**

1. Abre una terminal y desplázate al repositorio compartido:

   ```bash
   cd ~/genai-agent-labs
   ```

2. Activa el entorno virtual compartido:

   ```bash
   source .venv/bin/activate
   ```

3. Verifica la versión de Python:

   ```bash
   python --version
   ```

4. Crea los directorios requeridos:

   ```bash
   mkdir -p src data/raw data/processed tests reports config prompts
   ```

5. Si el repositorio aún no está inicializado, ejecútalo:

   ```bash
   git init
   ```

6. Crea o actualiza el archivo `.gitignore`:

   ```bash
   cat > .gitignore <<'EOF'
   .venv/
   __pycache__/
   .pytest_cache/
   *.pyc
   .env
   .env.*
   reports/*.log
   EOF
   ```

7. Instala las dependencias específicas de la práctica:

   ```bash
   pip install pandas==2.2.3 pydantic==2.10.4 jsonschema==4.23.0 pytest==8.3.4
   ```

8. Guarda las dependencias del laboratorio para reproducibilidad:

   ```bash
   cat > requirements-lab-04.txt <<'EOF'
   pandas==2.2.3
   pydantic==2.10.4
   jsonschema==4.23.0
   pytest==8.3.4
   EOF
   ```

**Salida esperada:**

- Existen los directorios `src`, `data/raw`, `data/processed`, `tests` y `reports`.
- El entorno virtual está activo.
- Las dependencias se instalan sin errores.

**Verificación:**

```bash
find src data tests reports -maxdepth 2 -type d | sort
python -c "import pandas, pydantic, jsonschema; print('Dependencias disponibles')"
```

---

### Paso 2. Crear la fuente de preguntas frecuentes

**Objetivo:** crear un archivo CSV de ejemplo con registros válidos, duplicados y registros que deban ser rechazados.

**Instrucciones:**

1. Crea el archivo `data/raw/faq_soporte.csv`:

   ```bash
   cat > data/raw/faq_soporte.csv <<'EOF'
   pregunta,respuesta
   "No puedo conectarme a la VPN corporativa.","1. Verifica que tengas conexión a Internet. 2. Confirma que el cliente VPN esté actualizado. 3. Comprueba que el nombre de usuario y la contraseña sean correctos. 4. Si el error continúa, comunica al soporte el código mostrado por la aplicación."
   "¿Cómo restablezco mi contraseña del portal?","1. Abre la página de restablecimiento de contraseña. 2. Introduce tu correo corporativo. 3. Completa la verificación de identidad. 4. Crea una contraseña nueva que cumpla la política de seguridad."
   "  No puedo conectarme a la VPN corporativa.  ","  1. Verifica que tengas conexión a Internet. 2. Confirma que el cliente VPN esté actualizado. 3. Comprueba que el nombre de usuario y la contraseña sean correctos. 4. Si el error continúa, comunica al soporte el código mostrado por la aplicación.  "
   "","1. Verifica que el navegador esté actualizado y vuelve a intentarlo."
   "¿Cómo reinicio mi equipo?","Reinicia."
   "¿Cómo uso <script> en el portal?","1. No ejecutes contenido no confiable. 2. Revisa la documentación oficial del portal. 3. Si necesitas una integración aprobada, solicita orientación al equipo de soporte."
   EOF
   ```

2. Revisa el contenido del archivo:

   ```bash
   column -s, -t < data/raw/faq_soporte.csv
   ```

**Salida esperada:**

El archivo contiene seis registros de entrada:

- Dos registros válidos.
- Un registro duplicado después de normalizar espacios.
- Un registro con pregunta vacía.
- Un registro con respuesta excesivamente corta.
- Un registro con caracteres no permitidos, como `<` y `>`.

**Verificación:**

```bash
wc -l data/raw/faq_soporte.csv
```

El resultado esperado es `7`, incluyendo la cabecera CSV.

---

### Paso 3. Implementar los modelos y reglas de validación

**Objetivo:** definir el formato conversacional de Fine-Tuning y las reglas de calidad del dataset.

**Instrucciones:**

1. Crea el archivo `src/dataset_pipeline.py`:

   ```bash
   cat > src/dataset_pipeline.py <<'PY'
   from __future__ import annotations

   import argparse
   import hashlib
   import json
   import re
   from collections import Counter
   from pathlib import Path
   from typing import Literal

   import pandas as pd
   from jsonschema import validate as validate_jsonschema
   from pydantic import BaseModel, Field, ValidationError, model_validator


   SYSTEM_MESSAGE = (
       "Eres un asistente de soporte técnico. Responde en español, "
       "con tono profesional y pasos numerados."
   )

   MIN_QUESTION_LENGTH = 12
   MAX_QUESTION_LENGTH = 300
   MIN_ANSWER_LENGTH = 40
   MAX_ANSWER_LENGTH = 1200

   # Permite letras Unicode, números, espacios y puntuación habitual.
   FORBIDDEN_CHARACTERS = re.compile(r"[<>{}\x00-\x08\x0b\x0c\x0e-\x1f\x7f]")
   NUMBERED_STEP = re.compile(r"^\s*1\.\s+", re.DOTALL)


   class Message(BaseModel):
       role: Literal["system", "user", "assistant"]
       content: str = Field(min_length=1, max_length=1200)


   class TrainingExample(BaseModel):
       messages: list[Message] = Field(min_length=3, max_length=3)

       @model_validator(mode="after")
       def validate_conversation_structure(self) -> "TrainingExample":
           expected_roles = ["system", "user", "assistant"]
           actual_roles = [message.role for message in self.messages]

           if actual_roles != expected_roles:
               raise ValueError(
                   "Los roles deben aparecer exactamente en el orden "
                   "system, user, assistant."
               )

           if self.messages[0].content != SYSTEM_MESSAGE:
               raise ValueError("El mensaje de sistema no coincide con la política definida.")

           return self


   def normalize_text(value: str) -> str:
       """Elimina espacios sobrantes y convierte secuencias de espacios en uno solo."""
       return " ".join(str(value).split())


   def validate_pair(question: str, answer: str) -> tuple[str, str]:
       """Normaliza y valida una pregunta-respuesta antes de crear un ejemplo."""
       normalized_question = normalize_text(question)
       normalized_answer = normalize_text(answer)

       errors: list[str] = []

       if not normalized_question:
           errors.append("pregunta_vacia")

       if not normalized_answer:
           errors.append("respuesta_vacia")

       if normalized_question and not (
           MIN_QUESTION_LENGTH <= len(normalized_question) <= MAX_QUESTION_LENGTH
       ):
           errors.append("longitud_pregunta_invalida")

       if normalized_answer and not (
           MIN_ANSWER_LENGTH <= len(normalized_answer) <= MAX_ANSWER_LENGTH
       ):
           errors.append("longitud_respuesta_invalida")

       if FORBIDDEN_CHARACTERS.search(normalized_question):
           errors.append("caracteres_no_permitidos_en_pregunta")

       if FORBIDDEN_CHARACTERS.search(normalized_answer):
           errors.append("caracteres_no_permitidos_en_respuesta")

       if normalized_answer and not NUMBERED_STEP.search(normalized_answer):
           errors.append("respuesta_sin_pasos_numerados")

       if errors:
           raise ValueError(";".join(errors))

       return normalized_question, normalized_answer


   def build_example(question: str, answer: str) -> TrainingExample:
       """Construye y valida un ejemplo de Fine-Tuning con Pydantic."""
       normalized_question, normalized_answer = validate_pair(question, answer)

       return TrainingExample(
           messages=[
               Message(role="system", content=SYSTEM_MESSAGE),
               Message(role="user", content=normalized_question),
               Message(role="assistant", content=normalized_answer),
           ]
       )


   def fingerprint(question: str, answer: str) -> str:
       """Genera una huella estable para detectar duplicados normalizados."""
       canonical_value = f"{question.casefold()}|{answer.casefold()}"
       return hashlib.sha256(canonical_value.encode("utf-8")).hexdigest()


   def write_jsonl(path: Path, records: list[dict]) -> None:
       path.parent.mkdir(parents=True, exist_ok=True)
       with path.open("w", encoding="utf-8") as file:
           for record in records:
               file.write(json.dumps(record, ensure_ascii=False) + "\n")


   def run_pipeline(
       input_path: Path,
       train_path: Path,
       rejected_path: Path,
       report_path: Path,
       catalog_path: Path,
   ) -> dict:
       """Procesa un CSV de FAQ y genera artefactos validados."""
       dataframe = pd.read_csv(input_path, dtype=str, keep_default_na=False)

       required_columns = {"pregunta", "respuesta"}
       missing_columns = required_columns.difference(dataframe.columns)
       if missing_columns:
           raise ValueError(
               f"Faltan columnas obligatorias en el CSV: {sorted(missing_columns)}"
           )

       training_records: list[dict] = []
       rejected_records: list[dict] = []
       catalog_records: list[dict] = []
       seen_fingerprints: set[str] = set()
       rejection_reasons: Counter[str] = Counter()
       duplicate_count = 0

       for row_number, row in enumerate(dataframe.to_dict(orient="records"), start=2):
           raw_question = row["pregunta"]
           raw_answer = row["respuesta"]

           try:
               normalized_question, normalized_answer = validate_pair(
                   raw_question, raw_answer
               )

               record_fingerprint = fingerprint(normalized_question, normalized_answer)
               if record_fingerprint in seen_fingerprints:
                   duplicate_count += 1
                   rejection_reasons["duplicado_normalizado"] += 1
                   rejected_records.append(
                       {
                           "source_row": row_number,
                           "question": normalized_question,
                           "answer": normalized_answer,
                           "rejection_reasons": ["duplicado_normalizado"],
                       }
                   )
                   continue

               example = build_example(normalized_question, normalized_answer)
               example_dict = example.model_dump()

               # Segunda validación independiente contra JSON Schema.
               validate_jsonschema(
                   instance=example_dict,
                   schema=TrainingExample.model_json_schema(),
               )

               seen_fingerprints.add(record_fingerprint)
               training_records.append(example_dict)

               catalog_records.append(
                   {
                       "id": f"faq-{len(catalog_records) + 1:03d}",
                       "question": normalized_question,
                       "answer": normalized_answer,
                   }
               )

           except (ValueError, ValidationError) as error:
               reasons = str(error).split(";")
               for reason in reasons:
                   rejection_reasons[reason] += 1

               rejected_records.append(
                   {
                       "source_row": row_number,
                       "question": normalize_text(raw_question),
                       "answer": normalize_text(raw_answer),
                       "rejection_reasons": reasons,
                   }
               )

       write_jsonl(train_path, training_records)
       write_jsonl(rejected_path, rejected_records)

       catalog_path.parent.mkdir(parents=True, exist_ok=True)
       catalog_path.write_text(
           json.dumps(catalog_records, ensure_ascii=False, indent=2) + "\n",
           encoding="utf-8",
       )

       total_records = len(dataframe)
       report = {
           "input_file": str(input_path),
           "total_records": total_records,
           "valid_records": len(training_records),
           "rejected_records": len(rejected_records),
           "duplicate_records": duplicate_count,
           "acceptance_rate": round(
               len(training_records) / total_records if total_records else 0, 4
           ),
           "rejection_reasons": dict(sorted(rejection_reasons.items())),
           "validation_rules": {
               "question_length": [MIN_QUESTION_LENGTH, MAX_QUESTION_LENGTH],
               "answer_length": [MIN_ANSWER_LENGTH, MAX_ANSWER_LENGTH],
               "required_roles": ["system", "user", "assistant"],
               "required_system_message": SYSTEM_MESSAGE,
               "assistant_response_requires_numbered_steps": True,
               "forbidden_characters_pattern": FORBIDDEN_CHARACTERS.pattern,
               "pydantic_validation": True,
               "json_schema_validation": True,
           },
           "artifacts": {
               "training_jsonl": str(train_path),
               "rejected_jsonl": str(rejected_path),
               "faq_catalog": str(catalog_path),
           },
       }

       report_path.parent.mkdir(parents=True, exist_ok=True)
       report_path.write_text(
           json.dumps(report, ensure_ascii=False, indent=2) + "\n",
           encoding="utf-8",
       )

       return report


   def main() -> None:
       parser = argparse.ArgumentParser(
           description="Prepara un dataset FAQ validado para Fine-Tuning."
       )
       parser.add_argument(
           "--input",
           type=Path,
           default=Path("data/raw/faq_soporte.csv"),
       )
       parser.add_argument(
           "--train-output",
           type=Path,
           default=Path("data/processed/finetuning_train.jsonl"),
       )
       parser.add_argument(
           "--rejected-output",
           type=Path,
           default=Path("data/processed/finetuning_rejected.jsonl"),
       )
       parser.add_argument(
           "--report-output",
           type=Path,
           default=Path("reports/dataset_quality_report.json"),
       )
       parser.add_argument(
           "--catalog-output",
           type=Path,
           default=Path("data/processed/faq_catalog.json"),
       )

       args = parser.parse_args()

       report = run_pipeline(
           input_path=args.input,
           train_path=args.train_output,
           rejected_path=args.rejected_output,
           report_path=args.report_output,
           catalog_path=args.catalog_output,
       )

       print(
           "Pipeline completado: "
           f"{report['valid_records']} válidos, "
           f"{report['rejected_records']} rechazados, "
           f"{report['duplicate_records']} duplicados."
       )


   if __name__ == "__main__":
       main()
   PY
   ```

2. Revisa la sintaxis del archivo:

   ```bash
   python -m py_compile src/dataset_pipeline.py
   ```

**Salida esperada:**

No se muestra salida si el archivo Python es sintácticamente válido.

**Verificación:**

Revisa que las reglas implementadas cubran estos controles:

- Campos vacíos.
- Longitudes mínimas y máximas.
- Caracteres no permitidos.
- Respuestas con pasos numerados.
- Roles obligatorios y orden correcto.
- Mensaje de sistema fijo.
- Validación Pydantic.
- Validación JSON Schema.
- Detección de duplicados tras normalización.

---

### Paso 4. Ejecutar el pipeline de preparación

**Objetivo:** generar los archivos JSONL, el catálogo FAQ y el informe de calidad.

**Instrucciones:**

1. Ejecuta el pipeline desde la raíz del repositorio:

   ```bash
   python src/dataset_pipeline.py
   ```

2. Lista los artefactos generados:

   ```bash
   find data/processed reports -maxdepth 1 -type f | sort
   ```

3. Visualiza el dataset de entrenamiento:

   ```bash
   cat data/processed/finetuning_train.jsonl
   ```

4. Visualiza los registros rechazados:

   ```bash
   cat data/processed/finetuning_rejected.jsonl
   ```

5. Visualiza el informe de calidad:

   ```bash
   python -m json.tool reports/dataset_quality_report.json
   ```

6. Visualiza el catálogo reutilizable de FAQ:

   ```bash
   python -m json.tool data/processed/faq_catalog.json
   ```

**Salida esperada:**

El comando principal debe indicar un resultado similar a:

```text
Pipeline completado: 2 válidos, 4 rechazados, 1 duplicados.
```

Deben existir estos archivos:

```text
data/processed/finetuning_train.jsonl
data/processed/finetuning_rejected.jsonl
data/processed/faq_catalog.json
reports/dataset_quality_report.json
```

**Verificación:**

Comprueba que el informe contiene valores equivalentes a los siguientes:

```json
{
  "total_records": 6,
  "valid_records": 2,
  "rejected_records": 4,
  "duplicate_records": 1,
  "acceptance_rate": 0.3333
}
```

El número exacto de razones de rechazo puede variar si se modificaron las reglas, pero deben aparecer causas relacionadas con:

- `duplicado_normalizado`
- `pregunta_vacia`
- `longitud_respuesta_invalida`
- `respuesta_sin_pasos_numerados`
- `caracteres_no_permitidos_en_pregunta`

---

### Paso 5. Inspeccionar el formato de Fine-Tuning y el catálogo FAQ

**Objetivo:** verificar que los artefactos generados sirven para propósitos distintos y no mezclan responsabilidades.

**Instrucciones:**

1. Comprueba que cada línea del archivo de entrenamiento es JSON válido:

   ```bash
   while IFS= read -r line; do
     echo "$line" | python -m json.tool > /dev/null
   done < data/processed/finetuning_train.jsonl

   echo "Todas las líneas JSONL son válidas."
   ```

2. Cuenta los registros del archivo JSONL:

   ```bash
   wc -l data/processed/finetuning_train.jsonl
   ```

3. Comprueba los roles de cada conversación:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   path = Path("data/processed/finetuning_train.jsonl")

   for index, line in enumerate(path.read_text(encoding="utf-8").splitlines(), start=1):
       record = json.loads(line)
       roles = [message["role"] for message in record["messages"]]
       print(f"Registro {index}: {roles}")
   PY
   ```

4. Comprueba que el catálogo FAQ no contiene el mensaje de sistema ni la estructura de entrenamiento:

   ```bash
   python - <<'PY'
   import json
   from pathlib import Path

   catalog = json.loads(
       Path("data/processed/faq_catalog.json").read_text(encoding="utf-8")
   )

   for item in catalog:
       assert set(item.keys()) == {"id", "question", "answer"}

   print(f"Catálogo válido con {len(catalog)} preguntas frecuentes.")
   PY
   ```

**Salida esperada:**

Cada registro de entrenamiento debe tener esta estructura conceptual:

```json
{
  "messages": [
    {
      "role": "system",
      "content": "Eres un asistente de soporte técnico..."
    },
    {
      "role": "user",
      "content": "Pregunta del usuario"
    },
    {
      "role": "assistant",
      "content": "1. Primer paso..."
    }
  ]
}
```

El catálogo debe tener una estructura más simple:

```json
[
  {
    "id": "faq-001",
    "question": "No puedo conectarme a la VPN corporativa.",
    "answer": "1. Verifica que tengas conexión a Internet..."
  }
]
```

**Verificación:**

Explica brevemente la diferencia:

- `finetuning_train.jsonl` representa ejemplos de comportamiento conversacional para entrenamiento supervisado.
- `faq_catalog.json` representa conocimiento recuperable que podrá ser consultado por una herramienta como `search_faq`.
- El catálogo no sustituye al Fine-Tuning: será útil para recuperar información concreta y actualizable, siguiendo una estrategia similar a RAG.

---

### Paso 6. Crear pruebas automatizadas

**Objetivo:** validar de forma automatizada los casos requeridos de calidad y estructura.

**Instrucciones:**

1. Crea el archivo `tests/test_dataset_pipeline.py`:

   ```bash
   cat > tests/test_dataset_pipeline.py <<'PY'
   import json
   from pathlib import Path

   import pytest
   from pydantic import ValidationError

   from src.dataset_pipeline import (
       SYSTEM_MESSAGE,
       Message,
       TrainingExample,
       build_example,
       run_pipeline,
       validate_pair,
   )


   VALID_QUESTION = "No puedo conectarme a la VPN corporativa."
   VALID_ANSWER = (
       "1. Verifica la conexión a Internet. "
       "2. Confirma que el cliente VPN esté actualizado. "
       "3. Revisa las credenciales corporativas. "
       "4. Comunica el código de error al soporte si el problema continúa."
   )


   def test_builds_valid_training_record():
       example = build_example(VALID_QUESTION, VALID_ANSWER)

       assert [message.role for message in example.messages] == [
           "system",
           "user",
           "assistant",
       ]
       assert example.messages[0].content == SYSTEM_MESSAGE
       assert example.messages[1].content == VALID_QUESTION
       assert example.messages[2].content == VALID_ANSWER


   def test_rejects_empty_messages():
       with pytest.raises(ValueError, match="pregunta_vacia"):
           validate_pair("", VALID_ANSWER)

       with pytest.raises(ValueError, match="respuesta_vacia"):
           validate_pair(VALID_QUESTION, "")


   def test_rejects_invalid_roles():
       with pytest.raises(ValidationError):
           Message(role="developer", content="Mensaje inválido")


   def test_rejects_invalid_role_order():
       with pytest.raises(ValidationError, match="roles deben aparecer"):
           TrainingExample(
               messages=[
                   Message(role="user", content=VALID_QUESTION),
                   Message(role="system", content=SYSTEM_MESSAGE),
                   Message(role="assistant", content=VALID_ANSWER),
               ]
           )


   def test_rejects_excessively_short_answer():
       with pytest.raises(ValueError, match="longitud_respuesta_invalida"):
           validate_pair("¿Cómo reinicio mi equipo?", "Reinicia.")


   def test_detects_normalized_duplicate(tmp_path: Path):
       csv_path = tmp_path / "faq.csv"
       train_path = tmp_path / "train.jsonl"
       rejected_path = tmp_path / "rejected.jsonl"
       report_path = tmp_path / "report.json"
       catalog_path = tmp_path / "catalog.json"

       csv_path.write_text(
           "pregunta,respuesta\n"
           f'"{VALID_QUESTION}","{VALID_ANSWER}"\n'
           f'"  {VALID_QUESTION}  ","  {VALID_ANSWER}  "\n',
           encoding="utf-8",
       )

       report = run_pipeline(
           input_path=csv_path,
           train_path=train_path,
           rejected_path=rejected_path,
           report_path=report_path,
           catalog_path=catalog_path,
       )

       assert report["valid_records"] == 1
       assert report["rejected_records"] == 1
       assert report["duplicate_records"] == 1

       rejected = json.loads(rejected_path.read_text(encoding="utf-8").strip())
       assert rejected["rejection_reasons"] == ["duplicado_normalizado"]
   PY
   ```

2. Ejecuta las pruebas:

   ```bash
   pytest -q
   ```

**Salida esperada:**

La salida debe indicar que todas las pruebas pasan. Un resultado típico es:

```text
......                                                                   [100%]
6 passed in <tiempo>s
```

**Verificación:**

Las pruebas deben cubrir explícitamente:

- Un registro válido.
- Preguntas o respuestas vacías.
- Roles inválidos.
- Orden inválido de roles.
- Duplicados después de normalizar espacios.
- Respuestas excesivamente cortas.

---

### Paso 7. Documentar criterios de calidad y límites de uso

**Objetivo:** dejar documentado que el dataset está preparado para revisión y carga posterior, no para entrenar automáticamente sin evaluación.

**Instrucciones:**

1. Crea el documento `reports/dataset_readiness.md`:

   ```bash
   cat > reports/dataset_readiness.md <<'EOF'
   # Preparación del dataset para Fine-Tuning

   ## Propósito

   El archivo `data/processed/finetuning_train.jsonl` contiene ejemplos
   conversacionales validados para una posible carga posterior a un proceso de
   Fine-Tuning supervisado.

   ## Controles aplicados

   - Normalización de espacios.
   - Validación de campos vacíos.
   - Límites de longitud para preguntas y respuestas.
   - Detección de caracteres no permitidos.
   - Requisito de respuestas con pasos numerados.
   - Validación de roles y orden conversacional.
   - Validación mediante Pydantic.
   - Validación independiente mediante JSON Schema.
   - Detección de duplicados normalizados.

   ## Limitaciones

   Este dataset es un conjunto de entrenamiento candidato. Antes de cualquier
   Fine-Tuning real debe revisarse por expertos del dominio, verificarse la
   ausencia de datos personales, secretos y contenido obsoleto, y separarse en
   conjuntos de entrenamiento, validación y prueba.

   ## Estrategia de personalización

   El Fine-Tuning se usaría para reforzar tono, estructura y pasos de soporte.
   Las preguntas frecuentes cambiantes deben recuperarse desde
   `faq_catalog.json` mediante una herramienta de búsqueda, no memorizarse
   únicamente en un modelo ajustado.
   EOF
   ```

2. Revisa el documento:

   ```bash
   cat reports/dataset_readiness.md
   ```

**Salida esperada:**

Existe documentación que diferencia claramente:

- Comportamiento y formato persistente: posible Fine-Tuning.
- Información cambiante: recuperación desde catálogo FAQ o RAG.
- Evaluación futura: comparación frente a una línea base y uso de conjuntos independientes.

**Verificación:**

Confirma que no se ha ejecutado ningún comando de entrenamiento, despliegue de Azure OpenAI ni carga de archivos a un servicio externo.

---

## Validación y pruebas

Ejecuta la validación completa desde la raíz del repositorio:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate

pytest -q
python src/dataset_pipeline.py
python -m json.tool reports/dataset_quality_report.json
```

Ejecuta además las siguientes comprobaciones de integridad:

```bash
test -s data/processed/finetuning_train.jsonl && echo "Dataset de entrenamiento generado"
test -s data/processed/finetuning_rejected.jsonl && echo "Dataset de rechazados generado"
test -s data/processed/faq_catalog.json && echo "Catálogo FAQ generado"
test -s reports/dataset_quality_report.json && echo "Informe de calidad generado"
```

Criterios de aceptación de la práctica:

| Criterio | Resultado esperado |
|---|---|
| Pipeline ejecutable | Finaliza sin excepciones |
| Dataset de entrenamiento | Contiene 2 registros JSONL válidos |
| Registros rechazados | Contiene 4 registros con razones de rechazo |
| Duplicados | Se detecta 1 duplicado normalizado |
| Catálogo FAQ | Contiene únicamente preguntas y respuestas aprobadas |
| Informe de calidad | Incluye métricas y reglas aplicadas |
| Pruebas | Todas pasan con `pytest -q` |
| Secretos | No existen claves, tokens ni credenciales en archivos generados |

Antes de finalizar, revisa los cambios:

```bash
git status
git diff -- src/dataset_pipeline.py tests/test_dataset_pipeline.py
```

Si el instructor ha definido un mensaje de commit para esta práctica, realiza el commit siguiendo esa convención. La lista de mensajes proporcionada para prácticas anteriores no incluye explícitamente `04-00-01`; no inventes un mensaje alternativo si tu instructor exige mensajes exactos.

---

## Resolución de problemas

### Problema 1. `ModuleNotFoundError: No module named 'src'` al ejecutar pytest

**Síntoma:**

```text
ModuleNotFoundError: No module named 'src'
```

**Causa:**

Las pruebas se ejecutaron desde un directorio distinto de `~/genai-agent-labs`, por lo que Python no puede resolver el paquete local `src`.

**Solución:**

Ejecuta las pruebas desde la raíz del repositorio:

```bash
cd ~/genai-agent-labs
source .venv/bin/activate
pytest -q
```

Si el problema continúa, crea un archivo vacío para declarar explícitamente el paquete:

```bash
touch src/__init__.py
pytest -q
```

### Problema 2. El pipeline rechaza más registros de los esperados

**Síntoma:**

El informe muestra una tasa de aceptación menor de la esperada o aparecen razones como:

```text
respuesta_sin_pasos_numerados
longitud_respuesta_invalida
caracteres_no_permitidos_en_pregunta
```

**Causa:**

Las reglas del laboratorio exigen que las respuestas tengan entre 40 y 1200 caracteres y comiencen con `1.` para representar pasos de soporte consistentes. También se rechazan caracteres como `<`, `>` o caracteres de control.

**Solución:**

Corrige los datos de origen en `data/raw/faq_soporte.csv` y vuelve a ejecutar el pipeline. Por ejemplo:

```csv
"¿Cómo reinicio mi equipo?","1. Guarda tu trabajo pendiente. 2. Abre el menú de apagado del sistema. 3. Selecciona Reiniciar. 4. Si el problema persiste después del reinicio, informa al equipo de soporte."
```

Después, ejecuta:

```bash
python src/dataset_pipeline.py
pytest -q
```

---

## Limpieza

No se deben eliminar los artefactos requeridos de la práctica, ya que serán reutilizados en laboratorios posteriores, especialmente:

```text
data/processed/faq_catalog.json
data/processed/finetuning_train.jsonl
reports/dataset_quality_report.json
```

Si deseas eliminar únicamente archivos temporales de Python y pruebas, ejecuta:

```bash
find src tests -type d -name "__pycache__" -prune -exec rm -rf {} +
rm -rf .pytest_cache
```

Desactiva el entorno virtual al terminar la sesión:

```bash
deactivate
```

---

## Resumen

En esta práctica construiste un pipeline reproducible de preparación de datos para Fine-Tuning. Transformaste preguntas y respuestas CSV en conversaciones JSONL con roles `system`, `user` y `assistant`; aplicaste normalización, validación de calidad, detección de duplicados y comprobaciones mediante Pydantic y JSON Schema.

También separaste los registros rechazados para revisión, generaste métricas de calidad y creaste un catálogo FAQ reutilizable. El resultado deja preparado un dataset candidato para evaluación humana y carga futura, evitando costes de entrenamiento durante el laboratorio.

Como criterio de diseño, recuerda:

- Usa Fine-Tuning para reforzar comportamiento, tono y estructura repetible.
- Usa recuperación de información, como `faq_catalog.json`, para conocimiento que pueda cambiar.
- Evalúa cualquier modelo ajustado contra una línea base y con datos de prueba no vistos.
- Mantén validaciones, controles de privacidad y supervisión humana antes de utilizar un dataset en producción.

### Recursos opcionales

- [Guía de Fine-Tuning de OpenAI](https://platform.openai.com/docs/guides/fine-tuning)
- [Guía de evaluaciones de OpenAI](https://platform.openai.com/docs/guides/evals)
- [Documentación de Pydantic](https://docs.pydantic.dev/)
- [Documentación de JSON Schema](https://json-schema.org/)
- [Principios de IA responsable de Microsoft](https://www.microsoft.com/ai/responsible-ai)
